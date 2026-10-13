# UT1 · Observabilidad de contenedores: métricas, logs y eventos

<p class="ut-meta">14 h · Sesiones 1 a 7 · RA1 CE a</p>

Esta es la primera unidad del módulo y trabaja sobre el servicio del curso, que se entrega el primer día en Aules como repositorio `servicio`: una API en Python con Flask, servida con gunicorn, detrás de nginx y con PostgreSQL, todo en un solo Docker Compose. El laboratorio llega por fases, y esta unidad se hace sobre un entorno provisional: hasta el 13 de octubre no existe ninguna VM y todo corre con Docker en el puesto del alumno; el 14 de octubre la asignatura de despliegue clona en la red del aula app01 (el servicio) y mon01 (la pila de monitorización), y las dos no pasan a la VPC dev, detrás del cortafuegos, hasta el 27 de noviembre. La pila de monitorización que se monta aquí, a partir de la mínima que se da en clase, es la de todo el curso: la [UT7 de Despliegue](https://victor-educ.github.io/apuntes-5166/ut/ut7-monitorizacion/), en marzo, trabaja sobre ella. Aquí no se despliega nada nuevo del servicio: se trata de sacar de él todo lo que cuenta sobre su estado (métricas de recursos, métricas de aplicación, logs y eventos), llevarlo al sistema de monitorización y demostrar con pruebas que llega entero, a tiempo y se guarda. Ese "demostrar" es la mitad de la unidad y el criterio de evaluación lo dice explícitamente: integrar y verificar comunicación, integridad y almacenamiento. Sin datos fiables, la UT2 (umbrales y alarmas) no tiene sobre qué trabajar.

Al servicio del curso, esté en el puesto o en app01, se le llama durante todo el módulo el **contenedor de referencia**. Se mantiene vivo todo el curso, y en la UT8 se da de baja de forma controlada su copia del entorno pre.

## Sesión 1 · Presentación y el contenedor de referencia

<p class="ut-meta" markdown>1 de octubre · Teoría y práctica · <span class="dur" tabindex="0" aria-label="Presentación de la asignatura · 25 min&#10;Qué expone un contenedor · 20 min&#10;A1.1 Presentación y el contenedor de referencia · 65 min" data-dur="Presentación de la asignatura · 25 min&#10;Qué expone un contenedor · 20 min&#10;A1.1 Presentación y el contenedor de referencia · 65 min">:material-school:<i class="dur-barra" style="--teoria:41%"></i>:material-flask:</span></p>

Al acabar, el servicio del curso queda en marcha y se ha visto en tres terminales cómo reaccionan métricas, logs y eventos cuando se cae la base de datos. Antes van la presentación de la asignatura y el mapa de los cuatro flujos que salen de un contenedor, que es lo único de teoría que necesita la hoja: sin ese mapa, lo que aparece en las terminales son piezas sueltas.

### Qué expone un contenedor

Aquí no se monta nada todavía: primero el mapa completo. Este apartado recoge qué cuatro cosas cuenta un contenedor de sí mismo, quién produce cada una, cómo se mira en la propia máquina y a qué sistema de mon01 llega.

Un contenedor en ejecución no es más que un proceso (o varios) con namespaces y cgroups (los dos mecanismos del kernel que lo aíslan del resto y le contabilizan el consumo). Todo lo que interesa saber de él sale por uno de estos cuatro caminos:

| Flujo | Quién lo produce | Cómo se lee en local | A dónde va |
|----|----|----|----|
| Métricas de recursos | El kernel, vía cgroups | `docker stats`, `/sys/fs/cgroup`, cAdvisor | Prometheus |
| Métricas de aplicación | La propia aplicación, en `/metrics` (formato de exposición de Prometheus) | `curl app01:9102/metrics` | Prometheus |
| Logs | stdout y stderr del proceso principal, capturados por el driver de logs de Docker | `docker logs -f app` | Loki (vía Promtail) |
| Eventos | El demonio Docker: start, die, oom, health_status... | `docker events` | Loki (vía un contenedor auxiliar y Promtail) |

Los dos primeros son series numéricas: se muestrean cada pocos segundos y se guardan comprimidas en una base de datos de series temporales. Los otros dos son texto con marca de tiempo: no se muestrean, se recogen línea a línea. Por eso hay dos sistemas de almacenamiento (Prometheus y Loki) y no uno. Grafana solo pinta lo que hay en ambos.

```mermaid
flowchart LR
    subgraph app01["Host del servicio · el puesto hasta el 13 oct, app01 después"]
        api["<b>API</b><br><small>/metrics :9102 · stdout JSON</small>"]:::pieza
        cad["<b>cAdvisor</b><br><small>:8081</small>"]:::pieza
        pgx["<b>postgres_exporter</b><br><small>:9187</small>"]:::pieza
        ev["<b>docker-events</b><br><small>contenedor auxiliar</small>"]:::pieza
        pt["<b>Promtail</b><br><small>:9080</small>"]:::pieza
        dk[("<b>dockerd</b><br><small>json-file</small>")]:::infra
        cg[("<b>cgroups v2</b>")]:::infra
        cg --> cad
        api --> dk
        ev --> dk
        dk --> pt
    end
    subgraph mon01["Monitorización · el puesto hasta el 13 oct, mon01 después"]
        prom["<b>Prometheus</b><br><small>:9090</small>"]:::act
        loki["<b>Loki</b><br><small>:3100</small>"]:::act
        graf["<b>Grafana</b><br><small>:3000</small>"]:::dato
        prom --> graf
        loki --> graf
    end
    prom -.->|scrape| api
    prom -.->|scrape| cad
    prom -.->|scrape| pgx
    pt -->|push HTTP| loki
    classDef act fill:#ea580c22,stroke:#ea580c,stroke-width:1.5px
    classDef pieza fill:#64748b22,stroke:#64748b,stroke-width:1.5px
    classDef dato fill:#2563eb22,stroke:#2563eb,stroke-width:1.5px
    classDef infra fill:#a1a1aa14,stroke:#a1a1aa,stroke-width:1.5px
    classDef ok fill:#16a34a22,stroke:#16a34a,stroke-width:1.5px
    classDef riesgo fill:#dc262622,stroke:#dc2626,stroke-width:1.5px
```

<p class="pie" markdown>Las métricas las va a buscar Prometheus; los logs los empuja Promtail. Esa diferencia de sentido es la que decide las reglas del cortafuegos en la UT3.</p>

El esquema es el mismo antes y después del 14 de octubre; lo único que cambia es dónde está cada caja. Hasta el 13 de octubre las dos están en el mismo host, el puesto del alumno, y las flechas son llamadas a `localhost` o al nombre del servicio dentro de la red de compose; desde la sesión 4 la de arriba es app01 y la de abajo mon01, y las flechas cruzan la red del aula.

Conviene fijarse en la dirección de las flechas: las métricas las **tira** Prometheus (pull), los logs los **empuja** Promtail (push). Esa diferencia lo condiciona todo cuando hay un corte de red y se comprueba al final de la unidad.

### A1.1 Presentación y el contenedor de referencia (sesión 1)

Los primeros 25 minutos son la presentación de la asignatura (evaluación, laboratorio compartido con Despliegue y calendario); después, 20 minutos para el mapa de los cuatro flujos, y el resto es esta hoja.

<span class="et et-obj">Objetivo</span> El servicio del curso corriendo con compose y, en tres terminales a la vez, cómo reaccionan métricas, logs y eventos cuando se cae la base de datos.

<span class="et et-pre">Antes de empezar</span>

- Docker Engine sobre Linux en tu puesto. Hoy no existe ninguna VM del laboratorio: todo se hace en localhost. Si el puesto es Windows o macOS, trabaja en una VM Debian en VirtualBox, porque la sesión 2 lee ficheros de cgroups que solo existen en un host Linux.
- De Aules, el servicio del curso (la API Flask en `api/` y un `compose.yml` con `nginx`, `app` y `db`) y la pila mínima de monitorización (Prometheus, Alertmanager y Grafana), que hoy no se usa.
- Lo explicado antes: [Qué expone un contenedor](#que-expone-un-contenedor).

<span class="et et-pas">Pasos</span>

1. Deja el servicio en `/opt/servicio` y la pila de monitorización en `/opt/monitoring`, cada una como repositorio Git con un commit de partida (así nacen hoy `servicio` y `monitoring`; el remoto llega en la A2.6), y crea `/opt/agentes` vacía para la sesión 2. Despliega el servicio y comprueba que responde en `http://localhost/health`.

    *Pista:* las contraseñas van en un `.env` copiado de `.env.example`, que no se sube al repositorio. Cámbialas **antes** del primer `up`: PostgreSQL fija la suya la primera vez que arranca.

2. Deja a la vista, en tres terminales, el consumo de los contenedores, los logs de la API y los eventos del demonio, con las herramientas de la columna «Cómo se lee en local» de [Qué expone un contenedor](#que-expone-un-contenedor). Desde un cuarto terminal, genera tráfico continuo contra `/health` y observa qué cambia en cada uno.

    *Pista:* `curl -s -o /dev/null -w '%{http_code}\n'` imprime solo el código de respuesta. Los contenedores se llaman `servicio-app-1`, `servicio-db-1` y `servicio-nginx-1`.

3. Para la base de datos y cronometra qué aparece en cada terminal y cuántos segundos tarda desde la parada. Después arráncala y observa la recuperación.

<span class="et et-com">Comprobación</span> `git log` tiene un commit en `/opt/servicio` y otro en `/opt/monitoring`; el bucle de `curl` devuelve 200 con la base de datos arriba y 500 (o 503) con ella parada; en el terminal de eventos aparece el `die` de `db` con `exitCode` 0 y después el `start`.

<span class="et et-ent">Entrega</span> Una tabla con tres filas (métricas, logs, eventos) y tres columnas (qué se vio al parar la BD, cuántos segundos tardó, qué se vio al arrancarla), en `tests/evidence/ut1/a1.1.md` del repositorio `servicio`.

<span class="et et-ext">Si te sobra tiempo</span> Repite el bucle con `/items`, que consulta la base de datos: ¿la API devuelve error o se queda colgada?

## Sesión 2 · Métricas de recursos

<p class="ut-meta" markdown>6 de octubre · Teoría y práctica · <span class="dur" tabindex="0" aria-label="Métricas de recursos: cgroups v2 y cAdvisor · 20 min&#10;A1.2 Métricas de recursos · 90 min" data-dur="Métricas de recursos: cgroups v2 y cAdvisor · 20 min&#10;A1.2 Métricas de recursos · 90 min">:material-school:<i class="dur-barra" style="--teoria:18%"></i>:material-flask:</span></p>

Al terminar, cAdvisor publica las métricas de recursos del host donde corre el servicio (hoy, el puesto; desde el 15 de octubre, app01), Prometheus las recoge con la etiqueta `service` limpia y Grafana pinta CPU y memoria por servicio. La hoja usa los cuatro apartados de abajo: los dos primeros para entender los números, el compose de cAdvisor para desplegarlo y las etiquetas y la cardinalidad para el scrape.

### Métricas de recursos: cgroups v2 y cAdvisor

Aquí se saca el primer flujo, el más sencillo porque ya existe: el kernel lleva la cuenta de lo que consume cada contenedor y solo hay que leerla. Primero se lee a mano para saber qué significa cada número, después se publica con cAdvisor para que Prometheus la recoja, y al final quedan las consultas en PromQL (el lenguaje de consulta de Prometheus) que irán al panel de Grafana. Lo que más problemas da es qué etiqueta identifica a cada contenedor y cuántas series se generan.

#### De dónde salen los números

```mermaid
flowchart TB
    K["<b>Kernel · cgroups v2</b><br><small>/sys/fs/cgroup/…/docker-&lt;id&gt;.scope<br>memory.current · cpu.stat · io.stat</small>"]:::infra
    C["<b>cAdvisor</b><br><small>lee esos ficheros cada pocos segundos</small>"]:::pieza
    E["<b>/metrics</b><br><small>texto plano en formato Prometheus</small>"]:::dato
    P["<b>Prometheus</b><br><small>hace scrape y lo guarda con su marca de tiempo</small>"]:::act
    G(["<b>Grafana</b><br><small>la gráfica que se consulta</small>"]):::ok
    K --> C --> E --> P --> G
    classDef act fill:#ea580c22,stroke:#ea580c,stroke-width:1.5px
    classDef pieza fill:#64748b22,stroke:#64748b,stroke-width:1.5px
    classDef dato fill:#2563eb22,stroke:#2563eb,stroke-width:1.5px
    classDef infra fill:#a1a1aa14,stroke:#a1a1aa,stroke-width:1.5px
    classDef ok fill:#16a34a22,stroke:#16a34a,stroke-width:1.5px
    classDef riesgo fill:#dc262622,stroke:#dc2626,stroke-width:1.5px
```

<p class="pie" markdown>Nadie inventa los números: el kernel ya los está contando para aplicar los límites. cAdvisor solo los lee y los publica.</p>

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

Con cgroups v1 (lo que todavía se encuentra en hosts viejos y en alguna documentación) la jerarquía estaba partida por controlador (`/sys/fs/cgroup/memory/docker/<id>/memory.usage_in_bytes`, `/sys/fs/cgroup/cpu/...`). En v2 hay un único árbol y los ficheros se llaman distinto. Cuando cAdvisor da un número raro, esto es la fuente de verdad.

`docker stats` lee exactamente estos ficheros cada segundo y los formatea. cAdvisor hace lo mismo pero para todos los contenedores a la vez, guarda un histórico corto en memoria y lo expone en `/metrics` en formato Prometheus. No hay magia: las métricas `container_*` son estos ficheros con etiquetas.

#### Memory usage frente a working set

Es la confusión más habitual con la memoria de un contenedor. `memory.current` (lo que cAdvisor expone como `container_memory_usage_bytes`) incluye la caché de páginas: si el contenedor lee un fichero de 200 MB, el kernel se queda esas páginas en caché y cuentan como memoria del cgroup aunque el proceso no las necesite y el kernel pueda soltarlas al instante. Por eso `usage` sube y sube en un contenedor que escribe logs o lee de disco, y la gente cree que tiene una fuga.

El **working set** es `memory.current` menos `inactive_file` de `memory.stat`: la memoria que el kernel no podría reclamar sin dolor. cAdvisor lo expone como `container_memory_working_set_bytes` y es la métrica que usa el propio OOM killer (el mecanismo del kernel que mata procesos cuando se agota la memoria) para decidir si un contenedor ha superado `memory.max`. Conclusión práctica: para alarmas y para comparar contra el límite, working set; `usage_bytes` solo para entender de qué está hecha la memoria. Para "memoria que realmente ha pedido el proceso" está además `container_memory_rss` (anon en v2).

```mermaid
flowchart TB
    subgraph U["container_memory_usage_bytes · memory.current"]
        direction TB
        RSS["<b>Memoria del proceso</b><br><small>anon / rss</small>"]:::dato
        CACHE["<b>Caché de páginas</b><br><small>ficheros leídos o escritos<br>el kernel la suelta sin dolor</small>"]:::infra
    end
    WS["<b>working set</b><br><small>usage − inactive_file<br>lo que el kernel NO puede reclamar</small>"]:::ok
    OOM["<b>Es lo que mira el OOM killer</b><br><small>para decidir si se ha pasado de memory.max</small>"]:::riesgo
    U -- "quitando la caché reclamable" --> WS --> OOM
    classDef act fill:#ea580c22,stroke:#ea580c,stroke-width:1.5px
    classDef pieza fill:#64748b22,stroke:#64748b,stroke-width:1.5px
    classDef dato fill:#2563eb22,stroke:#2563eb,stroke-width:1.5px
    classDef infra fill:#a1a1aa14,stroke:#a1a1aa,stroke-width:1.5px
    classDef ok fill:#16a34a22,stroke:#16a34a,stroke-width:1.5px
    classDef riesgo fill:#dc262622,stroke:#dc2626,stroke-width:1.5px
```

<p class="pie" markdown>Por eso `usage_bytes` sube y sube en un contenedor que escribe logs y parece una fuga que no existe. Para alarmas y para comparar contra el límite, **working set**.</p>


```promql
# porcentaje del límite que usa cada servicio de compose
container_memory_working_set_bytes{container_label_com_docker_compose_service!=""}
  / container_spec_memory_limit_bytes * 100
```

Si un contenedor no tiene límite, `container_spec_memory_limit_bytes` vale un número enorme y la división da casi cero. Es una pista, no un bug: hay que ponerle límite.

#### cAdvisor en compose

cAdvisor necesita ver el árbol de cgroups del host, el socket de Docker (`/var/run/docker.sock`, el fichero por el que se habla con el demonio; lo necesita para saber qué cgroup es qué contenedor y leer sus etiquetas) y el directorio de Docker (para métricas de disco). Se despliega en un compose aparte de agentes, en `/opt/agentes/compose.yml`, para que reiniciar el servicio no tumbe a los agentes. Es el fichero al que se van añadiendo Promtail y el recolector de eventos en las sesiones siguientes. El fichero es el mismo en el puesto del alumno (octubre) y en app01 (desde el 14 de octubre): lo que cambia es la máquina donde se ejecuta.

```yaml
# /opt/agentes/compose.yml, en el host del servicio
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

Cosas a saber de este fichero. `privileged` y `/dev/kmsg` hacen falta para que lea eventos OOM del kernel; sin ellos funciona pero `container_oom_events_total` nunca sube. `--docker_only` quita del listado los cgroups de systemd que no son contenedores (servicios del host), que no aportan nada y multiplican las series. `--housekeeping_interval=10s` baja el consumo de CPU del propio cAdvisor (por defecto muestrea cada segundo, y cAdvisor comiéndose el 5 % de la CPU del host es un clásico). El puerto: cAdvisor escucha en 8080 dentro del contenedor, pero en el host ese puerto suele estar ocupado por la API, así que se publica en 8081. Dentro de una red de compose se le sigue llamando por su nombre de servicio y su puerto interno, `cadvisor:8080`.

#### Etiquetas y cardinalidad

Cada serie de cAdvisor lleva `id` (la ruta del cgroup), `name` (nombre del contenedor, por ejemplo `servicio-app-1`) e `image`. Con `--store_container_labels` cAdvisor convierte todas las etiquetas Docker del contenedor en etiquetas `container_label_<clave>` con los puntos cambiados por guiones bajos. Compose pone bastantes etiquetas (proyecto, servicio, número de réplica, hash de la configuración, ruta del fichero...), y esa última, `com.docker.compose.config-hash`, cambia en cada `compose up` que modifique el servicio. Si se deja pasar, cada redespliegue crea series nuevas en Prometheus y las viejas quedan huérfanas: eso es **cardinalidad**, y es el enemigo número uno de una base de datos de series. Por eso el compose de arriba desactiva todas y solo deja pasar `com.docker.compose.service` y `com.docker.compose.project`.

```mermaid
flowchart TB
    subgraph MAL["Dejando pasar config-hash"]
        direction LR
        D1["<b>compose up</b><br><small>lunes</small>"]:::act --> S1["<b>serie nueva</b>"]:::riesgo
        D2["<b>compose up</b><br><small>martes</small>"]:::act --> S2["<b>serie nueva</b>"]:::riesgo
        D3["<b>compose up</b><br><small>miércoles</small>"]:::act --> S3["<b>serie nueva</b>"]:::riesgo
        S1 --- H["<b>Las anteriores quedan huérfanas</b><br><small>ocupan, ensucian y ralentizan las consultas</small>"]:::riesgo
        S2 --- H
        S3 --- H
    end
    subgraph BIEN["Dejando pasar solo service y project"]
        direction LR
        DD["<b>compose up</b><br><small>cualquier día</small>"]:::act --> SS(["<b>La misma serie</b><br><small>service=app · estable entre despliegues</small>"]):::ok
    end
    MAL ~~~ BIEN
    classDef act fill:#ea580c22,stroke:#ea580c,stroke-width:1.5px
    classDef pieza fill:#64748b22,stroke:#64748b,stroke-width:1.5px
    classDef dato fill:#2563eb22,stroke:#2563eb,stroke-width:1.5px
    classDef infra fill:#a1a1aa14,stroke:#a1a1aa,stroke-width:1.5px
    classDef ok fill:#16a34a22,stroke:#16a34a,stroke-width:1.5px
    classDef riesgo fill:#dc262622,stroke:#dc2626,stroke-width:1.5px
```

<p class="pie" markdown>Una etiqueta que cambia en cada despliegue multiplica las series. Es el enemigo número uno de una base de datos de series temporales.</p>


La etiqueta que interesa para todo el módulo es `container_label_com_docker_compose_service`: es estable entre despliegues (siempre `app`, `db`, `nginx`) mientras que `name` cambia al escalar o al renombrar. En Prometheus, en el scrape de cAdvisor, se renombra a algo manejable:

```yaml
# prometheus.yml (fragmento)
scrape_configs:
  - job_name: cadvisor
    scrape_interval: 15s
    static_configs:
      - targets: ["cadvisor:8080"]      # nombre del servicio en la red de compose
        labels: { host: puesto }
    metric_relabel_configs:
      - source_labels: [container_label_com_docker_compose_service]
        target_label: service
      - regex: container_label_.*
        action: labeldrop        # después de la copia: ya no hace falta ninguna
      - source_labels: [__name__]
        regex: container_(cpu_usage_seconds_total|memory_working_set_bytes|memory_usage_bytes|spec_memory_limit_bytes|network_(receive|transmit)_bytes_total|fs_(reads|writes)_bytes_total|oom_events_total|last_seen)
        action: keep
```

El target depende de dónde esté cada pieza. En el entorno provisional Prometheus y cAdvisor corren en el mismo puesto y comparten red de compose, así que basta el nombre del servicio (`cadvisor:8080`, con la etiqueta `host: puesto`). Desde la sesión 4, con Prometheus en mon01 y cAdvisor en app01, el target es la IP de app01 en vmbr0 con el puerto publicado (`<app01>:8081`) y la etiqueta pasa a `host: app01`. El resto del bloque no cambia.

El último bloque `keep` es opcional pero recomendable en un laboratorio con 2 o 3 GB de RAM en mon01: cAdvisor expone más de 100 métricas por contenedor y solo se usan diez. `container_last_seen` sirve para saber qué contenedores existen (y para detectar en UT2 los que han desaparecido).

Las tres consultas que se usan en Grafana en la sesión 2:

```promql
# CPU en núcleos por servicio (1 = un núcleo entero)
sum by (service) (rate(container_cpu_usage_seconds_total{service!=""}[2m]))
# memoria working set por servicio
sum by (service) (container_memory_working_set_bytes{service!=""})
# tráfico de red recibido, bytes/s
sum by (service) (rate(container_network_receive_bytes_total{service!=""}[2m]))
```

La ventana `[2m]` con scrape cada 15 s da 8 puntos por ventana; con `[30s]` salen dos puntos y las gráficas van a saltos. Regla: la ventana de `rate()` al menos cuatro veces el intervalo de scrape.

### A1.2 Métricas de recursos (sesión 2)

<span class="et et-obj">Objetivo</span> cAdvisor corriendo junto al servicio, Prometheus haciéndole scrape por el nombre del servicio de compose con la etiqueta `service` limpia, y dos paneles en Grafana con CPU y working set por servicio.

<span class="et et-pre">Antes de empezar</span>

- Sigue sin haber VM: todo en tu puesto, con el servicio de la A1.1 en `/opt/servicio` y la pila mínima en `/opt/monitoring`. Los agentes van en un tercer compose, `/opt/agentes`, para que reiniciar el servicio no los tumbe.
- Lo explicado antes: [de dónde salen los números](#de-donde-salen-los-numeros), [memory usage frente a working set](#memory-usage-frente-a-working-set), [cAdvisor en compose](#cadvisor-en-compose) y [etiquetas y cardinalidad](#etiquetas-y-cardinalidad).

<span class="et et-pas">Pasos</span>

1. Conecta las tres pilas a una red de Docker común llamada `obs`, para que Prometheus pueda llamar a cAdvisor por su nombre de servicio.

    *Pista:* la red se crea una vez con `docker network create` y cada compose la declara al final como su red por defecto, sin tocar ningún servicio:

    ```yaml
    networks:
      default:
        name: obs
        external: true
    ```

2. Despliega cAdvisor en `/opt/agentes` con el compose de [cAdvisor en compose](#cadvisor-en-compose) y busca en su `/metrics`, publicado en el 8081, qué valores toma la etiqueta del servicio de compose: tienen que salir `app`, `db` y `nginx`.

3. Calcula a mano el working set de la API a partir de su cgroup y compáralo con lo que da `docker stats`.

    *Pista:* la ruta del cgroup no se supone, porque cambia con el driver de cgroups: sale de `/proc/<PID>/cgroup` del proceso principal del contenedor.

4. Añade a Prometheus el job `cadvisor` de [Etiquetas y cardinalidad](#etiquetas-y-cardinalidad), con el target `cadvisor:8080`, y mide lo que ahorra el bloque `keep`: cuenta métricas distintas y series del job sin él, actívalo y vuelve a contar.

    *Pista:* `count(count by (__name__)({job="cadvisor"}))` cuenta métricas distintas; `count({job="cadvisor"})`, series.

5. Monta en Grafana un dashboard con dos paneles, CPU en núcleos y working set por servicio, con las consultas del final de [Etiquetas y cardinalidad](#etiquetas-y-cardinalidad), y añade al de memoria el porcentaje sobre el límite. Genera tráfico un par de minutos y comprueba que la CPU de `app` sube.

    *Pista:* en la fuente de datos, `localhost` sería el propio contenedor de Grafana: va `http://prometheus:9090`. Si el porcentaje sale casi cero, `app` no tiene límite de memoria; pónselo con `mem_limit: 256m`.

<span class="et et-com">Comprobación</span> La resta del paso 3 se queda a unos pocos MiB de `docker stats`; `cadvisor` en UP en Targets; las series llevan `service` con los valores `app`, `db` y `nginx`, y ninguna `container_label_*`; los dos paneles pintan una línea por servicio y el número de series tras el `keep` es varias veces menor.

<span class="et et-ent">Entrega</span> La comparación de memoria del paso 3, la captura de los dos paneles y los cuatro números (métricas distintas y series, antes y después del `keep`), en `tests/evidence/ut1/a1.2.md`. El compose de agentes y el fragmento de `prometheus.yml` van además al repositorio `monitoring`.

<span class="et et-ext">Si te sobra tiempo</span> Cambia la ventana del `rate()` a `[30s]` en el panel de CPU y observa cómo se rompe la gráfica; después vuelve a `[2m]`.

## Sesión 3 · Instrumentar la aplicación

<p class="ut-meta" markdown>13 de octubre · Teoría y práctica · <span class="dur" tabindex="0" aria-label="Métricas de aplicación · 20 min&#10;A1.3 Instrumentar la aplicación · 90 min" data-dur="Métricas de aplicación · 20 min&#10;A1.3 Instrumentar la aplicación · 90 min">:material-school:<i class="dur-barra" style="--teoria:18%"></i>:material-flask:</span></p>

Hoy se toca código: la API sirve `/metrics` con un counter y un histogram, y postgres_exporter publica lo que la base de datos no puede contar por sí misma. De la teoría de abajo la hoja usa los tipos de métrica, el ejemplo en Python, las convenciones de nombres y el apartado de exporters.

### Métricas de aplicación

![Prometheus](../img/prometheus-logo.svg){ .logo-inline } Las métricas de recursos dicen cuánto consume el contenedor, no si el servicio funciona. Una API puede estar al 2 % de CPU devolviendo 500 a todo el mundo. Para eso la aplicación se **instrumenta**: incorpora la librería cliente de Prometheus de su lenguaje, registra unas cuantas métricas y las sirve en un endpoint `/metrics` en texto plano, y Prometheus lo lee cada 15 s.

#### Tipos de métrica

- **Counter**: solo sube (peticiones totales, errores totales, bytes enviados). Nunca se consulta el valor crudo, que depende de cuándo arrancó el proceso, sino su velocidad con `rate()` o `increase()`. Si el proceso se reinicia, el contador vuelve a cero y `rate()` lo detecta y lo corrige.
- **Gauge**: sube y baja (conexiones activas, tamaño de una cola, temperatura, fecha del último backup en segundos epoch). Se consulta directo.
- **Histogram**: cuenta observaciones en cubos acumulativos (`le="0.1"`, `le="0.5"`...) y además suma y total. Permite calcular percentiles en el servidor con `histogram_quantile()` y, lo que importa, agregarlos entre instancias. Es lo que se usa para latencias.
- **Summary**: percentiles calculados en el cliente. Más preciso para una instancia, pero no se puede agregar entre varias réplicas. En la práctica, histogram casi siempre.

```mermaid
flowchart TB
    Q{"<b>¿Qué se quiere medir?</b>"}:::act
    C["<b>Counter</b><br><small>solo sube · peticiones, errores, bytes</small>"]:::dato
    G["<b>Gauge</b><br><small>sube y baja · conexiones, cola, temperatura</small>"]:::dato
    H["<b>Histogram</b><br><small>cubos acumulativos · percentiles en el servidor</small>"]:::dato
    S["<b>Summary</b><br><small>percentiles en el cliente · no se agrega</small>"]:::riesgo
    Q -- "algo que solo crece" --> C
    Q -- "un valor de ahora mismo" --> G
    Q -- "latencias" --> H
    Q -. "casi nunca" .-> S
    C -- "nunca el valor crudo" --> R(["<b>rate() o increase()</b>"]):::ok
    H --> P(["<b>histogram_quantile()</b><br><small>y se puede sumar entre réplicas</small>"]):::ok
    classDef act fill:#ea580c22,stroke:#ea580c,stroke-width:1.5px
    classDef pieza fill:#64748b22,stroke:#64748b,stroke-width:1.5px
    classDef dato fill:#2563eb22,stroke:#2563eb,stroke-width:1.5px
    classDef infra fill:#a1a1aa14,stroke:#a1a1aa,stroke-width:1.5px
    classDef ok fill:#16a34a22,stroke:#16a34a,stroke-width:1.5px
    classDef riesgo fill:#dc262622,stroke:#dc2626,stroke-width:1.5px
```

<p class="pie" markdown>El contador crudo depende de cuándo arrancó el proceso, así que no se consulta nunca: se consulta su velocidad.</p>


#### Python con prometheus_client

La API del curso es Flask, el microframework web de Python, servida con gunicorn. El endpoint `/metrics` lo sirve la propia aplicación en el mismo puerto 8080 o, mejor, en un puerto aparte (9102) con `start_http_server` para que no pase por el proxy y no cuente como tráfico de usuario:

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

Detalles que marcan la diferencia. La etiqueta `route` lleva la **plantilla** de la ruta (`/items/<int:id>`), nunca la URL real (`/items/48213`): con la URL real cada id distinto es una serie nueva y en una semana hay 100 000 series. Lo mismo con `status`: el código, no el mensaje. Los buckets del histogram se eligen pensando en el servicio: para una API con objetivo de 250 ms la lista de arriba es razonable; para un batch que tarda minutos no vale. Y `INFLIGHT` como gauge permite en UT2 detectar una API que se está atascando antes de que lo digan los errores.

gunicorn es el servidor que ejecuta la aplicación Python en producción, y en el curso arranca con un solo worker (proceso). Con varios workers en paralelo, cada proceso tendría su propio registro y Prometheus vería valores que saltan de un worker a otro. La librería lo resuelve con el modo multiproceso: variable de entorno `PROMETHEUS_MULTIPROC_DIR` apuntando a un directorio vacío y `MultiProcessCollector`. Está en la [documentación de prometheus_client](https://prometheus.github.io/client_python/multiprocess/); en el laboratorio del curso se activa el día que la API pase a más de un worker.

#### Nombres y etiquetas: convenciones

Prometheus no impone nombres, pero si no se siguen las convenciones las consultas se vuelven un infierno y los dashboards de la comunidad dejan de servir. Las reglas de [Metric and label naming](https://prometheus.io/docs/practices/naming/):

| Regla | Bien | Mal |
|----|----|----|
| Prefijo con el nombre de la aplicación o del subsistema | `app_http_requests_total` | `requests` |
| Unidad base en el nombre, sin prefijos SI | `_seconds`, `_bytes`, `_ratio` | `_ms`, `_kb`, `_percent` |
| Los counters terminan en `_total` | `app_errors_total` | `app_error_count` |
| Una métrica mide una cosa; las dimensiones van en etiquetas | `app_http_requests_total{status="500"}` | `app_http_500_total` |
| Etiquetas de cardinalidad acotada | `method`, `status`, `route` plantilla | `user_id`, `request_id`, URL completa |
| snake_case en minúsculas | `app_db_reachable` | `appDbReachable` |

Sobre cardinalidad, el cálculo que hay que hacer siempre: número de series = producto del número de valores de cada etiqueta. `method` (5) × `route` (20) × `status` (8) = 800 series para un counter. Asumible. Con `client_ip` (miles) se pasa del millón.

#### Exporters: lo que no se puede instrumentar

PostgreSQL, nginx, Redis o el propio host no van a incorporar una librería de Prometheus. Para ellos existe el patrón **exporter**: un proceso auxiliar que consulta al software por su interfaz nativa (SQL, una página de estado, ficheros del sistema) y traduce lo que obtiene al formato de Prometheus. Corre al lado del servicio, normalmente como sidecar (un contenedor auxiliar pegado al principal) en el mismo compose.

**postgres_exporter** (puerto 9187) consulta `pg_stat_database`, `pg_stat_activity`, `pg_stat_bgwriter`, `pg_locks` y compañía. Nunca se le da el usuario `postgres`: se crea un usuario de solo lectura con el rol `pg_monitor`, que existe desde PostgreSQL 10 precisamente para esto.

```sql
CREATE USER monitor WITH PASSWORD 'cámbiame';
GRANT pg_monitor TO monitor;
GRANT CONNECT ON DATABASE servicio TO monitor;
GRANT SELECT ON items TO monitor;   -- solo para la consulta propia de queries.yaml
```

```yaml
# junto al servicio db, en el mismo compose del servicio
  postgres_exporter:
    image: quay.io/prometheuscommunity/postgres-exporter:latest
    command: ["--extend.query-path=/etc/queries.yaml"]
    environment:
      DATA_SOURCE_URI: "db:5432/servicio?sslmode=disable"
      DATA_SOURCE_USER: monitor
      DATA_SOURCE_PASS_FILE: /run/secrets/pgx_pass
    volumes:
      - ./queries.yaml:/etc/queries.yaml:ro
    secrets: [pgx_pass]
    ports: ["9187:9187"]
    depends_on: [db]
# y al final del fichero
secrets:
  pgx_pass:
    file: ./pgx_pass.txt     # la contraseña de monitor; va en .gitignore
```

Las métricas que importan: `pg_up` (1 si consigue conectar; es la primera alarma de UT2), `pg_stat_database_numbackends` (conexiones abiertas, para comparar con `max_connections`), `pg_stat_database_xact_commit` y `xact_rollback` (con `rate()`), `pg_stat_database_blks_hit` frente a `blks_read` (acierto de caché), `pg_locks_count`, `pg_database_size_bytes`. Si hace falta algo propio (filas de una tabla, edad del registro más antiguo sin procesar), el exporter admite consultas personalizadas en un fichero YAML pasado con `--extend.query-path`; cada métrica se llama como la clave del fichero más el nombre de la columna, así que la clave empieza por `pg_` para que se agrupe con las demás. Ese mecanismo está marcado como obsoleto en las versiones actuales aunque sigue funcionando; si desaparece, la alternativa mantenida es [sql_exporter](https://github.com/burningalchemist/sql_exporter), que hace lo mismo para cualquier base de datos.

```yaml
# queries.yaml: sale como pg_items_filas (un gauge, así que sin _total)
pg_items:
  query: "SELECT count(*) AS filas FROM items"
  metrics:
    - filas: { usage: GAUGE, description: "Filas de la tabla items de la API" }
```

**node_exporter** (puerto 9100) es el exporter del propio host: CPU, memoria, disco y red de la VM, leídos de `/proc` y `/sys`. Viene instalado en la plantilla de la que salen app01 y mon01 (paquete `prometheus-node-exporter`), así que no se despliega: en la A1.4 solo se le añade su job en Prometheus.

**nginx-prometheus-exporter** (puerto 9113) hace lo mismo con nginx a partir de su módulo `stub_status`, pero da poco: conexiones activas y en espera, peticiones totales y `nginx_up`, sin códigos de respuesta ni latencias por ruta. Por eso en el curso lo que se sabe de nginx sale de su log de acceso en JSON, que llega a Loki en la sesión 4. Cómo se activa `stub_status` y se arranca el exporter está en [Para ampliar](../ampliacion.md#nginx-prometheus-exporter-y-stub_status).

### A1.3 Instrumentar la aplicación (sesión 3)

<span class="et et-obj">Objetivo</span> La API sirve `/metrics` en el 9102 con un counter y un histogram, postgres_exporter publica las métricas del servicio `db` del compose, los dos están en UP, y calculas en PromQL el p95 de latencia y el porcentaje de errores.

<span class="et et-pre">Antes de empezar</span>

- Última sesión sin VM: todo sigue en tu puesto, con las tres pilas de la A1.2 en la red `obs`.
- La API es Python con Flask, servida con gunicorn y un solo worker. PostgreSQL es el servicio `db` del mismo compose: se entra con `docker compose exec db psql -U postgres -d servicio`.
- Lo explicado antes: [tipos de métrica](#tipos-de-metrica), [Python con prometheus_client](#python-con-prometheus_client), [nombres y etiquetas](#nombres-y-etiquetas-convenciones) y [exporters](#exporters-lo-que-no-se-puede-instrumentar).

<span class="et et-pas">Pasos</span>

1. Instrumenta la API con el `metrics.py` de [Python con prometheus_client](#python-con-prometheus_client), con los mismos nombres de métrica y etiquetas, y publica su `/metrics` en el puerto 9102 del host.

    *Pista:* falta `prometheus_client` en `api/requirements.txt` y llamar a `instrument(app)` al crear la aplicación. Deja gunicorn con un solo worker: con varios, los contadores saltan entre scrapes.

2. Despliega postgres_exporter en el compose del servicio, con el usuario `monitor` y la consulta propia de [Exporters](#exporters-lo-que-no-se-puede-instrumentar), publicado en el 9187.

    *Pista:* el SQL, el servicio con su bloque `secrets` y `queries.yaml` están completos en ese apartado. `pgx_pass.txt` va al `.gitignore`.

3. Añade a Prometheus los jobs `app` y `postgres`, con las etiquetas `host: puesto` y `service` (`app` y `db`), y comprueba que los dos están en UP.

4. Genera tráfico variado (peticiones correctas, alguna ruta que no existe y medio minuto con la base de datos parada) y calcula en Prometheus el p95 de latencia y el porcentaje de respuestas 5xx.

    *Pista:* el p95 sale de `histogram_quantile(0.95, ...)` aplicado al `rate()` de `app_http_request_duration_seconds_bucket` sumado `by (le)`.

<span class="et et-com">Comprobación</span> `app` y `postgres` en UP en Targets; `app_http_requests_total` lleva `route` en forma de plantilla (`/items/<int:id>`, nunca `/items/48213`); `pg_up` vale 1 y la métrica personalizada aparece con prefijo `pg_`; el p95 sale en segundos y el porcentaje de 5xx sube solo mientras la base de datos está parada.

<span class="et et-ent">Entrega</span> En `tests/evidence/ut1/a1.3.md`: el código de la instrumentación, `queries.yaml`, los jobs de `prometheus.yml` y las dos consultas con su resultado. El código va además al repositorio `servicio` con un commit propio, que el 15 de octubre viaja con la carpeta a app01.

<span class="et et-ext">Si te sobra tiempo</span> Añade el gauge `app_db_reachable` (1 si la última consulta a PostgreSQL funcionó) y comprueba que baja a 0 al parar la base de datos.

## Sesión 4 · Logs estructurados

<p class="ut-meta" markdown>15 de octubre · Teoría y práctica · <span class="dur" tabindex="0" aria-label="Logs · 20 min&#10;A1.4 Logs estructurados · 90 min" data-dur="Logs · 20 min&#10;A1.4 Logs estructurados · 90 min">:material-school:<i class="dur-barra" style="--teoria:18%"></i>:material-flask:</span></p>

Al acabar, las tres pilas corren ya en app01 y mon01, los logs de la API y de nginx llegan a Loki como JSON con el mismo `request_id`, y se puede seguir una petición por los dos servicios desde Grafana Explore. La hoja empieza llevando las pilas del puesto a las VM y después recorre los cuatro apartados de abajo: limitar el driver de Docker, escribir JSON con `request_id`, montar Loki y desplegar Promtail.

### Logs

El tercer flujo es texto, no números, y por eso tiene su propio camino: Docker lo recoge, Promtail lo etiqueta y lo envía, y Loki lo guarda. Son cuatro cosas en orden: limitar el tamaño de los ficheros de log, conseguir que la API escriba una línea JSON por evento con un identificador de petición, configurar Promtail para que las lea y las envíe, y montar Loki en mon01 para guardarlas y consultarlas. Al terminar se puede seguir una petición desde nginx hasta la API por su identificador.

#### El driver de logs de Docker y por qué limitarlo

Docker captura stdout y stderr del proceso principal del contenedor y se los entrega a un **driver de logs**. El driver por defecto es `json-file`: escribe cada línea como un objeto JSON (`{"log":"...","stream":"stdout","time":"2026-10-13T09:12:44.120Z"}`) en `/var/lib/docker/containers/<id>/<id>-json.log`. Sin configuración adicional ese fichero **no rota nunca**. Una API que escriba 200 líneas por segundo de 300 bytes genera 5 GB al día.

!!! ojo "La causa más común de \"se ha caído el servidor\""
    En dos semanas `app01` se queda sin disco, Docker no puede escribir y el contenedor se para o se cuelga.
    No hay alarma de aplicación que avise de esto: el servicio está bien hasta que el disco se acaba. Conviene limitar
    el driver desde el primer despliegue, no cuando pase.

```mermaid
flowchart LR
    APP["<b>Contenedor</b><br><small>stdout y stderr</small>"]:::pieza
    DRV["<b>Driver json-file</b><br><small>una línea JSON por mensaje</small>"]:::pieza
    FIC["<b>&lt;id&gt;-json.log</b><br><small>sin configurar, no rota nunca</small>"]:::riesgo
    LLENO(["<b>Disco lleno</b><br><small>Docker no puede escribir · el contenedor se para</small>"]):::riesgo
    LIM["<b>max-size y max-file</b><br><small>en /etc/docker/daemon.json</small>"]:::act
    OK(["<b>Ocupación acotada</b>"]):::ok
    APP --> DRV --> FIC --> LLENO
    DRV -- "con límite" --> LIM --> OK
    classDef act fill:#ea580c22,stroke:#ea580c,stroke-width:1.5px
    classDef pieza fill:#64748b22,stroke:#64748b,stroke-width:1.5px
    classDef dato fill:#2563eb22,stroke:#2563eb,stroke-width:1.5px
    classDef infra fill:#a1a1aa14,stroke:#a1a1aa,stroke-width:1.5px
    classDef ok fill:#16a34a22,stroke:#16a34a,stroke-width:1.5px
    classDef riesgo fill:#dc262622,stroke:#dc2626,stroke-width:1.5px
```

<p class="pie" markdown>El log es el único recurso del contenedor que crece sin que nadie lo pida. Acotarlo es una línea de configuración.</p>


El límite se pone en `/etc/docker/daemon.json`, con los mismos valores en app01 y en mon01 (JSON no admite comentarios, así que el fichero lleva solo esto):

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "5" }
}
```

Tras cambiarlo, `systemctl restart docker`, y ojo: **solo aplica a contenedores creados después**. Los existentes conservan su driver y sus opciones hasta que se recrean (`docker compose up -d --force-recreate`). Con esos valores el techo por contenedor son 250 MB. La rotación es por tamaño, no por tiempo: si un contenedor apenas escribe, sus logs de hace meses siguen ahí.

Hay otros drivers (`local`, `journald`, el plugin `loki` y `none`), pero ninguno cambia la recomendación del curso: `json-file` con límites y un agente que lea los ficheros, añada contexto, reintente y no toque el demonio. Qué hace cada uno y por qué aquí no sirve está en [Para ampliar](../ampliacion.md#otros-drivers-de-logs-de-docker).

#### Logs estructurados: JSON con request_id

`docker logs` funciona con cualquier cosa que la aplicación escriba, pero un log que dice `Error procesando pedido 4821 para cliente 77` solo lo entiende una persona. Para filtrar en Loki por nivel, contar errores por ruta o seguir una petición desde nginx hasta la base de datos, la aplicación tiene que escribir **una línea JSON por evento** con campos fijos: `ts` (fecha en formato RFC 3339, como `2026-10-13T09:12:44.120Z`, con zona horaria y milisegundos), `level`, `msg`, `logger`, y los que aporten contexto: `request_id`, `method`, `route`, `status`, `duration_ms`, `user` (id, nunca nombre ni email).

El `request_id` es lo que permite la **correlación**. Lo genera nginx con la variable `$request_id` (32 caracteres hexadecimales, uno por petición), lo pasa a la API en una cabecera, la API lo incluye en todas sus líneas de log y lo devuelve al cliente en la respuesta. Cuando un usuario dice "me ha dado error a las 10:14", se busca ese id y aparece la petición completa. Hasta el 27 de noviembre nginx y la API son dos contenedores del mismo compose en app01, y la correlación es entre dos servicios; cuando el proxy pase a web01 el mecanismo será idéntico, solo que entre dos hosts.

```nginx
# nginx del servicio: en el fichero de /etc/nginx/conf.d/, que se incluye dentro del bloque http
proxy_set_header X-Request-ID $request_id;
add_header X-Request-ID $request_id always;

log_format json escape=json '{"ts":"$time_iso8601","level":"info","msg":"access",'
  '"request_id":"$request_id","method":"$request_method","uri":"$request_uri",'
  '"status":$status,"bytes":$body_bytes_sent,"duration_ms":$request_time,'
  '"upstream":"$upstream_addr","client":"$remote_addr"}';
access_log /dev/stdout json;
```

En un contenedor el log de acceso va a stdout, no a un fichero: así lo recoge el driver de logs de Docker y lo lee el mismo Promtail que lee la API, sin montar ningún volumen de logs. En la imagen oficial de nginx `/var/log/nginx/access.log` ya es un enlace a stdout; al cambiar el formato hay que escribir la ruta `/dev/stdout` de forma explícita.

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

#### Promtail

![Loki](../img/loki-logo.png){ .logo-inline } Promtail es el agente de Grafana Labs para enviar logs a Loki. Grafana lo ha declarado **al final de su vida** en favor de Grafana Alloy, pero se usa igualmente porque su configuración es YAML corto, se entiende en una sesión y sus conceptos se trasladan uno a uno a Alloy. Qué cambia con Alloy y cómo se pasa de uno a otro está en [Para ampliar](../ampliacion.md#promtail-y-grafana-alloy).

Promtail hace cuatro cosas en orden: **descubre** de dónde leer, **etiqueta** cada origen con relabel (reglas que copian, renombran o descartan etiquetas), **procesa** cada línea con un pipeline y **envía** lotes a Loki por HTTP, guardando en un fichero de posiciones hasta dónde ha leído.

```yaml
# /opt/agentes/promtail.yml en app01
server:
  http_listen_port: 9080

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: http://<mon01>:3100/loki/api/v1/push   # la IP de mon01 en vmbr0
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

**relabel_configs**. Las meta-etiquetas `__meta_docker_*` desaparecen al final del relabel; solo sobrevive lo que se copie a una etiqueta sin guiones bajos iniciales. Con estas dos reglas, más las `external_labels` del cliente, cada flujo queda como `{host="app01", env="dev", container="servicio-app-1", service="app"}`, y el pipeline le añade `level`. Son las etiquetas de todo el curso: `host`, `service`, `container` y `level`, siempre en minúsculas, más `env` para distinguir entornos. Las etiquetas en Loki son **índice**, y cada combinación distinta de valores es un stream con sus propios chunks (los bloques comprimidos en que Loki guarda el texto): unas pocas etiquetas con pocos valores cada una está bien; meter `request_id` o la ruta como etiqueta destroza Loki igual que destroza Prometheus. Para campos de cardinalidad alta que conviene filtrar rápido, Loki 3 tiene **structured metadata** (lo que hace la etapa `structured_metadata`): se guarda junto a la línea, no en el índice, y se consulta con `| request_id="..."`.

**pipeline_stages**. Se ejecutan por línea, en orden. `json` extrae campos a variables internas; `timestamp` usa el `ts` de la aplicación como marca de tiempo de la entrada en vez de la hora en que Promtail la leyó (importa cuando hay retraso: si Promtail estuvo 3 minutos sin poder enviar, las líneas se guardan con su hora real, no con la de reenvío); `labels` promociona `level` a etiqueta (5 valores posibles, aceptable y muy útil para `{service="app", level="error"}`). El segundo `match` marca las líneas que no son JSON (una traza de arranque, un `print` olvidado) para encontrarlas y corregirlas. Con `format: RFC3339Nano` Go acepta también RFC 3339 sin nanosegundos; si la API escribe otra cosa, el formato se expresa con la fecha de referencia de Go (`2006-01-02 15:04:05.000`).

**positions**. El fichero `positions.yaml` guarda, por origen, el último offset (o el último timestamp leído, con docker_sd) enviado con éxito. Si Promtail se reinicia, continúa desde ahí. Si el fichero está en `/tmp` dentro del contenedor y no en un volumen, cada reinicio de Promtail reenvía todo lo que Docker conserve del contenedor (hasta 250 MB con los límites de arriba), y Loki no deduplica: los `count_over_time` de la verificación salen multiplicados. Volumen persistente siempre.

**clients**. `batchwait` y `batchsize` deciden cuándo sale un lote: al segundo o al llegar a 1 MB, lo que ocurra antes. `backoff_config` es lo que salva en un corte de red: si el push falla, reintenta esperando 0,5 s, luego 1, 2, 4, 8... hasta 5 minutos entre intentos, y hasta 10 intentos por lote. La suma de las esperas con esos valores son unos 8,5 minutos: un corte de 3 minutos no pierde nada; uno más largo que esa suma sí, y se ve en `promtail_dropped_entries_total`. Ese es el límite de un corte: lo pone la suma de reintentos. `max-size` y `max-file` limitan otra cosa, cuánto puede reenviar Promtail tras un reinicio si el fichero de posiciones se queda atrás.

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

<p class="pie" markdown>Promtail sí tiene lote y reintentos, al revés que el scrape de Prometheus: solo avanza `positions.yaml` cuando Loki confirma el envío.</p>

#### Loki

Loki es "Prometheus para logs": mismo modelo de etiquetas, misma filosofía de indexar poco. Solo indexa las etiquetas del stream y el rango de tiempo; el texto se guarda comprimido en **chunks** y se busca por fuerza bruta dentro de los chunks que las etiquetas seleccionan. Por eso es barato y por eso las consultas sin etiquetas (`{host=~".+"} |= "error"` en 7 días) son lentas: hay que descomprimir todo.

En producción Loki se reparte en varios procesos, cada uno con su papel; aquí se ejecuta entero en uno (`-target=all`, el modo "monolítico") con el disco local de mon01. Ese reparto interno en componentes está descrito en [Para ampliar](../ampliacion.md#el-reparto-interno-de-loki-en-componentes). De la configuración conviene fijarse en `limits_config` (cuánto acepta y cuánto guarda) y en `compactor` (quién borra lo viejo); el resto es fijo:

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

La retención de 7 días (`168h`) se aplica cada 10 minutos por el compactor, y con la demora de 2 h. Con el servicio del curso, Loki ocupa entre 2 y 5 GB de disco con esa retención; en UT5, en la empresa, aparecen retenciones de 30 a 90 días por normativa y ahí el disco local ya no vale: se pasa a un object store (un almacén de ficheros por HTTP, como MinIO o S3) cambiando `object_store` y nada más. `reject_old_samples_max_age` es lo que rechaza líneas con reloj atrasado, y más adelante se cruza con chrony (el servicio que mantiene en hora la VM por NTP, el protocolo de hora en red).

**LogQL** (el lenguaje de consulta de Loki, primo de PromQL) para comprobar que las cosas llegan, que es lo que hace falta en esta unidad (los dashboards y alertas sobre logs van en UT2):

```text
{host="app01"}                                     # todo lo que envía app01, ¿llegan las etiquetas?
{service="app", level="error"}                     # errores de la API, con la etiqueta level de Promtail
{service="app"} | request_id="3f9a..."             # una petición completa (structured metadata)
{service=~"app|nginx"} |= "3f9a1c..."              # la misma petición en los dos servicios
count_over_time({service="app"}[5m])               # líneas en 5 min, para comparar con el origen
sum by (level) (rate({service="app"}[1m]))        # líneas/s por nivel
{service="app", malformed="true"}                  # lo que no era JSON
```

Desde línea de comandos, `logcli` (se instala en el puesto o en mon01 y habla con Loki por HTTP):

```bash
export LOKI_ADDR=http://<mon01>:3100
logcli labels                                  # qué etiquetas existen
logcli labels service                          # valores de una etiqueta
logcli query --since=5m '{service="app"}' --limit 20
logcli query --since=5m 'count_over_time({service="app"}[5m])'
logcli series '{host="app01"}'                 # streams que tiene Loki de app01: cardinalidad
```

En Grafana se añade Loki como fuente de datos (`http://loki:3100` si está en el mismo compose que Grafana, o `http://<mon01>:3100`) y en Explore (la pantalla de consultas libres de Grafana) se elige la fuente y se escriben las mismas consultas. Explore con la fuente Prometheus y la fuente Loki en pantalla partida, el mismo rango de tiempo, es la forma más rápida de correlacionar un pico de latencia con los errores de ese momento.

### A1.4 Logs estructurados (sesión 4)

<span class="et et-obj">Objetivo</span> Las tres pilas corriendo en app01 y mon01, y los logs de la API y del nginx del servicio llegando a Loki como JSON con el mismo `request_id` y las etiquetas `host`, `service`, `container` y `level`, de modo que puedes seguir una petición por los dos servicios desde Grafana Explore.

<span class="et et-pre">Antes de empezar</span>

- **Desde hoy se trabaja en app01 y mon01**, que Despliegue clonó ayer de la plantilla 9000: en vmbr0 con IP por DHCP, con el usuario `ops` (tu clave pública y `sudo` sin contraseña), Docker con compose y node_exporter en el 9100. mon01 tiene 3 GB de RAM. Llegan sin ningún servicio.
- web01 y db01 no se usan todavía: el proxy y la base de datos siguen siendo los servicios `nginx` y `db` del compose, ahora en app01. Entre las dos VM no hay cortafuegos.
- Pon nombre a las dos VM en el `/etc/hosts` de tu puesto y de mon01. En lo que sigue, `<app01>` y `<mon01>` son su IP, o su nombre allí donde vale ese fichero: en la shell y en el navegador sí; en lo que leen los contenedores (`targets` de Prometheus, `clients` de Promtail) no, porque un contenedor no lee el `/etc/hosts` de su host.
- Lo explicado antes: [el driver de logs y por qué limitarlo](#el-driver-de-logs-de-docker-y-por-que-limitarlo), [logs JSON con request_id](#logs-estructurados-json-con-request_id), [Promtail](#promtail) y [Loki](#loki).

<span class="et et-pas">Pasos</span>

1. Lleva las tres pilas del puesto a las VM con su `.git` (servicio y agentes a app01, monitoring a mon01) y levántalas, con los logs de Docker de app01 limitados por el `daemon.json` de [El driver de logs de Docker](#el-driver-de-logs-de-docker-y-por-que-limitarlo): 50 MB y 5 ficheros. Desde hoy esas son las copias de trabajo; las del puesto dejan de usarse.

    *Pista:* el límite solo afecta a los contenedores creados después de reiniciar Docker, así que va antes del primer `up`. La red `obs` hay que crearla en cada VM, y la base de datos de app01 nace vacía: el usuario `monitor` de la A1.3 se vuelve a crear.

2. Adapta los jobs de Prometheus de mon01 al nuevo sitio de cada pieza: `app`, `postgres` y `cadvisor` pasan a los puertos publicados en app01, con `host: app01`, y se añaden `node` (las dos VM, cada una con su `host`), `promtail` (en app01) y `loki` (por su nombre, porque comparte compose con Prometheus). `app`, `postgres`, `cadvisor` y los dos de `node` tienen que quedar en UP; `loki` y `promtail` siguen en DOWN hasta los pasos 4 y 5.

    *Pista:* el `metric_relabel_configs` de cAdvisor de la A1.2 se conserva sin tocar. El puerto de cada pieza está en la [tabla de puertos del laboratorio](../laboratorio.md#convenciones).

3. Haz que nginx genere un `request_id`, se lo pase a la API y escriba su log de acceso en JSON por stdout, y que la API escriba sus logs en JSON con ese mismo `request_id`. Las piezas de los dos lados están en [Logs estructurados: JSON con request_id](#logs-estructurados-json-con-request_id).

    *Pista:* en nginx, `log_format` y `access_log` van fuera del bloque `server`, y `proxy_set_header` y `add_header`, dentro. Para que cada petición deje una línea en la API, añade un `info("request done")` en el `after_request`.

4. Monta Loki en el compose de monitorización de mon01 con el `loki.yml` de [Loki](#loki) y sus datos en un volumen llamado `loki`: con el proyecto `monitoring` queda como `monitoring_loki`, que es el que mira la verificación de la sesión 6.

    *Pista:* imagen `grafana/loki:3.5.3`, arrancada con `-config.file=` apuntando al fichero montado. Tarda unos 15 s en responder `ready` en `/ready`.

5. Despliega Promtail en el compose de agentes de app01 con la configuración de [Promtail](#promtail), enviando a la IP de mon01, y comprueba en sus logs que no hay errores de socket ni respuestas 4xx de Loki.

    *Pista:* para leer el socket de Docker necesita `user: root` en el servicio, o `group_add` con el gid del grupo `docker`.

6. En Grafana, añade Loki como fuente de datos y sigue en Explore una petición por nginx y por la API a partir del `request_id` que devuelve su cabecera `X-Request-ID`.

<span class="et et-com">Comprobación</span> En Targets de mon01, `app`, `postgres`, `cadvisor` y `node` en UP, y las series `container_*` siguen llevando `service`; `docker inspect` muestra los límites del driver en app01; `{host="app01"}` devuelve líneas con `container` y `service`, las de la API también con `level`, y aparecen tanto `service="app"` como `service="nginx"` (las de nginx llegan enteras, todavía sin parsear: eso es lo primero de la sesión 5); la consulta por `request_id` devuelve dos líneas (nginx y API) con la misma marca de tiempo salvo milisegundos; `{service="app", malformed="true"}` está vacío o solo tiene líneas de arranque.

<span class="et et-ent">Entrega</span> En `tests/evidence/ut1/a1.4.md`: la captura de Targets en mon01, `promtail.yml` completo, la configuración de nginx, una captura de Explore con la petición correlada en los dos servicios y la salida de `docker inspect` con los límites del driver. `prometheus.yml`, `loki.yml` y una copia de `promtail.yml` van además al repositorio `monitoring`, y la configuración de nginx y el formateador, al repositorio `servicio`.

<span class="et et-ext">Si te sobra tiempo</span> Pon el mismo `daemon.json` en mon01, reinicia Docker y recrea su pila. Después localiza en Loki las líneas del servicio `db` al parar y arrancar la base de datos, y comprueba qué etiqueta `service` les pone Promtail.

## Sesión 5 · Eventos

<p class="ut-meta" markdown>20 de octubre · Teoría y práctica · <span class="dur" tabindex="0" aria-label="Eventos del demonio Docker · 10 min&#10;A1.5 Eventos · 100 min" data-dur="Eventos del demonio Docker · 10 min&#10;A1.5 Eventos · 100 min">:material-school:<i class="dur-barra" style="--teoria:9%"></i>:material-flask:</span></p>

Sesión con poca teoría: los eventos del demonio reutilizan el camino de los logs, así que casi todo el tiempo es laboratorio. Al terminar hay que haber provocado un `die`, un `oom` y un `health_status: unhealthy` en la API y haberlos localizado en Loki; el único apartado que necesita la hoja es el de eventos, con el contenedor auxiliar que los recoge.

### Eventos del demonio Docker

El cuarto flujo es el más corto y el que dice por qué se ha caído un contenedor: si lo mató el kernel por memoria, si terminó con error o si alguien lo paró. Se recoge reutilizando el camino de los logs, con un contenedor auxiliar y sin escribir código.

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

Cada línea es un JSON con `status`, `Action`, `Actor.ID`, `Actor.Attributes.name`, `Actor.Attributes.exitCode`, `Actor.Attributes.com.docker.compose.service`, y `time`. Un `match` en el pipeline de Promtail para `{service="docker-events"}` con `json` extrae `Action` a etiqueta `action` y el nombre del contenedor afectado a structured metadata. En Loki: `{service="docker-events", action=~"die|oom"}`. La alternativa es un exporter que convierta eventos en métricas (hay varios en GitHub, ninguno oficial); tiene la ventaja de poder alertar con PromQL en lugar de LogQL, pero se pierde el detalle (el exitCode, quién lo paró). En UT2 se alerta desde Loki con el ruler.

Dos avisos. El contenedor auxiliar solo captura eventos mientras está en marcha: si se reinicia el demonio, se pierden los de ese intervalo (poco importa: el `start` de todos los contenedores es lo primero que aparece). Y `docker events` desde un contenedor con el socket montado tiene acceso total al demonio; el `:ro` del montaje no limita la API, solo el fichero. Acotarlo exige un proxy de socket que solo deje pasar las llamadas de lectura, y eso queda fuera del curso: por eso el socket se monta solo en los agentes (cAdvisor, Promtail y docker-events) y nunca en un contenedor del servicio.

### A1.5 Eventos (sesión 5)

<span class="et et-obj">Objetivo</span> Los eventos del demonio Docker de app01 llegan a Loki con la etiqueta `action`, y has provocado y localizado un `die`, un `oom` y un `health_status: unhealthy` en la API.

<span class="et et-pre">Antes de empezar</span>

- A1.4 funcionando: Explore devuelve `{host="app01"}`.
- Lo explicado antes: [Eventos del demonio Docker](#eventos-del-demonio-docker).

<span class="et et-pas">Pasos</span>

1. Lleva a Loki los eventos del demonio de app01 con el contenedor `docker-events` de [Eventos del demonio Docker](#eventos-del-demonio-docker): `Action` como etiqueta `action`, y el nombre del contenedor afectado y su `exitCode` como structured metadata. Aprovecha para que el JSON de nginx, pendiente desde la sesión 4, se procese igual que el de la API.

    *Pista:* son dos `match` nuevos en el pipeline de `promtail.yml`, con la misma forma que el de `{service="app"}` en [Promtail](#promtail). Promtail no relee su fichero solo: después de cambiarlo, reinícialo.

2. Provoca en la API un `die`, un `oom` y un `health_status: unhealthy`, apuntando la hora de cada uno, y localízalos en Loki por su `action`. Al terminar, deja el compose como estaba.

    *Pista:* el `oom` sale con un `mem_limit` muy bajo (unos 32 MB) y unas decenas de peticiones a `/items` en paralelo; el `unhealthy`, con un healthcheck que pida una ruta que no existe.

3. Cruza cada evento con la métrica y el log de ese mismo minuto, en una tabla de tres filas, una por evento, y tres columnas: «evento en Loki», «qué hace la métrica» y «qué deja en los logs». Un evento solo dice que algo pasó; la métrica y el log dicen qué.

4. Dibuja el esquema del flujo de datos completo, que es el que pide el informe de la sesión 7: los contenedores del servicio y los agentes de app01, qué señal sale de cada uno, por qué puerto y en qué sentido. Guárdalo en `tests/evidence/ut1/flujo.md`, en Mermaid o como foto de un dibujo a mano.

<span class="et et-com">Comprobación</span> Loki devuelve al menos una línea de cada evento con la hora que apuntaste; el `oom` va seguido de un `die` con `exitCode` 137; `docker inspect` dice `unhealthy` cuando el evento aparece en Loki; `{service="nginx"}` trae la etiqueta `level`; la tabla del paso 3 tiene sus tres filas con una métrica y un log por evento, y el esquema de `flujo.md` recoge los cuatro flujos con su puerto y su sentido.

<span class="et et-ent">Entrega</span> En `tests/evidence/ut1/a1.5.md`: las tres consultas con su resultado, la lista de `exitCode` observados con su causa y la tabla de las tres señales del paso 3. El esquema va en `tests/evidence/ut1/flujo.md` y los dos `match` nuevos, al `promtail.yml` del repositorio `monitoring`.

<span class="et et-ext">Si te sobra tiempo</span> Mide cuánto tarda un evento en llegar: apunta la hora al lanzar un `docker restart` y compárala con la marca de tiempo de la línea `start` en Loki. Ese retardo es el del camino entero, y es uno de los números que la sesión 6 mide con la tabla de verificación.

## Sesión 6 · Integridad y almacenamiento

<p class="ut-meta" markdown>22 de octubre · Teoría y práctica · <span class="dur" tabindex="0" aria-label="Verificar la integración · 15 min&#10;A1.6 Integridad y almacenamiento · 95 min" data-dur="Verificar la integración · 15 min&#10;A1.6 Integridad y almacenamiento · 95 min">:material-school:<i class="dur-barra" style="--teoria:14%"></i>:material-flask:</span></p>

Es la sesión en la que se demuestra que la integración funciona: la tabla de comprobaciones completa y dos cortes de red de tres minutos para ver qué se recupera y qué se pierde. La hoja usa la tabla de verificación, chrony para los relojes y los comandos de nftables del corte; el apartado de scrape y remote_write no se explica en clase, pero hace falta para entender el hueco que el corte deja en las gráficas.

### Verificar la integración

Integrar no es configurar y olvidarse. Cada punto de la cadena puede fallar en silencio: el firewall descarta, Promtail lee pero no envía, Loki recibe pero rechaza por reloj, Prometheus guarda pero la retención borra. Verificar es ejecutar un comando en cada punto y saber qué significa lo que sale. Esta tabla es la que hay que completar entera en la sesión 6 y la que sale en la práctica.

| Comprobación | Comando | Lo esperado | Si no cuadra |
|----|----|----|----|
| Comunicación app01 → mon01 (logs) | Desde app01: `nc -zv <mon01> 3100` | `succeeded` | Hasta el 27 de noviembre no hay cortafuegos entre las dos VM: si falla, Loki está parado (`docker compose ps` en mon01) o el puerto no está publicado |
| Comunicación mon01 → app01 (métricas) | Desde mon01: `curl -s http://<app01>:9102/metrics \| head -5` y lo mismo a `:9187`, `:8081`, `:9080` y `:9100` | Texto `# HELP ...` | `Connection refused`: no escucha o no está publicado; se queda colgado: IP equivocada o VM apagada (desde el 27 de noviembre, con el cortafuegos, será la primera sospecha) |
| Recepción de métricas | `http://<mon01>:9090/targets` o `curl -s localhost:9090/api/v1/targets \| jq '.data.activeTargets[] \| {job, health, lastError}'` | Todos `up` | `lastError` dice el motivo; `scrape_samples_scraped` en PromQL dice cuántas series trae cada target |
| Recepción de logs | `logcli query --since=2m '{host="app01"}' --limit 5` | Líneas recientes con `container` y `service` | Vacío: mirar `docker logs promtail` (errores 4xx de Loki) y `promtail_targets_active_total` |
| Integridad de logs | En app01: `docker logs --since 5m servicio-app-1 \| wc -l`; en Loki en el mismo momento: `count_over_time({container="servicio-app-1"}[5m])` | Iguales o diferencia de un puñado por el desfase de segundos entre los dos comandos | Loki muy por debajo: pérdidas (`promtail_dropped_entries_total`) o rechazos (`loki_discarded_samples_total` por motivo); Loki por encima: duplicados por posiciones perdidas |
| Integridad de métricas | En app01: `curl -s localhost:9102/metrics \| grep 'app_http_requests_total{method="GET",route="/health",status="200"}'`; en Prometheus, la misma serie | El valor de Prometheus es el del último scrape (hasta 15 s más viejo), nunca mayor | Mayor o muy distinto: se está mirando otra instancia o etiquetas cambiadas por relabel |
| Marcas de tiempo | En cada host: `chronyc tracking \| grep 'System time'` | Desfase inferior a 100 ms | Ver la sección de chrony; Loki y las correlaciones en Grafana lo notan a partir de segundos |
| Almacenamiento Prometheus | `prometheus_tsdb_head_series`, `prometheus_tsdb_storage_blocks_bytes`, y consultar cualquier serie con `offset 24h` | Hay datos de hace 24 h; series estables (no crecen sin parar) | Series crecen: cardinalidad (etiqueta con demasiados valores); sin datos antiguos: retención demasiado corta o el volumen no persiste |
| Almacenamiento Loki | `logcli query --since=24h --limit 1 '{host="app01"}'` y `du -sh /var/lib/docker/volumes/monitoring_loki/_data` en mon01 | Hay líneas de ayer; el tamaño crece y se estabiliza al llegar a la retención | Sin líneas: el compactor ha borrado o el volumen no persiste |
| Pérdidas | `promtail_dropped_entries_total`, `loki_discarded_samples_total` y `count_over_time({service="app"}[1h])` antes y después de un corte | Todos a cero, o su crecimiento coincide con un corte documentado | Suben sin corte: revisar límites de ingestión de Loki (`ingestion_rate_mb`) y reloj |

Un detalle de la integridad de logs: `docker logs --since 5m` y la consulta de Loki no se ejecutan en el mismo instante ni miden exactamente la misma ventana. Para una prueba limpia se genera un número conocido de líneas (`for i in $(seq 1 500); do curl -s -o /dev/null http://<app01>/health; done` desde el puesto) con la API en silencio, se esperan diez segundos y se cuenta en las dos partes con una ventana holgada. Deben salir 500 y 500. Si la API escribe dos líneas por petición (una del middleware y otra del handler), 1000 y 1000.

#### Relojes: chrony

Prometheus pone la hora a las muestras con el reloj de mon01, así que las métricas solo dependen de un reloj. Los logs no: la hora la pone la aplicación (o Docker, si la aplicación no la escribe) con el reloj del host de app01, y Loki la compara con el suyo. Los contenedores no tienen reloj propio, usan el del kernel del host, así que basta con sincronizar las VM. Con un desfase de 30 s la correlación en Grafana entre un pico de latencia y sus errores queda desplazada y confunde; con más de `reject_old_samples_max_age` Loki rechaza las líneas; y con la hora en el futuro más de 10 minutos también (`creation_grace_period`).

Debian 13 trae `systemd-timesyncd`, que vale para un portátil pero no da diagnóstico. Conviene instalar chrony en las dos VM que existen en octubre, app01 y mon01. En el entorno provisional no hay OPNsense, así que la fuente es la de Internet y basta con la línea `pool`. Desde el 27 de noviembre, con las VM ya detrás del cortafuegos y sin salida a Internet por NTP, la fuente pasa a ser el `.1` de la zona de cada VM, que es el propio OPNsense sirviendo la hora:

```bash
sudo apt install chrony
sudo tee /etc/chrony/sources.d/lab.sources <<EOT
# server <el .1 de la zona de la VM> iburst prefer   # desde el 27 de noviembre, detrás de OPNsense
pool 2.debian.pool.ntp.org iburst
EOT
sudo systemctl restart chrony
chronyc sources -v      # ^* delante de la fuente elegida
chronyc tracking        # System time : 0.000213 seconds fast of NTP time
```

Las VM en Proxmox pierden tiempo al suspenderse o al migrarse; chrony por defecto solo corrige a saltos (`makestep`) en los tres primeros ajustes tras arrancar, y después ajusta la velocidad del reloj poco a poco (un desfase de 10 s tarda horas en corregirse). Para el laboratorio, donde se apagan y se encienden VM constantemente, conviene añadir `makestep 1 -1` en `/etc/chrony/conf.d/lab.conf`: corrige de golpe siempre que el desfase pase de 1 s. En producción se tiene apagado por lo que hace a las bases de datos un salto de reloj hacia atrás.

#### Qué pasa cuando se pierde la red

Es la prueba de la sesión 6 y de la práctica. En app01, con nftables (el cortafuegos del kernel de Linux; aquí solo se usa para cortar, y se ve a fondo en la [sesión 17](ut3-seguridad-monitorizacion.md#sesion-17-red-firewall-y-tls) de esta asignatura, el 1 de diciembre), se corta todo el tráfico con mon01 durante tres minutos:

```bash
sudo nft add table inet corte
sudo nft add chain inet corte out '{ type filter hook output priority 0; }'
sudo nft add chain inet corte in '{ type filter hook input priority 0; }'
sudo nft add rule inet corte out ip daddr <mon01> drop
sudo nft add rule inet corte in ip saddr <mon01> drop
date; sleep 180; date
sudo nft delete table inet corte
```

Lo que hay que observar y documentar, con capturas:

- **Métricas por scrape**: a los 15 s, en Targets todos los de app01 pasan a DOWN con `context deadline exceeded`; `up{host="app01"}` vale 0. Al restaurar, vuelven a UP en el siguiente scrape. En Grafana queda un hueco de tres minutos en todas las gráficas de app01. **No se recupera**: esos datos no existen en ningún sitio. Los contadores de la API han seguido subiendo, así que `increase(app_http_requests_total[10m])` sobre un rango que incluye el corte da el total correcto, solo falta el detalle de esos minutos.
- **Métricas por remote_write** (solo si se ha montado el Prometheus local de «Si te sobra tiempo»): la cola del emisor crece durante el corte y se vacía en segundos al restaurar, y en el central aparecen las muestras con su hora original. **Se recupera** sin hueco.
- **Logs**: `docker logs promtail` muestra los reintentos (`error sending batch, will retry`, con el tiempo de espera creciente). `promtail_sent_entries_total` deja de crecer, `promtail_dropped_entries_total` sigue a cero. Al restaurar, en menos de un minuto (el siguiente reintento) llegan a Loki todas las líneas del corte con sus marcas de tiempo reales. Se comprueba con `count_over_time` sobre la ventana del corte. **Se recupera** mientras el corte dure menos que la suma de esperas de los reintentos.
- **Eventos**: igual que los logs, porque van por el mismo camino.

Repetida la prueba con `max_retries: 3`, o con un corte más largo que la suma de esperas de los reintentos (unos 8,5 minutos con la configuración de Promtail de la sesión 4), aparecen pérdidas: `promtail_dropped_entries_total` sube y el conteo en Loki queda por debajo del de `docker logs`. Ese es el número que hay que poner en el informe: cuánto aguanta la configuración propia, calculado con la suma de esperas y comprobado con el corte.

### Enviar las métricas a un sistema remoto: scrape y remote_write

!!! consulta "Material de consulta"
    Esto no se explica en clase: lo necesitas para la hoja de práctica de esta sesión.

Hasta aquí se ha dado por hecho que las métricas llegan a mon01. Este apartado explica cómo viajan, porque hay dos formas con comportamientos distintos cuando la red falla: en la primera Prometheus va a buscarlas, en la segunda se las envían. La primera es la del curso y la que explica el hueco que deja el corte de red de la sesión 6; la segunda solo se monta si sobra tiempo.

<figure markdown="span">
  ![Arquitectura de Prometheus](../img/prometheus-arquitectura.svg){ width="640" }
  <figcaption>Arquitectura de Prometheus: descubrimiento, scrape de targets, TSDB local, reglas hacia Alertmanager y consulta desde Grafana. Fuente: Proyecto Prometheus, Apache 2.0.</figcaption>
</figure>

El camino normal es el **scrape**: Prometheus en mon01 tiene una lista de targets y cada `scrape_interval` (15 s en el laboratorio del curso) hace un GET a `/metrics` de cada uno, parsea el texto y guarda cada muestra con la hora del reloj de mon01. El target no sabe nada de Prometheus, solo responde a un GET. Esto tiene consecuencias que conviene tener claras:

```mermaid
sequenceDiagram
    participant P as Prometheus (mon01)
    participant T as API app01 :9102
    loop cada 15 s
        P->>T: GET /metrics (timeout 10 s)
        alt 200 OK
            T-->>P: texto de exposición (app_http_requests_total{...} 18342 ...)
            P->>P: parsea, añade job/instance, guarda con t = ahora
            P->>P: up{instance="app01:9102"} = 1
        else sin respuesta
            P->>P: up = 0, scrape_duration_seconds, no hay muestras
            P->>P: marca de "stale" en las series del target
        end
    end
```

<p class="pie" markdown>Un scrape es una petición HTTP con tiempo límite, sin cola y sin reintento: si falla, `up` vale 0 y en la serie queda el hueco.</p>

Un scrape que falla no se recupera: no hay cola, el target no guarda nada, y en la gráfica queda un hueco. Los contadores lo toleran bien (el siguiente scrape trae el valor acumulado y `rate()` lo reparte), pero un pico de un gauge que ocurrió en esos segundos se pierde para siempre. `up` es la métrica más importante de Prometheus: vale 1 si el último scrape del target tuvo éxito y 0 si no, y es lo primero que se mira en `http://<mon01>:9090/targets`.

Los jobs de la UT1 en mon01 son los del paso 2 de la [A1.4](#a14-logs-estructurados-sesion-4): todos apuntan a la IP de app01 con sus puertos publicados, salvo Loki, que se resuelve por su nombre porque comparte el compose de Prometheus, y el node_exporter de la propia mon01. Las dos VM están en la red del aula sin cortafuegos entre ellas, así que no hay que abrir nada. Cuando el 27 de noviembre la A3.3 de Despliegue las pase a la VPC dev, en OPNsense tendrán que existir reglas que permitan desde mon01 hacia esos puertos. Si un target sale `context deadline exceeded` en lugar de `connection refused`, los paquetes se están descartando sin respuesta: la VM está apagada o la IP es otra y, desde el 27 de noviembre, lo primero que hay que mirar es el cortafuegos. `connection refused` es que el puerto no está escuchando en la máquina.

**remote_write** es la alternativa cuando el scrape no puede hacerse: el target está detrás de NAT (una red privada cuyas direcciones mon01 no puede alcanzar directamente) o se busca tolerancia a cortes. Se despliega un Prometheus pequeño junto al servicio, en app01, que hace el scrape en local y **empuja** las muestras al central; guarda un WAL (diario de escrituras) en disco y, si el central no responde, reintenta y al volver envía lo acumulado. En el laboratorio la práctica se hace con scrape directo, que es lo que tiene cualquier empresa con una red plana, y el remote_write solo se monta si sobra tiempo en la sesión 6, para comparar el comportamiento ante el corte. La configuración del Prometheus local, el receptor del central y las métricas que dicen si su cola va bien están en [Para ampliar](../ampliacion.md#un-prometheus-local-con-remote_write).

### A1.6 Integridad y almacenamiento (sesión 6)

<span class="et et-obj">Objetivo</span> La tabla de verificación rellena con comandos y salidas reales, y el resultado de dos cortes de red de tres minutos, uno con la configuración normal de Promtail y otro con los reintentos reducidos: qué se recupera, qué se pierde y en cuánto tiempo.

<span class="et et-pre">Antes de empezar</span>

- Todos los flujos funcionando (A1.2 a A1.5) y `sudo` en app01 y mon01, las dos únicas VM que usa hoy esta asignatura.
- `logcli` en mon01 o en el puesto (binario de la [release de Loki](https://github.com/grafana/loki/releases)), con `LOKI_ADDR` apuntando a mon01.
- Lo explicado antes: [Verificar la integración](#verificar-la-integracion), [relojes: chrony](#relojes-chrony) y [qué pasa cuando se pierde la red](#que-pasa-cuando-se-pierde-la-red).

<span class="et et-pas">Pasos</span>

1. Pon en hora app01 y mon01 con chrony como en [Relojes: chrony](#relojes-chrony), todavía sin OPNsense (solo la línea `pool`) y con `makestep 1 -1`, y guarda la salida de `chronyc tracking` de las dos.

    *Pista:* instala a la vez `netcat-openbsd` y `jq`, que los usa la tabla del paso 2.

2. Ejecuta la tabla de [Verificar la integración](#verificar-la-integracion) fila a fila y anota el comando exacto, la salida y si cuadra. La integridad de logs se mide con la prueba limpia que se describe debajo de la tabla, con un número conocido de peticiones.

3. Corta la red entre app01 y mon01 durante tres minutos con los comandos de [Qué pasa cuando se pierde la red](#que-pasa-cuando-se-pierde-la-red), con tráfico en marcha, y documenta qué se recupera y qué se pierde: cuánto tardan los targets de app01 en caer y en volver, qué escribe Promtail, cuándo llegan a Loki las líneas del corte y si el hueco de las gráficas se rellena.

    *Pista:* apunta antes del corte `promtail_dropped_entries_total`, `loki_discarded_samples_total` y `count_over_time({service="app"}[1h])`, para compararlos después y contrastar el conteo de Loki con el de `docker logs`.

4. Repite el corte con `max_retries: 3` en Promtail y mide cuántas líneas se pierden. Al terminar, vuelve a `max_retries: 10`: con el valor bajo, cualquier corte corto de las unidades siguientes perdería logs.

    *Pista:* lo que aguanta la configuración normal es la suma de las esperas de sus reintentos, calculada en [Promtail](#promtail).

<span class="et et-com">Comprobación</span> Las diez filas de la tabla con resultado; desfase de reloj inferior a 100 ms en app01 y mon01; tras el primer corte los conteos de logs coinciden y `dropped_entries` no ha subido, mientras que las gráficas por scrape tienen un hueco que no se rellena; tras el segundo hay pérdida de logs medible, y `promtail.yml` vuelve a tener `max_retries: 10`.

<span class="et et-ent">Entrega</span> En `tests/evidence/ut1/a1.6.md`: la tabla rellena (comando, salida, cuadra o no), las salidas de `chronyc tracking`, por cada corte horas de inicio y fin, gráficas de `up` y de CPU, conteos y valor de `promtail_dropped_entries_total`, y cuánto aguanta la configuración normal. Es la base del informe de la sesión 7.

<span class="et et-ext">Si te sobra tiempo</span> Monta en app01 el Prometheus local de [Para ampliar](../ampliacion.md#un-prometheus-local-con-remote_write) (el central necesita `--web.enable-remote-write-receiver`), repite el corte de 3 minutos y compara: por remote_write no hay hueco.

## Sesión 7 · Práctica evaluable

<p class="ut-meta" markdown>27 de octubre · Práctica evaluable · <span class="dur" tabindex="0" aria-label="Explicación · 10 min&#10;Trabajo en la práctica · 100 min" data-dur="Explicación · 10 min&#10;Trabajo en la práctica · 100 min">:material-school:<i class="dur-barra" style="--teoria:9%"></i>:material-flask:</span></p>

Entrega un informe (máximo 5 páginas, PDF, en `tests/evidence/ut1/` del repositorio `servicio`) sobre el contenedor de referencia integrado en mon01. Lo que se documenta es el entorno que existe hoy: dos VM en la red del aula, app01 con el servicio del curso y sus agentes y mon01 con Prometheus, Loki y Grafana. web01 y db01 no se usan todavía, así que el proxy es el servicio `nginx` y la base de datos el servicio `db`, los dos en el compose de app01. El informe se monta con lo que ya está en `tests/evidence/ut1/` (de `a1.2.md` a `a1.6.md` y `flujo.md`): no pide ninguna prueba nueva. Tiene que contener:

- [ ] Esquema del flujo de datos (métricas, logs, eventos) desde app01 hasta mon01, con puertos y sentido de cada flujo (de la A1.5, paso 4). En app01 conviven los contenedores del compose del servicio (nginx, API, PostgreSQL y postgres_exporter) y los agentes (cAdvisor, Promtail y docker-events), así que el esquema tiene que distinguir qué sale de cada contenedor.
- [ ] Configuración: el compose del servicio y el de agentes, `promtail.yml`, los jobs de `prometheus.yml`, `daemon.json` y el fragmento de la instrumentación de la API (de la A1.2, paso 2; de la A1.3, pasos 1 y 2; de la A1.4, pasos 1, 2 y 5; de la A1.5, paso 1).
- [ ] Pruebas de comunicación, recepción, integridad y almacenamiento con evidencias (comandos y salidas, capturas de Targets y de Explore), siguiendo la tabla de verificación (de la A1.6, paso 2).
- [ ] Resultado de los dos cortes de red: qué se recupera, qué se pierde, qué métrica lo demuestra y cuánto aguanta la configuración normal (de la A1.6, pasos 3 y 4).
- [ ] Salida de `chronyc tracking` de app01 y de mon01 (de la A1.6, paso 1).

| Criterio (RA1 a) | Peso |
|----|----|
| Métricas de recursos y de aplicación llegan a Prometheus | 30 % |
| Logs y eventos llegan a Loki con etiquetas correctas | 30 % |
| Integridad de datos comprobada (conteos coinciden, relojes sincronizados) | 20 % |
| Almacenamiento en mon01 verificado (retención de Prometheus y Loki) y comportamiento ante pérdida de red documentado | 20 % |

Se valora que el informe se pueda reproducir: quien lo lea con acceso al laboratorio tiene que poder ejecutar los mismos comandos y obtener lo mismo. Las capturas sin el comando que las generó no cuentan como evidencia.

## Errores frecuentes en el laboratorio

**A1.1 · La API no arranca: `password authentication failed for user "postgres"`, y nginx devuelve 504.** PostgreSQL fija la contraseña la primera vez que arranca, en el volumen de datos, y después ignora `.env`: pasa si se arrancó sin `.env` o se cambió la contraseña tras el primer `up`. Arreglo: `docker compose down -v && docker compose up -d` en `/opt/servicio` (la API vuelve a crear sus datos de ejemplo). Con Grafana pasa lo mismo con la contraseña de `admin`. Si `docker compose up` dice `required variable DB_PASSWORD is missing a value`, falta copiar `.env.example` a `.env`.

**cAdvisor arranca pero no muestra contenedores, o todas las series llevan `name=""`.** Le falta el montaje de `/var/run` (no encuentra el socket para relacionar cgroups con contenedores) o `/sys`. Con cgroups v2 además necesita `privileged`. Conviene revisar `docker logs cadvisor`: suele decir `Failed to start container manager` o `unable to get docker info`.

**Prometheus hace scrape del puerto 8080 de app01 y trae las métricas de la API en vez de cAdvisor, o un 404.** El puerto 8080 de app01 es la API; cAdvisor está en 8081. Confusión clásica con la tabla de puertos del laboratorio.

**Los valores de `app_http_requests_total` bajan y suben entre scrapes.** gunicorn con varios workers y cada uno con su registro. Activar el modo multiproceso de prometheus_client o servir `/metrics` desde un único proceso.

**Promtail: `permission denied` al abrir `/var/run/docker.sock`.** El contenedor de Promtail corre con un usuario sin acceso al socket. Se añade `user: root` en el compose (aceptable en un agente de solo lectura) o `group_add` con el gid del grupo `docker` del host (`getent group docker`).

**Loki responde 400 `entry too far behind` o `timestamp too old`.** La primera: dentro de un mismo stream llegan líneas más antiguas que las ya aceptadas fuera de la ventana de desorden (2 h por defecto); suele ser un reenvío tras perder el fichero de posiciones. La segunda: la línea tiene más de `reject_old_samples_max_age`, normalmente por un reloj atrasado en app01 o por reenviar logs de hace días tras cambiar de configuración. `loki_discarded_samples_total` lleva la cuenta por motivo.

**Loki responde 429 `Ingestion rate limit exceeded` o `Maximum active stream limit exceeded`.** Se está enviando más de `ingestion_rate_mb` (subirlo si el disco lo permite, o bajar el nivel de log de la API), o hay en las etiquetas algo de cardinalidad alta (`request_id`, la ruta). `logcli series '{host="app01"}' | wc -l` dice cuántos streams hay; con el servicio del curso deberían ser unas decenas, no miles.

**Los conteos de Loki salen el doble que los de `docker logs`.** Promtail ha reenviado todo tras un reinicio porque el fichero de posiciones estaba en `/tmp` dentro del contenedor. Volumen persistente para `/var/lib/promtail`.

**`max-size` está puesto en daemon.json y el fichero json del contenedor sigue en 4 GB.** La opción solo se aplica a contenedores creados después del cambio. `docker compose up -d --force-recreate` y se comprueba con `docker inspect -f '{{.HostConfig.LogConfig}}' servicio-app-1`.

**A1.5 · El `oom` no salta, o la API ni siquiera arranca.** El límite de memoria no cuadra con lo que consume la API en ese puesto. Si con 32 MB no salta, se baja a 16 MB; si la API no llega a arrancar, se sube a 48 MB. Las peticiones a `/items` tienen que ir en paralelo (con `&` y un `wait` al final), porque de una en una la memoria se libera entre petición y petición.

**`docker logs` no muestra nada de la API.** La aplicación escribe a un fichero dentro del contenedor o el framework tiene el log por defecto a nivel WARNING. Todo a stdout y nivel INFO; en UT5 se afina.

**En Grafana, Explore con Loki dice "no logs found" pero `logcli` en mon01 sí devuelve.** Rango de tiempo del navegador (últimos 5 minutos con un desfase de reloj del portátil) o fuente de datos configurada con `localhost:3100` desde el contenedor de Grafana, donde `localhost` es el propio Grafana. Se usa el nombre del servicio de compose (`http://loki:3100`).

**Targets en DOWN con `context deadline exceeded` justo después de tocar OPNsense.** Regla de firewall sin la dirección o el puerto correcto, o creada en la interfaz equivocada. Se comprueba con `nc -zv` desde mon01 y se mira el live log del firewall filtrando por la IP de mon01.

**Las gráficas de CPU de cAdvisor muestran valores absurdos (4000 %).** Se está usando `container_cpu_usage_seconds_total` sin `rate()`, o `rate()` con una ventana menor que el intervalo de scrape.

Los enlaces para ampliar y los apartados que van más allá de lo que se hace en clase están en [Para ampliar](../ampliacion.md#ut1-observabilidad-de-contenedores-metricas-logs-y-eventos).
