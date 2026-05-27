# Configuración de canales seguros — Habilitar TLS 1.3 para Administration y Hardening de protocolos inseguros

## Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 120 minutos                                  |
| **Complejidad**  | Alta (Hard)                                  |
| **Nivel Bloom**  | Crear (Create)                               |
| **Laboratorio**  | 06-00-01                                     |
| **Prerequisito** | Lab 01-00-01 completado                      |

---

## Descripción General

En este laboratorio implementarás seguridad de transporte de extremo a extremo en WebLogic 15. Construirás una PKI mínima de laboratorio (CA raíz autofirmada + certificado de servidor con SANs), convertirás los certificados a los formatos PKCS12 y JKS requeridos por WebLogic, configurarás el Administration Server para usar esos keystores en el puerto 7002 y forzarás TLS 1.3 como versión mínima. Finalmente, verificarás que los protocolos inseguros (SSLv3, TLS 1.0, TLS 1.1) queden efectivamente rechazados y documentarás el proceso como runbook de seguridad reutilizable.

---

## Objetivos de Aprendizaje

- [ ] Generar una CA autofirmada y un certificado de servidor con Subject Alternative Names usando OpenSSL 3.0.
- [ ] Crear y configurar keystores PKCS12 e importarlos como Identity Store y Trust Store en WebLogic 15.
- [ ] Habilitar TLS 1.3 en el Administration Server y deshabilitar SSLv3, TLS 1.0 y TLS 1.1 a nivel de WebLogic y de JVM.
- [ ] Verificar la configuración TLS con `openssl s_client` y confirmar el rechazo de protocolos inseguros.
- [ ] Aplicar hardening adicional deshabilitando cipher suites débiles y configurando parámetros de JVM para reforzar la postura de seguridad.

---

## Requisitos Previos

### Conocimiento

| Área                          | Nivel requerido                                                     |
|-------------------------------|---------------------------------------------------------------------|
| PKI básica                    | CA, certificados X.509, firma digital, cadena de confianza          |
| SSL/TLS                       | Handshake, cipher suites, versiones de protocolo                    |
| WebLogic Administration       | Dominio operativo, Administration Server iniciado                   |
| Línea de comandos Linux       | Redirección, variables de entorno, edición de archivos              |

### Acceso y Software

| Requisito                     | Detalle                                                             |
|-------------------------------|---------------------------------------------------------------------|
| Lab 01-00-01 completado       | Dominio `wls_domain` con Administration Server en `localhost:7001`  |
| OpenSSL 3.0+                  | `openssl version` debe retornar 3.x                                 |
| keytool                       | `$JAVA_HOME/bin/keytool` disponible                                 |
| Usuario `oracle`              | Permisos de escritura sobre `$DOMAIN_HOME`                         |
| WebLogic 15 en ejecución      | Administration Server activo y accesible                            |

---

## Entorno del Laboratorio

### Variables de entorno de referencia

Todos los pasos asumen las siguientes variables. Ajústalas si tu instalación usa rutas distintas.

```bash
# ── Ejecutar como usuario oracle ──────────────────────────────────────────
export JAVA_HOME=/opt/oracle/java/jdk-21
export WL_HOME=/opt/oracle/middleware/wlserver
export DOMAIN_HOME=/opt/oracle/domains/wls_domain
export SSL_DIR=$DOMAIN_HOME/security/ssl
export ADMIN_HOST=localhost
export ADMIN_PORT_PLAIN=7001
export ADMIN_PORT_SSL=7002
export PATH=$JAVA_HOME/bin:$PATH
```

> **Nota:** Añade estas líneas a `~/.bashrc` o ejecútalas en cada terminal antes de comenzar.

### Verificación del entorno inicial

```bash
# Verificar Java 21
java -version

# Verificar OpenSSL 3.x
openssl version

# Verificar keytool
keytool -help | head -3

# Verificar que el Administration Server esté activo
curl -s -o /dev/null -w "%{http_code}" http://$ADMIN_HOST:$ADMIN_PORT_PLAIN/console/
# Debe retornar 200 o 302
```

---

## Pasos del Laboratorio

---

### Paso 1 — Preparar el directorio de trabajo SSL

**Objetivo:** Crear la estructura de directorios donde se almacenarán todos los artefactos criptográficos del laboratorio.

#### Instrucciones

1. Crea el directorio de trabajo y establece permisos restrictivos:

```bash
mkdir -p $SSL_DIR/{ca,server,keystores}
chmod 700 $SSL_DIR
cd $SSL_DIR
```

2. Crea el archivo de configuración OpenSSL para la CA (`ca/ca.cnf`):

```bash
cat > $SSL_DIR/ca/ca.cnf << 'EOF'
[ req ]
default_bits        = 4096
default_md          = sha256
prompt              = no
distinguished_name  = dn
x509_extensions     = v3_ca

[ dn ]
C  = ES
ST = Madrid
L  = Madrid
O  = WebLogic Lab CA
OU = Security Lab
CN = WebLogic Lab Root CA

[ v3_ca ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints       = critical, CA:true
keyUsage               = critical, digitalSignature, cRLSign, keyCertSign
EOF
```

3. Crea el archivo de configuración OpenSSL para el certificado de servidor (`server/server.cnf`). Incluye Subject Alternative Names (SANs) correctos:

```bash
cat > $SSL_DIR/server/server.cnf << 'EOF'
[ req ]
default_bits        = 2048
default_md          = sha256
prompt              = no
distinguished_name  = dn
req_extensions      = v3_req

[ dn ]
C  = ES
ST = Madrid
L  = Madrid
O  = WebLogic Lab
OU = Administration
CN = localhost

[ v3_req ]
subjectAltName = @alt_names
keyUsage       = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ alt_names ]
DNS.1 = localhost
DNS.2 = adminserver
IP.1  = 127.0.0.1
EOF
```

#### Resultado esperado

```
$SSL_DIR/
├── ca/
│   └── ca.cnf
├── server/
│   └── server.cnf
└── keystores/
```

#### Verificación

```bash
ls -la $SSL_DIR/ca/ $SSL_DIR/server/ $SSL_DIR/keystores/
# Deben existir los tres directorios y los dos archivos .cnf
```

---

### Paso 2 — Generar la CA raíz autofirmada

**Objetivo:** Crear la clave privada y el certificado autofirmado de la CA raíz de laboratorio.

#### Instrucciones

1. Genera la clave privada de la CA (4096 bits, RSA):

```bash
openssl genrsa -out $SSL_DIR/ca/ca.key 4096
chmod 400 $SSL_DIR/ca/ca.key
```

2. Genera el certificado autofirmado de la CA (válido 10 años para laboratorio):

```bash
openssl req -new -x509 \
  -key $SSL_DIR/ca/ca.key \
  -out $SSL_DIR/ca/ca.crt \
  -days 3650 \
  -config $SSL_DIR/ca/ca.cnf
```

3. Verifica el certificado generado:

```bash
openssl x509 -in $SSL_DIR/ca/ca.crt -text -noout | grep -E "Subject:|Issuer:|Not After|CA:"
```

#### Resultado esperado

```
Subject: C=ES, ST=Madrid, L=Madrid, O=WebLogic Lab CA, OU=Security Lab, CN=WebLogic Lab Root CA
Issuer:  C=ES, ST=Madrid, L=Madrid, O=WebLogic Lab CA, OU=Security Lab, CN=WebLogic Lab Root CA
Not After : <fecha 10 años futura>
CA:TRUE
```

#### Verificación

```bash
# El certificado debe ser autofirmado (Subject == Issuer)
openssl verify -CAfile $SSL_DIR/ca/ca.crt $SSL_DIR/ca/ca.crt
# Salida esperada: ca.crt: OK
```

---

### Paso 3 — Generar el certificado de servidor firmado por la CA

**Objetivo:** Crear la clave privada del servidor, generar una CSR y firmarla con la CA de laboratorio.

#### Instrucciones

1. Genera la clave privada del servidor (2048 bits):

```bash
openssl genrsa -out $SSL_DIR/server/server.key 2048
chmod 400 $SSL_DIR/server/server.key
```

2. Genera la Certificate Signing Request (CSR):

```bash
openssl req -new \
  -key $SSL_DIR/server/server.key \
  -out $SSL_DIR/server/server.csr \
  -config $SSL_DIR/server/server.cnf
```

3. Firma la CSR con la CA para obtener el certificado de servidor (válido 2 años):

```bash
openssl x509 -req \
  -in $SSL_DIR/server/server.csr \
  -CA $SSL_DIR/ca/ca.crt \
  -CAkey $SSL_DIR/ca/ca.key \
  -CAcreateserial \
  -out $SSL_DIR/server/server.crt \
  -days 730 \
  -sha256 \
  -extfile $SSL_DIR/server/server.cnf \
  -extensions v3_req
```

4. Verifica el certificado de servidor y sus SANs:

```bash
openssl x509 -in $SSL_DIR/server/server.crt -text -noout | \
  grep -A 4 "Subject Alternative Name"
```

#### Resultado esperado

```
X509v3 Subject Alternative Name:
    DNS:localhost, DNS:adminserver, IP Address:127.0.0.1
```

#### Verificación

```bash
# Verificar la cadena de confianza: servidor firmado por la CA
openssl verify -CAfile $SSL_DIR/ca/ca.crt $SSL_DIR/server/server.crt
# Salida esperada: server.crt: OK
```

---

### Paso 4 — Crear el Identity Keystore (PKCS12)

**Objetivo:** Empaquetar la clave privada del servidor y su certificado (junto con la cadena CA) en un keystore PKCS12 que WebLogic usará como Identity Store.

#### Instrucciones

1. Crea el keystore PKCS12 de identidad combinando la clave privada, el certificado del servidor y el certificado de la CA:

```bash
openssl pkcs12 -export \
  -in $SSL_DIR/server/server.crt \
  -inkey $SSL_DIR/server/server.key \
  -certfile $SSL_DIR/ca/ca.crt \
  -out $SSL_DIR/keystores/identity.p12 \
  -name "wls-server" \
  -passout pass:Welcome1
```

2. Verifica el contenido del keystore:

```bash
keytool -list -v \
  -keystore $SSL_DIR/keystores/identity.p12 \
  -storetype PKCS12 \
  -storepass Welcome1 | \
  grep -E "Alias|Entry type|Owner|Issuer"
```

#### Resultado esperado

```
Alias name: wls-server
Entry type: PrivateKeyEntry
Owner: CN=localhost, OU=Administration, O=WebLogic Lab, ...
Issuer: CN=WebLogic Lab Root CA, ...
```

#### Verificación

```bash
# El alias debe ser exactamente "wls-server" (se usará en la configuración WebLogic)
keytool -list \
  -keystore $SSL_DIR/keystores/identity.p12 \
  -storetype PKCS12 \
  -storepass Welcome1
# Debe listar: wls-server, PrivateKeyEntry
```

---

### Paso 5 — Crear el Trust Keystore (PKCS12)

**Objetivo:** Crear un keystore de confianza que contenga únicamente el certificado de la CA raíz. WebLogic lo usará para validar certificados de clientes y confirmar la cadena de confianza.

#### Instrucciones

1. Crea el keystore PKCS12 de confianza importando el certificado de la CA:

```bash
keytool -importcert \
  -alias "wls-lab-ca" \
  -file $SSL_DIR/ca/ca.crt \
  -keystore $SSL_DIR/keystores/trust.p12 \
  -storetype PKCS12 \
  -storepass Welcome1 \
  -noprompt
```

2. Verifica el contenido del Trust Store:

```bash
keytool -list -v \
  -keystore $SSL_DIR/keystores/trust.p12 \
  -storetype PKCS12 \
  -storepass Welcome1 | \
  grep -E "Alias|Entry type|Owner"
```

3. Establece permisos apropiados sobre los keystores:

```bash
chmod 640 $SSL_DIR/keystores/identity.p12
chmod 640 $SSL_DIR/keystores/trust.p12
ls -la $SSL_DIR/keystores/
```

#### Resultado esperado

```
Alias name: wls-lab-ca
Entry type: trustedCertEntry
Owner: CN=WebLogic Lab Root CA, ...
```

#### Verificación

```bash
# Ambos archivos deben existir y tener el tamaño apropiado
ls -lh $SSL_DIR/keystores/
# identity.p12  ~5KB
# trust.p12     ~3KB
```

---

### Paso 6 — Configurar SSL en el Administration Server mediante WLST

**Objetivo:** Usar WLST en modo online para configurar los keystores PKCS12, habilitar el puerto SSL 7002 y establecer TLS 1.3 como protocolo activo en el Administration Server.

#### Instrucciones

1. Asegúrate de que el Administration Server esté en ejecución:

```bash
curl -s -o /dev/null -w "%{http_code}" \
  http://$ADMIN_HOST:$ADMIN_PORT_PLAIN/console/
# Debe retornar 200 o 302
```

2. Crea el script WLST de configuración SSL:

```bash
cat > /tmp/configure_ssl.py << 'WLST_EOF'
# ─────────────────────────────────────────────────────────────────────────────
# Script WLST: Configuración TLS 1.3 en Administration Server
# Lab 06-00-01 — WebLogic 15
# ─────────────────────────────────────────────────────────────────────────────
import os

# Parámetros de conexión y rutas
ADMIN_URL     = 't3://localhost:7001'
WLS_USER      = 'weblogic'
WLS_PASS      = 'Welcome1'
SERVER_NAME   = 'AdminServer'
SSL_DIR       = os.environ.get('DOMAIN_HOME', '/opt/oracle/domains/wls_domain') + '/security/ssl'
IDENTITY_KS   = SSL_DIR + '/keystores/identity.p12'
TRUST_KS      = SSL_DIR + '/keystores/trust.p12'
KS_PASS       = 'Welcome1'
SSL_PORT      = 7002
KEY_ALIAS     = 'wls-server'

print("=== Conectando a WebLogic Administration Server ===")
connect(WLS_USER, WLS_PASS, ADMIN_URL)

edit()
startEdit()

print("=== Configurando Keystores en " + SERVER_NAME + " ===")
cd('/Servers/' + SERVER_NAME)

# Tipo de keystore: Custom Identity + Custom Trust
set('KeyStores', 'CustomIdentityAndCustomTrust')

# Identity Store (PKCS12)
set('CustomIdentityKeyStoreFileName', IDENTITY_KS)
set('CustomIdentityKeyStoreType', 'PKCS12')
set('CustomIdentityKeyStorePassPhrase', KS_PASS)

# Trust Store (PKCS12)
set('CustomTrustKeyStoreFileName', TRUST_KS)
set('CustomTrustKeyStoreType', 'PKCS12')
set('CustomTrustKeyStorePassPhrase', KS_PASS)

# Habilitar puerto SSL
set('SSLListenPortEnabled', Boolean(1))
set('SSLListenPort', SSL_PORT)

print("=== Configurando parámetros SSL/TLS ===")
cd('/Servers/' + SERVER_NAME + '/SSL/' + SERVER_NAME)

# Usar JSSE (obligatorio para TLS 1.3)
set('JSSEEnabled', Boolean(1))

# Alias de la clave privada (debe coincidir con el alias del keystore)
set('ServerPrivateKeyAlias', KEY_ALIAS)
set('ServerPrivateKeyPassPhrase', KS_PASS)

# Protocolos habilitados: solo TLS 1.3 y TLS 1.2
# TLS 1.3 es el primario; TLS 1.2 se mantiene para compatibilidad de clientes
set('EnabledProtocols', 'TLSv1.3,TLSv1.2')

# Cipher suites para TLS 1.2 (las de TLS 1.3 las gestiona JSSE automáticamente)
# Solo ECDHE con AES-GCM — sin CBC, sin RC4, sin 3DES
from jarray import array
from java.lang import String
ciphers = array([
    'TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384',
    'TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256'
], String)
set('Ciphersuites', ciphers)

# Verificación de hostname: BEA (compatible con localhost en laboratorio)
set('HostnameVerificationIgnored', Boolean(0))
set('HostnameVerifier', 'BEA Hostname Verifier')

print("=== Guardando y activando cambios ===")
save()
activate(block='true')

print("=== Configuración SSL completada exitosamente ===")
print("Identity Store : " + IDENTITY_KS)
print("Trust Store    : " + TRUST_KS)
print("SSL Port       : " + str(SSL_PORT))
print("Protocols      : TLSv1.3, TLSv1.2")
print("")
print("IMPORTANTE: Reinicia el Administration Server para aplicar los cambios.")

disconnect()
WLST_EOF
```

3. Ejecuta el script WLST:

```bash
$WL_HOME/common/bin/wlst.sh /tmp/configure_ssl.py
```

#### Resultado esperado

```
=== Conectando a WebLogic Administration Server ===
Connecting to t3://localhost:7001 with userid weblogic ...
Successfully connected to Administration Server...
=== Configurando Keystores en AdminServer ===
=== Configurando parámetros SSL/TLS ===
=== Guardando y activando cambios ===
Activating all your changes ...
The edit lock associated with this edit session is released ...
=== Configuración SSL completada exitosamente ===
Identity Store : /opt/oracle/domains/wls_domain/security/ssl/keystores/identity.p12
Trust Store    : /opt/oracle/domains/wls_domain/security/ssl/keystores/trust.p12
SSL Port       : 7002
Protocols      : TLSv1.3, TLSv1.2

IMPORTANTE: Reinicia el Administration Server para aplicar los cambios.
```

#### Verificación

```bash
# Verificar que los cambios se reflejan en config.xml
grep -A 5 "ssl-listen-port" $DOMAIN_HOME/config/config.xml
grep "CustomIdentity" $DOMAIN_HOME/config/config.xml | head -4
```

---

### Paso 7 — Configurar JVM para hardening adicional de TLS

**Objetivo:** Añadir parámetros de JVM al script de arranque del dominio para deshabilitar algoritmos inseguros a nivel de plataforma Java y reforzar la configuración TLS.

#### Instrucciones

1. Localiza el archivo `setDomainEnv.sh` del dominio:

```bash
ls $DOMAIN_HOME/bin/setDomainEnv.sh
```

2. Crea un archivo de extensión para no modificar directamente `setDomainEnv.sh` (mejor práctica):

```bash
cat > $DOMAIN_HOME/bin/setUserOverrides.sh << 'EOF'
#!/bin/bash
# ─────────────────────────────────────────────────────────────────────────────
# setUserOverrides.sh — Hardening TLS para WebLogic 15
# Lab 06-00-01
# ─────────────────────────────────────────────────────────────────────────────

echo "INFO: Aplicando parámetros de hardening TLS..."

# Deshabilitar protocolos inseguros y algoritmos débiles a nivel de JVM
# Esto complementa la configuración de WebLogic y actúa como segunda capa
TLS_HARDENING="-Djdk.tls.rejectClientInitiatedRenegotiation=true"

# Grupos de curvas elípticas permitidos (solo curvas modernas)
NAMED_GROUPS="-Djdk.tls.namedGroups=secp256r1,secp384r1,x25519"

# Verificación de hostname estricta para clientes JSSE
HOSTNAME_VERIF="-Dhttps.protocols=TLSv1.3,TLSv1.2"

# Deshabilitar algoritmos inseguros (refuerza java.security del JDK)
# Nota: En Java 21, TLSv1.0 y TLSv1.1 ya están deshabilitados por defecto,
# pero lo forzamos explícitamente para cumplimiento documentado.
DISABLED_ALGOS="-Djdk.tls.disabledAlgorithms=SSLv3,TLSv1,TLSv1.1,RC4,DES,3DES_EDE_CBC,MD5withRSA,EC keySize < 224,DH keySize < 2048,RSA keySize < 2048"

# Agregar todos los parámetros a JAVA_OPTIONS
JAVA_OPTIONS="${JAVA_OPTIONS} ${TLS_HARDENING} ${NAMED_GROUPS} ${HOSTNAME_VERIF} ${DISABLED_ALGOS}"

export JAVA_OPTIONS
echo "INFO: JAVA_OPTIONS TLS configuradas correctamente."
EOF

chmod +x $DOMAIN_HOME/bin/setUserOverrides.sh
```

3. Verifica que WebLogic cargará el archivo de overrides (debe existir la referencia en `setDomainEnv.sh`):

```bash
grep -i "setUserOverrides" $DOMAIN_HOME/bin/setDomainEnv.sh | head -5
```

> **Nota:** WebLogic 15 carga automáticamente `setUserOverrides.sh` si existe en `$DOMAIN_HOME/bin/`. Si tu versión no lo hace, añade la siguiente línea al final de `setDomainEnv.sh`:
> ```bash
> [ -f "${DOMAIN_HOME}/bin/setUserOverrides.sh" ] && . "${DOMAIN_HOME}/bin/setUserOverrides.sh"
> ```

4. Verifica el contenido del archivo `java.security` del JDK para confirmar los algoritmos ya deshabilitados por defecto:

```bash
grep "jdk.tls.disabledAlgorithms" $JAVA_HOME/conf/security/java.security
```

#### Resultado esperado

```
jdk.tls.disabledAlgorithms=SSLv3, TLSv1, TLSv1.1, RC4, DES, MD5withRSA, \
    DH keySize < 1024, EC keySize < 224, 3DES_EDE_CBC, anon, NULL
```

#### Verificación

```bash
# El archivo de overrides debe existir y ser ejecutable
ls -la $DOMAIN_HOME/bin/setUserOverrides.sh
# -rwxr-xr-x ... setUserOverrides.sh
```

---

### Paso 8 — Reiniciar el Administration Server y verificar arranque SSL

**Objetivo:** Aplicar todos los cambios de configuración reiniciando el Administration Server y confirmar que arranca con SSL habilitado en el puerto 7002.

#### Instrucciones

1. Detén el Administration Server de forma controlada:

```bash
# Si usas el script de stop estándar:
$DOMAIN_HOME/bin/stopWebLogic.sh

# O si tienes WLST disponible:
# $WL_HOME/common/bin/wlst.sh -e "connect('weblogic','Welcome1','t3://localhost:7001'); shutdown('AdminServer','Server',ignoreSessions='true')"
```

2. Inicia el Administration Server:

```bash
nohup $DOMAIN_HOME/bin/startWebLogic.sh > /tmp/adminserver.log 2>&1 &
echo "PID del proceso: $!"
```

3. Monitorea el log de arranque hasta que el servidor esté listo (espera hasta 3 minutos):

```bash
tail -f /tmp/adminserver.log | grep -E "RUNNING|SSL|TLS|7002|ERROR|CRITICAL" &
TAIL_PID=$!

# Esperar hasta 180 segundos
for i in $(seq 1 36); do
  sleep 5
  if grep -q "RUNNING" /tmp/adminserver.log 2>/dev/null; then
    echo "✓ Administration Server en estado RUNNING"
    kill $TAIL_PID 2>/dev/null
    break
  fi
  echo "Esperando arranque... ($((i*5))s)"
done
```

4. Verifica que el puerto SSL 7002 esté escuchando:

```bash
ss -tlnp | grep 7002
# o con netstat:
# netstat -tlnp | grep 7002
```

#### Resultado esperado en el log

```
<INFO> <WebLogicServer> ... SSL port 7002 is now listening...
<INFO> <WebLogicServer> ... <BEA-000360> <Server started in RUNNING mode>
```

#### Verificación

```bash
ss -tlnp | grep 7002
# Debe mostrar: LISTEN  0  ...  *:7002  ...  java
```

---

### Paso 9 — Verificar TLS 1.3 con openssl s_client

**Objetivo:** Confirmar usando `openssl s_client` que el Administration Server negocia correctamente TLS 1.3 y que el certificado de servidor es el generado en los pasos anteriores.

#### Instrucciones

1. Verifica la conexión TLS 1.3 (debe funcionar):

```bash
echo "Q" | openssl s_client \
  -connect $ADMIN_HOST:$ADMIN_PORT_SSL \
  -tls1_3 \
  -servername $ADMIN_HOST \
  -CAfile $SSL_DIR/ca/ca.crt \
  2>/dev/null | grep -E "Protocol|Cipher|subject|issuer|Verify"
```

2. Verifica la conexión TLS 1.2 (debe funcionar como fallback):

```bash
echo "Q" | openssl s_client \
  -connect $ADMIN_HOST:$ADMIN_PORT_SSL \
  -tls1_2 \
  -servername $ADMIN_HOST \
  -CAfile $SSL_DIR/ca/ca.crt \
  2>/dev/null | grep -E "Protocol|Cipher"
```

3. Verifica que TLS 1.1 es rechazado (debe fallar):

```bash
echo "Q" | openssl s_client \
  -connect $ADMIN_HOST:$ADMIN_PORT_SSL \
  -tls1_1 \
  -servername $ADMIN_HOST \
  -CAfile $SSL_DIR/ca/ca.crt \
  2>&1 | grep -E "Protocol|handshake failure|no protocols"
```

4. Verifica que TLS 1.0 es rechazado (debe fallar):

```bash
echo "Q" | openssl s_client \
  -connect $ADMIN_HOST:$ADMIN_PORT_SSL \
  -tls1 \
  -servername $ADMIN_HOST \
  -CAfile $SSL_DIR/ca/ca.crt \
  2>&1 | grep -E "Protocol|handshake failure|no protocols"
```

5. Muestra el certificado completo para confirmar SANs:

```bash
echo "Q" | openssl s_client \
  -connect $ADMIN_HOST:$ADMIN_PORT_SSL \
  -tls1_3 \
  -servername $ADMIN_HOST \
  -CAfile $SSL_DIR/ca/ca.crt \
  2>/dev/null | openssl x509 -noout -text | \
  grep -A 4 "Subject Alternative Name"
```

#### Resultado esperado

```
# Paso 1 — TLS 1.3 (DEBE FUNCIONAR):
Protocol  : TLSv1.3
Cipher    : TLS_AES_256_GCM_SHA384
subject=CN=localhost, OU=Administration, O=WebLogic Lab, ...
issuer=CN=WebLogic Lab Root CA, ...
Verify return code: 0 (ok)

# Paso 2 — TLS 1.2 (DEBE FUNCIONAR):
Protocol  : TLSv1.2
Cipher    : ECDHE-RSA-AES256-GCM-SHA384

# Paso 3 — TLS 1.1 (DEBE FALLAR):
140...:error:...handshake failure...
# O: no protocols available

# Paso 4 — TLS 1.0 (DEBE FALLAR):
140...:error:...handshake failure...

# Paso 5 — SANs:
X509v3 Subject Alternative Name:
    DNS:localhost, DNS:adminserver, IP Address:127.0.0.1
```

#### Verificación

```bash
# Resumen de verificación en una sola línea
echo "Q" | openssl s_client \
  -connect $ADMIN_HOST:$ADMIN_PORT_SSL \
  -tls1_3 \
  -CAfile $SSL_DIR/ca/ca.crt 2>/dev/null | \
  grep "Protocol" | grep -q "TLSv1.3" && \
  echo "✓ TLS 1.3 HABILITADO CORRECTAMENTE" || \
  echo "✗ ERROR: TLS 1.3 no negociado"
```

---

### Paso 10 — Verificar rechazo de cipher suites débiles

**Objetivo:** Confirmar que los cipher suites débiles (RC4, 3DES, NULL) son rechazados por el servidor.

#### Instrucciones

1. Intenta conectar forzando una cipher suite con 3DES (debe fallar):

```bash
echo "Q" | openssl s_client \
  -connect $ADMIN_HOST:$ADMIN_PORT_SSL \
  -cipher "DES-CBC3-SHA" \
  -tls1_2 \
  -CAfile $SSL_DIR/ca/ca.crt \
  2>&1 | grep -E "handshake|no cipher|alert"
```

2. Intenta conectar forzando RC4 (debe fallar):

```bash
echo "Q" | openssl s_client \
  -connect $ADMIN_HOST:$ADMIN_PORT_SSL \
  -cipher "RC4-SHA" \
  -tls1_2 \
  -CAfile $SSL_DIR/ca/ca.crt \
  2>&1 | grep -E "handshake|no cipher|alert"
```

3. Enumera los cipher suites que el servidor acepta en TLS 1.2:

```bash
# Prueba cada cipher suite disponible en OpenSSL contra el servidor
for cipher in $(openssl ciphers 'ALL:eNULL' | tr ':' ' '); do
  result=$(echo "Q" | openssl s_client \
    -connect $ADMIN_HOST:$ADMIN_PORT_SSL \
    -cipher "$cipher" \
    -tls1_2 \
    -CAfile $SSL_DIR/ca/ca.crt \
    2>/dev/null | grep "Cipher is")
  if [ -n "$result" ]; then
    echo "ACEPTADO: $cipher — $result"
  fi
done
```

#### Resultado esperado

```
# Paso 1 y 2: Deben fallar con error de handshake
SSL_ERROR_RX_RECORD_TOO_LONG
# o: handshake failure

# Paso 3: Solo deben aparecer:
ACEPTADO: ECDHE-RSA-AES256-GCM-SHA384 — Cipher is ECDHE-RSA-AES256-GCM-SHA384
ACEPTADO: ECDHE-RSA-AES128-GCM-SHA256 — Cipher is ECDHE-RSA-AES128-GCM-SHA256
```

#### Verificación

```bash
# Ningún cipher débil debe estar aceptado
echo "Q" | openssl s_client \
  -connect $ADMIN_HOST:$ADMIN_PORT_SSL \
  -cipher "RC4:3DES:DES:NULL" \
  -tls1_2 \
  -CAfile $SSL_DIR/ca/ca.crt \
  2>&1 | grep -c "handshake failure" | \
  grep -q "1" && echo "✓ Ciphers débiles RECHAZADOS" || \
  echo "✗ ATENCIÓN: Revisar configuración de ciphers"
```

---

### Paso 11 — Acceder a la consola de administración por HTTPS

**Objetivo:** Confirmar que la consola de WebLogic es accesible vía HTTPS en el puerto 7002 con TLS 1.3.

#### Instrucciones

1. Prueba el acceso HTTP a la consola de administración usando `curl` con el certificado de la CA:

```bash
curl -v \
  --cacert $SSL_DIR/ca/ca.crt \
  https://$ADMIN_HOST:$ADMIN_PORT_SSL/console/ \
  -o /dev/null \
  -w "\nHTTP Status: %{http_code}\nSSL Version: %{ssl_version}\nSSL Cipher: %{ssl_cipher}\n" \
  2>&1 | grep -E "HTTP Status|SSL Version|SSL Cipher|TLS|subject|issuer"
```

2. Verifica el acceso a la API REST de WebLogic por HTTPS:

```bash
curl -s \
  --cacert $SSL_DIR/ca/ca.crt \
  -u weblogic:Welcome1 \
  https://$ADMIN_HOST:$ADMIN_PORT_SSL/management/weblogic/latest/serverRuntime \
  | python3 -m json.tool 2>/dev/null | grep -E "name|state|listenPort" | head -6
```

#### Resultado esperado

```
HTTP Status: 200
SSL Version: TLSv1.3
SSL Cipher: TLS_AES_256_GCM_SHA384

# REST API:
"name": "AdminServer",
"state": "RUNNING",
"SSLListenPort": 7002,
```

#### Verificación

```bash
# El código HTTP debe ser 200 (consola accesible)
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
  --cacert $SSL_DIR/ca/ca.crt \
  https://$ADMIN_HOST:$ADMIN_PORT_SSL/console/)
[ "$HTTP_CODE" = "200" ] && \
  echo "✓ Consola accesible por HTTPS (TLS 1.3)" || \
  echo "✗ Error HTTP: $HTTP_CODE"
```

---

### Paso 12 — Documentar el runbook de seguridad

**Objetivo:** Generar un documento de runbook que resuma la configuración implementada, los comandos de verificación y los criterios de cumplimiento.

#### Instrucciones

1. Crea el runbook de seguridad TLS:

```bash
cat > $DOMAIN_HOME/security/TLS_RUNBOOK.md << 'EOF'
# Runbook de Seguridad TLS — WebLogic 15 Administration Server
## Fecha de implementación: $(date +%Y-%m-%d)
## Laboratorio: 06-00-01

## Resumen de Configuración

| Parámetro              | Valor                                          |
|------------------------|------------------------------------------------|
| Servidor               | AdminServer                                    |
| Puerto SSL             | 7002                                           |
| Protocolos habilitados | TLSv1.3, TLSv1.2                              |
| Protocolos deshabilitados | SSLv3, TLSv1.0, TLSv1.1                    |
| Identity Store         | $DOMAIN_HOME/security/ssl/keystores/identity.p12 |
| Trust Store            | $DOMAIN_HOME/security/ssl/keystores/trust.p12  |
| Alias clave privada    | wls-server                                     |
| Cipher suites TLS 1.2  | ECDHE-RSA-AES256-GCM-SHA384, ECDHE-RSA-AES128-GCM-SHA256 |
| Cipher suites TLS 1.3  | Gestionadas por JSSE (AES-GCM, ChaCha20-Poly1305) |

## Comandos de Verificación

### Verificar TLS 1.3 activo
```bash
echo "Q" | openssl s_client -connect localhost:7002 -tls1_3 \
  -CAfile $SSL_DIR/ca/ca.crt 2>/dev/null | grep Protocol
# Esperado: Protocol : TLSv1.3
```

### Verificar TLS 1.1 rechazado
```bash
echo "Q" | openssl s_client -connect localhost:7002 -tls1_1 \
  -CAfile $SSL_DIR/ca/ca.crt 2>&1 | grep "handshake failure"
# Esperado: handshake failure
```

## Criterios de Cumplimiento

- [x] TLS 1.3 habilitado y negociado correctamente
- [x] TLS 1.0 y 1.1 rechazados
- [x] SSLv3 rechazado
- [x] RC4 y 3DES no aceptados
- [x] Certificado con SANs correctos
- [x] Cadena de confianza válida

## Parámetros JVM aplicados

```
-Djdk.tls.rejectClientInitiatedRenegotiation=true
-Djdk.tls.namedGroups=secp256r1,secp384r1,x25519
-Dhttps.protocols=TLSv1.3,TLSv1.2
-Djdk.tls.disabledAlgorithms=SSLv3,TLSv1,TLSv1.1,RC4,DES,3DES_EDE_CBC,...
```

## Próxima revisión: $(date -d "+6 months" +%Y-%m-%d)
EOF

echo "Runbook generado en: $DOMAIN_HOME/security/TLS_RUNBOOK.md"
```

#### Resultado esperado

```
Runbook generado en: /opt/oracle/domains/wls_domain/security/TLS_RUNBOOK.md
```

#### Verificación

```bash
ls -la $DOMAIN_HOME/security/TLS_RUNBOOK.md
wc -l $DOMAIN_HOME/security/TLS_RUNBOOK.md
# Debe tener al menos 40 líneas
```

---

## Validación y Pruebas Finales

Ejecuta la siguiente batería de verificaciones para confirmar que toda la configuración es correcta:

```bash
#!/bin/bash
# ─────────────────────────────────────────────────────────────────────────────
# Validación final — Lab 06-00-01
# ─────────────────────────────────────────────────────────────────────────────
PASS=0
FAIL=0
CA_CERT=$SSL_DIR/ca/ca.crt
TARGET="$ADMIN_HOST:$ADMIN_PORT_SSL"

check() {
    local desc="$1"
    local cmd="$2"
    local expected="$3"
    if eval "$cmd" 2>/dev/null | grep -q "$expected"; then
        echo "✓ PASS: $desc"
        ((PASS++))
    else
        echo "✗ FAIL: $desc"
        ((FAIL++))
    fi
}

echo "════════════════════════════════════════════════"
echo "  VALIDACIÓN TLS — Lab 06-00-01"
echo "════════════════════════════════════════════════"

# 1. Puerto 7002 escuchando
check "Puerto SSL 7002 activo" \
  "ss -tlnp" "7002"

# 2. TLS 1.3 negociado
check "TLS 1.3 habilitado" \
  "echo Q | openssl s_client -connect $TARGET -tls1_3 -CAfile $CA_CERT" \
  "TLSv1.3"

# 3. TLS 1.2 como fallback
check "TLS 1.2 como fallback" \
  "echo Q | openssl s_client -connect $TARGET -tls1_2 -CAfile $CA_CERT" \
  "TLSv1.2"

# 4. TLS 1.1 rechazado
check "TLS 1.1 rechazado" \
  "echo Q | openssl s_client -connect $TARGET -tls1_1 -CAfile $CA_CERT 2>&1" \
  "handshake failure\|no protocols\|alert"

# 5. TLS 1.0 rechazado
check "TLS 1.0 rechazado" \
  "echo Q | openssl s_client -connect $TARGET -tls1 -CAfile $CA_CERT 2>&1" \
  "handshake failure\|no protocols\|alert"

# 6. Certificado con CN=localhost
check "Certificado CN correcto" \
  "echo Q | openssl s_client -connect $TARGET -tls1_3 -CAfile $CA_CERT" \
  "CN=localhost"

# 7. Cadena de confianza válida
check "Cadena de confianza válida" \
  "echo Q | openssl s_client -connect $TARGET -tls1_3 -CAfile $CA_CERT" \
  "Verify return code: 0"

# 8. SANs presentes
check "SANs configurados" \
  "echo Q | openssl s_client -connect $TARGET -tls1_3 -CAfile $CA_CERT \
   2>/dev/null | openssl x509 -noout -text" \
  "DNS:localhost"

# 9. Cipher suite AES-GCM en TLS 1.2
check "Cipher AES-GCM en TLS 1.2" \
  "echo Q | openssl s_client -connect $TARGET -tls1_2 -CAfile $CA_CERT" \
  "GCM"

# 10. Consola accesible por HTTPS
check "Consola HTTPS accesible" \
  "curl -s -o /dev/null -w '%{http_code}' --cacert $CA_CERT https://$TARGET/console/" \
  "200\|302"

echo "════════════════════════════════════════════════"
echo "  RESULTADO: $PASS PASS | $FAIL FAIL"
echo "════════════════════════════════════════════════"
[ $FAIL -eq 0 ] && exit 0 || exit 1
```

**Resultado esperado:** `RESULTADO: 10 PASS | 0 FAIL`

---

## Resolución de Problemas

### Problema 1 — El Administration Server arranca pero el puerto 7002 no aparece en `ss -tlnp`

**Síntomas:**
- El servidor está en estado `RUNNING` en el log.
- `ss -tlnp | grep 7002` no retorna ninguna línea.
- Los logs muestran mensajes como `SSL configuration failed` o `Cannot load keystore`.

**Causa probable:**
El keystore de identidad o confianza no puede ser leído por WebLogic. Esto ocurre cuando: (a) la ruta al archivo `.p12` es incorrecta o el archivo no existe, (b) la contraseña del keystore es incorrecta, o (c) el usuario que ejecuta WebLogic no tiene permisos de lectura sobre los archivos.

**Solución:**

```bash
# 1. Verificar que los archivos existen y son legibles
ls -la $SSL_DIR/keystores/identity.p12 $SSL_DIR/keystores/trust.p12

# 2. Verificar que el usuario oracle puede leerlos
sudo -u oracle keytool -list \
  -keystore $SSL_DIR/keystores/identity.p12 \
  -storetype PKCS12 \
  -storepass Welcome1

# 3. Verificar permisos y corregir si es necesario
chmod 640 $SSL_DIR/keystores/*.p12
chown oracle:oracle $SSL_DIR/keystores/*.p12

# 4. Buscar errores específicos de SSL en el log del servidor
grep -i "ssl\|keystore\|certificate\|PKCS12" /tmp/adminserver.log | tail -20

# 5. Verificar la configuración en config.xml
grep -A 3 "custom-identity-key-store-file-name" $DOMAIN_HOME/config/config.xml

# 6. Si la ruta en config.xml usa variables de entorno, verificar que están expandidas
# Corregir ejecutando de nuevo el script WLST con rutas absolutas
```

---

### Problema 2 — `openssl s_client -tls1_3` falla con "no protocols available" o "handshake failure"

**Síntomas:**
- El puerto 7002 está activo y escuchando.
- La conexión con `-tls1_2` funciona, pero `-tls1_3` falla.
- El log de WebLogic no muestra errores evidentes.

**Causa probable:**
TLS 1.3 no está habilitado correctamente en WebLogic porque: (a) `JSSEEnabled` no está en `true` (WebLogic usa su propio proveedor SSL nativo que no soporta TLS 1.3), (b) el parámetro `EnabledProtocols` no fue aplicado correctamente, o (c) los cambios WLST se guardaron pero no se activaron antes del reinicio.

**Solución:**

```bash
# 1. Verificar que JSSE está habilitado en config.xml
grep -i "jsse-enabled" $DOMAIN_HOME/config/config.xml
# Debe mostrar: <jsse-enabled>true</jsse-enabled>

# 2. Verificar el valor de EnabledProtocols
grep -i "enabled-protocols\|EnabledProtocols" $DOMAIN_HOME/config/config.xml
# Debe mostrar: <enabled-protocols>TLSv1.3,TLSv1.2</enabled-protocols>

# 3. Si no aparecen, re-ejecutar el script WLST con el servidor en RUNNING
$WL_HOME/common/bin/wlst.sh << 'EOF'
connect('weblogic', 'Welcome1', 't3://localhost:7001')
serverConfig()
cd('/Servers/AdminServer/SSL/AdminServer')
print("JSSEEnabled:", get('JSSEEnabled'))
print("EnabledProtocols:", get('EnabledProtocols'))
disconnect()
EOF

# 4. Si los valores son incorrectos, re-ejecutar el script de configuración
$WL_HOME/common/bin/wlst.sh /tmp/configure_ssl.py

# 5. Verificar que Java 21 soporta TLS 1.3 (debe indicar TLSv1.3 como soportado)
$JAVA_HOME/bin/java -Djavax.net.debug=ssl:handshake \
  -cp $WL_HOME/server/lib/weblogic.jar \
  weblogic.security.utils.SSLVersion 2>&1 | grep -i "tls1.3\|TLSv1.3" | head -5

# 6. Confirmar que JAVA_OPTIONS con -Djdk.tls.disabledAlgorithms NO incluye TLSv1.3
grep "disabledAlgorithms" $DOMAIN_HOME/bin/setUserOverrides.sh
# TLSv1.3 NO debe aparecer en la lista de algoritmos deshabilitados
```

---

## Limpieza del Laboratorio

> **Importante:** Este laboratorio modifica la configuración SSL del Administration Server. La limpieza revierte el servidor a configuración HTTP simple. **Toma un snapshot de la VM antes de ejecutar la limpieza** si deseas conservar la configuración TLS para laboratorios posteriores.

```bash
# ─────────────────────────────────────────────────────────────────────────────
# LIMPIEZA — Lab 06-00-01
# Ejecutar solo si necesitas revertir al estado anterior al laboratorio
# ─────────────────────────────────────────────────────────────────────────────

# 1. Revertir configuración SSL en WebLogic (volver a keystores demo por defecto)
$WL_HOME/common/bin/wlst.sh << 'WLST_CLEANUP'
connect('weblogic', 'Welcome1', 't3://localhost:7001')
edit()
startEdit()
cd('/Servers/AdminServer')
set('KeyStores', 'DemoIdentityAndDemoTrust')
set('SSLListenPortEnabled', Boolean(0))
cd('/Servers/AdminServer/SSL/AdminServer')
set('JSSEEnabled', Boolean(0))
save()
activate(block='true')
print("Configuración SSL revertida.")
disconnect()
WLST_CLEANUP

# 2. Eliminar el archivo de overrides de JVM
rm -f $DOMAIN_HOME/bin/setUserOverrides.sh

# 3. Eliminar archivos temporales (conservar $SSL_DIR para referencia futura)
rm -f /tmp/configure_ssl.py
rm -f /tmp/adminserver.log

# 4. Reiniciar el Administration Server para aplicar la reversión
$DOMAIN_HOME/bin/stopWebLogic.sh
sleep 5
nohup $DOMAIN_HOME/bin/startWebLogic.sh > /tmp/adminserver_clean.log 2>&1 &

# 5. Verificar que el servidor arranca sin SSL
sleep 60
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" \
  http://localhost:7001/console/
# Debe retornar: HTTP Status: 200

echo "Limpieza completada. Servidor revertido a configuración HTTP."
echo "Nota: Los archivos PKI en $SSL_DIR se conservan para referencia."
```

---

## Resumen

En este laboratorio has construido una solución completa de seguridad de transporte para WebLogic 15, cubriendo todo el ciclo desde la generación de la PKI hasta la verificación y documentación:

| Tarea completada                                             | Herramienta usada              |
|--------------------------------------------------------------|--------------------------------|
| PKI mínima: CA raíz autofirmada + certificado de servidor    | OpenSSL 3.0                    |
| Keystores PKCS12 de identidad y confianza                    | openssl pkcs12, keytool        |
| Configuración SSL en WebLogic (keystores, puerto, protocolos)| WLST (Jython)                  |
| Hardening de JVM (algoritmos deshabilitados, curvas)         | setUserOverrides.sh            |
| Verificación TLS 1.3 activo y protocolos inseguros rechazados| openssl s_client               |
| Verificación de cipher suites                                | openssl s_client               |
| Documentación como runbook de seguridad                      | Markdown                       |

### Conceptos clave reforzados

- **TLS 1.3** reduce el handshake a 1-RTT, elimina primitivas inseguras y aplica PFS mediante ECDHE obligatorio.
- **JSSE** debe estar habilitado (`JSSEEnabled=true`) en WebLogic para soportar TLS 1.3; el proveedor SSL nativo de WebLogic no lo soporta.
- **PKCS12** es el formato de keystore recomendado por su portabilidad e interoperabilidad frente a JKS.
- La deshabilitación de protocolos opera en **dos capas**: WebLogic (`EnabledProtocols`) y JVM (`-Djdk.tls.disabledAlgorithms`), siendo la segunda la barrera definitiva.
- Los **Subject Alternative Names (SANs)** son obligatorios en certificados modernos; el campo CN solo ya no es suficiente para la validación de hostname.

### Recursos adicionales

- [Oracle WebLogic Server — Configuring SSL](https://docs.oracle.com/en/middleware/standalone/weblogic-server/14.1.1.0/security/configure-ssl.html)
- [JSSE Reference Guide — Java 21](https://docs.oracle.com/en/java/javase/21/security/java-secure-socket-extension-jsse-reference-guide.html)
- [RFC 8446: TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446)
- [OWASP Transport Layer Protection Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Protection_Cheat_Sheet.html)
- [Mozilla Server Side TLS Guidelines](https://wiki.mozilla.org/Security/Server_Side_TLS)
- [testssl.sh — Auditoría completa de TLS](https://github.com/drwetter/testssl.sh)

---
*Lab 06-00-01 — Curso WebLogic 15 — Seguridad de Transporte TLS 1.3*
