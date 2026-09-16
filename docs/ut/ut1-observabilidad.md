# UT1 · Observabilidad de contenedores: métricas, logs y eventos

<p class="ut-meta">Módulo 5169 · 14 h · Sesiones 1 a 7 · RA1 CE a</p>

Esta es la primera unidad del módulo y se apoya en lo que se construye en la asignatura de despliegue: el servicio del curso (proxy nginx en web01, API con Docker Compose en app01 y PostgreSQL en db01) y una VM mon01 (10.10.0.20) con una pila de Prometheus y Grafana (la base de datos de métricas y el visor de gráficas). La versión definitiva de esa pila se monta en [5166 UT7](https://victor-educ.github.io/apuntes-5166/ut/ut7-monitorizacion/), que este curso llega en abril, y la VPC dev con su firewall no existe hasta diciembre; por eso en octubre trabajamos sobre un entorno provisional (lo detalla el aviso de más abajo): en la sesión 1 levantamos una pila mínima con el compose que os doy en clase y la cambiáis por la definitiva cuando llegue. Aquí no desplegamos nada nuevo del servicio: lo que hacemos es sacar de él todo lo que cuenta sobre su estado (métricas de recursos, métricas de aplicación, logs y eventos), llevarlo a mon01 y demostrar con pruebas que llega entero, a tiempo y se guarda. Ese "demostrar" es la mitad de la unidad y el criterio de evaluación lo dice explícitamente: integrar y verificar comunicación, integridad y almacenamiento. Sin datos fiables, la UT2 (umbrales y alarmas) no tiene sobre qué trabajar.

Al servicio desplegado en app01 lo llamaremos durante todo el módulo el **contenedor de referencia**. Se mantiene vivo hasta la UT8, donde se da de baja de forma controlada.

## Qué tienes que saber hacer al terminar

- Explicar de dónde salen las métricas de recursos de un contenedor (cgroups v2) y leerlas a mano en `/sys/fs/cgroup` antes de fiarte de una gráfica.
- Desplegar cAdvisor en app01 con los montajes correctos y configurar su scrape en Prometheus, sabiendo qué etiquetas identifican cada servicio de compose.
- Instrumentar la API con un endpoint `/metrics` (counter e histogram con buenas prácticas de nombres y etiquetas) y añadir exporters a lo que no se puede instrumentar (PostgreSQL, nginx).
- Configurar el driver de logs de Docker con límites, hacer que la aplicación escriba logs JSON con `request_id`, y enviarlos a Loki con Promtail con las etiquetas `host`, `container` y `service`.
- Recoger los eventos del demonio Docker (die, oom, health_status) en Loki.
- Verificar la integración con comandos concretos: comunicación, recepción, integridad (conteos y relojes), almacenamiento y pérdidas, e interpretar lo que sale.
- Explicar y documentar qué ocurre con métricas y logs cuando se corta la red entre app01 y mon01.

## Antes de entrar en detalle

Un jueves a las tres de la tarde la API del servicio del curso empieza a devolver errores 500. Nadie se entera hasta el viernes, cuando un compañero intenta usarla y avisa por el grupo. Entras en app01, haces `docker logs` y el fichero pesa 4 GB y no hay quien lo lea; miras `docker stats` y la CPU va bien, así que no sabes si fue la base de datos, la memoria o un despliegue a medias. Lo que pasó se ha perdido y solo puedes reiniciar y cruzar los dedos. Esta unidad existe para que esa escena no se repita: al terminarla, todo lo que el contenedor cuenta de sí mismo (cuánto consume, cuántas peticiones atiende y cuánto tardan, qué escribe en sus logs y cuándo arranca, muere o se reinicia) llega a una máquina aparte, mon01, se guarda ahí varios días y puedes demostrar con comandos que llega completo y a tiempo. En una frase: que el servicio se pueda vigilar desde fuera sin entrar en la máquina.

| Herramienta o concepto | Qué es, en una frase | Para qué la usamos en esta unidad |
|---|---|---|
| cgroups v2 | La contabilidad del kernel de Linux: unos ficheros en `/sys/fs/cgroup` donde apunta cuánta CPU, memoria y disco gasta cada grupo de procesos | Leer a mano el consumo real de un contenedor antes de fiarte de una gráfica |
| cAdvisor | Un contenedor de Google que lee esos ficheros para todos los contenedores del host y los publica en una página de texto | Sacar las métricas de recursos de app01 sin escribir código |
| Prometheus y PromQL | Una base de datos de series numéricas que cada 15 s va a buscar los datos a cada máquina; PromQL es su lenguaje de consulta | Guardar todas las métricas en mon01 y hacer las consultas de comprobación |
| prometheus_client y prom-client | Las librerías oficiales de Prometheus para Python y Node: unas pocas líneas que cuentan peticiones y miden tiempos dentro de la API | Instrumentar la API para saber si funciona, no solo cuánto consume |
| Exporters (postgres_exporter, nginx-prometheus-exporter) | Programas auxiliares que preguntan a PostgreSQL o a nginx por su interfaz nativa y traducen la respuesta al formato de Prometheus | Vigilar lo que no podemos modificar por dentro |
| Driver de logs json-file | El mecanismo con que Docker recoge lo que el contenedor escribe por pantalla y lo guarda en un fichero por contenedor | Limitar el tamaño de esos ficheros y dejarlos listos para que un agente los lea |
| Promtail | Un agente de Grafana Labs que lee los logs de los contenedores, les pone etiquetas (host, contenedor, servicio) y los envía por HTTP a Loki | Sacar los logs de app01 hacia mon01, con reintentos si la red falla |
| Loki, LogQL y logcli | La base de datos de logs de Grafana Labs, que guarda el texto comprimido e indexa solo unas pocas etiquetas; LogQL es su lenguaje de consulta y logcli su cliente de terminal | Guardar y buscar los logs en mon01 |
| Grafana | El visor: una web que dibuja gráficas a partir de Prometheus y de Loki | Ver métricas y logs del mismo instante en la misma pantalla |
| docker events | El flujo de avisos del demonio Docker: un contenedor ha arrancado, ha muerto, se ha quedado sin memoria | Recoger esos avisos en Loki con un contenedor auxiliar, sin programar nada |
| remote_write | El modo en que un Prometheus pequeño envía sus muestras a otro central, con cola en disco por si la red se corta | Comparar en la sesión 6 qué se pierde con scrape y qué con remote_write |
| chrony y nftables | chrony mantiene en hora la VM por NTP; nftables es el cortafuegos del kernel de Linux | Sincronizar relojes para que Loki no rechace líneas, y cortar la red a propósito para ver qué aguanta la cadena |

**Cómo está organizada la unidad.** Empezamos por "Qué expone un contenedor", el mapa de los cuatro flujos (métricas de recursos, métricas de aplicación, logs y eventos) y de las flechas entre app01 y mon01; sin ese mapa el resto son piezas sueltas. Después van los cuatro flujos en orden de dificultad: primero las métricas de recursos, porque ya existen y solo hay que leerlas; luego las de aplicación, porque hay que tocar código; después los logs, que traen su propio agente y su propia base de datos; y por último los eventos, que reutilizan el camino de los logs. Sigue el apartado de scrape y remote_write, que explica cómo viajan las métricas y por qué eso importa cuando la red falla. Cierra la verificación, que necesita todas las piezas montadas y es lo que más pesa en la práctica.

!!! info "Lo que necesitas de la otra asignatura"
    Esta unidad se hace sobre un entorno provisional, porque la VPC y el firewall de [5166 UT2 y UT3](https://victor-educ.github.io/apuntes-5166/ut/ut2-vpc/) no existen hasta noviembre y diciembre, y la pila de monitorización de [5166 UT7](https://victor-educ.github.io/apuntes-5166/ut/ut7-monitorizacion/) no llega hasta abril. Lo que tienes que traer de 5166 son las dos VM creadas en la [sesión 3 de 5166 UT1 desde la plantilla cloud-init](https://victor-educ.github.io/apuntes-5166/ut/ut1-virtualizacion/), en el bridge del aula (vmbr0): `app01` con el servicio del curso en compose y `mon01` con Prometheus, Alertmanager y Grafana levantados con el compose que os doy en la sesión 1. Loki lo añadimos aquí.
    Las IP 10.10.x.x de los ejemplos son las de la VPC dev; en el entorno provisional sustituidlas por las que tengan vuestras VM en vmbr0, y las reglas de OPNsense no aplican todavía. Cuando 5166 termine la UT3 (9 dic), las VM se mueven a la VPC dev detrás del firewall: eso se hace en la UT3 de este módulo, que coincide en fechas.

## Qué expone un contenedor

En este apartado no montamos nada todavía: el objetivo es que tengas el mapa completo antes de tocar configuración. Vamos a ver cuáles son las cuatro cosas que un contenedor cuenta de sí mismo, quién produce cada una, cómo se mira en la propia máquina con un comando y a qué sistema de mon01 acaba llegando. Las herramientas de la tabla son las de la tabla de arriba; aquí lo que importa es el recorrido de cada flujo.

Un contenedor en ejecución no es más que un proceso (o varios) con namespaces y cgroups (los dos mecanismos del kernel que lo aíslan del resto y le contabilizan el consumo). Todo lo que queremos saber de él sale por uno de estos cuatro caminos:

| Flujo | Quién lo produce | Cómo se lee en local | A dónde va |
|----|----|----|----|
| Métricas de recursos | El kernel, vía cgroups | `docker stats`, `/sys/fs/cgroup`, cAdvisor | Prometheus |
| Métricas de aplicación | La propia aplicación, en `/metrics` (formato de exposición de Prometheus) | `curl app01:9102/metrics` | Prometheus |
| Logs | stdout y stderr del proceso principal, capturados por el driver de logs de Docker | `docker logs -f app` | Loki (vía Promtail) |
| Eventos | El demonio Docker: start, die, oom, health_status... | `docker events` | Loki (vía un contenedor auxiliar y Promtail) |

Los dos primeros son series numéricas: se muestrean cada pocos segundos y se guardan comprimidas en una base de datos de series temporales. Los otros dos son texto con marca de tiempo: no se muestrean, se recogen línea a línea. Por eso hay dos sistemas de almacenamiento (Prometheus y Loki) y no uno. Grafana solo pinta lo que hay en ambos.

```mermaid
flowchart LR
    subgraph app01[app01 · 10.10.2.x]
        api[API<br>/metrics :9102<br>stdout JSON]
        cad[cAdvisor :8081]
        ev[docker-events<br>contenedor auxiliar]
        pt[Promtail :9080]
        dk[(dockerd<br>json-file)]
        cg[(cgroups v2)]
        cg --> cad
        api --> dk
        ev --> dk
        dk --> pt
    end
    subgraph db01[db01 · 10.10.3.x]
        pgx[postgres_exporter :9187]
    end
    subgraph mon01[mon01 · 10.10.0.20]
        prom[Prometheus :9090]
        loki[Loki :3100]
        graf[Grafana :3000]
        prom --> graf
        loki --> graf
    end
    prom -.->|scrape| api
    prom -.->|scrape| cad
    prom -.->|scrape| pgx
    pt -->|push HTTP| loki
```

Fijaos en la dirección de las flechas: las métricas las **tira** Prometheus (pull), los logs los **empuja** Promtail (push). Esa diferencia lo condiciona todo cuando hay un corte de red y lo veremos al final de la unidad.

## Métricas de recursos: cgroups v2 y cAdvisor

Aquí sacamos el primer flujo, el más sencillo porque ya existe sin que hagamos nada: el kernel lleva la cuenta de lo que consume cada contenedor y solo hay que leerla. Primero la leemos a mano para saber qué significa cada número, después la publicamos con cAdvisor para que Prometheus la recoja, y al final hacemos las tres consultas en PromQL (el lenguaje de consulta de Prometheus) que van a estar en el panel de Grafana toda la unidad. Lo que más problemas da no es el consumo en sí, sino qué etiqueta identifica a cada contenedor y cuántas series generamos.

### De dónde salen los números

Cuando Docker arranca un contenedor, le pide al kernel un cgroup (control group) y mete dentro su proceso principal. El cgroup es a la vez el sitio donde se aplican límites (`--memory`, `--cpus`) y el sitio donde el kernel contabiliza lo consumido. En Debian 13 con Docker Engine 28 el sistema usa **cgroups v2** con el driver systemd, así que cada contenedor aparece como una unidad `docker-<id>.scope` colgando de `system.slice`:

```bash
ID=$(docker inspect -f '{{.Id}}' servicio-app-1)
ls /sys/fs/cgroup/system.slice/docker-$ID.scope/
# cgroup.procs  cpu.max  cpu.stat  io.stat  memory.current  memory.max  memory.stat  pids.current ...

cat /sys/fs/cgroup/system.slice/docker-$ID.scope/memory.current   # bytes en uso ahora mismo
cat /sys/fs/cgroup/system.slice/docker-$ID.scope/memory.max       # "max" si no hay límite
cat /sys/fs/cgroup/system.slice/docker-$ID.scope/cpu.stat
# usage_usec 184392110
# user_usec 121003320
# system_usec 63388790
# nr_throttled 0
grep -E '^(anon|file|inactive_file|active_file) ' /sys/fs/cgroup/system.slice/docker-$ID.scope/memory.stat
```

Con cgroups v1 (lo que os encontraréis todavía en hosts viejos y en alguna documentación) la jerarquía estaba partida por controlador (`/sys/fs/cgroup/memory/docker/<id>/memory.usage_in_bytes`, `/sys/fs/cgroup/cpu/...`). En v2 hay un único árbol y los ficheros se llaman distinto. Merece la pena saber leerlos a mano porque cuando cAdvisor da un número raro, esto es la fuente de verdad.

`docker stats` lee exactamente estos ficheros cada segundo y los formatea. cAdvisor hace lo mismo pero para todos los contenedores a la vez, guarda un histórico corto en memoria y lo expone en `/metrics` en formato Prometheus. No hay magia: las métricas `container_*` son estos ficheros con etiquetas.

### Memory usage frente a working set

Es la confusión más habitual con la memoria de un contenedor. `memory.current` (lo que cAdvisor expone como `container_memory_usage_bytes`) incluye la caché de páginas: si el contenedor lee un fichero de 200 MB, el kernel se queda esas páginas en caché y cuentan como memoria del cgroup aunque el proceso no las necesite y el kernel pueda soltarlas al instante. Por eso `usage` sube y sube en un contenedor que escribe logs o lee de disco, y la gente cree que tiene una fuga.

El **working set** es `memory.current` menos `inactive_file` de `memory.stat`: la memoria que el kernel no podría reclamar sin dolor. cAdvisor lo expone como `container_memory_working_set_bytes` y es la métrica que usa el propio OOM killer (el mecanismo del kernel que mata procesos cuando se agota la memoria) para decidir si un contenedor ha superado `memory.max`. Conclusión práctica: para alarmas y para comparar contra el límite, working set; `usage_bytes` solo para entender de qué está hecha la memoria. Para "memoria que realmente ha pedido el proceso" también tenéis `container_memory_rss` (anon en v2).

```promql
# porcentaje del límite que usa cada servicio de compose
container_memory_working_set_bytes{container_label_com_docker_compose_service!=""}
  / container_spec_memory_limit_bytes * 100
```

Si un contenedor no tiene límite, `container_spec_memory_limit_bytes` vale un número enorme y la división da casi cero. Es una pista, no un bug: ponedle límite.

### cAdvisor en compose

cAdvisor necesita ver el árbol de cgroups del host, el socket de Docker (`/var/run/docker.sock`, el fichero por el que se habla con el demonio; lo necesita para saber qué cgroup es qué contenedor y leer sus etiquetas) y el directorio de Docker (para métricas de disco). Se despliega en el mismo compose que el servicio, o en un compose aparte de "agentes" en app01; yo prefiero lo segundo para que reiniciar el servicio no tumbe los agentes.

```yaml
# /opt/agentes/compose.yml en app01
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.52.1
    container_name: cadvisor
    privileged: true
    devices:
      - /dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    command:
      - --docker_only=true
      - --housekeeping_interval=10s
      - --store_container_labels=false
      - --whitelisted_container_labels=com.docker.compose.service,com.docker.compose.project
      - --disable_metrics=percpu,sched,tcp,udp,hugetlb,referenced_memory,cpu_topology,resctrl
    ports:
      - "8081:8080"
    restart: unless-stopped
```

Cosas a saber de este fichero. `privileged` y `/dev/kmsg` hacen falta para que lea eventos OOM del kernel; sin ellos funciona pero `container_oom_events_total` nunca sube. `--docker_only` quita del listado los cgroups de systemd que no son contenedores (servicios del host), que no aportan nada y multiplican las series. `--housekeeping_interval=10s` baja el consumo de CPU del propio cAdvisor (por defecto muestrea cada segundo, y cAdvisor comiéndose el 5 % de la CPU de app01 es un clásico). El puerto: cAdvisor escucha en 8080 dentro del contenedor, pero en app01 ese puerto ya lo ocupa la API que consume web01, así que lo publicamos en 8081 (la tabla de puertos del laboratorio dice 8080 para cAdvisor; en app01 no puede ser).

### Etiquetas y cardinalidad

Cada serie de cAdvisor lleva `id` (la ruta del cgroup), `name` (nombre del contenedor, por ejemplo `servicio-app-1`) e `image`. Con `--store_container_labels` cAdvisor convierte todas las etiquetas Docker del contenedor en etiquetas `container_label_<clave>` con los puntos cambiados por guiones bajos. Compose pone bastantes etiquetas (proyecto, servicio, número de réplica, hash de la configuración, ruta del fichero...), y esa última, `com.docker.compose.config-hash`, cambia en cada `compose up` que modifique el servicio. Si la dejas pasar, cada redespliegue crea series nuevas en Prometheus y las viejas quedan huérfanas: eso es **cardinalidad**, y es el enemigo número uno de una base de datos de series. Por eso el compose de arriba desactiva todas y solo deja pasar `com.docker.compose.service` y `com.docker.compose.project`.

La etiqueta que os interesa para todo el módulo es `container_label_com_docker_compose_service`: es estable entre despliegues (siempre `app`, `db`, `nginx`) mientras que `name` cambia si escaláis o renombráis. En Prometheus, en el scrape de cAdvisor, la renombramos a algo manejable:

```yaml
# prometheus.yml en mon01 (fragmento)
scrape_configs:
  - job_name: cadvisor
    scrape_interval: 15s
    static_configs:
      - targets: ["10.10.2.10:8081"]
        labels: { host: app01 }
    metric_relabel_configs:
      - source_labels: [container_label_com_docker_compose_service]
        target_label: service
      - regex: container_label_com_docker_compose_project
        action: labeldrop
      - source_labels: [__name__]
        regex: container_(cpu_usage_seconds_total|memory_working_set_bytes|memory_usage_bytes|spec_memory_limit_bytes|network_(receive|transmit)_bytes_total|fs_(reads|writes)_bytes_total|oom_events_total|last_seen)
        action: keep
```

El último bloque `keep` es opcional pero recomendable en un laboratorio con 3 GB de RAM en mon01: cAdvisor expone más de 100 métricas por contenedor y solo vais a usar diez. `container_last_seen` sirve para saber qué contenedores existen (y para detectar en UT2 los que han desaparecido).

Las tres consultas que usaréis en Grafana en la sesión 2:

```promql
# CPU en núcleos por servicio (1 = un núcleo entero)
sum by (service) (rate(container_cpu_usage_seconds_total{service!=""}[2m]))
# memoria working set por servicio
sum by (service) (container_memory_working_set_bytes{service!=""})
# tráfico de red recibido, bytes/s
sum by (service) (rate(container_network_receive_bytes_total{service!=""}[2m]))
```

La ventana `[2m]` con scrape cada 15 s da 8 puntos por ventana; con `[30s]` tendrías dos puntos y las gráficas saldrían a saltos. Regla: la ventana de `rate()` al menos cuatro veces el intervalo de scrape.

## Métricas de aplicación

![Prometheus](../img/prometheus-logo.svg){ .logo-inline } Las métricas de recursos dicen cuánto consume el contenedor, no si el servicio funciona. Una API puede estar al 2 % de CPU devolviendo 500 a todo el mundo. Para eso la aplicación se **instrumenta**: incorpora la librería cliente de Prometheus de su lenguaje, registra unas cuantas métricas y las sirve en un endpoint `/metrics` en texto plano, y Prometheus lo lee cada 15 s.

### Tipos de métrica

- **Counter**: solo sube (peticiones totales, errores totales, bytes enviados). Nunca se consulta el valor crudo, que depende de cuándo arrancó el proceso, sino su velocidad con `rate()` o `increase()`. Si el proceso se reinicia, el contador vuelve a cero y `rate()` lo detecta y lo corrige.
- **Gauge**: sube y baja (conexiones activas, tamaño de una cola, temperatura, fecha del último backup en segundos epoch). Se consulta directo.
- **Histogram**: cuenta observaciones en cubos acumulativos (`le="0.1"`, `le="0.5"`...) y además suma y total. Permite calcular percentiles en el servidor con `histogram_quantile()` y, lo que importa, agregarlos entre instancias. Es lo que se usa para latencias.
- **Summary**: percentiles calculados en el cliente. Más preciso para una instancia, pero no se puede agregar entre varias réplicas. En la práctica, histogram casi siempre.

### Python con prometheus_client

Suponemos que la API es Flask, el microframework web de Python (si es FastAPI, el patrón es idéntico con un middleware). El endpoint `/metrics` lo sirve la propia aplicación en el mismo puerto 8080 o, mejor, en un puerto aparte (9102) con `start_http_server` para que no pase por el proxy y no cuente como tráfico de usuario:

```python
# metrics.py
import time
from flask import Flask, request, g
from prometheus_client import Counter, Histogram, Gauge, start_http_server

REQUESTS = Counter(
    "app_http_requests_total",
    "Peticiones HTTP atendidas",
    ["method", "route", "status"],
)
LATENCY = Histogram(
    "app_http_request_duration_seconds",
    "Duración de la petición en segundos",
    ["method", "route"],
    buckets=(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5),
)
INFLIGHT = Gauge("app_http_requests_in_flight", "Peticiones en curso")
DB_UP = Gauge("app_db_reachable", "1 si la última consulta a PostgreSQL funcionó")

def instrument(app: Flask) -> None:
    @app.before_request
    def _start():
        g.t0 = time.perf_counter()
        INFLIGHT.inc()

    @app.after_request
    def _end(resp):
        route = request.url_rule.rule if request.url_rule else "unmatched"
        LATENCY.labels(request.method, route).observe(time.perf_counter() - g.t0)
        REQUESTS.labels(request.method, route, str(resp.status_code)).inc()
        INFLIGHT.dec()
        return resp

    start_http_server(9102)   # sirve /metrics en 0.0.0.0:9102
```

Detalles que marcan la diferencia. La etiqueta `route` lleva la **plantilla** de la ruta (`/items/<int:id>`), nunca la URL real (`/items/48213`): con la URL real cada id distinto es una serie nueva y en una semana tienes 100 000 series. Lo mismo con `status`: el código, no el mensaje. Los buckets del histogram se eligen pensando en el servicio: para una API con objetivo de 250 ms la lista de arriba es razonable; para un batch que tarda minutos no vale. Y `INFLIGHT` como gauge os permitirá en UT2 detectar una API que se está atascando antes de que lo digan los errores.

Si la API corre con gunicorn (el servidor que ejecuta la aplicación Python en producción) y varios workers (procesos en paralelo), cada proceso tiene su propio registro y Prometheus vería valores que saltan de un worker a otro. La librería lo resuelve con el modo multiproceso: variable de entorno `PROMETHEUS_MULTIPROC_DIR` apuntando a un directorio vacío y `MultiProcessCollector`. Está en la [documentación de prometheus_client](https://prometheus.github.io/client_python/multiprocess/); en clase lo activaremos si la API va con más de un worker.

### Node con prom-client

Si la API es Node con Express, la librería oficial es prom-client y el patrón es el mismo: un contador y un histograma por petición y un servidor aparte en 9102. Fíjate en que aquí el endpoint `/metrics` se sirve a mano y en que `collectDefaultMetrics` da métricas del proceso sin registrar nada:

```javascript
// metrics.js
const client = require("prom-client");
const express = require("express");

client.collectDefaultMetrics({ prefix: "app_" });   // CPU, memoria, event loop, GC del proceso Node

const requests = new client.Counter({
  name: "app_http_requests_total",
  help: "Peticiones HTTP atendidas",
  labelNames: ["method", "route", "status"],
});
const latency = new client.Histogram({
  name: "app_http_request_duration_seconds",
  help: "Duración de la petición en segundos",
  labelNames: ["method", "route"],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
});

function instrument(app) {
  app.use((req, res, next) => {
    const end = latency.startTimer();
    res.on("finish", () => {
      const route = req.route ? req.route.path : "unmatched";
      end({ method: req.method, route });
      requests.inc({ method: req.method, route, status: String(res.statusCode) });
    });
    next();
  });
}

const metricsApp = express();
metricsApp.get("/metrics", async (_req, res) => {
  res.set("Content-Type", client.register.contentType);
  res.end(await client.register.metrics());
});
metricsApp.listen(9102);

module.exports = { instrument };
```

Con esto, `curl localhost:9102/metrics` devuelve las mismas series que en la versión Python más las `app_nodejs_*` del proceso. `collectDefaultMetrics` es gratis y muy útil: `app_nodejs_eventloop_lag_seconds` es el primer síntoma de una API Node saturada, mucho antes que la CPU.

### Nombres y etiquetas: convenciones

Prometheus no impone nombres, pero si no seguís las convenciones las consultas se vuelven un infierno y los dashboards de la comunidad no os valen. Las reglas de [Metric and label naming](https://prometheus.io/docs/practices/naming/):

| Regla | Bien | Mal |
|----|----|----|
| Prefijo con el nombre de la aplicación o del subsistema | `app_http_requests_total` | `requests` |
| Unidad base en el nombre, sin prefijos SI | `_seconds`, `_bytes`, `_ratio` | `_ms`, `_kb`, `_percent` |
| Los counters terminan en `_total` | `app_errors_total` | `app_error_count` |
| Una métrica mide una cosa; las dimensiones van en etiquetas | `app_http_requests_total{status="500"}` | `app_http_500_total` |
| Etiquetas de cardinalidad acotada | `method`, `status`, `route` plantilla | `user_id`, `request_id`, URL completa |
| snake_case en minúsculas | `app_db_reachable` | `appDbReachable` |

Sobre cardinalidad, el cálculo que hay que hacer siempre: número de series = producto del número de valores de cada etiqueta. `method` (5) × `route` (20) × `status` (8) = 800 series para un counter. Asumible. Añade `client_ip` (miles) y estás en el millón.

### Exporters: lo que no puedes instrumentar

PostgreSQL, nginx, Redis o el propio host no van a incorporar una librería de Prometheus. Para ellos existe el patrón **exporter**: un proceso auxiliar que consulta al software por su interfaz nativa (SQL, `stub_status`, comandos) y traduce lo que obtiene al formato de Prometheus. Corre al lado del servicio, normalmente como sidecar (un contenedor auxiliar pegado al principal) en el mismo compose.

**postgres_exporter** (puerto 9187) consulta `pg_stat_database`, `pg_stat_activity`, `pg_stat_bgwriter`, `pg_locks` y compañía. Nunca le deis el usuario `postgres`: se crea un usuario de solo lectura con el rol `pg_monitor`, que existe desde PostgreSQL 10 precisamente para esto.

```sql
CREATE USER postgres_exporter WITH PASSWORD 'cámbiame';
GRANT pg_monitor TO postgres_exporter;
GRANT CONNECT ON DATABASE servicio TO postgres_exporter;
```

```yaml
# en el compose de db01
  postgres_exporter:
    image: quay.io/prometheuscommunity/postgres-exporter:latest
    environment:
      DATA_SOURCE_URI: "db:5432/servicio?sslmode=disable"
      DATA_SOURCE_USER: postgres_exporter
      DATA_SOURCE_PASS_FILE: /run/secrets/pgx_pass
    secrets: [pgx_pass]
    ports: ["9187:9187"]
    depends_on: [db]
```

Las métricas que os importan: `pg_up` (1 si consigue conectar; es la primera alarma de UT2), `pg_stat_database_numbackends` (conexiones abiertas, comparadlas con `max_connections`), `pg_stat_database_xact_commit` y `xact_rollback` (con `rate()`), `pg_stat_database_blks_hit` frente a `blks_read` (acierto de caché), `pg_locks_count`, `pg_database_size_bytes`. Si necesitáis algo propio (filas de una tabla de pedidos, edad del pedido más antiguo sin procesar), el exporter admite consultas personalizadas en un fichero YAML pasado con `--extend.query-path`. Ese mecanismo está marcado como obsoleto en las versiones actuales aunque sigue funcionando; si desaparece, la alternativa mantenida es [sql_exporter](https://github.com/burningalchemist/sql_exporter), que hace exactamente eso y para cualquier base de datos.

```yaml
# queries.yaml
pedidos_pendientes:
  query: "SELECT count(*) AS total, EXTRACT(EPOCH FROM now() - min(creado)) AS edad_max FROM pedidos WHERE estado = 'pendiente'"
  metrics:
    - total:    { usage: GAUGE, description: "Pedidos sin procesar" }
    - edad_max: { usage: GAUGE, description: "Segundos del pedido pendiente más antiguo" }
```

**nginx-prometheus-exporter** (puerto 9113) lee el módulo `stub_status` de nginx, que hay que activar en web01 y proteger para que solo lo vea la subred de gestión:

```nginx
# /etc/nginx/conf.d/status.conf en web01
server {
    listen 127.0.0.1:8081;
    location = /stub_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
```

```bash
docker run -d --name nginx-exporter --network host \
  nginx/nginx-prometheus-exporter:latest \
  --nginx.scrape-uri=http://127.0.0.1:8081/stub_status
```

Lo que da `stub_status` es poco: `nginx_connections_active`, `nginx_connections_waiting`, `nginx_http_requests_total` y `nginx_up`. No hay códigos de respuesta ni latencias por ruta; para eso, en nginx de código abierto, la vía es el log de acceso en JSON parseado en Loki (lo hacemos en la sección de logs) o el módulo VTS de terceros. Es una limitación real que os encontraréis en empresa: nginx Plus sí expone todo eso por API, la versión libre no.

## Logs

El tercer flujo es texto, no números, y por eso tiene su propio camino: Docker lo recoge, Promtail lo etiqueta y lo envía, y Loki lo guarda. En este apartado hacemos cuatro cosas en orden: limitar el tamaño de los ficheros de log para que no llenen el disco, conseguir que la API escriba una línea JSON por evento con un identificador de petición, configurar Promtail para que lea esas líneas y las envíe, y montar Loki en mon01 para recibirlas y consultarlas. Al terminar tienes que poder seguir una petición desde nginx hasta la API por su identificador.

### El driver de logs de Docker y por qué limitarlo

Docker captura stdout y stderr del proceso principal del contenedor y se los entrega a un **driver de logs**. El driver por defecto es `json-file`: escribe cada línea como un objeto JSON (`{"log":"...","stream":"stdout","time":"2026-10-13T09:12:44.120Z"}`) en `/var/lib/docker/containers/<id>/<id>-json.log`. Sin configuración adicional ese fichero **no rota nunca**. Una API que escriba 200 líneas por segundo de 300 bytes genera 5 GB al día; en dos semanas app01 se queda sin disco, Docker no puede escribir y el contenedor se para o se cuelga. Es la causa más común de "se ha caído el servidor" en despliegues pequeños.

```json
// /etc/docker/daemon.json en app01, db01 y web01
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "5" }
}
```

Tras cambiarlo, `systemctl restart docker`, y ojo: **solo aplica a contenedores creados después**. Los existentes conservan su driver y sus opciones hasta que se recrean (`docker compose up -d --force-recreate`). Con esos valores el techo por contenedor son 250 MB. La rotación es por tamaño, no por tiempo: si un contenedor apenas escribe, sus logs de hace meses siguen ahí.

Otros drivers que vais a ver:

| Driver | Qué hace | Cuándo |
|----|----|----|
| `local` | Como json-file pero en formato binario propio, comprimido, con rotación por defecto (100 MB en 5 ficheros) | Cuando no necesitas que otro agente lea el fichero. Docker lo recomienda sobre json-file, pero Promtail y Fluent Bit leen json-file y no `local`, así que para nosotros no |
| `journald` | Envía las líneas al journal de systemd del host con metadatos del contenedor | Hosts donde ya centralizas el journal; consultas con `journalctl CONTAINER_NAME=app` |
| `loki` | Plugin de Docker que empuja directamente a Loki sin agente intermedio | Tentador, pero si Loki no responde el plugin bloquea el demonio o pierde líneas según configuración; y si el plugin falla, `docker` entero se resiente. Prefiero el agente |
| `none` | Descarta todo | Contenedores muy ruidosos cuya salida no importa |

Desde Docker 20.10 existe el **dual logging**: aunque uses `journald` o `loki`, Docker guarda una copia local para que `docker logs` siga funcionando. Está bien saberlo para no llevarse sustos, pero no cambia la recomendación: `json-file` con límites y un agente que lea los ficheros. El agente añade contexto (host, contenedor, servicio), transforma, reintenta y no toca el demonio.

### Logs estructurados: JSON con request_id

`docker logs` funciona con cualquier cosa que la aplicación escriba, pero un log que dice `Error procesando pedido 4821 para cliente 77` solo lo entiende una persona. Para filtrar en Loki por nivel, contar errores por ruta o seguir una petición desde nginx hasta la base de datos, la aplicación tiene que escribir **una línea JSON por evento** con campos fijos: `ts` (fecha en formato RFC 3339, como `2026-10-13T09:12:44.120Z`, con zona horaria y milisegundos), `level`, `msg`, `logger`, y los que aporten contexto: `request_id`, `method`, `route`, `status`, `duration_ms`, `user` (id, nunca nombre ni email).

El `request_id` es lo que permite la **correlación**. Lo genera nginx en web01 con la variable `$request_id` (32 caracteres hexadecimales, uno por petición), lo pasa a la API en una cabecera, la API lo incluye en todas sus líneas de log y lo devuelve al cliente en la respuesta. Cuando un usuario dice "me ha dado error a las 10:14", buscas ese id y ves la petición completa en los dos hosts.

```nginx
# web01: en el server del proxy
proxy_set_header X-Request-ID $request_id;
add_header X-Request-ID $request_id always;

log_format json escape=json '{"ts":"$time_iso8601","level":"info","msg":"access",'
  '"request_id":"$request_id","method":"$request_method","uri":"$request_uri",'
  '"status":$status,"bytes":$body_bytes_sent,"duration_ms":$request_time,'
  '"upstream":"$upstream_addr","client":"$remote_addr"}';
access_log /var/log/nginx/access.json json;
```

En la API, con Python, un formateador JSON en la librería `logging` estándar y un filtro que inyecte el id que llega en la cabecera (si no llega, se genera uno):

```python
import json, logging, uuid, sys
from flask import request, g

class JsonFormatter(logging.Formatter):
    def format(self, record):
        d = {
            "ts": self.formatTime(record, "%Y-%m-%dT%H:%M:%S") + f".{int(record.msecs):03d}Z",
            "level": record.levelname.lower(),
            "logger": record.name,
            "msg": record.getMessage(),
            "request_id": getattr(g, "request_id", None) if request else None,
        }
        if record.exc_info:
            d["exc"] = self.formatException(record.exc_info)
        return json.dumps(d, ensure_ascii=False)

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JsonFormatter())
logging.basicConfig(level=logging.INFO, handlers=[handler])

@app.before_request
def _rid():
    g.request_id = request.headers.get("X-Request-ID", uuid.uuid4().hex)

@app.after_request
def _rid_out(resp):
    resp.headers["X-Request-ID"] = g.request_id
    return resp
```

Tres normas para que esto funcione después en Loki: una línea por evento (nada de trazas multilínea sueltas; van dentro del campo `exc`), siempre a stdout (no a un fichero dentro del contenedor, que nadie va a leer y que llena el disco), y el mismo nombre de campo en todos los servicios (`request_id`, no `requestId` en uno y `rid` en otro).

### Promtail

![Loki](../img/loki-logo.png){ .logo-inline } Promtail es el agente de Grafana Labs para enviar logs a Loki. Tiene una advertencia que hay que hacer en 2026: Grafana lo ha declarado **al final de su vida** (sin cambios desde febrero de 2025, fin de soporte en marzo de 2026) en favor de [Grafana Alloy](https://grafana.com/docs/alloy/latest/), que hace lo mismo y además métricas y trazas con una configuración en su propio lenguaje. Lo usamos igualmente porque su configuración es YAML corto, se entiende en una sesión y todos los conceptos (descubrimiento, relabel, pipeline, posiciones, reintentos) se trasladan uno a uno a Alloy; en empresa os encontraréis las dos cosas y muchos Promtail que nadie ha migrado. Para pasar de uno a otro existe `alloy convert --source-format=promtail`.

Promtail hace cuatro cosas en orden: **descubre** de dónde leer, **etiqueta** cada origen con relabel (reglas que copian, renombran o descartan etiquetas), **procesa** cada línea con un pipeline y **envía** lotes a Loki por HTTP, guardando en un fichero de posiciones hasta dónde ha leído.

```yaml
# /opt/agentes/promtail.yml en app01
server:
  http_listen_port: 9080

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: http://10.10.0.20:3100/loki/api/v1/push
    external_labels:
      host: app01
      env: dev
    batchwait: 1s
    batchsize: 1048576
    backoff_config:
      min_period: 500ms
      max_period: 5m
      max_retries: 10
    timeout: 10s

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: [__meta_docker_container_name]
        regex: '/(.*)'
        target_label: container
      - source_labels: [__meta_docker_container_label_com_docker_compose_service]
        target_label: service
      - source_labels: [__meta_docker_container_label_com_docker_compose_project]
        target_label: project
      - source_labels: [__meta_docker_container_log_stream]
        target_label: stream
    pipeline_stages:
      - match:
          selector: '{service="app"}'
          stages:
            - json:
                expressions:
                  level: level
                  ts: ts
                  request_id: request_id
            - timestamp:
                source: ts
                format: RFC3339Nano
            - labels:
                level:
            - structured_metadata:
                request_id:
      - match:
          selector: '{service="app"} !~ "^\\{"'
          stages:
            - static_labels:
                malformed: "true"
```

Por partes:

**docker_sd_configs**. En lugar de leer los ficheros `*-json.log` (que también se puede, con `static_configs` y `__path__: /var/lib/docker/containers/*/*.log`), Promtail pregunta al demonio por el socket qué contenedores hay y lee sus logs por la API de Docker. Ventaja: obtiene el nombre y las etiquetas del contenedor sin tener que parsear rutas, y sigue a contenedores nuevos a los 5 s de arrancar. Necesita acceso al socket, así que el contenedor de Promtail lo monta en solo lectura y corre como root o con el gid del grupo `docker`.

**relabel_configs**. Las meta-etiquetas `__meta_docker_*` desaparecen al final del relabel; solo sobrevive lo que copiéis a una etiqueta sin guiones bajos iniciales. Con estas cuatro reglas cada flujo queda como `{host="app01", env="dev", container="servicio-app-1", service="app", project="servicio", stream="stdout"}`. Las etiquetas en Loki son **índice**, y cada combinación distinta de valores es un stream con sus propios chunks (los bloques comprimidos en que Loki guarda el texto): tres o cuatro etiquetas de pocos valores está bien; meter `request_id` o la ruta como etiqueta destroza Loki igual que destroza Prometheus. Para campos de cardinalidad alta que queréis filtrar rápido, Loki 3 tiene **structured metadata** (lo que hace la etapa `structured_metadata`): se guarda junto a la línea, no en el índice, y se consulta con `| request_id="..."`.

**pipeline_stages**. Se ejecutan por línea, en orden. `json` extrae campos a variables internas; `timestamp` usa el `ts` de la aplicación como marca de tiempo de la entrada en vez de la hora en que Promtail la leyó (importa cuando hay retraso: si Promtail estuvo 3 minutos sin poder enviar, las líneas se guardan con su hora real, no con la de reenvío); `labels` promociona `level` a etiqueta (5 valores posibles, aceptable y muy útil para `{service="app", level="error"}`). El segundo `match` marca las líneas que no son JSON (una traza de arranque, un `print` olvidado) para encontrarlas y corregirlas. Con `format: RFC3339Nano` Go acepta también RFC 3339 sin nanosegundos; si vuestra API escribe otra cosa, el formato se expresa con la fecha de referencia de Go (`2006-01-02 15:04:05.000`).

**positions**. El fichero `positions.yaml` guarda, por origen, el último offset (o el último timestamp leído, con docker_sd) enviado con éxito. Si Promtail se reinicia, continúa desde ahí. Si el fichero está en `/tmp` dentro del contenedor y no en un volumen, cada reinicio de Promtail reenvía todo lo que Docker conserve del contenedor (hasta 250 MB con nuestros límites), y Loki no deduplica: los `count_over_time` de la verificación os saldrán multiplicados. Volumen persistente siempre.

**clients**. `batchwait` y `batchsize` deciden cuándo sale un lote: al segundo o al llegar a 1 MB, lo que ocurra antes. `backoff_config` es lo que os salva en un corte de red: si el push falla, reintenta esperando 0,5 s, luego 1, 2, 4, 8... hasta 5 minutos entre intentos, y hasta 10 intentos por lote. La suma de las esperas con esos valores son unos 8,5 minutos: un corte de 3 minutos no pierde nada; uno de 15 sí, y lo veréis en `promtail_dropped_entries_total`. Mientras reintenta, Promtail deja de leer el origen (no avanza posiciones), así que lo que no ha enviado sigue en el fichero de Docker; el límite real de lo que aguanta un corte lo ponen `max-size` y `max-file`.

```yaml
# en /opt/agentes/compose.yml
  promtail:
    image: grafana/promtail:3.5.3
    container_name: promtail
    command: -config.file=/etc/promtail/promtail.yml
    volumes:
      - ./promtail.yml:/etc/promtail/promtail.yml:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - promtail-positions:/var/lib/promtail
    ports:
      - "9080:9080"
    restart: unless-stopped
volumes:
  promtail-positions:
```

Promtail expone sus propias métricas en `:9080/metrics` y hay que hacerles scrape desde mon01: `promtail_sent_entries_total`, `promtail_dropped_entries_total`, `promtail_request_duration_seconds_count{status_code!="204"}` (pushes fallidos) y `promtail_targets_active_total`. Quien vigila al vigilante.

```mermaid
sequenceDiagram
    participant D as dockerd (app01)
    participant P as Promtail
    participant L as Loki (mon01)
    D->>P: líneas nuevas del contenedor app (API /containers/{id}/logs?follow)
    P->>P: relabel + pipeline (json, timestamp, labels)
    P->>P: acumula lote (1 s o 1 MB)
    P->>L: POST /loki/api/v1/push (snappy + protobuf)
    alt Loki responde 204
        L-->>P: 204 No Content
        P->>P: avanza positions.yaml
    else Loki no responde o 5xx
        L--xP: timeout / error
        P->>P: espera backoff (0,5 s, 1, 2, ... 5 min) y reintenta hasta 10 veces
        P->>P: si agota reintentos: promtail_dropped_entries_total +N
    end
```

### Loki

Loki es "Prometheus para logs": mismo modelo de etiquetas, misma filosofía de indexar poco. Solo indexa las etiquetas del stream y el rango de tiempo; el texto se guarda comprimido en **chunks** y se busca por fuerza bruta dentro de los chunks que las etiquetas seleccionan. Por eso es barato y por eso las consultas sin etiquetas (`{host=~".+"} |= "error"` en 7 días) son lentas: hay que descomprimir todo.

Internamente tiene varios componentes (distributor recibe los push, ingester los acumula en memoria y los escribe como chunks, querier responde consultas, compactor aplica retención, y el índice TSDB, el mismo formato que usa Prometheus). En producción se reparten en varios procesos; para el laboratorio y para muchas empresas pequeñas se ejecuta todo en un binario (`-target=all`, el modo "monolítico") con almacenamiento en disco local. Del fichero, fíjate en `limits_config` (cuánto acepta y cuánto tiempo guarda) y en `compactor` (quién borra lo viejo); el resto es la parte fija del modo monolítico:

```yaml
# /opt/monitoring/loki/loki.yml en mon01
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2026-10-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 168h
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  allow_structured_metadata: true
  ingestion_rate_mb: 8
  ingestion_burst_size_mb: 16
  max_global_streams_per_user: 5000

compactor:
  working_directory: /loki/compactor
  delete_request_store: filesystem
  retention_enabled: true
  compaction_interval: 10m
  retention_delete_delay: 2h
```

La retención de 7 días (`168h`) se aplica cada 10 minutos por el compactor, y con la demora de 2 h. Con el servicio del curso, Loki ocupa entre 2 y 5 GB de disco con esa retención; en UT5, en la empresa, veréis retenciones de 30 a 90 días por normativa y ahí el disco local ya no vale: se pasa a un object store (un almacén de ficheros por HTTP, como MinIO o S3) cambiando `object_store` y nada más. `reject_old_samples_max_age` es lo que rechaza líneas con reloj atrasado, luego lo cruzamos con chrony (el servicio que mantiene en hora la VM por NTP, el protocolo de hora en red).

**LogQL** (el lenguaje de consulta de Loki, primo de PromQL) para comprobar que las cosas llegan, que es lo que necesitáis en esta unidad (los dashboards y alertas sobre logs van en UT2):

```text
{host="app01"}                                     # todo lo que envía app01, ¿llegan las etiquetas?
{service="app"} | json | level="error"             # errores de la API parseando la línea
{service="app"} | request_id="3f9a..."             # una petición completa (structured metadata)
{service=~"app|nginx"} |= "3f9a1c..."              # la misma petición en los dos hosts
count_over_time({service="app"}[5m])               # líneas en 5 min, para comparar con el origen
sum by (level) (rate({service="app"} | json [1m])) # líneas/s por nivel
{service="app", malformed="true"}                  # lo que no era JSON
```

Desde línea de comandos, `logcli` (se instala en el puesto o en mon01 y habla con Loki por HTTP):

```bash
export LOKI_ADDR=http://10.10.0.20:3100
logcli labels                                  # qué etiquetas existen
logcli labels service                          # valores de una etiqueta
logcli query --since=5m '{service="app"}' --limit 20
logcli query --since=5m 'count_over_time({service="app"}[5m])'
logcli series '{host="app01"}'                 # streams que tiene Loki de app01: cardinalidad
```

En Grafana, añadís Loki como fuente de datos (`http://loki:3100` si está en el mismo compose que Grafana, o `http://10.10.0.20:3100`) y en Explore (la pantalla de consultas libres de Grafana) elegís la fuente y escribís las mismas consultas. Explore con la fuente Prometheus y la fuente Loki en pantalla partida, el mismo rango de tiempo, es la forma más rápida de correlacionar un pico de latencia con los errores de ese momento.

## Eventos del demonio Docker

El cuarto flujo es el más corto y el que menos gente recoge, y es una pena, porque es el que dice por qué se ha caído un contenedor: si lo mató el kernel por memoria, si terminó con error o si alguien lo paró. En este apartado lo recogemos reutilizando el camino de los logs, con un contenedor auxiliar y sin escribir código.

El demonio emite un evento cada vez que cambia el estado de un contenedor, imagen, volumen o red. Para el mantenimiento los que importan son los de contenedor:

| Evento | Qué significa | Dato útil en `Actor.Attributes` |
|----|----|----|
| `die` | El proceso principal ha terminado | `exitCode` (0 normal, 137 SIGKILL, 143 SIGTERM, 1 error de la app) |
| `oom` | El kernel ha matado un proceso del contenedor por exceder `memory.max` | Siempre seguido de un `die` con 137 |
| `health_status: unhealthy` | El healthcheck ha fallado `retries` veces seguidas | Nombre del contenedor |
| `restart` | Docker lo ha reiniciado por la política `restart` | Cuántas veces, comparando con `start` |
| `kill`, `stop` | Alguien (o compose) lo ha parado | `signal` |

```bash
docker events --since 30m --filter type=container --format '{{json .}}'
docker events --filter event=die --filter event=oom --filter event=health_status
```

No hay una fuente de datos "eventos" en Promtail ni en Prometheus, pero no hace falta: la forma más simple de recogerlos es un contenedor auxiliar que ejecute `docker events` con salida JSON. Su stdout pasa por el driver de logs como cualquier otro y Promtail lo envía a Loki con `service="docker-events"`. Un contenedor, cero código:

```yaml
# en /opt/agentes/compose.yml
  docker-events:
    image: docker:28-cli
    container_name: docker-events
    command: docker events --filter type=container --format '{{json .}}'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    restart: unless-stopped
```

Cada línea es un JSON con `status`, `Action`, `Actor.ID`, `Actor.Attributes.name`, `Actor.Attributes.exitCode`, `Actor.Attributes.com.docker.compose.service`, y `time`. Un `match` en el pipeline de Promtail para `{service="docker-events"}` con `json` extrae `Action` a etiqueta `action` y el nombre del contenedor afectado a structured metadata. En Loki: `{service="docker-events", action=~"die|oom"}`. La alternativa es un exporter que convierta eventos en métricas (hay varios en GitHub, ninguno oficial); tiene la ventaja de poder alertar con PromQL en lugar de LogQL, pero pierdes el detalle (el exitCode, quién lo paró). En UT2 alertaremos desde Loki con el ruler.

Dos avisos. El contenedor auxiliar solo captura eventos mientras está en marcha: si se reinicia el demonio, se pierden los de ese intervalo (poco importa: el `start` de todos los contenedores es lo primero que verás). Y `docker events` desde un contenedor con el socket montado tiene acceso total al demonio; el `:ro` del montaje no limita la API, solo el fichero. En UT3 se restringe con un proxy de socket.

## Enviar las métricas a un sistema remoto: scrape y remote_write

Hasta aquí hemos dado por hecho que las métricas llegan a mon01. Este apartado explica cómo viajan, porque hay dos formas de hacerlo con comportamientos distintos cuando la red falla: en la primera, Prometheus va a buscarlas; en la segunda, se las envían. Tienes que entender las dos para interpretar la prueba de corte de red de la sesión 6.

<figure markdown="span">
  ![Arquitectura de Prometheus](../img/prometheus-arquitectura.svg){ width="640" }
  <figcaption>Arquitectura de Prometheus: descubrimiento, scrape de targets, TSDB local, reglas hacia Alertmanager y consulta desde Grafana. Fuente: Proyecto Prometheus, Apache 2.0.</figcaption>
</figure>

El camino normal es el **scrape**: Prometheus en mon01 tiene una lista de targets y cada `scrape_interval` (15 s en nuestro caso) hace un GET a `/metrics` de cada uno, parsea el texto y guarda cada muestra con la hora del reloj de mon01. El target no sabe nada de Prometheus, solo responde a un GET. Esto tiene consecuencias que conviene tener claras:

```mermaid
sequenceDiagram
    participant P as Prometheus (mon01)
    participant T as API app01 :9102
    loop cada 15 s
        P->>T: GET /metrics (timeout 10 s)
        alt 200 OK
            T-->>P: texto de exposición (app_http_requests_total{...} 18342 ...)
            P->>P: parsea, añade job/instance, guarda con t = ahora
            P->>P: up{instance="10.10.2.10:9102"} = 1
        else sin respuesta
            P->>P: up = 0, scrape_duration_seconds, no hay muestras
            P->>P: marca de "stale" en las series del target
        end
    end
```

Un scrape que falla no se recupera: no hay cola, el target no guarda nada, y en la gráfica queda un hueco. Los contadores lo toleran bien (el siguiente scrape trae el valor acumulado y `rate()` lo reparte), pero un pico de un gauge que ocurrió en esos segundos se pierde para siempre. `up` es la métrica más importante de Prometheus: vale 1 si el último scrape del target tuvo éxito y 0 si no, y es lo primero que se mira en `http://10.10.0.20:9090/targets`.

```yaml
# prometheus.yml en mon01, jobs de la UT1
scrape_configs:
  - job_name: app
    static_configs:
      - targets: ["10.10.2.10:9102"]
        labels: { host: app01, service: app }
  - job_name: postgres
    static_configs:
      - targets: ["10.10.3.10:9187"]
        labels: { host: db01, service: db }
  - job_name: nginx
    static_configs:
      - targets: ["10.10.1.10:9113"]
        labels: { host: web01, service: web }
  - job_name: promtail
    static_configs:
      - targets: ["10.10.2.10:9080"]
        labels: { host: app01 }
  - job_name: loki
    static_configs:
      - targets: ["loki:3100"]
```

Todos los targets están en subredes distintas de la de gestión, así que en OPNsense tienen que existir reglas que permitan desde 10.10.0.20 hacia esos puertos. Esto aplica cuando las VM estén en la VPC; en el entorno provisional del bridge del aula no hay firewall entre ellas y este paso todavía no toca. Si un target sale `context deadline exceeded` en lugar de `connection refused`, casi seguro es el firewall (los paquetes se descartan sin respuesta); `connection refused` es que el puerto no está escuchando en la máquina.

**remote_write** es la alternativa cuando el scrape no puede hacerse: el target está detrás de NAT (una red privada cuyas direcciones mon01 no puede alcanzar directamente), en una red que mon01 no alcanza, o queréis tolerancia a cortes. Se despliega un Prometheus pequeño junto al servicio (en app01, con retención de horas) que hace el scrape en local y **empuja** las muestras al central. Ese Prometheus local guarda un WAL (write-ahead log) en disco y una cola en memoria: si el central no responde, reintenta con backoff y, al volver, envía lo acumulado desde el WAL. Con dos horas de WAL aguanta cortes largos sin perder nada.

```yaml
# prometheus.yml del Prometheus local en app01
global:
  scrape_interval: 15s
  external_labels: { host: app01, env: dev }
scrape_configs:
  - job_name: app
    static_configs: [{ targets: ["app:9102"] }]
  - job_name: cadvisor
    static_configs: [{ targets: ["cadvisor:8080"] }]
remote_write:
  - url: http://10.10.0.20:9090/api/v1/write
    queue_config:
      capacity: 10000
      max_shards: 4
      max_samples_per_send: 2000
      batch_send_deadline: 5s
      min_backoff: 30ms
      max_backoff: 5s
```

El Prometheus central tiene que arrancar con `--web.enable-remote-write-receiver` para aceptar escrituras en `/api/v1/write`. En Prometheus 3 el protocolo de remote write tiene versión 2.0 (más compacto, con metadatos); por defecto se sigue usando la 1.0 y no hace falta tocar nada. Las métricas del emisor que dicen si va bien: `prometheus_remote_storage_samples_pending` (cola), `prometheus_remote_storage_samples_retried_total`, `prometheus_remote_storage_samples_failed_total` (perdidas de verdad) y `prometheus_remote_storage_highest_timestamp_in_seconds` frente a `prometheus_remote_storage_queue_highest_sent_timestamp_seconds` (retraso en segundos).

En el laboratorio la práctica se hace con scrape directo, que es lo que tiene cualquier empresa con una red plana; el remote_write lo montamos en la sesión 6 en paralelo para comparar el comportamiento ante el corte. Es el mismo mecanismo que usan los servicios gestionados (Grafana Cloud, Amazon Managed Prometheus, Thanos Receive): un Prometheus local que escribe hacia fuera.

## Verificar la integración

Integrar no es configurar y olvidarse. Cada punto de la cadena puede fallar en silencio: el firewall descarta, Promtail lee pero no envía, Loki recibe pero rechaza por reloj, Prometheus guarda pero la retención borra. Verificar es ejecutar un comando en cada punto y saber qué significa lo que sale. Esta tabla es la que hay que completar entera en la sesión 6 y la que sale en la práctica.

| Comprobación | Comando | Lo esperado | Si no cuadra |
|----|----|----|----|
| Comunicación app01 → mon01 (logs) | Desde app01: `nc -zv 10.10.0.20 3100` | `succeeded` | Firewall (regla back → gestión 3100) o Loki parado (`docker compose ps` en mon01) |
| Comunicación mon01 → app01 (métricas) | Desde mon01: `curl -s http://10.10.2.10:9102/metrics \| head -5` y lo mismo a `:8081` y `:9080` | Texto `# HELP ...` | `Connection refused`: no escucha o no está publicado; se queda colgado: firewall |
| Recepción de métricas | `http://10.10.0.20:9090/targets` o `curl -s localhost:9090/api/v1/targets \| jq '.data.activeTargets[] \| {job, health, lastError}'` | Todos `up` | `lastError` dice el motivo; `scrape_samples_scraped` en PromQL dice cuántas series trae cada target |
| Recepción de logs | `logcli query --since=2m '{host="app01"}' --limit 5` | Líneas recientes con `container` y `service` | Vacío: mirar `docker logs promtail` (errores 4xx de Loki) y `promtail_targets_active_total` |
| Integridad de logs | En app01: `docker logs --since 5m servicio-app-1 \| wc -l`; en Loki en el mismo momento: `count_over_time({container="servicio-app-1"}[5m])` | Iguales o diferencia de un puñado por el desfase de segundos entre los dos comandos | Loki muy por debajo: pérdidas (`promtail_dropped_entries_total`) o rechazos (`loki_discarded_samples_total` por motivo); Loki por encima: duplicados por posiciones perdidas |
| Integridad de métricas | En app01: `curl -s localhost:9102/metrics \| grep 'app_http_requests_total{method="GET",route="/health",status="200"}'`; en Prometheus, la misma serie | El valor de Prometheus es el del último scrape (hasta 15 s más viejo), nunca mayor | Mayor o muy distinto: estáis mirando otra instancia o etiquetas cambiadas por relabel |
| Marcas de tiempo | En cada host: `chronyc tracking \| grep 'System time'` | Desfase inferior a 100 ms | Ver la sección de chrony; Loki y las correlaciones en Grafana lo notan a partir de segundos |
| Almacenamiento Prometheus | `prometheus_tsdb_head_series`, `prometheus_tsdb_storage_blocks_bytes`, y consultar cualquier serie con `offset 24h` | Hay datos de hace 24 h; series estables (no crecen sin parar) | Series crecen: cardinalidad (etiqueta con demasiados valores); sin datos antiguos: retención demasiado corta o el volumen no persiste |
| Almacenamiento Loki | `logcli query --since=24h --limit 1 '{host="app01"}'` y `du -sh /var/lib/docker/volumes/monitoring_loki/_data` en mon01 | Hay líneas de ayer; el tamaño crece y se estabiliza al llegar a la retención | Sin líneas: el compactor ha borrado o el volumen no persiste |
| Pérdidas | `promtail_dropped_entries_total`, `loki_discarded_samples_total`, `prometheus_remote_storage_samples_failed_total`, y `count_over_time({service="app"}[1h])` antes y después de un corte | Todos a cero, o su crecimiento coincide con un corte documentado | Suben sin corte: revisar límites de ingestión de Loki (`ingestion_rate_mb`) y reloj |

Un detalle de la integridad de logs: `docker logs --since 5m` y la consulta de Loki no se ejecutan en el mismo instante ni miden exactamente la misma ventana. Para una prueba limpia, generad un número conocido de líneas (`for i in $(seq 1 500); do curl -s -o /dev/null http://10.10.1.10/health; done` desde el puesto) con la API en silencio, esperad diez segundos, y contad en las dos partes con una ventana holgada. Deben salir 500 y 500. Si la API escribe dos líneas por petición (una del middleware y otra del handler), 1000 y 1000.

### Relojes: chrony

Prometheus pone la hora a las muestras con el reloj de mon01, así que las métricas solo dependen de un reloj. Los logs no: la hora la pone la aplicación (o Docker, si la aplicación no la escribe) con el reloj del host de app01, y Loki la compara con el suyo. Los contenedores no tienen reloj propio, usan el del kernel del host, así que basta con sincronizar las VM. Con un desfase de 30 s la correlación en Grafana entre un pico de latencia y sus errores queda desplazada y confunde; con más de `reject_old_samples_max_age` Loki rechaza las líneas; y con la hora en el futuro más de 10 minutos también (`creation_grace_period`).

Debian 13 trae `systemd-timesyncd`, que vale para un portátil pero no da diagnóstico. Instalad chrony en las cuatro VM y en mon01 y apuntadlo al OPNsense (que a su vez sincroniza con Internet) o a `pool.ntp.org` si el firewall deja salir UDP 123 (en el entorno provisional, sin OPNsense, dejad solo la línea `pool`):

```bash
sudo apt install chrony
sudo tee /etc/chrony/sources.d/lab.sources <<EOT
server 10.10.0.1 iburst prefer
pool 2.debian.pool.ntp.org iburst
EOT
sudo systemctl restart chrony
chronyc sources -v      # ^* delante de la fuente elegida
chronyc tracking        # System time : 0.000213 seconds fast of NTP time
```

Las VM en Proxmox pierden tiempo al suspenderse o al migrarse; chrony por defecto solo corrige a saltos (`makestep`) en los tres primeros ajustes tras arrancar, y después ajusta la velocidad del reloj poco a poco (un desfase de 10 s tarda horas en corregirse). Para el laboratorio, donde apagáis y encendéis VM constantemente, añadid `makestep 1 -1` en `/etc/chrony/conf.d/lab.conf`: corrige de golpe siempre que el desfase pase de 1 s. En producción se tiene apagado por lo que hace a las bases de datos un salto de reloj hacia atrás.

### Qué pasa cuando se pierde la red

Es la prueba de la sesión 6 y de la práctica. En app01, con nftables (el cortafuegos del kernel de Linux, que se ve a fondo en 5166 UT3), cortamos todo el tráfico con mon01 durante tres minutos:

```bash
sudo nft add table inet corte
sudo nft add chain inet corte out '{ type filter hook output priority 0; }'
sudo nft add chain inet corte in '{ type filter hook input priority 0; }'
sudo nft add rule inet corte out ip daddr 10.10.0.20 drop
sudo nft add rule inet corte in ip saddr 10.10.0.20 drop
date; sleep 180; date
sudo nft delete table inet corte
```

Lo que tenéis que observar y documentar, con capturas:

- **Métricas por scrape**: a los 15 s, en Targets los tres de app01 pasan a DOWN con `context deadline exceeded`; `up{host="app01"}` vale 0. Al restaurar, vuelven a UP en el siguiente scrape. En Grafana queda un hueco de tres minutos en todas las gráficas de app01. **No se recupera**: esos datos no existen en ningún sitio. Los contadores de la API han seguido subiendo, así que `increase(app_http_requests_total[10m])` sobre un rango que incluye el corte da el total correcto, solo falta el detalle de esos minutos.
- **Métricas por remote_write** (si habéis montado el Prometheus local): `prometheus_remote_storage_samples_pending` sube durante el corte, `samples_retried_total` cuenta reintentos, `samples_failed_total` se queda a cero. Al restaurar, la cola se vacía en segundos y en el central aparecen las muestras con su hora original. **Se recupera** sin hueco.
- **Logs**: `docker logs promtail` muestra los reintentos (`error sending batch, will retry`, con el tiempo de espera creciente). `promtail_sent_entries_total` deja de crecer, `promtail_dropped_entries_total` sigue a cero. Al restaurar, en menos de un minuto (el siguiente reintento) llegan a Loki todas las líneas del corte con sus marcas de tiempo reales. Comprobadlo con `count_over_time` sobre la ventana del corte. **Se recupera** mientras el corte dure menos que la suma de reintentos y menos de lo que aguanten los ficheros de Docker.
- **Eventos**: igual que los logs, porque van por el mismo camino.

Repetid la prueba con un corte de 12 minutos, o bajando `max_retries` a 3, y ya veréis pérdidas: `promtail_dropped_entries_total` sube y el conteo en Loki queda por debajo del de `docker logs`. Ese es el número que hay que poner en el informe: cuánto aguanta vuestra configuración.

## Errores frecuentes en el laboratorio

**cAdvisor arranca pero no muestra contenedores, o todas las series llevan `name=""`.** Le falta el montaje de `/var/run` (no encuentra el socket para relacionar cgroups con contenedores) o `/sys`. Con cgroups v2 además necesita `privileged`. Revisad `docker logs cadvisor`: suele decir `Failed to start container manager` o `unable to get docker info`.

**Prometheus hace scrape del puerto 8080 de app01 y trae las métricas de la API en vez de cAdvisor, o un 404.** El puerto 8080 de app01 es la API; cAdvisor está en 8081. Confusión clásica con la tabla de puertos del laboratorio.

**Los valores de `app_http_requests_total` bajan y suben entre scrapes.** Gunicorn o uvicorn con varios workers y cada uno con su registro. Activar el modo multiproceso de prometheus_client o servir `/metrics` desde un único proceso.

**Promtail: `permission denied` al abrir `/var/run/docker.sock`.** El contenedor de Promtail corre con un usuario sin acceso al socket. Añadid `user: root` en el compose (aceptable en un agente de solo lectura) o `group_add` con el gid del grupo `docker` del host (`getent group docker`).

**Loki responde 400 `entry too far behind` o `timestamp too old`.** La primera: dentro de un mismo stream llegan líneas más antiguas que las ya aceptadas fuera de la ventana de desorden (2 h por defecto); suele ser un reenvío tras perder el fichero de posiciones. La segunda: la línea tiene más de `reject_old_samples_max_age`, normalmente por un reloj atrasado en app01 o por reenviar logs de hace días tras cambiar de configuración. `loki_discarded_samples_total` lleva la cuenta por motivo.

**Loki responde 429 `Ingestion rate limit exceeded` o `Maximum active stream limit exceeded`.** Estáis enviando más de `ingestion_rate_mb` (subirlo si el disco lo permite, o bajar el nivel de log de la API), o habéis metido en las etiquetas algo de cardinalidad alta (`request_id`, la ruta). `logcli series '{host="app01"}' | wc -l` dice cuántos streams tenéis; con el servicio del curso deberían ser unas decenas, no miles.

**Los conteos de Loki salen el doble que los de `docker logs`.** Promtail ha reenviado todo tras un reinicio porque el fichero de posiciones estaba en `/tmp` dentro del contenedor. Volumen persistente para `/var/lib/promtail`.

**He puesto `max-size` en daemon.json y el fichero json del contenedor sigue en 4 GB.** La opción solo se aplica a contenedores creados después del cambio. `docker compose up -d --force-recreate` y comprobad con `docker inspect -f '{{.HostConfig.LogConfig}}' servicio-app-1`.

**`docker logs` no muestra nada de la API.** La aplicación escribe a un fichero dentro del contenedor o el framework tiene el log por defecto a nivel WARNING. Todo a stdout y nivel INFO; en UT5 se afina.

**En Grafana, Explore con Loki dice "no logs found" pero `logcli` en mon01 sí devuelve.** Rango de tiempo del navegador (últimos 5 minutos con un desfase de reloj de tu portátil) o fuente de datos configurada con `localhost:3100` desde el contenedor de Grafana, donde `localhost` es el propio Grafana. Usad el nombre del servicio de compose (`http://loki:3100`).

**Targets en DOWN con `context deadline exceeded` justo después de tocar OPNsense.** Regla de firewall sin la dirección o el puerto correcto, o creada en la interfaz equivocada. Comprobad con `nc -zv` desde mon01 y mirad el live log del firewall filtrando por la IP de mon01.

**Las gráficas de CPU de cAdvisor muestran valores absurdos (4000 %).** Estáis usando `container_cpu_usage_seconds_total` sin `rate()`, o `rate()` con una ventana menor que el intervalo de scrape.

## Actividades

### A1.1 El contenedor de referencia (sesión 1)

Despliega el servicio del curso en app01 con compose (si no lo tienes de la 5166, el de clase). Abre tres terminales en app01 con `docker stats`, `docker logs -f servicio-app-1` y `docker events --format '{{json .}}'`, y desde tu puesto genera tráfico con un bucle de `curl` contra web01. Para la base de datos (`docker compose stop db` en db01) y anota qué ves en cada terminal y con cuántos segundos de diferencia. Localiza en `/sys/fs/cgroup` el cgroup del contenedor de la API y compara `memory.current` y `memory.stat` con la columna MEM de `docker stats`. Entrega: una tabla con los tres flujos y lo que vio cada uno durante la caída.

### A1.2 Métricas de recursos (sesión 2)

Despliega cAdvisor en app01 con el compose de agentes. Configura el job en Prometheus con el relabel que deja `service`. En Grafana, un panel de CPU (núcleos) y otro de working set por servicio de compose, con la memoria comparada contra el límite. Localiza en `/metrics` de cAdvisor la etiqueta que identifica cada servicio y comprueba con `count(count by (__name__)({job="cadvisor"}))` cuántas métricas distintas entran antes y después del `keep`. Entrega: captura de los paneles y el número de series antes y después.

### A1.3 Instrumentar la aplicación (sesión 3)

Añade a la API un endpoint `/metrics` en el puerto 9102 con un counter de peticiones por método, ruta y estado y un histogram de latencia con buckets adecuados. Añade postgres_exporter en db01 con un usuario `pg_monitor` y una consulta personalizada sobre una tabla del servicio. Ambos como targets en Prometheus en UP. Calcula en PromQL el p95 de latencia (`histogram_quantile(0.95, sum by (le) (rate(app_http_request_duration_seconds_bucket[5m])))`) y el porcentaje de respuestas 5xx. Entrega: el código de la instrumentación, `prometheus.yml` y las dos consultas con su resultado.

### A1.4 Logs estructurados (sesión 4)

Configura la API para escribir logs JSON con `ts`, `level`, `msg` y `request_id` recibido de nginx por `X-Request-ID`. Pon el log de acceso de nginx en JSON con el mismo `request_id`. Limita el driver `json-file` en daemon.json de las tres VM y recrea los contenedores. Despliega Promtail en app01 (y en web01 leyendo `/var/log/nginx/access.json` con `static_configs`) y comprueba en Grafana → Explore que llegan con `host`, `container` y `service`. Sigue una petición concreta por su `request_id` en los dos hosts. Entrega: `promtail.yml`, una captura de la petición correlada y el `docker inspect` que muestra los límites del driver.

### A1.5 Eventos (sesión 5)

Añade el contenedor `docker-events` al compose de agentes y el `match` en Promtail que extrae `action`. Provoca tres eventos en la API: `die` (`docker kill -s SIGKILL`), `oom` (límite de memoria de 32 MB en el compose y una petición que cargue datos) y `health_status: unhealthy` (rompe el healthcheck cambiando la ruta). Localiza los tres en Loki con una consulta cada uno y apunta el `exitCode` de cada `die`. Entrega: las tres consultas y sus resultados.

### A1.6 Integridad y almacenamiento (sesión 6)

Ejecuta la tabla de verificación completa y anota el resultado de cada fila con el comando exacto y su salida. Instala chrony en todas las VM y documenta `chronyc tracking`. Corta la red entre app01 y mon01 durante 3 minutos con nftables y observa qué pasa con las métricas por scrape, los logs y los eventos al restaurarla: qué se recupera, qué se pierde, en cuánto tiempo. Repite con 12 minutos. Si te da tiempo, monta el Prometheus local con remote_write y compara. Entrega: la tabla rellena y las gráficas del corte.

## Práctica evaluable

**Sesión 7.** Entrega un informe (máximo 5 páginas, PDF, en `tests/evidence/ut1/` del repositorio `servicio`) sobre el contenedor de referencia integrado en mon01. Tiene que contener:

- [ ] Esquema del flujo de datos (métricas, logs, eventos) desde app01, db01 y web01 hasta mon01, con puertos y sentido de cada flujo.
- [ ] Configuración: el compose de agentes, `promtail.yml`, los jobs de `prometheus.yml`, `daemon.json` y el fragmento de la instrumentación de la API.
- [ ] Pruebas de comunicación, recepción, integridad y almacenamiento con evidencias (comandos y salidas, capturas de Targets y de Explore), siguiendo la tabla de verificación.
- [ ] Resultado de la prueba de corte de red de 3 y de 12 minutos: qué se recupera, qué se pierde, qué métrica lo demuestra.
- [ ] Salida de `chronyc tracking` de las cinco VM.

| Criterio (RA1 a) | Peso |
|----|----|
| Métricas de recursos y de aplicación llegan a Prometheus | 30 % |
| Logs y eventos llegan a Loki con etiquetas correctas | 30 % |
| Integridad de datos comprobada (conteos coinciden, relojes sincronizados) | 20 % |
| Almacenamiento remoto verificado y comportamiento ante pérdida de red documentado | 20 % |

Se valora que el informe se pueda reproducir: quien lo lea con acceso al laboratorio tiene que poder ejecutar los mismos comandos y obtener lo mismo. Las capturas sin el comando que las generó no cuentan como evidencia.

## Para ampliar

- [cgroups v2, man 7 cgroups](https://man7.org/linux/man-pages/man7/cgroups.7.html): referencia del kernel con todos los ficheros de `memory.*`, `cpu.*` e `io.*` que lee cAdvisor.
- [Documentación de cgroup v2 del kernel](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html): explica `memory.stat` campo a campo, incluido `inactive_file` y por qué el working set se calcula así.
- [Prometheus metrics for cAdvisor](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md): lista completa de métricas `container_*` con su tipo y qué flag las activa.
- [Metric and label naming](https://prometheus.io/docs/practices/naming/) e [Instrumentation](https://prometheus.io/docs/practices/instrumentation/): las convenciones que hemos seguido y cuándo usar cada tipo de métrica.
- [prometheus_client para Python](https://prometheus.github.io/client_python/) y [prom-client para Node](https://github.com/siimon/prom-client): documentación oficial de las dos librerías, con el modo multiproceso y las métricas por defecto.
- [postgres_exporter](https://github.com/prometheus-community/postgres_exporter): variables de entorno, colectores disponibles y el formato de las consultas personalizadas.
- [Configure logging drivers, Docker docs](https://docs.docker.com/engine/logging/configure/): todos los drivers, sus opciones y el dual logging.
- [Promtail configuration](https://grafana.com/docs/loki/latest/send-data/promtail/configuration/): referencia de `docker_sd_configs`, `relabel_configs`, cada etapa del pipeline y `backoff_config`.
- [Loki configuration](https://grafana.com/docs/loki/latest/configure/) y [LogQL](https://grafana.com/docs/loki/latest/query/): referencia del fichero de Loki (retención, límites) y del lenguaje de consulta.
- [Prometheus remote write](https://prometheus.io/docs/practices/remote_write/): cómo dimensionar la cola y qué significan sus métricas.
- [chrony, documentación oficial](https://chrony-project.org/documentation.html): `chronyc tracking`, `sources` y `makestep` explicados.
