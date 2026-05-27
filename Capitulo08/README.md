# Simular incidencia, recopilar trazas y resolver problemas.

## 1. Metadatos

| Campo            | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 90 minutos                                 |
| **Complejidad**  | Alta                                       |
| **Nivel Bloom**  | Crear (Create)                             |
| **Módulo**       | 8 — Diagnóstico avanzado en WebLogic 15    |
| **Laboratorio**  | 08-00-01                                   |

---

## 2. Descripción General

Este laboratorio integra las habilidades de diagnóstico avanzado adquiridas a lo largo del curso. Partiendo del dominio con cluster operativo del Lab 07, el alumno configurará un módulo WLDF completo con Harvester, Watch Rules y Notifications; después simulará tres incidencias controladas —memory leak, thread starvation y deadlock— y las diagnosticará con jstack, Eclipse MAT y las imágenes de diagnóstico de WLDF. Al finalizar, habrá documentado un runbook estructurado de troubleshooting aplicable a entornos de producción reales con WebLogic 15.

---

## 3. Objetivos de Aprendizaje

- [ ] Configurar un módulo WLDF (Harvester + Watch Rules + Notifications) y verificar que genera alertas automáticas ante condiciones anómalas.
- [ ] Capturar y analizar thread dumps con `jstack` para identificar deadlocks, thread contention y stuck threads.
- [ ] Simular un memory leak bajo carga JMeter, capturar un heap dump y localizar la clase responsable con Eclipse MAT.
- [ ] Aplicar el proceso de diagnóstico estructurado (síntomas → herramientas → análisis → solución) para cada incidencia.
- [ ] Producir un runbook de troubleshooting documentando los hallazgos de las tres incidencias.

---

## 4. Prerrequisitos

### Conocimiento previo

| Área                              | Detalle requerido                                                     |
|-----------------------------------|-----------------------------------------------------------------------|
| Lab 05-00-01 completado           | Experiencia con heap dumps y análisis en Eclipse MAT                 |
| Lab 07-00-01 completado           | Dominio `wl15-cluster` con AdminServer + MS1 + MS2 operativos        |
| Concurrencia Java                 | Threads, `synchronized`, `Lock`, deadlock, estados de thread         |
| Estados de thread JVM             | `RUNNABLE`, `WAITING`, `BLOCKED`, `TIMED_WAITING`                    |
| WLDF (lección 8.1)                | Harvester, Archive, Watch & Notification, Diagnostic Image           |
| JMeter básico                     | Crear Thread Group, HTTP Sampler, ejecutar test plan                  |

### Acceso y herramientas

| Herramienta             | Versión mínima | Verificación                          |
|-------------------------|----------------|---------------------------------------|
| Oracle JDK 21           | 21.0.3+        | `java -version`                       |
| WebLogic 15             | 15.0           | AdminServer arrancado en `7001`       |
| jstack                  | JDK 21         | `$JAVA_HOME/bin/jstack -version`      |
| Eclipse MAT             | 1.14.x         | Aplicación de escritorio abierta      |
| Apache JMeter           | 5.6.x          | `$JMETER_HOME/bin/jmeter -version`    |
| curl + jq               | Sistema        | `curl --version && jq --version`      |

---

## 5. Entorno de Laboratorio

### Topología de referencia

```
AdminServer  :7001  (host: localhost)
MS1          :7003  (Cluster-1)
MS2          :7005  (Cluster-1)
DOMAIN_HOME  : /u01/domains/wl15-cluster
JAVA_HOME    : /u01/java/jdk21
APP_DIR      : /u01/lab08/apps
SCRIPTS_DIR  : /u01/lab08/scripts
```

### Variables de entorno necesarias

Define estas variables en tu sesión antes de iniciar:

```bash
export JAVA_HOME=/u01/java/jdk21
export WL_HOME=/u01/oracle/wls15/wlserver
export DOMAIN_HOME=/u01/domains/wl15-cluster
export APP_DIR=/u01/lab08/apps
export SCRIPTS_DIR=/u01/lab08/scripts
export PATH=$JAVA_HOME/bin:$PATH

# Credenciales de laboratorio (NUNCA usar en producción)
export WL_USER=weblogic
export WL_PASS=Welcome1
export ADMIN_URL=http://localhost:7001
```

### Preparación del directorio de trabajo

```bash
mkdir -p $APP_DIR $SCRIPTS_DIR
mkdir -p /u01/lab08/runbook
```

### Verificación del entorno antes de comenzar

```bash
# 1. Confirmar que AdminServer responde
curl -s -u $WL_USER:$WL_PASS \
  "$ADMIN_URL/management/weblogic/latest/serverRuntime?links=none" | jq '.name'
# Esperado: "AdminServer"

# 2. Confirmar que MS1 y MS2 están RUNNING
curl -s -u $WL_USER:$WL_PASS \
  "$ADMIN_URL/management/weblogic/latest/domainRuntime/serverLifeCycleRuntimes?links=none" \
  | jq '.items[] | {name: .name, state: .state}'

# 3. Confirmar jstack disponible
$JAVA_HOME/bin/jstack -version
```

> **Snapshot recomendado:** Toma un snapshot de la VM ahora, antes de iniciar cualquier cambio.

---

## 6. Pasos del Laboratorio

---

### PASO 1 — Configurar el módulo WLDF `DiagLab08`

**Objetivo:** Crear un WLDF System Resource con Harvester (ThreadPool + JVM heap), Watch Rules (heap > 80 % y stuck threads > 5) y Notifications (Log Action + Diagnostic Image).

#### 1.1 Crear el módulo WLDF mediante WLST

```bash
# Iniciar WLST
$WL_HOME/common/bin/wlst.sh << 'WLST_EOF'
connect('weblogic', 'Welcome1', 't3://localhost:7001')
edit()
startEdit()

# Crear el WLDF System Resource
wldfMod = create('DiagLab08', 'WLDFSystemResource')
wldfMod.setTargets(getMBean('/Clusters/Cluster-1'))

# Obtener la raíz de configuración WLDF
wldfRes = wldfMod.getWLDFResource()

# ── HARVESTER ────────────────────────────────────────────────────────────────
harvester = wldfRes.getHarvester()
harvester.setEnabled(true)
harvester.setSamplePeriod(30000)   # 30 segundos

# Tipo 1: ThreadPoolRuntimeMBean
ht1 = harvester.createHarvestedType()
ht1.setName('weblogic.management.runtime.ThreadPoolRuntimeMBean')
ht1.setEnabled(true)
ht1.setAttributes(['QueueLength','ExecuteThreadTotalCount','HoggingThreadCount','StuckThreadCount'])

# Tipo 2: JVMRuntimeMBean (heap)
ht2 = harvester.createHarvestedType()
ht2.setName('weblogic.management.runtime.JVMRuntimeMBean')
ht2.setEnabled(true)
ht2.setAttributes(['HeapSizeCurrent','HeapFreeCurrent','HeapSizeMax'])

# ── WATCH & NOTIFICATION ─────────────────────────────────────────────────────
watchNotif = wldfRes.getWatchNotification()

# Acción 1: Log Action
logAction = watchNotif.createAction('LogAlertAction')
logAction.setType('LogAction')

# Acción 2: Diagnostic Image Notification
imgAction = watchNotif.createAction('ImageCaptureAction')
imgAction.setType('ImageNotification')

# Watch 1: Heap usado > 80%
# Expresión: cuando HeapFreeCurrent < 20% de HeapSizeMax
w1 = watchNotif.createWatch('WatchHeapHigh')
w1.setEnabled(true)
w1.setRuleType('Harvester')
w1.setRuleExpression("${JVM:HeapFreeCurrent} < (${JVM:HeapSizeMax} * 0.20)")
w1.setSeverity('Warning')
w1.setActions(['LogAlertAction','ImageCaptureAction'])

# Watch 2: Stuck threads > 5
w2 = watchNotif.createWatch('WatchStuckThreads')
w2.setEnabled(true)
w2.setRuleType('Harvester')
w2.setRuleExpression("${ThreadPool:StuckThreadCount} > 5")
w2.setSeverity('Critical')
w2.setActions(['LogAlertAction','ImageCaptureAction'])

# Guardar y activar
save()
activate(block='true')
print('>>> Modulo DiagLab08 creado y activado correctamente')
disconnect()
exit()
WLST_EOF
```

#### 1.2 Verificar el módulo desde la consola REST

```bash
curl -s -u $WL_USER:$WL_PASS \
  "$ADMIN_URL/management/weblogic/latest/edit/wldfSystemResources/DiagLab08?links=none" \
  | jq '{name: .name, targets: .targets}'
```

**Salida esperada:**

```json
{
  "name": "DiagLab08",
  "targets": ["Cluster-1"]
}
```

#### 1.3 Confirmar que el Harvester está activo en MS1

```bash
curl -s -u $WL_USER:$WL_PASS \
  "http://localhost:7003/management/weblogic/latest/serverRuntime/threadPoolRuntime?links=none" \
  | jq '{queueLength: .queueLength, stuckThreadCount: .stuckThreadCount}'
```

**Verificación:** Si el JSON devuelve valores numéricos (aunque sean 0), el runtime de MS1 está respondiendo y WLDF puede muestrearlo.

---

### PASO 2 — Preparar y desplegar las aplicaciones de simulación

**Objetivo:** Compilar y desplegar tres mini-aplicaciones Jakarta EE 10 que simulan las tres incidencias: memory leak, thread starvation y deadlock.

#### 2.1 Aplicación 1 — Memory Leak (`MemLeakServlet`)

```bash
mkdir -p $APP_DIR/memleak/WEB-INF
```

Crea el archivo `$APP_DIR/memleak/WEB-INF/web.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee
           https://jakarta.ee/xml/ns/jakartaee/web-app_6_0.xsd"
         version="6.0">
  <servlet>
    <servlet-name>MemLeakServlet</servlet-name>
    <servlet-class>lab08.MemLeakServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>MemLeakServlet</servlet-name>
    <url-pattern>/leak</url-pattern>
  </servlet-mapping>
</web-app>
```

Crea `$APP_DIR/memleak/WEB-INF/classes/lab08/MemLeakServlet.java`:

```java
package lab08;

import jakarta.servlet.*;
import jakarta.servlet.http.*;
import java.io.*;
import java.util.*;

public class MemLeakServlet extends HttpServlet {

    // Lista estática: los objetos nunca se liberan -> memory leak
    private static final List<byte[]> LEAK_BUCKET = new ArrayList<>();

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        // Acumular 1 MB por petición
        LEAK_BUCKET.add(new byte[1024 * 1024]);
        resp.setContentType("text/plain");
        resp.getWriter().printf("Leak size: %d MB%n", LEAK_BUCKET.size());
    }
}
```

Compilar y empaquetar:

```bash
mkdir -p $APP_DIR/memleak/WEB-INF/classes/lab08

javac -cp $WL_HOME/server/lib/wls-api.jar \
      -source 21 -target 21 \
      -d $APP_DIR/memleak/WEB-INF/classes \
      $APP_DIR/memleak/WEB-INF/classes/lab08/MemLeakServlet.java

cd $APP_DIR/memleak
jar cvf $APP_DIR/memleak.war .
cd -
```

#### 2.2 Aplicación 2 — Thread Starvation (`SlowServlet` con Work Manager)

```bash
mkdir -p $APP_DIR/starvation/WEB-INF/classes/lab08
```

`$APP_DIR/starvation/WEB-INF/web.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         version="6.0">
  <servlet>
    <servlet-name>SlowServlet</servlet-name>
    <servlet-class>lab08.SlowServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>SlowServlet</servlet-name>
    <url-pattern>/slow</url-pattern>
  </servlet-mapping>
</web-app>
```

`$APP_DIR/starvation/WEB-INF/weblogic.xml` — Work Manager con restricción de 2 threads:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<weblogic-web-app
    xmlns="http://xmlns.oracle.com/weblogic/weblogic-web-app"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <work-manager>
    <name>LimitedWM</name>
    <max-threads-constraint>
      <name>MaxTwoThreads</name>
      <count>2</count>
    </max-threads-constraint>
  </work-manager>
</weblogic-web-app>
```

`$APP_DIR/starvation/WEB-INF/classes/lab08/SlowServlet.java`:

```java
package lab08;

import jakarta.servlet.*;
import jakarta.servlet.http.*;
import java.io.*;

public class SlowServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        try {
            // Simula trabajo lento: 15 segundos por petición
            Thread.sleep(15_000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        resp.setContentType("text/plain");
        resp.getWriter().println("Done");
    }
}
```

Compilar y empaquetar:

```bash
javac -cp $WL_HOME/server/lib/wls-api.jar \
      -source 21 -target 21 \
      -d $APP_DIR/starvation/WEB-INF/classes \
      $APP_DIR/starvation/WEB-INF/classes/lab08/SlowServlet.java

cd $APP_DIR/starvation
jar cvf $APP_DIR/starvation.war .
cd -
```

#### 2.3 Aplicación 3 — Deadlock (`DeadlockServlet`)

```bash
mkdir -p $APP_DIR/deadlock/WEB-INF/classes/lab08
```

`$APP_DIR/deadlock/WEB-INF/web.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee" version="6.0">
  <servlet>
    <servlet-name>DeadlockServlet</servlet-name>
    <servlet-class>lab08.DeadlockServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>DeadlockServlet</servlet-name>
    <url-pattern>/deadlock</url-pattern>
  </servlet-mapping>
</web-app>
```

`$APP_DIR/deadlock/WEB-INF/classes/lab08/DeadlockServlet.java`:

```java
package lab08;

import jakarta.servlet.*;
import jakarta.servlet.http.*;
import java.io.*;

public class DeadlockServlet extends HttpServlet {

    // Dos locks compartidos entre peticiones concurrentes
    private static final Object LOCK_A = new Object();
    private static final Object LOCK_B = new Object();

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {

        String order = req.getParameter("order");
        PrintWriter out = resp.getWriter();
        resp.setContentType("text/plain");

        if ("AB".equals(order)) {
            synchronized (LOCK_A) {
                out.println("Thread " + Thread.currentThread().getName() + " holds LOCK_A");
                out.flush();
                try { Thread.sleep(500); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                synchronized (LOCK_B) {
                    out.println("Thread acquired LOCK_B");
                }
            }
        } else {
            synchronized (LOCK_B) {
                out.println("Thread " + Thread.currentThread().getName() + " holds LOCK_B");
                out.flush();
                try { Thread.sleep(500); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                synchronized (LOCK_A) {
                    out.println("Thread acquired LOCK_A");
                }
            }
        }
    }
}
```

Compilar y empaquetar:

```bash
javac -cp $WL_HOME/server/lib/wls-api.jar \
      -source 21 -target 21 \
      -d $APP_DIR/deadlock/WEB-INF/classes \
      $APP_DIR/deadlock/WEB-INF/classes/lab08/DeadlockServlet.java

cd $APP_DIR/deadlock
jar cvf $APP_DIR/deadlock.war .
cd -
```

#### 2.4 Desplegar las tres aplicaciones en el Cluster

```bash
$WL_HOME/common/bin/wlst.sh << 'WLST_EOF'
connect('weblogic', 'Welcome1', 't3://localhost:7001')

# Desplegar memleak.war
deploy('memleak',   '/u01/lab08/apps/memleak.war',   targets='Cluster-1', block='true')
deploy('starvation','/u01/lab08/apps/starvation.war', targets='Cluster-1', block='true')
deploy('deadlock',  '/u01/lab08/apps/deadlock.war',   targets='Cluster-1', block='true')

print('>>> Tres aplicaciones desplegadas en Cluster-1')
disconnect()
exit()
WLST_EOF
```

**Verificación rápida de despliegue:**

```bash
# Comprobar que MS1 responde a cada app
curl -s http://localhost:7003/memleak/leak
curl -s http://localhost:7003/starvation/slow &   # En background (tarda 15s)
curl -s "http://localhost:7003/deadlock/deadlock?order=AB"
```

---

### PASO 3 — Incidencia 1: Memory Leak

**Objetivo:** Generar carga con JMeter hasta que WLDF dispare `WatchHeapHigh`, capturar heap dump y analizar con MAT.

#### 3.1 Crear el test plan JMeter para carga de memory leak

Crea el archivo `$SCRIPTS_DIR/memleak_test.jmx`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0">
  <hashTree>
    <TestPlan testname="MemLeak Load" enabled="true">
      <hashTree>
        <ThreadGroup testname="LeakGroup" enabled="true">
          <stringProp name="ThreadGroup.num_threads">20</stringProp>
          <stringProp name="ThreadGroup.ramp_time">10</stringProp>
          <boolProp name="ThreadGroup.scheduler">false</boolProp>
          <stringProp name="ThreadGroup.duration">120</stringProp>
          <hashTree>
            <HTTPSamplerProxy testname="LeakRequest" enabled="true">
              <stringProp name="HTTPSampler.domain">localhost</stringProp>
              <stringProp name="HTTPSampler.port">7003</stringProp>
              <stringProp name="HTTPSampler.path">/memleak/leak</stringProp>
              <stringProp name="HTTPSampler.method">GET</stringProp>
              <hashTree/>
            </HTTPSamplerProxy>
          </hashTree>
        </ThreadGroup>
      </hashTree>
    </TestPlan>
  </hashTree>
</jmeterTestPlan>
```

#### 3.2 Ejecutar la carga (modo headless)

```bash
# Asegúrate de que MS1 tiene heap limitado para acelerar el leak
# Si no está ya configurado, editar setDomainEnv o usar -Xmx512m en el inicio de MS1

$JMETER_HOME/bin/jmeter -n \
  -t $SCRIPTS_DIR/memleak_test.jmx \
  -l /u01/lab08/results/memleak_results.jtl \
  -e -o /u01/lab08/results/memleak_report &

JMETER_PID=$!
echo "JMeter PID: $JMETER_PID"
```

#### 3.3 Monitorear el heap mientras corre la carga

```bash
# Monitoreo continuo del heap en MS1 cada 10 segundos
watch -n 10 'curl -s -u weblogic:Welcome1 \
  "http://localhost:7003/management/weblogic/latest/serverRuntime/JVMRuntime?links=none" \
  | jq "{heapUsedMB: (.heapSizeCurrent - .heapFreeCurrent)/1048576 | floor, heapMaxMB: (.heapSizeMax/1048576 | floor)}"'
```

#### 3.4 Capturar heap dump cuando el heap supere el 80%

```bash
# Obtener el PID de MS1
MS1_PID=$(jps | grep 'MS1' | awk '{print $1}')
echo "MS1 PID: $MS1_PID"

# Capturar heap dump con jmap (incluido en JDK 21)
$JAVA_HOME/bin/jmap -dump:format=b,file=/u01/lab08/heapdump_ms1.hprof $MS1_PID

echo "Heap dump guardado en: /u01/lab08/heapdump_ms1.hprof"
ls -lh /u01/lab08/heapdump_ms1.hprof
```

> **Alternativa automática:** Si configuraste `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/u01/lab08/` en las JVM options de MS1, el dump se generará automáticamente al producirse OOM.

#### 3.5 Detener la carga JMeter

```bash
kill $JMETER_PID 2>/dev/null || true
```

#### 3.6 Verificar que WLDF disparó la alerta

```bash
# Buscar la alerta WatchHeapHigh en el server log de MS1
grep -i "WatchHeapHigh\|DiagLab08\|WLDF" \
  $DOMAIN_HOME/servers/MS1/logs/MS1.log | tail -20

# Verificar imagen de diagnóstico generada automáticamente
ls -lh $DOMAIN_HOME/servers/MS1/logs/diagnostic_images/
```

#### 3.7 Analizar el heap dump con Eclipse MAT

Abre Eclipse MAT y sigue estos pasos:

1. **File → Open Heap Dump** → selecciona `/u01/lab08/heapdump_ms1.hprof`
2. Selecciona **"Leak Suspects Report"** cuando se ofrezca.
3. En el reporte, busca el **Dominator Tree**: ordena por "Retained Heap" descendente.
4. Localiza la clase `lab08.MemLeakServlet` o el campo `LEAK_BUCKET` (tipo `java.util.ArrayList`).
5. Haz clic derecho → **"List objects → with outgoing references"** para ver los `byte[]` acumulados.

**Comando alternativo (MAT en modo batch):**

```bash
# Generar reporte de leak suspects desde línea de comandos
$MAT_HOME/ParseHeapDump.sh \
  /u01/lab08/heapdump_ms1.hprof \
  org.eclipse.mat.api:suspects \
  -Xmx4g

# El reporte HTML queda en el mismo directorio
ls /u01/lab08/heapdump_ms1_Leak_Suspects/
```

#### 3.8 Documentar la incidencia 1 en el runbook

```bash
cat > /u01/lab08/runbook/incidencia1_memleak.md << 'EOF'
# Incidencia 1: Memory Leak

## Síntomas observados
- Heap de MS1 creció progresivamente hasta superar el 80% de HeapSizeMax.
- WLDF disparó Watch `WatchHeapHigh` con severidad Warning.
- Se generó Diagnostic Image automáticamente.
- Eventualmente: OutOfMemoryError en logs de MS1.

## Herramientas usadas
- Apache JMeter (generación de carga)
- WLDF Watch & Notification (detección automática)
- jmap / -XX:+HeapDumpOnOutOfMemoryError (captura de heap dump)
- Eclipse MAT (análisis del heap dump)

## Análisis realizado
- MAT Dominator Tree: clase `lab08.MemLeakServlet` retiene > 90% del heap.
- Campo estático `LEAK_BUCKET` (ArrayList<byte[]>) acumula 1 MB por petición.
- Ningún objeto es elegible para GC porque la referencia estática los ancla.

## Solución aplicada
- Eliminar el campo estático `LEAK_BUCKET` o limitarlo con un tamaño máximo.
- Usar caché con política de expiración (ej. Caffeine, Ehcache).
- Configurar `-XX:+HeapDumpOnOutOfMemoryError` como medida preventiva.
- Ajustar alertas WLDF al 70% de heap para reacción más temprana.
EOF
echo "Runbook incidencia 1 guardado."
```

---

### PASO 4 — Incidencia 2: Thread Starvation

**Objetivo:** Saturar el Work Manager de 2 threads con carga concurrente, observar threads STUCK en WebLogic y analizarlos con thread dump.

#### 4.1 Crear el test plan JMeter para thread starvation

```bash
cat > $SCRIPTS_DIR/starvation_test.jmx << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0">
  <hashTree>
    <TestPlan testname="Thread Starvation Load" enabled="true">
      <hashTree>
        <ThreadGroup testname="StarvationGroup" enabled="true">
          <stringProp name="ThreadGroup.num_threads">30</stringProp>
          <stringProp name="ThreadGroup.ramp_time">5</stringProp>
          <boolProp name="ThreadGroup.scheduler">false</boolProp>
          <stringProp name="ThreadGroup.duration">90</stringProp>
          <hashTree>
            <HTTPSamplerProxy testname="SlowRequest" enabled="true">
              <stringProp name="HTTPSampler.domain">localhost</stringProp>
              <stringProp name="HTTPSampler.port">7003</stringProp>
              <stringProp name="HTTPSampler.path">/starvation/slow</stringProp>
              <stringProp name="HTTPSampler.method">GET</stringProp>
              <stringProp name="HTTPSampler.connect_timeout">5000</stringProp>
              <stringProp name="HTTPSampler.response_timeout">60000</stringProp>
              <hashTree/>
            </HTTPSamplerProxy>
          </hashTree>
        </ThreadGroup>
      </hashTree>
    </TestPlan>
  </hashTree>
</jmeterTestPlan>
EOF
```

#### 4.2 Ejecutar la carga

```bash
mkdir -p /u01/lab08/results/starvation_report

$JMETER_HOME/bin/jmeter -n \
  -t $SCRIPTS_DIR/starvation_test.jmx \
  -l /u01/lab08/results/starvation_results.jtl \
  -e -o /u01/lab08/results/starvation_report &

JMETER_STARV_PID=$!
echo "JMeter Starvation PID: $JMETER_STARV_PID"
```

#### 4.3 Monitorear el estado del thread pool en MS1

```bash
# Monitoreo del pool de threads cada 5 segundos
watch -n 5 'curl -s -u weblogic:Welcome1 \
  "http://localhost:7003/management/weblogic/latest/serverRuntime/threadPoolRuntime?links=none" \
  | jq "{queueLength: .queueLength, stuckThreadCount: .stuckThreadCount, \
         hoggingThreadCount: .hoggingThreadCount, executeThreadTotalCount: .executeThreadTotalCount}"'
```

**Salida esperada después de ~30 segundos de carga:**

```json
{
  "queueLength": 28,
  "stuckThreadCount": 2,
  "hoggingThreadCount": 2,
  "executeThreadTotalCount": 25
}
```

#### 4.4 Capturar thread dump con jstack

```bash
# Obtener PID de MS1
MS1_PID=$(jps -l | grep 'weblogic.Server' | head -1 | awk '{print $1}')
echo "MS1 PID: $MS1_PID"

# Capturar tres thread dumps con intervalo de 10 segundos (patrón recomendado)
for i in 1 2 3; do
  echo "=== Thread Dump $i - $(date) ===" >> /u01/lab08/threaddump_ms1.txt
  $JAVA_HOME/bin/jstack -l $MS1_PID >> /u01/lab08/threaddump_ms1.txt
  echo "" >> /u01/lab08/threaddump_ms1.txt
  [ $i -lt 3 ] && sleep 10
done

echo "Thread dumps guardados en: /u01/lab08/threaddump_ms1.txt"
wc -l /u01/lab08/threaddump_ms1.txt
```

> **Alternativa con kill -3 (señal SIGQUIT):** En sistemas Linux, `kill -3 $MS1_PID` envía la señal que provoca que la JVM imprima el thread dump en su stdout/stderr. WebLogic redirige esto al archivo de log del servidor.

```bash
# Método alternativo: kill -3 (el dump aparece en MS1.out o MS1.log)
kill -3 $MS1_PID
sleep 2
grep -A 50 "Full thread dump" $DOMAIN_HOME/servers/MS1/logs/MS1.out | head -80
```

#### 4.5 Analizar el thread dump

```bash
# Buscar threads STUCK o en estado WAITING/BLOCKED en SlowServlet
grep -A 10 "SlowServlet\|STUCK\|TIMED_WAITING" /u01/lab08/threaddump_ms1.txt | head -60

# Contar threads bloqueados esperando el Work Manager
grep -c "LimitedWM\|MaxTwoThreads\|WAITING" /u01/lab08/threaddump_ms1.txt
```

**Patrón esperado en el thread dump:**

```
"[STUCK] ExecuteThread: '0' for queue: 'weblogic.kernel.Default'" #45 daemon prio=5
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.base/java.lang.Thread.sleep(Native Method)
        at lab08.SlowServlet.doGet(SlowServlet.java:14)
        ...
```

#### 4.6 Verificar alerta WLDF WatchStuckThreads

```bash
grep -i "WatchStuckThreads\|StuckThread\|DiagLab08" \
  $DOMAIN_HOME/servers/MS1/logs/MS1.log | tail -10
```

#### 4.7 Detener JMeter y documentar

```bash
kill $JMETER_STARV_PID 2>/dev/null || true

cat > /u01/lab08/runbook/incidencia2_starvation.md << 'EOF'
# Incidencia 2: Thread Starvation

## Síntomas observados
- La cola del thread pool (queueLength) creció hasta 28+ peticiones pendientes.
- stuckThreadCount y hoggingThreadCount = 2 (igual al límite del Work Manager).
- Tiempo de respuesta HTTP: > 15 segundos o timeout para la mayoría de peticiones.
- WLDF disparó Watch `WatchStuckThreads` con severidad Critical.

## Herramientas usadas
- Apache JMeter (carga concurrente de 30 usuarios)
- REST API de WebLogic (monitoreo threadPoolRuntime en tiempo real)
- jstack (captura de 3 thread dumps con intervalo de 10s)
- WLDF Watch & Notification (detección automática)

## Análisis realizado
- Thread dump muestra 2 threads en estado TIMED_WAITING dentro de SlowServlet.doGet().
- El Work Manager `LimitedWM` con max-threads-constraint=2 limita la concurrencia.
- Las 28 peticiones restantes esperan en cola sin poder ejecutarse.
- Patrón: threads no están bloqueados por locks, sino por diseño de throttling.

## Solución aplicada
- Incrementar max-threads-constraint a un valor acorde al throughput esperado.
- Optimizar SlowServlet para reducir el tiempo de procesamiento.
- Implementar patrón async (AsyncContext de Servlet 3.1+) para liberar threads.
- Configurar un timeout de stuck thread en WebLogic (StuckThreadMaxTime).
EOF
echo "Runbook incidencia 2 guardado."
```

---

### PASO 5 — Incidencia 3: Deadlock

**Objetivo:** Provocar un deadlock entre dos threads concurrentes, capturar el thread dump y localizar el ciclo de espera con jstack.

#### 5.1 Provocar el deadlock

```bash
# Lanzar dos peticiones concurrentes con orden de locks opuesto
# Petición 1: adquiere LOCK_A primero, luego intenta LOCK_B
curl -s "http://localhost:7003/deadlock/deadlock?order=AB" &
CURL_PID1=$!

# Pequeña pausa para que el primer thread adquiera su lock
sleep 0.3

# Petición 2: adquiere LOCK_B primero, luego intenta LOCK_A
curl -s "http://localhost:7003/deadlock/deadlock?order=BA" &
CURL_PID2=$!

echo "Deadlock provocado. PIDs curl: $CURL_PID1 $CURL_PID2"
echo "Esperando 5 segundos para que el deadlock se consolide..."
sleep 5
```

#### 5.2 Capturar thread dump con jstack (detección de deadlock)

```bash
MS1_PID=$(jps -l | grep 'weblogic.Server' | head -1 | awk '{print $1}')

# jstack con la opción -l incluye información de locks (java.util.concurrent)
$JAVA_HOME/bin/jstack -l $MS1_PID > /u01/lab08/threaddump_deadlock.txt

echo "Thread dump con deadlock guardado."

# jstack detecta deadlocks automáticamente y los reporta al final del dump
grep -A 30 "Found.*deadlock\|Java-level deadlock" /u01/lab08/threaddump_deadlock.txt
```

**Salida esperada de jstack al detectar deadlock:**

```
Found one Java-level deadlock:
=============================
"ExecuteThread: '1' for queue: 'weblogic.kernel.Default'":
  waiting to lock monitor 0x00007f8a1c003a80 (object 0x00000000e1234560, a java.lang.Object),
  which is held by "ExecuteThread: '2' for queue: 'weblogic.kernel.Default'"
"ExecuteThread: '2' for queue: 'weblogic.kernel.Default'":
  waiting to lock monitor 0x00007f8a1c004b90 (object 0x00000000e1234580, a java.lang.Object),
  which is held by "ExecuteThread: '1' for queue: 'weblogic.kernel.Default'"

Java stack information for the threads listed above:
===================================================
"ExecuteThread: '1'":
        at lab08.DeadlockServlet.doGet(DeadlockServlet.java:28)
        - waiting to lock <0x00000000e1234580> (a java.lang.Object)
        - locked <0x00000000e1234560> (a java.lang.Object)
"ExecuteThread: '2'":
        at lab08.DeadlockServlet.doGet(DeadlockServlet.java:38)
        - waiting to lock <0x00000000e1234560> (a java.lang.Object)
        - locked <0x00000000e1234580> (a java.lang.Object)

Found 1 deadlock.
```

#### 5.3 Analizar el ciclo de espera

```bash
# Extraer el bloque completo del deadlock
awk '/Found.*deadlock/,/^$/' /u01/lab08/threaddump_deadlock.txt

# Identificar los objetos involucrados (direcciones de monitor)
grep -E "waiting to lock|locked <" /u01/lab08/threaddump_deadlock.txt | \
  grep -v "java.util.concurrent" | head -10
```

#### 5.4 Resolver el deadlock (simulación de fix)

La solución estándar para deadlocks es **imponer un orden consistente de adquisición de locks**. Documenta la corrección:

```java
// CÓDIGO CORREGIDO: siempre adquirir LOCK_A antes que LOCK_B,
// independientemente del parámetro 'order'
// En DeadlockServlet.java - versión corregida:
synchronized (LOCK_A) {
    synchronized (LOCK_B) {
        // lógica de negocio
    }
}
```

#### 5.5 Limpiar los procesos curl colgados

```bash
kill $CURL_PID1 $CURL_PID2 2>/dev/null || true
# Forzar si es necesario
pkill -f "curl.*deadlock" 2>/dev/null || true
```

#### 5.6 Documentar la incidencia 3

```bash
cat > /u01/lab08/runbook/incidencia3_deadlock.md << 'EOF'
# Incidencia 3: Deadlock

## Síntomas observados
- Dos peticiones HTTP se cuelgan indefinidamente sin responder.
- Los threads asociados nunca liberan CPU pero tampoco progresan.
- La consola de WebLogic muestra threads en estado BLOCKED.
- Después de StuckThreadMaxTime, WebLogic reporta threads STUCK.

## Herramientas usadas
- curl (provocación del deadlock con dos peticiones concurrentes)
- jstack -l (captura de thread dump con información de locks)
- Análisis manual del ciclo de espera en el dump

## Análisis realizado
- jstack detectó automáticamente "Found 1 deadlock".
- Thread 1 mantiene LOCK_A y espera LOCK_B.
- Thread 2 mantiene LOCK_B y espera LOCK_A.
- Ciclo de espera circular: ningún thread puede progresar.
- Origen: DeadlockServlet.java líneas 28 y 38 — orden de locks invertido.

## Solución aplicada
- Imponer orden consistente de adquisición: siempre LOCK_A → LOCK_B.
- Alternativa: usar java.util.concurrent.locks.Lock con tryLock() y timeout.
- En producción: revisar todos los bloques synchronized que adquieren múltiples locks.
- Configurar StuckThreadMaxTime en WebLogic para detección automática temprana.
EOF
echo "Runbook incidencia 3 guardado."
```

---

### PASO 6 — Captura manual de Diagnostic Image y análisis

**Objetivo:** Capturar una Diagnostic Image bajo demanda y examinar su contenido para correlacionar con las incidencias anteriores.

#### 6.1 Capturar Diagnostic Image vía REST

```bash
# Capturar imagen de diagnóstico de MS1 bajo demanda
curl -s -u $WL_USER:$WL_PASS \
  -X POST \
  -H "Content-Type: application/json" \
  "$ADMIN_URL/management/weblogic/latest/domainRuntime/serverRuntimes/MS1/captureImage" \
  | jq '.'
```

#### 6.2 Verificar y examinar el contenido del ZIP

```bash
# Esperar unos segundos para que se genere el archivo
sleep 5

# Listar imágenes disponibles
ls -lt $DOMAIN_HOME/servers/MS1/logs/diagnostic_images/ | head -5

# Obtener el nombre del ZIP más reciente
DIAG_ZIP=$(ls -t $DOMAIN_HOME/servers/MS1/logs/diagnostic_images/*.zip 2>/dev/null | head -1)
echo "Imagen de diagnóstico: $DIAG_ZIP"

# Listar contenido del ZIP
unzip -l "$DIAG_ZIP"
```

#### 6.3 Extraer y analizar el thread dump de la imagen

```bash
# Extraer el thread dump incluido en la imagen
unzip -p "$DIAG_ZIP" "*/Thread_Dump.txt" > /u01/lab08/diag_image_threaddump.txt 2>/dev/null || \
unzip -p "$DIAG_ZIP" "*thread*" > /u01/lab08/diag_image_threaddump.txt 2>/dev/null

# Buscar threads hogging o stuck
grep -i "HOGGING\|STUCK\|deadlock" /u01/lab08/diag_image_threaddump.txt | head -20

# Resumen del contenido de la imagen
unzip -p "$DIAG_ZIP" "*/JVM_Info.txt" 2>/dev/null | head -30
```

---

## 7. Validación y Pruebas

Ejecuta estas verificaciones para confirmar que el laboratorio se completó correctamente:

```bash
echo "========================================"
echo "  VALIDACIÓN FINAL - Lab 08-00-01"
echo "========================================"

# V1: Módulo WLDF activo
echo -n "[V1] Módulo DiagLab08 existe: "
curl -s -u $WL_USER:$WL_PASS \
  "$ADMIN_URL/management/weblogic/latest/edit/wldfSystemResources/DiagLab08?links=none" \
  | jq -r '.name' 2>/dev/null | grep -q "DiagLab08" && echo "OK" || echo "FALLO"

# V2: Heap dump generado
echo -n "[V2] Heap dump de memory leak existe: "
[ -f /u01/lab08/heapdump_ms1.hprof ] && echo "OK" || echo "FALLO"

# V3: Thread dumps capturados
echo -n "[V3] Thread dump de starvation existe: "
[ -f /u01/lab08/threaddump_ms1.txt ] && echo "OK" || echo "FALLO"

# V4: Thread dump de deadlock con detección
echo -n "[V4] Deadlock detectado en thread dump: "
grep -q "Found.*deadlock\|Java-level deadlock" /u01/lab08/threaddump_deadlock.txt \
  && echo "OK" || echo "FALLO"

# V5: Diagnostic Image generada
echo -n "[V5] Diagnostic Image capturada: "
ls $DOMAIN_HOME/servers/MS1/logs/diagnostic_images/*.zip &>/dev/null \
  && echo "OK" || echo "FALLO"

# V6: Alertas WLDF registradas en logs
echo -n "[V6] Alertas WLDF en server log: "
grep -qi "DiagLab08\|WatchHeapHigh\|WatchStuckThreads" \
  $DOMAIN_HOME/servers/MS1/logs/MS1.log 2>/dev/null \
  && echo "OK" || echo "POSIBLEMENTE NO DISPARADA (normal si no se alcanzó el umbral)"

# V7: Runbook documentado
echo -n "[V7] Runbook con 3 incidencias documentadas: "
FILES=$(ls /u01/lab08/runbook/incidencia*.md 2>/dev/null | wc -l)
[ "$FILES" -eq 3 ] && echo "OK ($FILES archivos)" || echo "INCOMPLETO ($FILES/3 archivos)"

echo "========================================"
echo "  Validación completada"
echo "========================================"
```

**Resultado esperado:** Todos los checks en `OK`. El check V6 puede mostrar el mensaje alternativo si la JVM tiene heap grande y no se alcanzó el umbral durante el tiempo del laboratorio.

---

## 8. Resolución de Problemas

### Problema 1: jstack falla con "Unable to open socket file" o "Process not found"

**Síntomas:**
```
Unable to open socket file: target process not responding or HotSpot VM not loaded
```

**Causa:**
El PID obtenido con `jps` no corresponde al proceso de MS1, o el proceso de MS1 fue iniciado por un usuario diferente al que ejecuta jstack. En entornos con múltiples JVMs, `jps` puede devolver varios procesos `weblogic.Server`.

**Solución:**

```bash
# Paso 1: Identificar el PID correcto de MS1 (buscar por puerto de escucha)
MS1_PID=$(ss -tlnp | grep ':7003' | grep -oP 'pid=\K[0-9]+' | head -1)
echo "MS1 PID por puerto: $MS1_PID"

# Paso 2: Si MS1 fue iniciado por otro usuario, usar sudo
sudo $JAVA_HOME/bin/jstack -l $MS1_PID > /u01/lab08/threaddump_ms1.txt

# Paso 3: Alternativa con kill -3 (no requiere mismo usuario si tienes sudo)
sudo kill -3 $MS1_PID
# El dump aparece en el log de stdout de MS1
tail -200 $DOMAIN_HOME/servers/MS1/logs/MS1.out | grep -A 100 "Full thread dump"

# Paso 4: Verificar que el attach mechanism esté habilitado
# Añadir a las JVM options de MS1 si es necesario:
# -Djdk.attach.allowAttachSelf=true
```

---

### Problema 2: WLDF Watch no dispara la alerta aunque el heap supera el umbral

**Síntomas:**
- El heap de MS1 supera visiblemente el 80% según la REST API.
- No aparecen entradas de `WatchHeapHigh` en `MS1.log`.
- No se genera ninguna Diagnostic Image automática.

**Causa:**
La expresión de la Watch Rule hace referencia a atributos del Harvester que no coinciden exactamente con los nombres de los MBeans configurados en el Harvested Types. WLDF requiere que los atributos en la expresión de la Watch coincidan con los tipos y atributos declarados en el Harvester. Si el tipo `JVMRuntimeMBean` no está correctamente targetado o los nombres de atributo tienen mayúsculas/minúsculas incorrectas, la regla nunca se evalúa.

**Solución:**

```bash
# Paso 1: Verificar los atributos harvested disponibles en runtime
curl -s -u $WL_USER:$WL_PASS \
  "http://localhost:7003/management/weblogic/latest/serverRuntime/JVMRuntime?links=none" \
  | jq 'keys'
# Observa los nombres exactos de atributos devueltos

# Paso 2: Corregir la expresión de la Watch en WLST
$WL_HOME/common/bin/wlst.sh << 'WLST_FIX'
connect('weblogic', 'Welcome1', 't3://localhost:7001')
edit()
startEdit()

# Navegar a la Watch y corregir la expresión
wldfRes = getMBean('/WLDFSystemResources/DiagLab08/WLDFResource/DiagLab08')
watchNotif = wldfRes.getWatchNotification()
w1 = watchNotif.lookupWatch('WatchHeapHigh')

# Expresión corregida usando nombres exactos del MBean
# HeapFreeCurrent y HeapSizeMax son los nombres correctos en WebLogic 15
w1.setRuleExpression("${WLDFHarvesterData:Type=weblogic.management.runtime.JVMRuntimeMBean,HeapFreeCurrent} < (${WLDFHarvesterData:Type=weblogic.management.runtime.JVMRuntimeMBean,HeapSizeMax} * 0.20)")

save()
activate(block='true')
print('Watch corregida')
disconnect()
exit()
WLST_FIX

# Paso 3: Verificar que el módulo está activo y harvesting
grep -i "DiagLab08\|harvester\|WLDF" $DOMAIN_HOME/servers/MS1/logs/MS1.log | tail -10

# Paso 4: Forzar una captura manual de imagen para verificar que las notificaciones funcionan
curl -s -u $WL_USER:$WL_PASS \
  -X POST \
  "$ADMIN_URL/management/weblogic/latest/domainRuntime/serverRuntimes/MS1/captureImage" \
  | jq '.messages'
```

---

## 9. Limpieza del Entorno

Ejecuta estos pasos al finalizar el laboratorio para dejar el entorno en estado limpio:

```bash
echo ">>> Iniciando limpieza del Lab 08-00-01..."

# 1. Detener cualquier proceso JMeter que siga activo
pkill -f "jmeter" 2>/dev/null && echo "JMeter detenido." || echo "JMeter ya estaba detenido."

# 2. Desplegar (undeploy) las aplicaciones de simulación
$WL_HOME/common/bin/wlst.sh << 'WLST_CLEAN'
connect('weblogic', 'Welcome1', 't3://localhost:7001')
try:
    undeploy('memleak',    targets='Cluster-1', block='true')
    print('memleak desinstalado')
except:
    print('memleak no encontrado (ya desinstalado)')
try:
    undeploy('starvation', targets='Cluster-1', block='true')
    print('starvation desinstalado')
except:
    print('starvation no encontrado')
try:
    undeploy('deadlock',   targets='Cluster-1', block='true')
    print('deadlock desinstalado')
except:
    print('deadlock no encontrado')
disconnect()
exit()
WLST_CLEAN

# 3. Eliminar el módulo WLDF DiagLab08 (opcional: conservarlo si se quiere monitoreo continuo)
read -p "¿Eliminar el módulo WLDF DiagLab08? (s/N): " RESP
if [[ "$RESP" =~ ^[sS]$ ]]; then
  $WL_HOME/common/bin/wlst.sh << 'WLST_WLDF'
connect('weblogic', 'Welcome1', 't3://localhost:7001')
edit()
startEdit()
delete('DiagLab08', 'WLDFSystemResource')
save()
activate(block='true')
print('Módulo DiagLab08 eliminado')
disconnect()
exit()
WLST_WLDF
fi

# 4. Archivar los artefactos del laboratorio
ARCHIVE_DIR=/u01/lab08/archive_$(date +%Y%m%d_%H%M%S)
mkdir -p $ARCHIVE_DIR
cp /u01/lab08/heapdump_ms1.hprof $ARCHIVE_DIR/ 2>/dev/null || true
cp /u01/lab08/threaddump_ms1.txt $ARCHIVE_DIR/ 2>/dev/null || true
cp /u01/lab08/threaddump_deadlock.txt $ARCHIVE_DIR/ 2>/dev/null || true
cp -r /u01/lab08/runbook $ARCHIVE_DIR/ 2>/dev/null || true
echo "Artefactos archivados en: $ARCHIVE_DIR"

# 5. Limpiar archivos temporales grandes (heap dump puede ser varios GB)
read -p "¿Eliminar el heap dump (.hprof) para liberar espacio? (s/N): " RESP2
if [[ "$RESP2" =~ ^[sS]$ ]]; then
  rm -f /u01/lab08/heapdump_ms1.hprof
  echo "Heap dump eliminado."
fi

# 6. Verificar estado final del dominio
curl -s -u $WL_USER:$WL_PASS \
  "$ADMIN_URL/management/weblogic/latest/domainRuntime/serverLifeCycleRuntimes?links=none" \
  | jq '.items[] | {name: .name, state: .state}'

echo ">>> Limpieza completada."
```

---

## 10. Resumen

### Lo que has construido y practicado

En este laboratorio has completado el ciclo completo de diagnóstico avanzado en WebLogic 15:

| Fase                     | Actividad realizada                                           | Herramienta principal          |
|--------------------------|---------------------------------------------------------------|--------------------------------|
| **Observabilidad**       | Módulo WLDF con Harvester, Watch Rules y Notifications        | WLST + WLDF                    |
| **Incidencia 1**         | Memory leak detectado, heap dump capturado y analizado        | JMeter + jmap + Eclipse MAT    |
| **Incidencia 2**         | Thread starvation por Work Manager, threads STUCK analizados  | JMeter + jstack + REST API     |
| **Incidencia 3**         | Deadlock detectado automáticamente con jstack -l              | jstack + análisis manual       |
| **Evidencia forense**    | Diagnostic Image capturada manualmente y examinada            | REST API + unzip                |
| **Documentación**        | Runbook estructurado con síntomas, análisis y solución        | Markdown                       |

### Conceptos clave consolidados

- **WLDF como primera línea de defensa:** el Harvester + Watch proporciona detección automática sin agentes externos; las Diagnostic Images son evidencia forense inmediata.
- **Tres thread dumps con intervalo:** un solo dump es insuficiente; tres dumps con 10 segundos de separación permiten distinguir threads realmente bloqueados de los que están en tránsito.
- **jstack -l para deadlocks:** la opción `-l` incluye información de `java.util.concurrent.locks` y provoca que jstack reporte automáticamente los ciclos de deadlock al final del dump.
- **MAT Dominator Tree:** el árbol de dominadores es el punto de partida para cualquier análisis de memory leak; los objetos con mayor "Retained Heap" son los sospechosos principales.
- **Work Manager con max-threads-constraint:** es una herramienta válida de throttling, pero debe dimensionarse con datos reales de carga para evitar starvation.

### Runbook de producción — estructura mínima recomendada

Para cualquier incidencia en WebLogic 15, aplica este proceso:

```
1. DETECTAR   → WLDF Watch alert / monitoreo externo / queja de usuario
2. TRIAGEAR   → curl REST: threadPoolRuntime, JVMRuntime, serverState
3. CAPTURAR   → 3x jstack (10s intervalo) + Diagnostic Image + heap dump si sospecha OOM
4. ANALIZAR   → jstack: buscar BLOCKED/WAITING/deadlock | MAT: Dominator Tree
5. RESOLVER   → fix de código / configuración / reinicio controlado
6. DOCUMENTAR → síntomas, herramientas, análisis, solución, prevención
```

### Recursos adicionales

- [WLDF Configuration and Use Guide (WebLogic 14.1.1)](https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.1.0/wldfc/index.html)
- [Diagnosing Problems in Oracle WebLogic Server](https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.1.0/wlprg/diagnosing-problems.html)
- [jstack Tool Reference — JDK 21](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jstack.html)
- [Eclipse Memory Analyzer Tool — Getting Started](https://help.eclipse.org/latest/topic/org.eclipse.mat.ui.help/welcome.html)
- [Work Managers, Request Classes, and Constraints (WebLogic)](https://docs.oracle.com/en/middleware/fusion-middleware/weblogic-server/14.1.1.0/cnfgd/self_tuned.html)

---
*Lab 08-00-01 — Curso WebLogic 15: Administración y Diagnóstico Avanzado*
