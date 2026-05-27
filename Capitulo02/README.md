# Crear y revisar recursos en un dominio, utilizando exclusivamente WRC sin usar la consola tradicional.

## 1. Metadatos

| Campo            | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 150 minutos                                |
| **Complejidad**  | Media                                      |
| **Nivel Bloom**  | Crear (Create)                             |
| **Laboratorio**  | 02-00-01                                   |
| **Dependencia**  | Lab 01-00-01 completado                    |

---

## 2. Descripción General

En este laboratorio explorarás y operarás un dominio WebLogic 15 **sin utilizar en ningún momento la consola web tradicional** (`/console`). Utilizarás WebLogic Remote Console (WRC) como herramienta principal para instalar, conectar y crear recursos reales: un Managed Server adicional, un JDBC Data Source (H2 embebida), un JMS Module con una Queue y un Work Manager con restricciones de capacidad. A lo largo del laboratorio correlacionarás cada acción en WRC con los cambios producidos en el archivo `config.xml`, consolidando así la comprensión de la arquitectura del dominio y el ciclo Edit/Activate de WebLogic.

---

## 3. Objetivos de Aprendizaje

Al finalizar este laboratorio serás capaz de:

- [ ] Instalar y conectar WebLogic Remote Console (WRC) al Administration Server mediante la REST Management API.
- [ ] Crear y configurar recursos de dominio (Managed Server, JDBC Data Source, JMS Module/Queue, Work Manager) exclusivamente desde WRC.
- [ ] Correlacionar cada operación en WRC con los cambios reflejados en `config.xml` y los ficheros auxiliares del dominio.
- [ ] Aplicar *targeting* de recursos a servidores y clusters usando WRC.
- [ ] Desplegar la aplicación de muestra Jakarta EE 10 desde WRC validando el ciclo completo de administración moderna.

---

## 4. Prerrequisitos

### Conocimientos previos

- Haber completado el **Lab 01-00-01**: WebLogic 15 instalado, dominio `base_domain` creado y Administration Server arrancado en el puerto 7001.
- Conceptos básicos de JDBC (driver, URL de conexión, pool de conexiones).
- Conceptos básicos de mensajería JMS (módulos, fábricas de conexión, colas).
- Comprensión del modelo REST (verbos HTTP: GET, POST, PUT, DELETE).
- Familiaridad con la línea de comandos Linux (bash).

### Acceso y recursos necesarios

- Administration Server arrancado y accesible en `http://localhost:7001`.
- Usuario `weblogic` con contraseña `Welcome1` (credenciales de laboratorio).
- Acceso a internet **o** archivo `wrc-2.4.x.zip` pre-descargado en el entorno.
- Aplicación de muestra `sample-jakarta-ee10.war` proporcionada por el instructor (disponible en `~/lab-resources/`).
- Driver H2 JAR: `h2-2.x.x.jar` disponible en `~/lab-resources/` (o descargable durante el lab).

> ⚠️ **Restricción de laboratorio:** La URL `http://localhost:7001/console` queda **prohibida** durante todo este laboratorio. Si la abres accidentalmente, ciérrala de inmediato y continúa con WRC.

---

## 5. Entorno de Laboratorio

### Hardware mínimo recomendado

| Recurso       | Mínimo          | Recomendado     |
|---------------|-----------------|-----------------|
| RAM           | 8 GB            | 16 GB           |
| CPU           | 4 núcleos       | 8 núcleos       |
| Disco libre   | 20 GB (SSD)     | 40 GB (SSD)     |
| Red           | Loopback (lab)  | 1 GbE           |

### Software requerido

| Componente                        | Versión          | Ubicación / Notas                                   |
|-----------------------------------|------------------|-----------------------------------------------------|
| Oracle JDK                        | 21 LTS           | `/u01/jdk21`                                        |
| Oracle WebLogic Server            | 15.0             | `/u01/wls15`                                        |
| Dominio base                      | `base_domain`    | `/u01/domains/base_domain`                          |
| WebLogic Remote Console (WRC)     | 2.4.x            | Se instalará en `~/tools/wrc`                       |
| H2 Database JAR                   | 2.x              | `~/lab-resources/h2-2.x.x.jar`                      |
| Aplicación muestra Jakarta EE 10  | sample v1.0      | `~/lab-resources/sample-jakarta-ee10.war`           |
| curl                              | 7.76+            | Sistema operativo (incluido en Oracle Linux 8/9)    |
| Visual Studio Code (opcional)     | 1.89+            | Para editar y visualizar `config.xml`               |

### Variables de entorno de referencia

Estas variables se asumen configuradas desde el Lab 01. Verifica su existencia antes de comenzar:

```bash
echo $JAVA_HOME       # /u01/jdk21
echo $WL_HOME         # /u01/wls15/wlserver
echo $DOMAIN_HOME     # /u01/domains/base_domain
```

Si alguna no está definida, agrégalas a `~/.bashrc` y ejecuta `source ~/.bashrc`.

### Verificación del estado inicial

```bash
# Verificar que el Administration Server está RUNNING
curl -s -u weblogic:Welcome1 \
  http://localhost:7001/management/weblogic/latest/domainRuntime/serverLifeCycleRuntimes/AdminServer \
  | python3 -m json.tool | grep '"state"'
```

**Salida esperada:**
```json
"state": "RUNNING"
```

Si el Administration Server no está arrancado, inícialo antes de continuar:

```bash
cd $DOMAIN_HOME
nohup ./bin/startWebLogic.sh > /tmp/adminserver.log 2>&1 &
# Esperar ~60 segundos y volver a verificar
```

---

## 6. Procedimiento Paso a Paso

---

### Paso 1: Verificar y habilitar el endpoint REST del Administration Server

**Objetivo:** Confirmar que la REST Management API de WebLogic está activa y accesible, ya que WRC depende exclusivamente de ella.

#### Instrucciones

1. Abre una terminal y ejecuta la siguiente consulta REST para verificar que el endpoint de gestión responde correctamente:

```bash
curl -s -u weblogic:Welcome1 \
  http://localhost:7001/management/weblogic/latest \
  | python3 -m json.tool | head -20
```

2. Si el endpoint devuelve un JSON con el campo `"links"`, la REST API está activa. Si obtienes un error 404 o de conexión, verifica que el Administration Server esté en estado `RUNNING` (ver sección de entorno).

3. Anota la versión del dominio que aparece en la respuesta:

```bash
curl -s -u weblogic:Welcome1 \
  http://localhost:7001/management/weblogic/latest/domainConfig \
  | python3 -m json.tool | grep -E '"name"|"domainVersion"'
```

4. Toma una **instantánea de referencia** del `config.xml` antes de cualquier cambio:

```bash
cp $DOMAIN_HOME/config/config.xml /tmp/config_snapshot_inicial.xml
echo "Snapshot inicial guardado en /tmp/config_snapshot_inicial.xml"
wc -l /tmp/config_snapshot_inicial.xml
```

**Salida esperada (ejemplo):**
```
Snapshot inicial guardado en /tmp/config_snapshot_inicial.xml
87 /tmp/config_snapshot_inicial.xml
```

#### Verificación

```bash
curl -o /dev/null -s -w "HTTP Status: %{http_code}\n" \
  -u weblogic:Welcome1 \
  http://localhost:7001/management/weblogic/latest
```
Debes obtener `HTTP Status: 200`.

---

### Paso 2: Instalar y lanzar WebLogic Remote Console (WRC)

**Objetivo:** Descargar, instalar y ejecutar WRC 2.4.x para conectarlo al Administration Server.

#### Instrucciones

1. Crea el directorio de instalación de WRC:

```bash
mkdir -p ~/tools/wrc
cd ~/tools/wrc
```

2. **Opción A – Descarga desde internet:**

```bash
# Obtener la URL de la última release desde GitHub
WRC_VERSION="2.4.9"  # Ajusta a la versión disponible en el entorno
wget -O wrc-linux.zip \
  "https://github.com/oracle/weblogic-remote-console/releases/download/v${WRC_VERSION}/weblogic-remote-console-linux.zip"
unzip wrc-linux.zip
```

**Opción B – Usar archivo pre-descargado (sin internet):**

```bash
cp ~/lab-resources/wrc-linux.zip ~/tools/wrc/
cd ~/tools/wrc
unzip wrc-linux.zip
ls -la
```

3. Verifica el contenido extraído:

```bash
ls -la ~/tools/wrc/
# Debes ver: WebLogicRemoteConsole (ejecutable), lib/, etc.
```

4. Otorga permisos de ejecución si es necesario:

```bash
chmod +x ~/tools/wrc/WebLogicRemoteConsole
```

5. Lanza WRC. Al ser una aplicación Electron/desktop, se abrirá una ventana gráfica:

```bash
~/tools/wrc/WebLogicRemoteConsole &
```

> 💡 **Nota:** Si el entorno es headless (sin GUI), WRC también puede ejecutarse en modo servidor embebido. En ese caso, consulta al instructor para obtener la URL de acceso al WRC compartido del laboratorio.

6. Una vez abierta la interfaz de WRC, localiza la opción **"Add Admin Server Connection Provider"** en el panel izquierdo (o en el menú de conexiones).

7. Configura la conexión con los siguientes valores:

| Campo                  | Valor                        |
|------------------------|------------------------------|
| Connection Name        | `base_domain_lab`            |
| Username               | `weblogic`                   |
| Password               | `Welcome1`                   |
| URL                    | `http://localhost:7001`      |

8. Haz clic en **"Add"** o **"Connect"**. WRC establecerá la conexión REST con el Administration Server.

#### Salida esperada

En el panel de WRC deberás ver:
- El nombre de la conexión `base_domain_lab` activa (icono verde o estado "Connected").
- El árbol de navegación del dominio con nodos como: **Environment**, **Services**, **Deployments**, **Security**.

#### Verificación

En la terminal, confirma que WRC está consultando la API:

```bash
# WRC genera tráfico REST; puedes verificar con un curl equivalente
curl -s -u weblogic:Welcome1 \
  http://localhost:7001/management/weblogic/latest/domainConfig/servers \
  | python3 -m json.tool | grep '"name"'
```

Debes ver al menos `"AdminServer"` en la lista de servidores.

---

### Paso 3: Explorar la arquitectura del dominio desde WRC y correlacionar con config.xml

**Objetivo:** Navegar la interfaz de WRC para identificar los componentes del dominio y correlacionarlos con la estructura del `config.xml`.

#### Instrucciones

1. En WRC, expande el nodo **Environment** → **Servers**. Debes ver `AdminServer` listado.

2. Haz clic sobre `AdminServer` y observa sus propiedades: puerto de escucha, estado, modo de arranque.

3. Abre una segunda terminal y examina el `config.xml` actual:

```bash
grep -A 10 "<server>" $DOMAIN_HOME/config/config.xml
```

4. Localiza en el XML el bloque `<server>` correspondiente a `AdminServer` y compara los valores con los que muestra WRC (puerto, nombre, tipo de SSL).

5. En WRC, navega a **Environment** → **Domain** para ver las propiedades globales del dominio: nombre, modo de producción, versión de WebLogic.

6. Regresa al `config.xml` y ubica el elemento raíz `<domain>`:

```bash
head -30 $DOMAIN_HOME/config/config.xml
```

7. Documenta en tu cuaderno de laboratorio la correspondencia entre la interfaz WRC y el XML. Completa la siguiente tabla:

| Elemento en WRC                       | Elemento en config.xml               |
|---------------------------------------|--------------------------------------|
| Environment → Servers → AdminServer   | `<server><name>AdminServer</name>`   |
| Environment → Domain → Name           | `<name>base_domain</name>`           |
| Services → Data Sources               | `<jdbc-system-resource>` (futuro)    |
| Services → Messaging → JMS Modules   | `<jms-system-resource>` (futuro)     |

#### Verificación

```bash
# Verificar nombre del dominio desde REST API
curl -s -u weblogic:Welcome1 \
  http://localhost:7001/management/weblogic/latest/domainConfig \
  | python3 -m json.tool | grep '"name"'
```

---

### Paso 4: Crear un Managed Server adicional desde WRC

**Objetivo:** Añadir un nuevo Managed Server `ms1` al dominio y observar el impacto en `config.xml`.

#### Instrucciones

1. Toma un snapshot del `config.xml` antes de este paso:

```bash
cp $DOMAIN_HOME/config/config.xml /tmp/config_antes_ms1.xml
```

2. En WRC, navega a **Edit Tree** (árbol de edición). WRC iniciará automáticamente una sesión de edición en el Administration Server.

> 💡 **Concepto clave:** En WebLogic, los cambios de configuración se realizan en una **sesión de edición**. Los cambios no son efectivos hasta que se ejecuta **Activate Changes**. WRC gestiona este ciclo de forma transparente.

3. En el árbol de edición, navega a **Environment** → **Servers** y haz clic en el botón **"New"** o **"+"**.

4. Rellena los campos del nuevo servidor:

| Campo              | Valor            |
|--------------------|------------------|
| Name               | `ms1`            |
| Listen Address     | `localhost`      |
| Listen Port        | `7003`           |

5. Haz clic en **"Create"** para crear el servidor.

6. Verifica que `ms1` aparece en la lista de servidores dentro de WRC.

7. Haz clic en **"Activate Changes"** en WRC (botón en la barra superior o en el panel de cambios pendientes).

8. Tras la activación, compara el `config.xml`:

```bash
diff /tmp/config_antes_ms1.xml $DOMAIN_HOME/config/config.xml
```

9. Localiza en el diff el bloque `<server>` nuevo correspondiente a `ms1`:

```bash
grep -A 8 "<name>ms1</name>" $DOMAIN_HOME/config/config.xml
```

**Salida esperada del diff (fragmento):**
```xml
+  <server>
+    <name>ms1</name>
+    <listen-address>localhost</listen-address>
+    <listen-port>7003</listen-port>
+  </server>
```

#### Verificación

```bash
curl -s -u weblogic:Welcome1 \
  http://localhost:7001/management/weblogic/latest/domainConfig/servers \
  | python3 -m json.tool | grep '"name"'
```

Debes ver tanto `"AdminServer"` como `"ms1"`.

---

### Paso 5: Crear un JDBC Data Source apuntando a H2 embebida

**Objetivo:** Configurar un JDBC Data Source genérico usando la base de datos H2 embebida y aplicarle *targeting* al servidor `ms1`.

#### Instrucciones

1. Copia el driver H2 al directorio de librerías del dominio para que WebLogic pueda cargarlo:

```bash
cp ~/lab-resources/h2-2.x.x.jar $DOMAIN_HOME/lib/
ls -la $DOMAIN_HOME/lib/
```

> 💡 Si no tienes el JAR de H2, descárgalo:
> ```bash
> wget -P ~/lab-resources/ \
>   https://repo1.maven.org/maven2/com/h2database/h2/2.2.224/h2-2.2.224.jar
> cp ~/lab-resources/h2-2.2.224.jar $DOMAIN_HOME/lib/
> ```

2. Toma un snapshot del `config.xml`:

```bash
cp $DOMAIN_HOME/config/config.xml /tmp/config_antes_jdbc.xml
```

3. En WRC, navega a **Edit Tree** → **Services** → **Data Sources** y haz clic en **"New"**.

4. Selecciona el tipo **"Generic Data Source"** y rellena el asistente con los siguientes valores:

**Página 1 – Propiedades del Data Source:**

| Campo              | Valor                        |
|--------------------|------------------------------|
| Name               | `ds-h2-lab`                  |
| JNDI Name          | `jdbc/h2Lab`                 |
| Database Type      | `Other`                      |

**Página 2 – Driver JDBC:**

| Campo              | Valor                                            |
|--------------------|--------------------------------------------------|
| Driver Class Name  | `org.h2.Driver`                                  |
| URL                | `jdbc:h2:mem:labdb;DB_CLOSE_DELAY=-1`            |
| DB User            | `sa`                                             |
| DB Password        | `sa`                                             |
| Confirm Password   | `sa`                                             |

**Página 3 – Pool de conexiones (valores mínimos para lab):**

| Campo                   | Valor |
|-------------------------|-------|
| Initial Capacity        | `1`   |
| Maximum Capacity        | `5`   |
| Minimum Capacity        | `1`   |

**Página 4 – Targeting:**

- Selecciona **`ms1`** como target del Data Source.
- Haz clic en **"Finish"** o **"Create"**.

5. Activa los cambios haciendo clic en **"Activate Changes"** en WRC.

6. Verifica los cambios en el sistema de ficheros:

```bash
# El Data Source se guarda en un fichero auxiliar en config/jdbc/
ls -la $DOMAIN_HOME/config/jdbc/
cat $DOMAIN_HOME/config/jdbc/ds-h2-lab-jdbc.xml
```

7. Verifica también la referencia en `config.xml`:

```bash
grep -A 5 "jdbc-system-resource" $DOMAIN_HOME/config/config.xml
```

**Salida esperada:**
```xml
<jdbc-system-resource>
  <name>ds-h2-lab</name>
  <target>ms1</target>
  <descriptor-file-name>jdbc/ds-h2-lab-jdbc.xml</descriptor-file-name>
</jdbc-system-resource>
```

#### Verificación vía REST API

```bash
curl -s -u weblogic:Welcome1 \
  http://localhost:7001/management/weblogic/latest/domainConfig/JDBCSystemResources \
  | python3 -m json.tool | grep '"name"'
```

Debes ver `"ds-h2-lab"` en la respuesta.

---

### Paso 6: Crear un JMS Module con una Queue

**Objetivo:** Configurar un JMS System Module con una Connection Factory y una Queue, y aplicar *targeting* al servidor `ms1`.

#### Instrucciones

1. Toma un snapshot del `config.xml`:

```bash
cp $DOMAIN_HOME/config/config.xml /tmp/config_antes_jms.xml
```

2. En WRC, navega a **Edit Tree** → **Services** → **Messaging** → **JMS Modules** y haz clic en **"New"**.

3. Crea el JMS Module con los siguientes valores:

| Campo   | Valor          |
|---------|----------------|
| Name    | `jms-lab-module` |

4. En la página de targeting, selecciona `ms1` y haz clic en **"Finish"**.

5. Una vez creado el módulo, haz clic sobre `jms-lab-module` para editarlo.

6. Navega a la sub-sección **Connection Factories** dentro del módulo y crea una nueva:

| Campo      | Valor                    |
|------------|--------------------------|
| Name       | `cf-lab`                 |
| JNDI Name  | `jms/cf-lab`             |

7. Navega a la sub-sección **Queues** dentro del módulo y crea una nueva Queue:

| Campo      | Valor                    |
|------------|--------------------------|
| Name       | `queue-pedidos`          |
| JNDI Name  | `jms/queue-pedidos`      |

> 💡 **Nota:** En WebLogic, las Queues dentro de un JMS Module requieren un **Subdeployment** para asociarse a un JMS Server. Si WRC solicita un Subdeployment, crea uno llamado `sd-ms1` y asígnalo a `ms1`.

8. Activa los cambios en WRC.

9. Verifica los ficheros JMS generados:

```bash
ls -la $DOMAIN_HOME/config/jms/
cat $DOMAIN_HOME/config/jms/jms-lab-module-jms.xml
```

10. Compara con el snapshot anterior:

```bash
diff /tmp/config_antes_jms.xml $DOMAIN_HOME/config/config.xml | head -30
```

**Fragmento esperado en `jms-lab-module-jms.xml`:**
```xml
<weblogic-jms>
  <connection-factory name="cf-lab">
    <jndi-params>
      <jndi-name>jms/cf-lab</jndi-name>
    </jndi-params>
  </connection-factory>
  <queue name="queue-pedidos">
    <jndi-params>
      <jndi-name>jms/queue-pedidos</jndi-name>
    </jndi-params>
  </queue>
</weblogic-jms>
```

#### Verificación

```bash
curl -s -u weblogic:Welcome1 \
  http://localhost:7001/management/weblogic/latest/domainConfig/JMSSystemResources \
  | python3 -m json.tool | grep '"name"'
```

---

### Paso 7: Crear un Work Manager con restricciones de capacidad

**Objetivo:** Configurar un Work Manager personalizado con una restricción `Max Threads Constraint` y asignarlo al servidor `ms1`.

#### Instrucciones

1. Toma un snapshot del `config.xml`:

```bash
cp $DOMAIN_HOME/config/config.xml /tmp/config_antes_wm.xml
```

2. En WRC, navega a **Edit Tree** → **Environment** → **Work Managers** y haz clic en **"New"**.

3. Crea el Work Manager con los siguientes valores:

| Campo   | Valor             |
|---------|-------------------|
| Name    | `wm-lab-pedidos`  |

4. Tras crear el Work Manager, edítalo para añadir una restricción de capacidad máxima de hilos:

   a. Dentro del Work Manager, busca la sección **"Max Threads Constraint"**.
   b. Crea una nueva restricción:

| Campo  | Valor               |
|--------|---------------------|
| Name   | `mtc-pedidos`       |
| Count  | `5`                 |

5. Asigna la restricción `mtc-pedidos` al Work Manager `wm-lab-pedidos`.

6. En la sección de targeting del Work Manager, selecciona `ms1`.

7. Activa los cambios en WRC.

8. Verifica el cambio en `config.xml`:

```bash
diff /tmp/config_antes_wm.xml $DOMAIN_HOME/config/config.xml
```

```bash
grep -A 15 "work-manager" $DOMAIN_HOME/config/config.xml
```

**Fragmento esperado:**
```xml
<work-manager>
  <name>wm-lab-pedidos</name>
  <target>ms1</target>
  <max-threads-constraint>
    <name>mtc-pedidos</name>
    <count>5</count>
  </max-threads-constraint>
</work-manager>
```

#### Verificación

```bash
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainConfig/selfTuningDeploymentConfig/workManagers" \
  | python3 -m json.tool | grep '"name"'
```

---

### Paso 8: Iniciar el Managed Server ms1

**Objetivo:** Arrancar `ms1` para que los recursos con *targeting* a este servidor queden activos y verificables.

#### Instrucciones

1. Abre una nueva terminal y arranca `ms1` manualmente (sin Node Manager, modo laboratorio):

```bash
cd $DOMAIN_HOME
nohup ./bin/startManagedWebLogic.sh ms1 http://localhost:7001 \
  > /tmp/ms1.log 2>&1 &
echo "PID de ms1: $!"
```

2. Monitoriza el arranque de `ms1`:

```bash
tail -f /tmp/ms1.log
# Espera hasta ver: <Server started in RUNNING mode>
# Presiona Ctrl+C para dejar de seguir el log
```

> ⏱️ El arranque puede tardar entre 30 y 90 segundos dependiendo del hardware.

3. Verifica el estado de `ms1` desde WRC: navega a **Monitoring Tree** → **Environment** → **Servers** → `ms1` y comprueba que el estado sea **RUNNING**.

4. Verifica también mediante REST:

```bash
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime/serverLifeCycleRuntimes/ms1" \
  | python3 -m json.tool | grep '"state"'
```

**Salida esperada:**
```json
"state": "RUNNING"
```

5. Verifica que el Data Source `ds-h2-lab` está activo en `ms1`:

```bash
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime/serverRuntimes/ms1/JDBCServiceRuntime/JDBCDataSourceRuntimeMBeans/ds-h2-lab" \
  | python3 -m json.tool | grep -E '"state"|'"'"'name'"'"
```

---

### Paso 9: Desplegar la aplicación de muestra Jakarta EE 10 desde WRC

**Objetivo:** Desplegar `sample-jakarta-ee10.war` en el servidor `ms1` utilizando exclusivamente WRC, sin tocar la consola tradicional.

#### Instrucciones

1. Verifica que el archivo WAR existe y es accesible:

```bash
ls -lh ~/lab-resources/sample-jakarta-ee10.war
file ~/lab-resources/sample-jakarta-ee10.war
```

2. En WRC, navega a **Edit Tree** → **Deployments** y haz clic en **"New"** o **"Deploy"**.

3. Selecciona la opción para subir un archivo desde el sistema local y localiza `~/lab-resources/sample-jakarta-ee10.war`.

4. Configura el despliegue:

| Campo              | Valor                        |
|--------------------|------------------------------|
| Name               | `sample-jakartaee10`         |
| Deployment Path    | `sample-jakarta-ee10.war`    |
| Target             | `ms1`                        |
| Staging Mode       | `Default (Staged)`           |

5. Haz clic en **"Deploy"** o **"Finish"**.

6. Activa los cambios en WRC.

7. Verifica el estado del despliegue desde WRC: navega a **Monitoring Tree** → **Deployments** → `sample-jakartaee10` y comprueba que el estado sea **Active**.

8. Verifica mediante REST:

```bash
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime/deploymentManager/deploymentProgressObjects" \
  | python3 -m json.tool
```

9. Verifica el estado del despliegue:

```bash
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainConfig/appDeployments/sample-jakartaee10" \
  | python3 -m json.tool | grep -E '"name"|'"'"'deploymentState'"'"
```

10. Prueba la aplicación desplegada accediendo a su URL en `ms1` (puerto 7003):

```bash
curl -o /dev/null -s -w "HTTP Status: %{http_code}\n" \
  http://localhost:7003/sample-jakartaee10/
```

**Salida esperada:** `HTTP Status: 200` (o el código de respuesta de la página de bienvenida de la aplicación).

#### Verificación en config.xml

```bash
grep -A 8 "sample-jakartaee10" $DOMAIN_HOME/config/config.xml
```

---

### Paso 10: Análisis final de config.xml y revisión de la arquitectura

**Objetivo:** Realizar un análisis comparativo completo del `config.xml` para consolidar la comprensión de cómo WRC y la REST API persisten los cambios de configuración.

#### Instrucciones

1. Genera el diff completo entre el estado inicial y el estado final del dominio:

```bash
diff /tmp/config_snapshot_inicial.xml $DOMAIN_HOME/config/config.xml > /tmp/diff_completo.txt
cat /tmp/diff_completo.txt
```

2. Cuenta las líneas añadidas en el diff:

```bash
grep "^>" /tmp/diff_completo.txt | wc -l
```

3. Lista todos los ficheros auxiliares de configuración creados durante el laboratorio:

```bash
echo "=== Ficheros JDBC creados ==="
ls -la $DOMAIN_HOME/config/jdbc/

echo "=== Ficheros JMS creados ==="
ls -la $DOMAIN_HOME/config/jms/

echo "=== Directorio config completo ==="
find $DOMAIN_HOME/config -name "*.xml" | sort
```

4. Revisa la estructura de directorios del servidor `ms1`:

```bash
ls -la $DOMAIN_HOME/servers/ms1/
ls -la $DOMAIN_HOME/servers/ms1/logs/ 2>/dev/null || echo "Logs aún no generados"
```

5. Realiza una consulta REST para obtener el resumen completo del dominio y guárdalo:

```bash
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainConfig?links=none&fields=name,domainVersion,productionModeEnabled" \
  | python3 -m json.tool > /tmp/domain_summary.json
cat /tmp/domain_summary.json
```

6. Documenta en tu cuaderno el inventario final de recursos creados:

| Recurso             | Nombre              | Target | JNDI / Puerto        |
|---------------------|---------------------|--------|----------------------|
| Managed Server      | `ms1`               | —      | `localhost:7003`     |
| JDBC Data Source    | `ds-h2-lab`         | `ms1`  | `jdbc/h2Lab`         |
| JMS Module          | `jms-lab-module`    | `ms1`  | —                    |
| JMS Connection Fac. | `cf-lab`            | `ms1`  | `jms/cf-lab`         |
| JMS Queue           | `queue-pedidos`     | `ms1`  | `jms/queue-pedidos`  |
| Work Manager        | `wm-lab-pedidos`    | `ms1`  | —                    |
| Aplicación          | `sample-jakartaee10`| `ms1`  | `/sample-jakartaee10`|

---

## 7. Validación y Pruebas

Una vez completados todos los pasos, ejecuta la siguiente batería de verificaciones para confirmar el estado correcto del laboratorio:

```bash
#!/bin/bash
# Script de validación Lab 02-00-01
BASE_URL="http://localhost:7001/management/weblogic/latest"
CREDS="weblogic:Welcome1"

echo "============================================"
echo "VALIDACIÓN LAB 02-00-01"
echo "============================================"

# 1. Administration Server RUNNING
echo -n "[1] AdminServer RUNNING: "
STATE=$(curl -s -u $CREDS "$BASE_URL/domainRuntime/serverLifeCycleRuntimes/AdminServer" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('state','ERROR'))")
[ "$STATE" = "RUNNING" ] && echo "✅ $STATE" || echo "❌ $STATE"

# 2. ms1 RUNNING
echo -n "[2] ms1 RUNNING: "
STATE=$(curl -s -u $CREDS "$BASE_URL/domainRuntime/serverLifeCycleRuntimes/ms1" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('state','ERROR'))")
[ "$STATE" = "RUNNING" ] && echo "✅ $STATE" || echo "❌ $STATE"

# 3. JDBC Data Source existe
echo -n "[3] JDBC ds-h2-lab existe: "
RESULT=$(curl -s -u $CREDS "$BASE_URL/domainConfig/JDBCSystemResources/ds-h2-lab" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('name','NOT_FOUND'))")
[ "$RESULT" = "ds-h2-lab" ] && echo "✅ $RESULT" || echo "❌ $RESULT"

# 4. JMS Module existe
echo -n "[4] JMS Module jms-lab-module existe: "
RESULT=$(curl -s -u $CREDS "$BASE_URL/domainConfig/JMSSystemResources/jms-lab-module" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('name','NOT_FOUND'))")
[ "$RESULT" = "jms-lab-module" ] && echo "✅ $RESULT" || echo "❌ $RESULT"

# 5. Aplicación desplegada
echo -n "[5] Aplicación sample-jakartaee10 desplegada: "
RESULT=$(curl -s -u $CREDS "$BASE_URL/domainConfig/appDeployments/sample-jakartaee10" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('name','NOT_FOUND'))")
[ "$RESULT" = "sample-jakartaee10" ] && echo "✅ $RESULT" || echo "❌ $RESULT"

# 6. Aplicación accesible en ms1
echo -n "[6] Aplicación accesible en ms1 (puerto 7003): "
HTTP_CODE=$(curl -o /dev/null -s -w "%{http_code}" http://localhost:7003/sample-jakartaee10/)
[ "$HTTP_CODE" = "200" ] && echo "✅ HTTP $HTTP_CODE" || echo "⚠️  HTTP $HTTP_CODE (verifica contexto)"

# 7. config.xml actualizado
echo -n "[7] config.xml contiene ms1: "
grep -q "<name>ms1</name>" $DOMAIN_HOME/config/config.xml \
  && echo "✅ Encontrado" || echo "❌ No encontrado"

# 8. Fichero JDBC auxiliar existe
echo -n "[8] Fichero JDBC auxiliar existe: "
[ -f "$DOMAIN_HOME/config/jdbc/ds-h2-lab-jdbc.xml" ] \
  && echo "✅ Encontrado" || echo "❌ No encontrado"

echo "============================================"
echo "Validación completada."
```

Guarda el script y ejecútalo:

```bash
chmod +x ~/lab-resources/validate_lab02.sh
# O copia el contenido en un fichero y ejecútalo:
bash /tmp/validate_lab02.sh
```

**Resultado esperado:** Todos los checks deben mostrar ✅. Si alguno muestra ❌, consulta la sección de Troubleshooting.

---

## 8. Solución de Problemas

### Problema 1: WRC no puede conectarse al Administration Server ("Connection refused" o "401 Unauthorized")

**Síntomas:**
- WRC muestra un error de conexión al intentar añadir el *Connection Provider*.
- El mensaje indica `Connection refused`, `401 Unauthorized` o `Failed to connect to http://localhost:7001`.
- Los comandos `curl` también fallan con errores similares.

**Causa probable:**
Existen dos causas frecuentes:
- **"Connection refused"**: El Administration Server no está en estado `RUNNING`. El proceso puede haber fallado durante el arranque o no haberse iniciado.
- **"401 Unauthorized"**: Las credenciales introducidas en WRC son incorrectas, o el usuario `weblogic` tiene la contraseña bloqueada por intentos fallidos previos.

**Solución:**

```bash
# Verificar si el proceso del AdminServer está activo
ps aux | grep weblogic | grep -v grep

# Si no está activo, revisar el log de arranque
tail -50 $DOMAIN_HOME/servers/AdminServer/logs/AdminServer.log 2>/dev/null \
  || tail -50 /tmp/adminserver.log

# Reiniciar el Administration Server si es necesario
cd $DOMAIN_HOME
nohup ./bin/startWebLogic.sh > /tmp/adminserver.log 2>&1 &

# Esperar 60 segundos y verificar
sleep 60
curl -s -u weblogic:Welcome1 \
  http://localhost:7001/management/weblogic/latest \
  | python3 -m json.tool | head -5

# Si el error es 401, verificar credenciales en boot.properties
cat $DOMAIN_HOME/servers/AdminServer/security/boot.properties
# El fichero debe contener (cifrado tras primer arranque):
# username=<cifrado>
# password=<cifrado>
# Si está en texto plano (primer arranque), verificar que dice:
# username=weblogic
# password=Welcome1
```

Si el problema persiste con las credenciales, regenera el `boot.properties`:

```bash
cat > $DOMAIN_HOME/servers/AdminServer/security/boot.properties << EOF
username=weblogic
password=Welcome1
EOF
# WebLogic cifrará este fichero en el próximo arranque
```

---

### Problema 2: Los recursos creados en WRC no aparecen en `config.xml` o el Activate Changes falla

**Síntomas:**
- Después de crear un recurso en WRC (Data Source, JMS Module, Work Manager), el `diff` del `config.xml` no muestra cambios.
- WRC muestra un mensaje de error al hacer "Activate Changes" como: `Activation failed` o `There are validation errors`.
- El recurso aparece en WRC como "pendiente" pero nunca se activa.

**Causa probable:**
El ciclo **Edit/Activate** de WebLogic requiere que no haya errores de validación en la configuración pendiente. Las causas más comunes son:
- El puerto del Managed Server `ms1` (7003) ya está en uso por otro proceso.
- Falta un campo obligatorio en el recurso (por ejemplo, la URL JDBC o el nombre JNDI).
- Una sesión de edición anterior quedó "bloqueada" (lock) por un error previo sin descartar.

**Solución:**

```bash
# 1. Verificar si el puerto 7003 está en uso
ss -tlnp | grep 7003
# Si está ocupado, cambiar el puerto de ms1 a 7005 desde WRC

# 2. Descartar cambios pendientes desde WRC:
# En WRC → Edit Tree → botón "Discard Changes" o "Cancel Edit"

# 3. Si la sesión de edición está bloqueada, liberarla vía REST:
curl -s -u weblogic:Welcome1 \
  -X POST \
  -H "Content-Type: application/json" \
  "http://localhost:7001/management/weblogic/latest/edit/releaseEditSession" \
  | python3 -m json.tool

# 4. Verificar errores de validación específicos vía REST:
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/edit" \
  | python3 -m json.tool | grep -A 5 '"validationErrors"'

# 5. Después de resolver el error, volver a crear el recurso en WRC
# y activar de nuevo.
```

Si el problema es un campo obligatorio faltante, revisa en WRC el recurso pendiente y asegúrate de que todos los campos marcados con `*` estén rellenos antes de intentar activar de nuevo.

---

## 9. Limpieza del Entorno

Al finalizar el laboratorio, ejecuta los siguientes pasos para dejar el entorno en un estado limpio y controlado, listo para los laboratorios siguientes.

> ⚠️ **Importante:** No elimines el dominio `base_domain` ni el Administration Server. Los laboratorios posteriores dependen de este dominio. Solo detén los procesos y conserva la configuración.

```bash
# 1. Detener el Managed Server ms1 (si está en ejecución)
# Desde WRC: Monitoring Tree → Servers → ms1 → botón "Shutdown" (Graceful)
# O desde la línea de comandos (obtener PID):
PID_MS1=$(ps aux | grep "weblogic.Name=ms1" | grep -v grep | awk '{print $2}')
if [ -n "$PID_MS1" ]; then
  echo "Deteniendo ms1 (PID: $PID_MS1)..."
  kill -SIGTERM $PID_MS1
  sleep 10
  echo "ms1 detenido."
else
  echo "ms1 no está en ejecución."
fi

# 2. Verificar que ms1 se ha detenido
curl -s -u weblogic:Welcome1 \
  "http://localhost:7001/management/weblogic/latest/domainRuntime/serverLifeCycleRuntimes/ms1" \
  | python3 -m json.tool | grep '"state"' 2>/dev/null || echo "ms1 ya no responde (detenido)"

# 3. El Administration Server puede mantenerse activo para los próximos laboratorios.
# Si necesitas detenerlo:
# PID_ADMIN=$(ps aux | grep "weblogic.Name=AdminServer" | grep -v grep | awk '{print $2}')
# kill -SIGTERM $PID_ADMIN

# 4. Guardar los snapshots del laboratorio para referencia futura
mkdir -p ~/lab-artifacts/lab02-00-01/
cp /tmp/config_snapshot_inicial.xml ~/lab-artifacts/lab02-00-01/
cp /tmp/diff_completo.txt ~/lab-artifacts/lab02-00-01/
cp /tmp/domain_summary.json ~/lab-artifacts/lab02-00-01/
echo "Artefactos guardados en ~/lab-artifacts/lab02-00-01/"

# 5. Limpiar archivos temporales
rm -f /tmp/config_antes_*.xml /tmp/config_snapshot_inicial.xml

# 6. Cerrar WRC desde la interfaz gráfica (File → Exit o cerrar la ventana)

echo "============================================"
echo "Limpieza completada. Estado final:"
echo "- Administration Server: ACTIVO (puerto 7001)"
echo "- ms1: DETENIDO"
echo "- Recursos creados: PERSISTIDOS en config.xml"
echo "- Artefactos: ~/lab-artifacts/lab02-00-01/"
echo "============================================"
```

---

## 10. Resumen

### Puntos Clave del Laboratorio

En este laboratorio has practicado la administración completa de un dominio WebLogic 15 **sin utilizar la consola web tradicional**, demostrando que WRC es una herramienta completamente capaz de sustituirla:

1. **WRC y la REST API** son la interfaz moderna de gestión de WebLogic. Toda operación realizada en WRC se traduce en llamadas REST al Administration Server, que persiste los cambios en `config.xml` y sus ficheros auxiliares.

2. **El ciclo Edit/Activate** es fundamental: los cambios no son efectivos hasta que se activan. WRC gestiona este ciclo de forma visual, pero el mecanismo subyacente es el mismo que en la consola tradicional y WLST.

3. **El archivo `config.xml`** es el repositorio central de configuración del dominio. Cada recurso creado (Managed Server, JDBC, JMS, Work Manager) genera entradas en este fichero y, para recursos complejos, ficheros auxiliares en subdirectorios (`config/jdbc/`, `config/jms/`).

4. **El targeting** es el mecanismo por el que los recursos se asocian a servidores o clusters. Un recurso sin target no estará disponible para las aplicaciones.

5. **La arquitectura del dominio** (AdminServer → Managed Servers → Node Manager) establece una separación clara entre administración y ejecución de carga, que debe respetarse en producción.

### Diferencias clave observadas: WRC vs. Consola Tradicional

| Aspecto                    | Consola Tradicional (`/console`) | WebLogic Remote Console (WRC)          |
|----------------------------|----------------------------------|----------------------------------------|
| Tecnología                 | JSF / ADF (Java EE)              | Electron + REST API                    |
| Protocolo                  | HTTP/HTTPS + sesión web          | REST API pura (JSON)                   |
| Instalación                | Incluida en WLS                  | Descarga independiente de GitHub       |
| Acceso multi-dominio       | Un dominio por sesión            | Múltiples conexiones simultáneas       |
| Automatización             | No directa                       | Basada en REST, fácilmente scriptable  |
| Soporte futuro             | Legacy (deprecación planificada) | Herramienta estratégica Oracle         |

### Recursos de Referencia

- [WebLogic Remote Console – GitHub Oficial](https://github.com/oracle/weblogic-remote-console)
- [WebLogic REST Management API Reference](https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.1.0/wlrer/index.html)
- [Configuring JDBC Data Sources](https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.1.0/jdbca/index.html)
- [Configuring and Managing JMS for Oracle WebLogic Server](https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.1.0/jmspg/index.html)
- [Understanding Domain Configuration for Oracle WebLogic Server](https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.1.0/domcf/index.html)
- [Using Work Managers to Optimize Scheduled Work](https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.1.0/cnfgd/self_tuned.html)

---
