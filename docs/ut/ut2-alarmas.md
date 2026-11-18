# UT2 · Umbrales, agregación y gestión de alarmas

<p class="ut-meta">8 h · Sesiones 12 a 15 · RA1 CE b, c, d, e</p>

La UT1 dejó mon01 recogiendo los datos del contenedor de referencia: en Prometheus, las métricas de recursos, las de la API y las de PostgreSQL; en Loki, los logs y los eventos del demonio Docker. Pero esos datos solo sirven si alguien mira las gráficas justo cuando algo falla, y a las tres de la madrugada no mira nadie. Esta unidad los convierte en alarmas: condiciones que se evalúan solas, avisan a quien tiene que actuar sin repetirse de más y dejan rastro en un gestor de incidencias.

La unidad monta la cadena en el orden en que la recorre una alarma: qué condiciones vigilar (sesión 12), cómo se calculan los indicadores y se escriben las reglas (sesión 13), a quién se avisa y cómo queda registrada cada alarma (sesión 14). La práctica evaluable monta el registro de incidencias y prueba la cadena entera de punta a punta.

## Sesión 12 · Umbrales sobre contadores y cadenas en logs

<p class="ut-meta" markdown>12 de noviembre · Teoría y práctica · <span class="dur" tabindex="0" aria-label="De la métrica a la alarma · 10 min&#10;Umbrales sobre contadores · 10 min&#10;Cadenas en logs y eventos · 10 min&#10;A2.1 Umbrales y cadenas · 80 min" data-dur="De la métrica a la alarma · 10 min&#10;Umbrales sobre contadores · 10 min&#10;Cadenas en logs y eventos · 10 min&#10;A2.1 Umbrales y cadenas · 80 min">:material-school:<i class="dur-barra" style="--teoria:27%"></i>:material-flask:</span></p>

Hasta ahora, para enterarse de que el servicio iba mal había que estar mirando. Una alarma empieza por una condición escrita de antemano que el sistema comprueba sin nadie delante. Esta sesión escribe las primeras sobre las dos fuentes que ya llegan a mon01: los números de Prometheus y el texto de Loki. Primero, el mapa de la cadena y el criterio para decidir qué merece ser alarma; después, cómo se escribe un umbral sobre un contador en PromQL; por último, cómo se busca en los logs un mensaje concreto y se convierte en un número. La hoja mide el valor normal de cada indicador, busca en el código de la API los mensajes a vigilar y provoca un fallo para verlos reaccionar.

### De la métrica a la alarma

**Qué es una alarma.** Una condición sobre los datos que, mantenida durante un tiempo, exige que alguien haga algo. Sin condición escrita es un panel que hay que mirar; sin tiempo mínimo, un pico de diez segundos despierta a alguien para nada; sin alguien a quien avisar, no se entera nadie; y sin algo que hacer, el aviso solo molesta.

**La cadena.** Entre un número que sube en Prometheus y un mensaje en el móvil hay varias piezas, cada una con su trabajo: Prometheus y el ruler de Loki calculan y deciden si una condición se cumple; **Alertmanager**, un programa aparte del proyecto Prometheus, decide a quién se avisa y cuándo. Gracias a ese reparto se puede cambiar el canal de aviso sin tocar ninguna regla.

```mermaid
flowchart TB
    M["<b>Métrica o log</b><br><small>Prometheus / Loki</small>"]:::dato
    I["<b>Indicador</b><br><small>recording rule</small>"]:::dato
    R["<b>Regla de alerta</b><br><small>condición + umbral + for</small>"]:::pieza
    AM["<b>Alertmanager</b><br><small>agrupa, enruta, calla</small>"]:::pieza
    N(["<b>Notificación</b><br><small>correo, Telegram</small>"]):::ok
    T(["<b>Incidencia en Gitea</b><br><small>se abre y se cierra sola</small>"]):::ok
    M --> I --> R --> AM
    AM --> N
    AM -- "webhook" --> T
    classDef pieza fill:#64748b22,stroke:#64748b,stroke-width:1.5px
    classDef dato fill:#2563eb22,stroke:#2563eb,stroke-width:1.5px
    classDef ok fill:#16a34a22,stroke:#16a34a,stroke-width:1.5px
```

<p class="pie" markdown>El umbral de cada regla no lo decide la herramienta: sale de la documentación del servicio.</p>

Todo lo que se escribe en la unidad va al repositorio `alerting`, en `/opt/alerting` de mon01:

| Eslabón | Dónde se escribe | Cuándo |
|---|---|---|
| Umbrales y cadenas, con su justificación | `docs/umbrales.md` y `docs/cadenas.md` | 12 nov |
| Indicadores (recording rules) | `prometheus/rules.yml` | 17 nov |
| Reglas de alerta | `prometheus/alerts.yml` y `loki/loki-alerts.yml` | 17 nov |
| A quién se avisa y cuándo | `alertmanager/alertmanager.yml` | 19 nov |
| Incidencias | el receptor `tickets/` | 24 nov |

El umbral sale de la documentación del servicio o de un acuerdo de nivel de servicio (lo que el servicio promete a quien lo usa). Alertmanager ya corre en mon01 desde la UT1, pero sin configurar: no avisa a nadie.

#### Alertar por síntomas, no por causas

**El problema.** Con PromQL se le puede poner un umbral a cualquier cosa. Si se le pone a todo, cada día salta algo, casi nunca importa y el equipo deja de hacer caso.

**El criterio.** El capítulo de monitorización del libro de SRE de Google (así llama Google a operar servicios en producción; [sre.google/sre-book/monitoring-distributed-systems](https://sre.google/sre-book/monitoring-distributed-systems/)) lo resume así: las alarmas que despiertan a alguien responden a **síntomas** que sufre el usuario (errores, lentitud, servicio caído), no a **causas** posibles (CPU alta, muchas conexiones). Una CPU al 95 % que sirve todo en 80 ms no es un problema; una CPU al 30 % con un 5 % de errores sí lo es.

**Qué se hace con las causas.** Se vigilan con menos urgencia. Lo marca la etiqueta **severidad**: `critical` exige actuar ya y va al canal de guardia (el de quien atiende los avisos ese turno, aunque sea de noche); `warning` puede esperar al día siguiente. La buena práctica: cada alarma `critical` tiene que pedir una acción ahora que no se pueda automatizar; si no, baja a `warning` o sobra.

#### Fatiga de alertas

**Qué es.** Lo que pasa cuando el canal de avisos recibe tantos mensajes que el equipo deja de leerlos: si de cuarenta avisos diarios treinta y ocho no piden nada, el que importa se pierde entre los demás. Por eso ningún umbral se pone bajo "por si acaso". Cuando una alarma salta sin motivo se hace una de tres cosas: subir el umbral, alargar el `for` (el tiempo que la condición tiene que durar antes de avisar) o bajarla de `critical` a `warning`.

### Umbrales sobre contadores

![Prometheus](../img/prometheus-logo.svg){ .logo-inline } **Un umbral es un filtro.** Una condición de alarma es una consulta PromQL con una comparación al final. La comparación deja pasar solo las series que la cumplen; si no pasa ninguna, el resultado está vacío y no hay nada de qué avisar. En una regla de alerta, cada serie que pasa el filtro es una alerta:

```promql
up{job="node"}          # una serie por máquina: 1 si su node_exporter responde, 0 si no
up{job="node"} == 0     # solo las máquinas que no responden; con todas en pie, ninguna
```

**Contadores y gauges.** Casi todo lo que hay que vigilar (peticiones, errores) es un **contador**, una métrica que solo sube ([Tipos de métrica](ut1-observabilidad.md#tipos-de-metrica)). Su valor solo dice cuánto lleva arrancado el proceso, así que el umbral se pone sobre su velocidad. La memoria o las conexiones abiertas son **gauges**: suben y bajan, y se comparan tal cual. El tipo de cada métrica está en su línea `# TYPE` del `/metrics`.

#### rate, increase y la ventana

**Qué hacen.** `rate(c[5m])` es cuánto sube por segundo el contador `c`, de media en los últimos cinco minutos; `increase(c[5m])` es cuánto ha subido en total en esa ventana. Las dos corrigen los reinicios del contador. Con `rate` se calculan el p95 y el porcentaje de errores de la UT1 ([Las dos consultas de la sesión](ut1-observabilidad.md#las-dos-consultas-de-la-sesion)).

**La ventana.** Para alertar, entre 2 y 10 minutos: con menos, dos peticiones fallidas parecen una avería; con más, el aviso llega tarde. El curso usa `[5m]` en todos los indicadores.

**Reinicios: `changes`.** Para contar cuántas veces ha arrancado la API se usa `process_start_time_seconds`, un gauge con la hora de arranque del proceso: cada arranque cambia su valor, y `changes()` cuenta los cambios en la ventana. La métrica parecida de cAdvisor, `container_start_time_seconds`, no sirve: guarda la hora en que se creó el contenedor, que no cambia al reiniciarlo.

```promql
changes(process_start_time_seconds{job="app"}[1h]) > 3
```

#### Operadores con etiquetas

**Por qué casi todo es una división.** 50 errores por minuto son un desastre con 60 peticiones y una minucia con 60 000. Por eso los indicadores son proporciones: errores entre peticiones, memoria usada entre límite, conexiones entre máximo.

**Cómo divide PromQL.** Empareja cada serie de la izquierda con la de la derecha que tenga **las mismas etiquetas**; las que no encuentran pareja desaparecen sin error, y la división sale vacía. Tres casos:

- `sum by (service)(...) / sum by (service)(...)`: las dos partes se quedan solo con `service` y emparejan. Es el del ratio de errores.
- `a / ignoring(id) b` empareja sin mirar `id`; `a / on(service) b`, solo por `service`. Con cAdvisor, `id` es la primera etiqueta que hay que ignorar.
- `a / on(service) group_left b`: varias series a la izquierda y una a la derecha, que se reutiliza para cada una.

#### Una consulta vacía no es un cero

**El problema.** Si la consulta sale vacía, la alarma no salta, y desde fuera no se distingue "todo va bien" de "no hay nada que medir". Cuatro motivos, y solo el primero es bueno:

- **La comparación lo ha filtrado todo**: todo está bien.
- **La serie aún no existe**: hasta el primer error 5xx de la API no hay serie de errores, y el ratio sale vacío en lugar de 0.
- **La serie ha desaparecido**: si un target deja de existir, `up{job="app"} == 0` no devuelve nada. Para eso está `absent(up{job="app"})`, que devuelve 1 cuando no hay ninguna serie. La condición de servicio caído usa las dos: `up{job="app"} == 0 or absent(up{job="app"})`.
- **La métrica no se guarda**: el bloque `keep` del [job de cAdvisor](ut1-observabilidad.md#etiquetas-y-cardinalidad) solo deja pasar unas pocas métricas.

La buena práctica: ejecutar la consulta sin la comparación y, si sigue vacía, cada lado de la división por separado, hasta encontrar el que no devuelve nada.

#### Tabla de umbrales del contenedor de referencia

**De dónde salen los números.** De la documentación del servicio: límite de memoria de 512 MiB para `app`, latencia p95 de 500 ms (el tiempo por debajo del cual queda el 95 % de las peticiones), disponibilidad del 99 % (se admite un 1 % de errores) y `max_connections` de PostgreSQL a 100.

| Indicador | Consulta | Umbral orientativo | Severidad |
|---|---|---|---|
| Tasa de errores | `sum(rate(app_http_requests_total{status=~"5.."}[5m])) / sum(rate(app_http_requests_total[5m]))` | > 1 % durante 5 min | critical |
| Latencia p95 | `histogram_quantile(0.95, sum by (le)(rate(app_http_request_duration_seconds_bucket[5m])))` | > 0,5 s durante 10 min | critical |
| Servicio ausente | `up{job="app"} == 0 or absent(up{job="app"})` | 2 min | critical |
| Memoria del contenedor | `container_memory_working_set_bytes{service="app"} / container_spec_memory_limit_bytes{service="app"}` | > 90 % durante 5 min | warning |
| Reinicios | `changes(process_start_time_seconds{job="app"}[1h])` | > 3 en 1 h | warning |
| Conexiones a PostgreSQL | `sum by (service)(pg_stat_activity_count) / on(service) group_left pg_settings_max_connections` | > 80 % | warning |

Los tres primeros son síntomas y son `critical`; los otros tres son causas que anuncian un síntoma cercano (sin memoria, el kernel mata el proceso; sin conexiones, la API empieza a fallar) y son `warning`. Los contenedores se eligen con `service="app"`, la etiqueta que copia el job de cAdvisor del servicio de compose, y no con `name`, que cambia si se recrea el contenedor.

**Orientativos y definitivos.** Los valores de partida vienen de la documentación; los definitivos se ajustan mirando el valor normal del servicio. Un umbral por debajo del valor normal es una alarma permanente.

#### Cómo elegir el for

**El problema: los picos.** Una condición puede cumplirse un momento por casualidad. El `for` es el tiempo que tiene que mantenerse cierta, sin interrupción, antes de avisar. Mientras tanto la alerta está en espera (`pending`); al cumplirlo pasa a disparada (`firing`) y se notifica.

**Tres criterios:**

1. **Los retrasos se suman.** Con `rate(...[5m])` y `for: 5m`, el aviso puede llegar diez minutos después de empezar el fallo. La pregunta es si el usuario aguanta ese tiempo.
2. **Cero para lo que ya ha pasado.** Un OOM kill (el kernel mata el contenedor por pasarse de memoria) es un suceso: esperar no aporta nada. Sin `for`.
3. **Proporcional al coste de una falsa alarma.** Si despierta a alguien, `for` largo; si abre una incidencia que se mira mañana, corto.

### Cadenas en logs y eventos

![Loki](../img/loki-logo.png){ .logo-inline } **Qué es una cadena.** Un texto fijo cuya aparición en un log significa algo: un mensaje de error de la API o un evento de Docker como `oom`. Sale del código de la aplicación o de su documentación, que es donde está escrito qué mensaje se escribe en cada caso. Se consulta en Loki con LogQL, un lenguaje parecido a PromQL.

#### Selectores de stream y filtros de línea

**Cómo es una consulta.** Dos partes. El **selector**, entre llaves, elige los flujos de log (*streams*) por sus etiquetas; es lo único que Loki tiene indexado, y es barato. Detrás van los **filtros**, que leen línea a línea lo que el selector ha dejado; eso es lo caro. La buena práctica: el selector lo más estrecho posible.

```text
{service="app", level="error"}                         # líneas de error de la API
{service="app"} |= "timeout"                           # que contienen timeout
{service="app"} |= "timeout" != "/health"              # ...pero no /health
{service="app"} |~ "(?i)timeout|refused"               # expresión regular, sin distinguir mayúsculas
{service="app", level="error"} | json | msg=~"timeout.*"   # filtra por el campo msg del JSON
```

Los filtros son `|=` (contiene), `!=` (no contiene), `|~` y `!~` (cumple o no una expresión regular), y distinguen mayúsculas: la API escribe `"level":"error"` en minúsculas. Las etiquetas disponibles (`host`, `service`, `container`, `level` y, en los eventos de Docker, `action`) las pone Promtail; `logcli labels` dice las que existen. Como la API escribe sus logs en JSON, `| json` convierte sus campos (`level`, `msg`, `request_id`...) en etiquetas temporales con las que se puede filtrar.

#### Métricas sobre logs

**Del texto al número.** Una línea no se compara con un umbral. Para eso se envuelve la consulta en `count_over_time(...[ventana])`, que cuenta las líneas de la ventana, igual que `rate()` envuelve un contador:

```text
sum(count_over_time({service="app", level="error"} [5m])) > 10
sum(count_over_time({service="app", level="error"} | json | msg=~"timeout.*" [5m])) > 0
sum by (host)(count_over_time({service="docker-events", action="oom"} [10m])) > 0
```

El `sum` junta los streams; con `sum by (host)` se conserva la máquina, para que la alarma diga dónde ha sido.

**Dónde se guardan las cadenas.** En `docs/cadenas.md` del repositorio, una entrada por cadena con su consulta, de dónde sale y qué significa que aparezca: dentro de seis meses nadie recordará por qué se vigila un mensaje concreto.

### A2.1 Umbrales y cadenas (sesión 12)

<span class="et et-obj">Objetivo</span> **Qué vas a conseguir:** saber cuándo tu servicio está mal. Al terminar tienes, por escrito, el valor a partir del cual cada indicador es un problema y los mensajes de log que delatan una avería, y los has visto reaccionar a un fallo provocado por ti.

**Para qué sirve en un trabajo:** una alarma con un umbral inventado salta de más o no salta nunca. En una empresa, cada umbral se mide sobre el servicio real y se puede justificar; si no, el equipo acaba ignorando las alarmas.

<span class="et et-pre">Antes de empezar</span>

- app01 y mon01 encendidas, con todos los targets de `http://mon01:29090` en UP y los logs y eventos de app01 llegando a Loki.
- Los nombres `app01` y `mon01` en el `/etc/hosts` de tu VM `puesto` y de mon01. Si no están, o el DHCP les ha cambiado la IP, añade o corrige esta línea en los dos (la IP de cada VM sale en su *Summary* de Proxmox):

    ```bash
    echo "<IP de app01> app01
    <IP de mon01> mon01" | sudo tee -a /etc/hosts
    ```

- En tu VM `puesto`, `jq` (lee JSON en la terminal) y `logcli` (el cliente de consola de Loki), de la misma versión que el Loki de mon01. Si no los tienes:

    ```bash
    sudo apt install -y jq unzip
    curl -LO https://github.com/grafana/loki/releases/download/v3.5.3/logcli-linux-amd64.zip
    unzip logcli-linux-amd64.zip && sudo install logcli-linux-amd64 /usr/local/bin/logcli
    echo 'export LOKI_ADDR=http://mon01:23100' >> ~/.bashrc && source ~/.bashrc
    logcli labels                     # debe listar host, service, level...
    ```

- La documentación del servicio (la da el profesor) y el código de la API, en `/opt/servicio/api/` de app01.
- La teoría de hoy: [Alertar por síntomas](#alertar-por-sintomas-no-por-causas), [Tabla de umbrales](#tabla-de-umbrales-del-contenedor-de-referencia), [Una consulta vacía no es un cero](#una-consulta-vacia-no-es-un-cero), [Cómo elegir el for](#como-elegir-el-for), [Selectores de stream y filtros de línea](#selectores-de-stream-y-filtros-de-linea) y [Métricas sobre logs](#metricas-sobre-logs).

<span class="et et-pas">Pasos</span>

Trabajas entre mon01, app01 y tu VM `puesto`. El recorrido: preparas el repositorio y un tráfico de fondo (paso 1), mides lo normal (paso 2), buscas los mensajes de error en el código (paso 3), provocas dos averías para ver reaccionar números y logs (pasos 4 y 5) y lo dejas todo escrito (paso 6).

1. **Prepara el repositorio y el tráfico.** *En mon01 y en tu VM `puesto`.*

    Todo lo de la unidad va al repositorio `alerting` de mon01, que entregas en la práctica evaluable. Nace con `secrets/` ignorado, porque ahí irán los tokens de Telegram y de Gitea. En mon01 (la primera línea solo hace falta si Git aún no sabe tu nombre en esa máquina):

    ```bash
    git config --global user.name ops && git config --global user.email ops@lab
    sudo mkdir -p /opt/alerting && sudo chown "$USER" /opt/alerting && cd /opt/alerting
    git init && mkdir -p docs secrets && echo "secrets/" > .gitignore
    echo "# alerting. Sin remoto todavía: lo recibe en la práctica evaluable." > README.md
    git add . && git commit -m "Repositorio alerting"
    ```

    Sin peticiones, el porcentaje de errores no tiene nada que dividir. En tu VM `puesto`, deja este bucle corriendo toda la sesión (las hojas siguientes también lo usan). Pide una ruta inexistente de cada diez, para comprobar que un 404 (página no encontrada) no cuenta como error del servicio:

    ```bash
    i=0; while true; do
      curl -s -o /dev/null -w "%{http_code} " http://app01:20080/items
      i=$((i+1)); [ $((i % 10)) -eq 0 ] && curl -s -o /dev/null http://app01:20080/no-existe
      sleep 1
    done
    ```

    **Sale bien si:** el bucle imprime `200` sin parar.

2. **Mide el valor normal de cada indicador.** *En tu VM `puesto` o en el navegador.*

    Antes de decidir cuándo algo va mal, hay que saber cuánto marca cuando va bien. Ejecuta las seis consultas de la [tabla de umbrales](#tabla-de-umbrales-del-contenedor-de-referencia) en *Graph* de `http://mon01:29090`, o desde el puesto con esta función `q`, y apunta el valor que se mantiene con el tráfico de fondo. La función queda en tu `~/.bashrc`, porque la A2.2 la vuelve a usar:

    ```bash
    echo 'q() { curl -s http://mon01:29090/api/v1/query --data-urlencode "query=$1" | jq -c ".data.result[] | [.metric, .value[1]]"; }' >> ~/.bashrc
    source ~/.bashrc
    q 'sum(rate(app_http_requests_total{status=~"5.."}[5m])) / sum(rate(app_http_requests_total[5m]))'
    ```

    Si una sale vacía, busca el motivo en [Una consulta vacía no es un cero](#una-consulta-vacia-no-es-un-cero) antes de seguir. Dos casos son normales: la de servicio ausente sale vacía mientras la API responde (su número se lee en `up{job="app"}`), y la de errores puede salir vacía si la API aún no ha dado ningún 5xx (error del servidor); apúntalo así.

    **Sale bien si:** tienes un valor (o «vacía» con su motivo) para cada una de las seis.

3. **Busca las cadenas en su origen.** *En app01 y en tu VM `puesto`.*

    Una cadena es un texto del log que delata una avería. Saca del código de la API los dos mensajes que escribe cuando falla la base de datos, con su fichero y su línea: un error (`log.error`), que sale en cada petición que no llega a la base de datos, y un aviso (`log.warning`), que sale cuando la API arranca y no la encuentra. Para cada uno, escribe una consulta LogQL (el lenguaje de consultas de Loki), al menos una con `| json` (lee la línea como JSON para filtrar por sus campos):

    ```bash
    grep -rnE "log(ger)?\.(warning|error|exception)\(" /opt/servicio/api/     # en app01
    logcli labels service                                                       # en el puesto
    logcli query --since=1h '{service="app", level="error"} |= "<texto del error>"'
    logcli query --since=1h '{service="app", level="warning"} | json | msg=~"<inicio del aviso>.*"'
    ```

    `base de datos lista` también es un `log.warning`, pero dice que todo va bien: no es una cadena.

    **Sale bien si:** las dos consultas se ejecutan sin error de sintaxis. Todavía no tienen por qué devolver líneas: aparecen en el paso siguiente.

4. **Provoca un fallo y mira reaccionar números y texto.** *En app01 y en tu VM `puesto`.*

    Una condición que no se ha visto saltar no está probada. Con el bucle corriendo, para `db` dos minutos y vuelve a arrancarla. Mientras está parada, reinicia `app` para que escriba el aviso de arranque, y repite la consulta de errores del paso 2 y tus dos consultas LogQL del paso 3:

    ```bash
    cd /opt/servicio && docker compose stop db      # en app01
    docker compose restart app                       # con db parada: la API se queda esperándola
    docker compose start db                          # dos minutos después
    ```

    Mientras `db` está parada, el bucle se queda esperando unos segundos en cada petición: la API tarda en darse cuenta de que el nombre `db` ya no existe.

    **Sale bien si:** el ratio de errores sube y tus dos consultas devuelven líneas. Si una no devuelve ninguna, no está probada: cambia el texto o el selector y repite.

5. **Provoca un `oom` y convierte las consultas en números.** *En app01 y en tu VM `puesto`.*

    Un `oom` (*out of memory*) es un contenedor que el sistema mata por pasarse de memoria; Docker lo anota como evento. Lanza un contenedor de usar y tirar que muere así. Después busca el evento y pasa la consulta del `oom` y una de la API a consultas métricas (cuentan líneas en vez de enseñarlas), como en [Métricas sobre logs](#metricas-sobre-logs):

    ```bash
    docker run --rm -m 32m python:3-slim python -c "x = bytearray(200 * 1024 * 1024)"   # en app01
    logcli query --since=15m '{service="docker-events", action="oom"}'
    logcli instant-query 'sum by (host)(count_over_time({service="docker-events", action="oom"} [10m]))'
    logcli instant-query 'sum(count_over_time(<tu consulta de la API> [10m]))'
    ```

    **Sale bien si:** las dos consultas métricas dan más de cero.

6. **Fija los umbrales y documenta las cadenas.** *En mon01.*

    Con el valor normal y la documentación delante, decide para cada indicador el umbral, el `for` (cuánto tiempo tiene que durar el problema antes de avisar, mira [Cómo elegir el for](#como-elegir-el-for)) y la severidad (`critical` o `warning`). Escríbelo en `docs/umbrales.md`:

    ```markdown
    | Indicador | Consulta | Valor normal | Umbral | for | Severidad | De dónde sale |
    |---|---|---|---|---|---|---|
    | Tasa de errores | `<consulta>` | <medido> | <umbral> | <for> | critical | Disponibilidad del 99 % (documentación, <página>) |
    ```

    Y en `docs/cadenas.md`, una entrada por cadena (las dos de la API y la del `oom`):

    ```markdown
    ## <nombre corto>
    - Consulta: `<consulta métrica>`
    - De dónde sale: `<fichero>:<línea>` (o el evento `oom` de Docker)
    - Qué significa que aparezca: <qué ha pasado y qué hay que mirar>
    ```

    Haz commit de los dos.

    **Sale bien si:** `git log` muestra el commit y ningún umbral está por debajo de su valor normal.

<span class="et et-com">Comprobación</span>

- [ ] Las seis consultas devuelven lo esperado con el tráfico de fondo.
- [ ] El ratio de errores reaccionó a la parada de `db`.
- [ ] Cada consulta de `docs/cadenas.md` ha devuelto al menos una línea, y la del evento encuentra el `oom`.
- [ ] Las dos consultas métricas dan más de cero.
- [ ] `docs/umbrales.md` está en el repositorio con las siete columnas rellenas.

<span class="et et-ent">Para practicar</span>

- **Guarda** `docs/umbrales.md` y `docs/cadenas.md` en el repositorio `alerting`. Las reglas de la A2.2, el 17 de noviembre, salen de ellos, y la práctica evaluable los revisa para el criterio b.
- **Piensa:** si pusieras el umbral de errores justo en el valor normal, ¿cuántas veces al día te despertaría la alarma?

<span class="et et-ext">Si te sobra tiempo</span> Para probar la alarma de reinicios: reinicia `app` cuatro veces, con medio minuto entre una y otra para que Prometheus vea cada arranque, y comprueba que la consulta de reinicios marca 4. Para tener una cadena más: provoca un `unhealthy` (la comprobación de salud del contenedor falla) con un contenedor de usar y tirar con `--health-cmd "exit 1"` y un intervalo corto, y bórralo después. Su `action` es `health_status: unhealthy`, así que se filtra con `action=~"health_status.*"`. Añade la cadena a `docs/cadenas.md`.

## Sesión 13 · Recording rules y reglas de alerta

<p class="ut-meta" markdown>17 de noviembre · Teoría y práctica · <span class="dur" tabindex="0" aria-label="Agregación y correlación: recording rules · 10 min&#10;Reglas de alerta · 15 min&#10;A2.2 Recording rules y reglas de alerta · 85 min" data-dur="Agregación y correlación: recording rules · 10 min&#10;Reglas de alerta · 15 min&#10;A2.2 Recording rules y reglas de alerta · 85 min">:material-school:<i class="dur-barra" style="--teoria:23%"></i>:material-flask:</span></p>

Las seis consultas de la [tabla de umbrales](#tabla-de-umbrales-del-contenedor-de-referencia) funcionan, pero son largas, se copian en cada panel y cada alerta, y el ratio de errores no existe como métrica en ningún sitio. Las **recording rules** calculan cada indicador una vez y lo guardan con nombre propio. Después, las **reglas de alerta** convierten las condiciones en algo que Prometheus y el ruler de Loki evalúan solos, cada pocos segundos, y mandan a Alertmanager con unas etiquetas que dicen qué es cada alerta y cuánto urge. La teoría va en ese orden: el fichero de recording rules, el de reglas de alerta con sus etiquetas, cómo se prueba una regla sin esperar a que falle nada y las reglas sobre logs del ruler de Loki.

### Agregación y correlación: recording rules

**Qué es.** Una regla que Prometheus evalúa solo, cada cierto tiempo: ejecuta una consulta y guarda el resultado como una serie nueva con nombre propio. Desde ese momento es una métrica más, que se consulta, se grafica y se usa en alertas, pero ya viene calculada.

#### Por qué precalcular

- **Correlar.** El ratio de errores no está en ninguna métrica: sale de dividir dos contadores. La regla lo convierte en una serie con nombre e histórico. Es lo que pide el criterio c: obtener indicadores nuevos a partir de los contadores.
- **Agregar.** Pasar de una serie por réplica a una por servicio con `sum by (service)`. Con una sola réplica no cambia nada, pero la regla queda preparada para más.
- **Aligerar.** Un percentil se calcula una vez cada 30 s, no cada vez que alguien abre un panel.
- **Coherencia.** El panel y la alarma leen la misma serie, así que no pueden dejar de coincidir.

#### Convención de nombres

Las métricas grabadas se nombran `nivel:métrica:operaciones`, con dos puntos, que las métricas normales no usan, así que se reconocen a simple vista. En el laboratorio, `nivel` es el servicio (`app`, `db`), `métrica` lo que se mide (`requests`, `errors`, `latency_p95`) y `operaciones` lo que se le ha hecho con su ventana (`rate5m`, `ratio5m`). `app:errors:ratio5m` se lee "en app, ratio de errores, ventana de 5 minutos".

#### El fichero rules.yml

**Qué lleva.** Cuatro indicadores de la tabla (errores, latencia, memoria y conexiones) más `app:requests:rate5m`, auxiliar para los paneles de tráfico. La ausencia y los reinicios no se graban, porque no calculan ningún indicador. Cada grupo tiene un nombre, un intervalo y sus reglas; cada regla, un `record` (el nombre nuevo) y un `expr` (la consulta):

```yaml
groups:
  - name: app_rules
    interval: 30s
    rules:
      - record: app:requests:rate5m
        expr: sum by (service)(rate(app_http_requests_total[5m]))
      - record: app:errors:ratio5m
        expr: |
          sum by (service)(rate(app_http_requests_total{status=~"5.."}[5m]))
          /
          sum by (service)(rate(app_http_requests_total[5m]))
      - record: app:latency_p95:5m
        expr: histogram_quantile(0.95, sum by (service, le)(rate(app_http_request_duration_seconds_bucket[5m])))
      - record: app:memory:ratio
        expr: |
          container_memory_working_set_bytes{service="app"}
          / ignoring(id)
          container_spec_memory_limit_bytes{service="app"}
      - record: db:connections:ratio
        expr: sum by (service)(pg_stat_activity_count) / on(service) group_left pg_settings_max_connections
```

**Cómo se carga.** Se referencia en `prometheus.yml` con `rule_files: [ "rules.yml", "alerts.yml" ]`, se valida con `promtool check rules` (la utilidad que viene en la imagen de Prometheus) y se recarga con un `POST` a `/-/reload`. Una recarga con errores no tumba Prometheus: se queda con lo anterior, y lo dice `prometheus_config_last_reload_successful`, que vale 0.

**La norma del repositorio.** Las alertas sobre indicadores calculados (ratios, percentiles, porcentajes) usan su métrica grabada, no la consulta cruda. Las de ausencia (`AppDown`, `HostDown`) y la de reinicios (`AppRestarting`) van sobre la serie cruda, porque no calculan nada. La práctica evaluable lo revisa.

### Reglas de alerta

**Qué es.** Lo mismo que una recording rule, pero el resultado se usa como condición: cada serie que devuelve la expresión es una alerta, y si la sigue devolviendo durante el `for`, Prometheus la manda a Alertmanager. Dónde está Alertmanager lo dice el bloque `alerting` de `prometheus.yml`, que se añade en la hoja si la pila no lo trae.

**El fichero.** `alerts.yml` traduce la tabla de umbrales a seis reglas más `HostDown`, que avisa si una máquina deja de contestar al job `node`. Los umbrales y los `for` son los orientativos; en la hoja se cambian por los de cada `docs/umbrales.md`:

```yaml
groups:
  - name: app_alerts
    rules:
      - alert: AppHighErrorRate
        expr: app:errors:ratio5m > 0.01
        for: 5m
        labels:
          severity: critical
          team: ops
          service: app
          origen: prometheus
          env: dev
        annotations:
          summary: "Errores 5xx al {{ $value | humanizePercentage }} en {{ $labels.service }}"
          description: "El ratio de errores lleva 5 min por encima del 1 % (SLA). Revisa la API y el servicio db en app01."
          runbook: "https://wiki.lab/runbooks/app-errors"
      - alert: AppHighLatency
        expr: app:latency_p95:5m > 0.5
        for: 10m
        labels: { severity: critical, team: ops, service: app, origen: prometheus, env: dev }
        annotations:
          summary: "p95 de {{ $value | humanizeDuration }} en {{ $labels.service }}"
          runbook: "https://wiki.lab/runbooks/app-latency"
      - alert: AppDown
        expr: up{job="app"} == 0 or absent(up{job="app"})
        for: 2m
        labels: { severity: critical, team: ops, service: app, origen: prometheus, env: dev }
        annotations:
          summary: "El exporter de app no responde desde hace 2 min"
      - alert: HostDown
        expr: up{job="node"} == 0
        for: 2m
        labels: { severity: critical, team: ops, service: node, origen: prometheus, env: dev }
        annotations:
          summary: "{{ $labels.host }} no responde al node_exporter desde hace 2 min"
      - alert: AppMemoryHigh
        expr: app:memory:ratio > 0.9
        for: 5m
        labels: { severity: warning, team: dev, service: app, origen: cadvisor, env: dev }
        annotations:
          summary: "Memoria al {{ $value | humanizePercentage }} del límite en {{ $labels.name }}"
      - alert: AppRestarting
        expr: changes(process_start_time_seconds{job="app"}[1h]) > 3
        for: 0m
        labels: { severity: warning, team: dev, service: app, origen: prometheus, env: dev }
        annotations:
          summary: "app reiniciado {{ $value }} veces en 1 h"
      - alert: DbConnectionsHigh
        expr: db:connections:ratio > 0.8
        for: 5m
        labels: { severity: warning, team: ops, service: db, origen: prometheus, env: dev }
        annotations:
          summary: "PostgreSQL al {{ $value | humanizePercentage }} de max_connections"
```

Cargado el fichero, las siete reglas salen en la página *Alerts* de Prometheus en verde. Si alguna sale en amarillo o rojo sin haber provocado nada, su umbral está por debajo del valor normal.

**El catálogo cerrado.** Estas siete y las tres del [ruler de Loki](#alertas-en-el-ruler-de-loki) (`AppLogErrorsBurst`, `AppDbTimeout` y `ContainerOOM`) son las diez alarmas de la unidad. El resto del curso las cita por estos nombres, y una alarma que no esté en la lista no existe en el repositorio `alerting`. Solo crece el 12 de enero, cuando el catálogo de la UT4 (indicadores y pruebas) le añade cinco.

#### Etiquetas y anotaciones

**Las etiquetas se deciden en la regla.** Son lo único que Alertmanager mira para enrutar y callar alertas, y lo que usa el receptor de incidencias para clasificarlas. Todas las reglas llevan las mismas cinco:

| Etiqueta | Valores | Para qué la usa la cadena |
|---|---|---|
| `severity` | critical, warning, info | Elige el canal: Telegram para `critical`, correo para el resto |
| `team` | ops, dev | Las de `dev` llegan además al correo del equipo de desarrollo |
| `service` | app, db, web, y `node` para las de la máquina | Agrupa las alertas del mismo servicio |
| `origen` | prometheus, cadvisor, loki, docker-events | Dice de dónde sale el dato |
| `env` | dev, pre | Separa entornos |

Es la categorización que pide el criterio e, decidida en el origen. Prometheus añade además las etiquetas de la serie (`instance`, `job`, `host`...) y `alertname`, el nombre de la regla.

**Las anotaciones son para la persona.** Texto con huecos entre dobles llaves: `{{ $labels.service }}` inserta una etiqueta y `{{ $value }}` el valor; `humanizePercentage` convierte 0.0234 en `2.34%`. La convención: `summary` de una línea, lo que cabe en un móvil; `description` con el contexto; y `runbook` con la URL del procedimiento de qué hacer, que de momento apunta a una wiki de ejemplo (los runbooks se escriben el 12 de enero, en la UT4).

#### Estados de una alerta

```mermaid
stateDiagram-v2
  [*] --> inactive
  inactive --> pending : la expresión devuelve series
  pending --> inactive : deja de cumplirse antes del for
  pending --> firing : se cumple durante todo el for
  firing --> inactive : deja de cumplirse (resolved)
```

<p class="pie" markdown>`inactive` (verde), `pending` (amarillo) y `firing` (rojo) se ven en la página *Alerts* de Prometheus.</p>

Prometheus manda la alerta a Alertmanager en cuanto entra en `firing` y la reenvía mientras siga así. Cuando deja de cumplirse, la manda una última vez con su hora de fin, y Alertmanager la da por resuelta (`resolved`).

#### Probar las reglas con promtool

**Para qué.** Esperar a que el servicio falle para ver si una regla dispara es lento y obliga a romper algo cada vez que cambia un umbral. `promtool test rules` inventa series, simula el paso del tiempo y comprueba qué alertas tienen que estar disparadas. Tarda un segundo y se queda en el repositorio como prueba.

**Cómo es.** Este fichero comprueba que, con un 5 % constante de errores, `AppHighErrorRate` no ha disparado a los 2 minutos (su `for` es de 5) y sí a los 6, con sus etiquetas y anotaciones exactas:

```yaml
rule_files:
  - ../alerts.yml
evaluation_interval: 15s
tests:
  - interval: 15s
    input_series:
      - series: 'app:errors:ratio5m{service="app"}'
        values: '0.05x40'
    alert_rule_test:
      - eval_time: 2m
        alertname: AppHighErrorRate
        exp_alerts: []
      - eval_time: 6m
        alertname: AppHighErrorRate
        exp_alerts:
          - exp_labels: { service: app, severity: critical, team: ops, origen: prometheus, env: dev }
            exp_annotations:
              summary: "Errores 5xx al 5% en app"
              description: "El ratio de errores lleva 5 min por encima del 1 % (SLA). Revisa la API y el servicio db en app01."
              runbook: "https://wiki.lab/runbooks/app-errors"
```

- `values: '0.05x40'` es el valor 0,05 repetido cada 15 s durante diez minutos.
- En cada `eval_time`, `exp_alerts` es la lista de alertas en `firing` que se espera: vacía a los 2 minutos, una a los 6.
- Las anotaciones tienen que coincidir letra a letra con las que da la plantilla.

#### Alertas en el ruler de Loki

**Para qué.** Las consultas de [Métricas sobre logs](#metricas-sobre-logs) solo se ejecutan a mano. El **ruler** de Loki las evalúa cada minuto, como Prometheus hace con las suyas, y manda las que se cumplen al mismo Alertmanager. Así las alarmas sobre texto entran en la misma cadena.

**Cómo se activa.** Con este bloque en `loki.yml`:

```yaml
ruler:
  storage:
    type: local
    local:
      directory: /loki/rules              # de dónde lee las reglas
  rule_path: /tmp/loki-rules-scratch      # directorio de trabajo, no se toca
  alertmanager_url: http://alertmanager:29093
  enable_api: true                        # para consultar las reglas cargadas
  evaluation_interval: 1m
```

**Dónde van las reglas.** En `/loki/rules/fake/`: con `auth_enabled: false` Loki tiene un solo cliente, que se llama `fake`, y lee las reglas del subdirectorio con ese nombre. El fichero `loki-alerts.yml` tiene la forma de uno de Prometheus, con expresiones LogQL y las mismas cinco etiquetas:

```yaml
groups:
  - name: app_logs
    interval: 1m
    rules:
      - alert: AppLogErrorsBurst
        expr: sum(count_over_time({service="app", level="error"} [5m])) > 10
        for: 2m
        labels:
          severity: warning
          team: dev
          service: app
          origen: loki
          env: dev
        annotations:
          summary: "Más de 10 líneas de error en 5 min en la API"
          runbook: "https://wiki.lab/runbooks/app-log-errors"
      - alert: AppDbTimeout
        expr: sum(count_over_time({service="app", level="error"} | json | msg=~"timeout.*" [5m])) > 0
        for: 0m
        labels:
          severity: critical
          team: ops
          service: app
          origen: loki
          env: dev
        annotations:
          summary: "La API registra timeouts contra la base de datos"
      - alert: ContainerOOM
        expr: sum by (host)(count_over_time({service="docker-events", action="oom"} [10m])) > 0
        labels:
          severity: critical
          team: ops
          service: app
          origen: docker-events
          env: dev
        annotations:
          summary: "OOM kill en {{ $labels.host }}"
```

La cadena de `AppDbTimeout` es un ejemplo: la buena sale del `docs/cadenas.md` de la A2.1. `ContainerOOM` no lleva `for` porque un OOM ya ha pasado.

**Cómo se comprueba.** Antes de cargarlo, `lokitool rules lint -n` (`lokitool` es un binario de la misma release que `logcli`; sin `-n` reescribe el fichero con su propio formato). Las reglas se aplican reiniciando Loki, y las cargadas se ven en `/loki/api/v1/rules`.

### A2.2 Recording rules y reglas de alerta (sesión 13)

<span class="et et-obj">Objetivo</span> **Qué vas a conseguir:** que Prometheus y Loki vigilen el servicio **solos**, sin que nadie mire las gráficas. Al terminar, las diez alarmas del catálogo están cargadas y has visto una dispararse de verdad.

**Para qué sirve en un trabajo:** nadie está mirando Grafana a las tres de la madrugada. Una alarma bien escrita es la que avisa por ti, y saber cargarla, probarla y comprobar que salta es el trabajo diario de quien mantiene un servicio.

<span class="et et-pre">Antes de empezar</span>

- De la A2.1: el repositorio `/opt/alerting` con `docs/umbrales.md` y `docs/cadenas.md`, el bucle de tráfico corriendo en tu VM `puesto` y la función `q` en su `~/.bashrc`. Si te falta el repositorio, créalo con el [paso 1 de la A2.1](#a21-umbrales-y-cadenas-sesion-12); si te falta `q`, con el comienzo de su paso 2.
- La pila de mon01 como repositorio Git, porque hoy cambias su `compose.yml`, `prometheus.yml` y `loki.yml` y esos cambios tienen que quedar registrados. Si `/opt/monitoring` todavía no lo es (`git status` dice `not a git repository`), conviértelo con un primer commit de lo que hay. Las contraseñas no entran: si `git status` enseña un `.env` o un directorio de datos que no esté en la lista, añádelo al `.gitignore` antes del commit:

    ```bash
    cd /opt/monitoring
    git init && printf '.env\nsecrets/\ntls/\n' >> .gitignore
    git status                        # solo ficheros de configuración
    git add . && git commit -m "Pila de monitorización tal como la dejó la UT1"
    ```

- Instala `lokitool` en mon01. Es la herramienta que revisa las reglas de Loki:

    ```bash
    sudo apt install -y unzip
    curl -LO https://github.com/grafana/loki/releases/download/v3.5.3/lokitool-linux-amd64.zip
    unzip lokitool-linux-amd64.zip && sudo install lokitool-linux-amd64 /usr/local/bin/lokitool
    ```

- Ten a mano la teoría de hoy, porque vas a copiar de ella: [El fichero rules.yml](#el-fichero-rulesyml), [Reglas de alerta](#reglas-de-alerta), [Probar las reglas con promtool](#probar-las-reglas-con-promtool) y [Alertas en el ruler de Loki](#alertas-en-el-ruler-de-loki).

<span class="et et-pas">Pasos</span>

Hoy trabajas en **mon01** (salvo un momento en app01 para parar la base de datos). El recorrido es este: escribes las reglas, las cargas, provocas una avería para ver una alarma dispararse, pruebas la regla sin avería, haces lo mismo con Loki y lo guardas todo en Git.

1. **Escribe los dos ficheros de reglas.** *En mon01.*

    Hay dos ficheros: `rules.yml` calcula los indicadores (el % de errores, el p95...) y les pone nombre; `alerts.yml` dice cuándo uno de esos indicadores es una alarma.
    Crea los directorios de hoy con `mkdir -p /opt/alerting/prometheus/tests /opt/alerting/loki`, copia los dos ficheros de la teoría en `/opt/alerting/prometheus/` y cambia **solo** los números: los umbrales (`> 0.01`...) y los `for` que decidiste en tu `docs/umbrales.md`. Las etiquetas se quedan como están.

    **Sale bien si:** las alarmas de porcentajes y tiempos usan un nombre con dos puntos (`app:errors:ratio5m`, `db:connections:ratio`...), no la consulta larga.

2. **Cárgalos en Prometheus.** *En mon01.*

    Prometheus tiene que poder leer los ficheros (se los montas en el contenedor) y saber que existen (se lo dices en `prometheus.yml`). En el `compose.yml` de `/opt/monitoring`, añade estas dos líneas a los `volumes` de `prometheus`:

    ```yaml
          - /opt/alerting/prometheus/rules.yml:/etc/prometheus/rules.yml:ro
          - /opt/alerting/prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro
    ```

    En `prometheus.yml`, al nivel de `scrape_configs` (sin sangría), añade la lista de ficheros de reglas y, si no está ya, el bloque `alerting`, que dice a Prometheus adónde mandar las alertas:

    ```yaml
    rule_files: [ "rules.yml", "alerts.yml" ]
    alerting:
      alertmanagers:
        - static_configs:
            - targets: ["alertmanager:29093"]
    ```

    Después revisa los ficheros con el `promtool` de la propia imagen de Prometheus (así no descargas otra) y recrea Prometheus:

    ```bash
    cd /opt/monitoring
    docker compose run --rm --no-deps --entrypoint promtool prometheus check rules /etc/prometheus/rules.yml /etc/prometheus/alerts.yml
    docker compose up -d prometheus
    ```

    **Sale bien si:** `promtool` dice `SUCCESS` y en `http://mon01:29090` → *Alerts* aparecen las siete alarmas en verde (`inactive`, es decir, todo normal).

3. **Provoca una avería y mira cómo salta la alarma.** *En app01 y en el navegador.*

    Es la prueba de verdad: si la alarma no salta con una avería que provocas tú, no saltará con una real. Con el bucle de tráfico corriendo, para la base de datos en app01:

    ```bash
    cd /opt/servicio && docker compose stop db
    ```

    Mira `AppHighErrorRate` en *Alerts* y **apunta la hora** de cada cambio: cuando paras `db`, cuando se pone amarilla (`pending`: se cumple, pero espera el `for` por si es un pico pasajero) y cuando se pone roja (`firing`: avisa). Entonces arranca `db` con `docker compose start db` y apunta cuándo vuelve a verde. La vuelta tarda unos minutos, hasta que los errores salen de la ventana de 5 min: mientras, sigue con el paso 4 y vuelve a mirar al terminarlo.

    Mientras esperas, comprueba en tu VM `puesto` que el nombre corto calcula lo mismo que la consulta larga, con la función `q` de la A2.1:

    ```bash
    q 'app:errors:ratio5m'
    q 'sum by (service)(rate(app_http_requests_total{status=~"5.."}[5m])) / sum by (service)(rate(app_http_requests_total[5m]))'
    ```

    **Sale bien si:** la alarma pasa por amarillo y rojo, y vuelve a verde; las dos consultas dan el mismo número.

4. **Prueba la regla sin romper nada.** *En mon01.*

    Parar la base de datos cada vez que cambias un umbral es lento. `promtool test rules` se inventa los datos y comprueba en un segundo si la alarma saltaría. Copia la prueba de la teoría en `prometheus/tests/alerts_test.yml`, ajusta el `eval_time` y el texto esperado a tu `AppHighErrorRate`, y pásala:

    ```bash
    cd /opt/monitoring
    docker compose run --rm --no-deps -v /opt/alerting/prometheus:/w -w /w/tests --entrypoint promtool prometheus test rules alerts_test.yml
    ```

    **Sale bien si:** dice `SUCCESS`. Si falla, enseña lo que esperabas y lo que ha salido: casi siempre es una letra distinta en el texto del aviso.

5. **Haz lo mismo con los logs: las alarmas de Loki.** *En mon01.*

    Tres alarmas vigilan texto, no números: muchas líneas de error, un mensaje concreto de la API y el `oom` de Docker. Las evalúa Loki, con su *ruler*.
    Añade a `/opt/monitoring/loki/loki.yml` el bloque `ruler` de la teoría. Copia las tres reglas en `/opt/alerting/loki/loki-alerts.yml` y cambia la cadena de `AppDbTimeout` por una de tu `docs/cadenas.md`. Móntalo con esta línea en los `volumes` de `loki`:

    ```yaml
          - /opt/alerting/loki/loki-alerts.yml:/loki/rules/fake/loki-alerts.yml:ro
    ```

    ```bash
    lokitool rules lint -n /opt/alerting/loki/loki-alerts.yml
    cd /opt/monitoring && docker compose up -d loki
    curl -s http://localhost:23100/loki/api/v1/rules
    ```

    **Sale bien si:** el `lint` no se queja y el `curl` enseña tus tres reglas.

6. **Guárdalo todo en Git.** *En mon01.*

    Apunta en `docs/umbrales.md`, junto a `AppHighErrorRate`, las horas del paso 3. Después haz un commit en cada repositorio:

    ```bash
    cd /opt/alerting && git add prometheus loki docs && git commit -m "A2.2: recording rules y alarmas"
    cd /opt/monitoring && git add compose.yml prometheus.yml loki/loki.yml && git commit -m "A2.2: reglas en Prometheus y ruler de Loki"
    ```

    **Sale bien si:** `git log` enseña los dos commits.

<span class="et et-com">Comprobación</span>

- [ ] `promtool check rules` y `promtool test rules` dicen `SUCCESS`.
- [ ] Las siete alarmas de Prometheus están en *Alerts*, y las tres de Loki salen en `/loki/api/v1/rules`.
- [ ] Cada nombre corto da el mismo valor que su consulta larga.
- [ ] Tienes apuntadas las horas de la avería: parada, `pending`, `firing` y vuelta a verde.

<span class="et et-ent">Para practicar</span>

- **Guarda** en el repositorio `alerting`: `prometheus/rules.yml`, `prometheus/alerts.yml`, `prometheus/tests/alerts_test.yml` y `loki/loki-alerts.yml`. Las usa la práctica evaluable, y `rules.yml` es la evidencia del criterio c.
- **Piensa:** ¿cuánto tardó la alarma en ponerse roja desde que paraste `db`? ¿De dónde sale ese retraso?
- **Para el 19 de noviembre**, trae creado un bot de Telegram: [Receptores y plantillas](#receptores-y-plantillas) explica cómo.

<span class="et et-ext">Si te sobra tiempo</span> Crea en Grafana un dashboard `Alarmas` con tres paneles de serie temporal, uno por consulta: `app:errors:ratio5m`, `app:latency_p95:5m` y `app:memory:ratio`. Así el panel y la alarma leen la misma serie y no pueden contradecirse. Exporta el dashboard (*Share → Export → Save to file*) a `grafana/dashboard.json` de `alerting`.

## Sesión 14 · Alertmanager e incidencias

<p class="ut-meta" markdown>19 de noviembre · Teoría y práctica · <span class="dur" tabindex="0" aria-label="Alertmanager · 15 min&#10;Categorizar, notificar y tratar · 5 min&#10;Demostración: incidencias en Gitea · 15 min&#10;A2.3 Alertmanager · 75 min" data-dur="Alertmanager · 15 min&#10;Categorizar, notificar y tratar · 5 min&#10;Demostración: incidencias en Gitea · 15 min&#10;A2.3 Alertmanager · 75 min">:material-school:<i class="dur-barra" style="--teoria:32%"></i>:material-flask:</span></p>

Prometheus y el ruler de Loki ya mandan sus alertas a Alertmanager, pero Alertmanager sigue sin configurar y no avisa a nadie. Y si cada alerta fuera un mensaje, una máquina caída daría diez mensajes repetidos a todo el mundo. Alertmanager decide con qué alertas va cada una, a quién se manda, cuándo y qué se calla. Como no guarda historial, la cadena termina en un **gestor de incidencias** donde cada alarma queda registrada. La teoría sigue ese camino: rutas, tiempos, inhibición, silencios, receptores y pruebas con alertas sintéticas; después, qué datos clasifican cada alarma y una demostración que lleva las alarmas a incidencias de Gitea. La hoja monta las rutas y las prueba; las incidencias se montan en la práctica evaluable.

### Alertmanager

**Prometheus evalúa; Alertmanager decide.** Alertmanager recibe las alertas de Prometheus y del ruler de Loki, las agrupa, decide a quién van, calla las que sobran y notifica por los receptores configurados, repitiendo mientras sigan activas. Toda su configuración está en `alertmanager.yml`, que se valida con `amtool check-config` (`amtool` viene en la imagen de Alertmanager) y se recarga con un `POST` a `/-/reload`.

```mermaid
flowchart TB
    IN["<b>Llega una alerta</b><br><small>de Prometheus o del ruler de Loki</small>"]:::dato
    RU["<b>Rutas</b><br><small>a quién, según sus etiquetas</small>"]:::pieza
    GR["<b>Grupo y tiempos</b><br><small>con quién viaja y cuándo sale</small>"]:::pieza
    IS{"<b>¿Inhibida o silenciada?</b>"}:::act
    CA(["<b>Se calla</b><br><small>se ve en la interfaz</small>"]):::infra
    NO(["<b>Notificación</b><br><small>correo, Telegram, webhook</small>"]):::ok
    IN --> RU --> GR --> IS
    IS -- "sí" --> CA
    IS -- "no" --> NO
    classDef act fill:#ea580c22,stroke:#ea580c,stroke-width:1.5px
    classDef pieza fill:#64748b22,stroke:#64748b,stroke-width:1.5px
    classDef dato fill:#2563eb22,stroke:#2563eb,stroke-width:1.5px
    classDef infra fill:#a1a1aa14,stroke:#a1a1aa,stroke-width:1.5px
    classDef ok fill:#16a34a22,stroke:#16a34a,stroke-width:1.5px
```

<p class="pie" markdown>El camino de una alerta dentro de Alertmanager. Inhibir o silenciar no borra la alerta: solo impide que se notifique.</p>

#### El árbol de rutas

**Para qué.** Una alerta `critical` tiene que llegar al móvil; una `warning` puede esperar en el correo; las del equipo de desarrollo, a ese equipo. Las **rutas** lo deciden: una raíz con rutas hijas, cada una con condiciones sobre las etiquetas (`matchers`) y un **receptor**, el destino.

**El fichero del laboratorio.** Cinco bloques: `global` (valores comunes, aquí el correo), `templates` (las plantillas de los mensajes), `route` (las rutas), `inhibit_rules` (qué se calla) y `receivers` (los destinos):

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: mailpit:1025
  smtp_from: alertmanager@lab
  smtp_require_tls: false

templates:
  - /etc/alertmanager/templates/*.tmpl

route:
  receiver: mail
  group_by: [alertname, service]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - matchers: [severity="critical"]
      receiver: critical
      continue: true
    - matchers: [severity="warning"]
      receiver: mail
    - matchers: [team="dev"]
      receiver: dev-chat
      group_by: [alertname, service, env]

inhibit_rules:
  - source_matchers: [alertname="HostDown"]
    target_matchers: [severity=~"warning|critical"]
    equal: [host]
  - source_matchers: [alertname="AppDown"]
    target_matchers: [service="app"]
    equal: [env]

receivers:
  - name: mail
    email_configs:
      - to: ops@lab
        send_resolved: true
  - name: critical
    telegram_configs:
      - bot_token_file: /etc/alertmanager/secrets/tg_token
        chat_id: -100123456789
        parse_mode: HTML
        message: '{{ template "lab.telegram" . }}'
        send_resolved: true
  - name: dev-chat
    email_configs:
      - to: dev@lab
        send_resolved: true
```

**Cómo se recorre.** Las rutas hijas se prueban en orden y la primera que coincide se queda la alerta, salvo que lleve `continue: true`, que sigue probando las siguientes. Así, una alerta `critical` de `team=dev` va a Telegram y también a `dev-chat`; una `warning` se queda en `mail`, sea del equipo que sea; y lo que no coincide con ninguna cae en el receptor de la raíz. La buena práctica: el token del bot va en un fichero (`bot_token_file`) fuera del repositorio, nunca en el YAML.

```mermaid
flowchart TB
    C{"<b>severity=critical</b>"}:::dato
    CR["<b>critical</b><br><small>Telegram · continue: true</small>"]:::riesgo
    W{"<b>severity=warning</b>"}:::dato
    MW["<b>mail</b><br><small>ops@lab</small>"]:::pieza
    D{"<b>team=dev</b>"}:::dato
    DEV["<b>dev-chat</b><br><small>dev@lab</small>"]:::pieza
    MAIL["<b>mail</b><br><small>la raíz</small>"]:::pieza
    C -- sí --> CR --> W
    C -- no --> W
    W -- sí --> MW
    W -- no --> D
    D -- sí --> DEV
    D -- no --> MAIL
    classDef pieza fill:#64748b22,stroke:#64748b,stroke-width:1.5px
    classDef dato fill:#2563eb22,stroke:#2563eb,stroke-width:1.5px
    classDef riesgo fill:#dc262622,stroke:#dc2626,stroke-width:1.5px
```

<p class="pie" markdown>Sin el `continue: true` de la rama crítica, una alerta grave del equipo de desarrollo se quedaría en Telegram y no llegaría a `dev-chat`.</p>

`amtool config routes test severity=critical team=dev` dice a qué receptores llegaría una alerta con esas etiquetas, sin provocar nada.

#### group_wait, group_interval y repeat_interval

**Para qué.** Si se cae una máquina llegan diez alertas casi a la vez, y lo útil es un mensaje con las diez. `group_by` dice qué etiquetas tienen que compartir las alertas para viajar juntas: con `[alertname, service]`, las de la misma regla y el mismo servicio. Tres tiempos deciden cuándo sale cada mensaje:

| Tiempo | Qué hace | En el laboratorio |
|---|---|---|
| `group_wait` | Espera tras la primera alerta de un grupo, por si llegan más, y las manda juntas | 30 s |
| `group_interval` | Cada cuánto se manda lo que ha cambiado en un grupo ya avisado (alertas nuevas o resueltas) | 5 min |
| `repeat_interval` | Cada cuánto se recuerda un grupo que sigue igual | 4 h |

Por eso la resolución de una alarma no llega al instante, sino en el siguiente `group_interval`.

#### Inhibición

**Para qué.** Cuando una máquina se cae, todo lo que corre en ella dispara su alerta, y el aviso que importa (la máquina está caída) se pierde entre los demás. Una regla de inhibición dice: mientras esté activa una alerta que cumpla `source_matchers`, calla las que cumplan `target_matchers` y tengan el mismo valor en las etiquetas de `equal`.

**Las dos del laboratorio.** Con `HostDown{host="app01"}` en `firing`, se callan las alertas de app01; con `AppDown`, las demás de `service="app"` del mismo entorno. Las inhibidas se ven en la interfaz, pero no llegan a ningún receptor. Se usa `host` y no `instance` porque `host` lo llevan todas las alertas de la misma máquina, mientras que `instance` cambia con el puerto de cada exporter.

!!! ojo "Sin `equal`, la inhibición calla de más"
    Sin `equal`, un host caído callaría las alertas de todas las demás máquinas, y una segunda avería pasaría
    sin aviso.

#### Silencios

**Para qué.** Hay fallos que se provocan a propósito: una actualización, un reinicio programado, una prueba. Un **silencio** calla unas alertas durante un tiempo, con autor y un comentario de por qué. Se crea desde la web de Alertmanager (puerto 29093) o con `amtool`:

```bash
amtool --alertmanager.url=http://mon01:29093 silence add \
  service=db env=dev --duration=2h \
  --author=ops --comment="Mantenimiento PostgreSQL 17.x, ticket #142"
amtool --alertmanager.url=http://mon01:29093 silence query
amtool --alertmanager.url=http://mon01:29093 silence expire <id>
```

Los silencios se guardan en el directorio de datos de Alertmanager (`--storage.path`): con un volumen, sobreviven a un reinicio. Lo que no guarda es el historial de alertas, y por eso las alarmas terminan en incidencias ([Categorizar, notificar y tratar](#categorizar-notificar-y-tratar)).

#### Receptores y plantillas

**Los tres canales del laboratorio:**

- **Correo**, para lo que puede esperar, contra Mailpit, un servidor de correo de pruebas. En una empresa sería su servidor de correo, con usuario y TLS.
- **Telegram**, para lo que tiene que sonar en el móvil. El bot se crea hablando con @BotFather, que entrega su token; se mete el bot en un grupo, se escribe algo y `https://api.telegram.org/bot<TOKEN>/getUpdates` devuelve el `chat_id` del grupo (negativo).
- **Webhook**, un `POST` con JSON a una URL. Es el que lleva las alarmas al gestor de incidencias.

La buena práctica: `send_resolved: true` explícito en todos, para que también se avise cuando la alerta se resuelve.

**La plantilla propia.** Las de fábrica son largas para un móvil. Esta, en `alertmanager/templates/lab.tmpl`, deja solo lo necesario, con el estado (FIRING o RESOLVED) delante:

```text
{{ define "lab.title" }}[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.alertname }} ({{ .GroupLabels.service }}){{ end }}

{{ define "lab.telegram" }}
<b>{{ template "lab.title" . }}</b>
{{ range .Alerts -}}
<b>{{ .Status | toUpper }} · {{ .Labels.severity }}</b> {{ .Annotations.summary }}
Desde: {{ .StartsAt.Format "02/01 15:04" }} · Origen: {{ .Labels.origen }}
{{ if .Annotations.runbook }}<a href="{{ .Annotations.runbook }}">Runbook</a>{{ end }}
{{ end }}
{{ end }}
```

`define` da nombre a un trozo de plantilla, y el receptor lo usa con `message: '{{ template "lab.telegram" . }}'`.

#### Mailpit como SMTP de pruebas

**Para qué.** Probar avisos por correo sin mandar correos de verdad. Mailpit acepta todo lo que le llega por el puerto 1025, no lo entrega a nadie y lo enseña en una web. En el compose de mon01 solo se publica la web, en el 28025 del curso; el 1025 se queda dentro de la red del compose, que es por donde llega Alertmanager:

```yaml
  mailpit:
    image: axllent/mailpit:v1.22.3
    ports: ["28025:28025"]
    environment:
      MP_UI_BIND_ADDR: "0.0.0.0:28025"    # la web escucha en el puerto del curso
      MP_MAX_MESSAGES: 500
```

#### Alertas sintéticas

**Para qué.** Probar rutas, grupos, inhibiciones y plantillas sin romper el servicio ni esperar minutos. Una alerta sintética se mete a mano en Alertmanager con las etiquetas que se quiera:

```bash
amtool --alertmanager.url=http://mon01:29093 alert add alertname=PruebaAviso severity=warning service=app \
  --annotation=summary="Prueba de la ruta warning"
```

Para Alertmanager son alertas como las demás: pasan por rutas, grupos, inhibiciones y silencios. No salen en *Alerts* de Prometheus, y se resuelven solas a los 5 minutos (`resolve_timeout`).

### Categorizar, notificar y tratar

**Qué es tratar una alarma.** Notificar es avisar a alguien; tratar es que el aviso se convierta en una **incidencia**, un registro con dueño desde que el problema aparece hasta que se cierra. Para eso cada alarma llega ya clasificada, con datos que salen de las etiquetas de la regla y de Alertmanager:

| Parámetro | Valores | De dónde sale |
|---|---|---|
| Fecha de creación | `startsAt` (hora UTC) | Alertmanager, del primer envío |
| Fecha de resolución | `endsAt` | Alertmanager, cuando se resuelve |
| Origen | prometheus, cadvisor, loki, docker-events | Etiqueta `origen` |
| Criticidad | critical, warning, info | Etiqueta `severity` |
| Servicio / entorno | app, db, web, node; dev, pre | Etiquetas `service`, `env` |
| Equipo | ops, dev | Etiqueta `team` |
| Estado | firing, resolved | Campo `status` |
| Identidad | `fingerprint` | Alertmanager; empareja el disparo con su resolución |

Es lo que se rellena para cada alarma en la tabla de verificación de la práctica evaluable.

#### El JSON del webhook

**Qué llega.** Un receptor con `webhook_configs` manda a una URL un `POST` con los datos del grupo de alertas. Este es un ejemplo, que la demostración usa como fichero de prueba:

```json
{
  "version": "4",
  "groupKey": "{}/{severity=\"critical\"}:{alertname=\"AppHighErrorRate\", service=\"app\"}",
  "truncatedAlerts": 0,
  "status": "firing",
  "receiver": "critical",
  "groupLabels": { "alertname": "AppHighErrorRate", "service": "app" },
  "commonLabels": {
    "alertname": "AppHighErrorRate", "env": "dev", "origen": "prometheus",
    "service": "app", "severity": "critical", "team": "ops"
  },
  "commonAnnotations": {
    "summary": "Errores 5xx al 2.4% en app",
    "runbook": "https://wiki.lab/runbooks/app-errors"
  },
  "externalURL": "http://mon01:29093",
  "alerts": [
    {
      "status": "firing",
      "labels": {
        "alertname": "AppHighErrorRate", "env": "dev", "origen": "prometheus",
        "service": "app", "severity": "critical", "team": "ops"
      },
      "annotations": {
        "summary": "Errores 5xx al 2.4% en app",
        "description": "El ratio de errores lleva 5 min por encima del 1 % (SLA). Revisa la API y el servicio db en app01.",
        "runbook": "https://wiki.lab/runbooks/app-errors"
      },
      "startsAt": "2026-11-12T09:41:30.512Z",
      "endsAt": "0001-01-01T00:00:00Z",
      "generatorURL": "http://mon01:29090/graph?g0.expr=app%3Aerrors%3Aratio5m+%3E+0.01",
      "fingerprint": "5c2f8a1e9b0d4c77"
    }
  ]
}
```

Lo que usa el receptor está en la lista `alerts`: de cada alerta, su `status` (el de fuera es el del grupo entero), sus etiquetas y anotaciones, sus fechas y su `fingerprint`, que es siempre el mismo para la misma alerta y sirve para saber qué incidencia cerrar.

#### Receptor Flask que abre y cierra incidencias en Gitea

**Por qué un programa en medio.** Alertmanager manda su JSON y Gitea tiene una API para crear y cerrar incidencias, pero alguien tiene que traducir uno a otro y recordar qué alerta es qué incidencia. En el laboratorio lo hace un script corto en Python con Flask (una librería para montar un servicio web en pocas líneas), que corre como contenedor con el nombre `tickets`.

**Dónde van las incidencias.** Gitea es un servidor de Git ligero con repositorios, incidencias (*issues*) y una API, como un GitHub propio. Corre en un contenedor de mon01 desde la práctica evaluable de esta unidad; el 12 de febrero Despliegue lo lleva a `gitea01` en su unidad de integración continua, y ese día solo hay que cambiar `GITEA_URL`.

```python
import json, os, requests
from flask import Flask, request

GITEA = os.environ.get("GITEA_URL", "http://gitea:23001")
REPO = os.environ.get("GITEA_REPO", "ops/incidencias")
HDR = {"Authorization": "token " + os.environ["GITEA_TOKEN"]}
API = f"{GITEA}/api/v1/repos/{REPO}/issues"
STATE_FILE = os.environ.get("STATE_FILE", "/data/issues.json")

app = Flask(__name__)
state = json.load(open(STATE_FILE)) if os.path.exists(STATE_FILE) else {}

def save():
    json.dump(state, open(STATE_FILE, "w"))

def abrir(a):
    l, an = a["labels"], a.get("annotations", {})
    titulo = f"[{l.get('severity','?')}][{l.get('service','?')}] {l['alertname']}: {an.get('summary','')}"
    cuerpo = (f"**Inicio:** {a['startsAt']}\n**Origen:** {l.get('origen','?')} · "
              f"**Entorno:** {l.get('env','?')} · **Equipo:** {l.get('team','?')}\n\n"
              f"{an.get('description','')}\n\nRunbook: {an.get('runbook','-')}\n"
              f"Consulta: {a.get('generatorURL','')}\n\n`fingerprint: {a['fingerprint']}`")
    r = requests.post(API, json={"title": titulo, "body": cuerpo}, headers=HDR, timeout=10)
    r.raise_for_status()
    state[a["fingerprint"]] = r.json()["number"]

def cerrar(a):
    num = state.pop(a["fingerprint"], None)
    if num is None:
        return
    requests.post(f"{API}/{num}/comments", headers=HDR, timeout=10,
                  json={"body": f"Resuelta automáticamente. Fin: {a['endsAt']}"})
    requests.patch(f"{API}/{num}", json={"state": "closed"}, headers=HDR, timeout=10)

@app.post("/alertmanager")
def alertmanager():
    for a in request.get_json(force=True)["alerts"]:
        if a["status"] == "firing" and a["fingerprint"] not in state:
            abrir(a)
        elif a["status"] == "resolved":
            cerrar(a)
    save()
    return "", 204

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=25000)
```

Leído de arriba abajo:

- **La configuración sale de variables de entorno**, sobre todo el token: el código va al repositorio y el token no.
- **`state`** guarda en disco qué issue corresponde a cada `fingerprint`: así no abre dos issues para la misma alerta y sabe cuál cerrar.
- **`abrir`** crea la issue con la severidad, el servicio y el nombre en el título, y la categorización completa en el cuerpo.
- **`cerrar`** comenta la hora de fin y cierra la issue.
- **`/alertmanager`** recorre las alertas mirando el `status` de cada una y responde 204; si falla, Alertmanager reintenta.
- **`app.run`** escucha en el 25000, su puerto del curso, y en `0.0.0.0`: con el `127.0.0.1` de fábrica de Flask, Alertmanager no le llegaría desde otro contenedor.

### Demostración: incidencias en Gitea

**Qué se ve.** El último eslabón de la cadena: una alarma que abre una incidencia al dispararse y la cierra sola al resolverse. Se hace en un mon01 con la cadena de las A2.1 a A2.3 montada, y cada uno la repite en su mon01 en la práctica evaluable. Se construye desde el final, probando cada tramo por separado para saber dónde está el fallo si algo no funciona.

**1. Gitea en la pila de mon01.** Como servicio del compose de `/opt/monitoring`, en el 23001 (el 23000 es de Grafana). `ROOT_URL` es la dirección de sus enlaces, con el nombre `gitea.lab`, que se añade a la línea de mon01 del `/etc/hosts` de la VM `puesto`, mon01, app01 y el puesto de administración:

```yaml
  gitea:
    image: gitea/gitea:1.24
    ports: ["23001:23001"]
    environment:
      GITEA__server__HTTP_PORT: "23001"
      GITEA__server__ROOT_URL: "http://gitea.lab:23001/"
    volumes:
      - gitea_data:/data
```

En la primera visita a `http://gitea.lab:23001` se elige SQLite como base de datos y se crea el administrador. Después se crea la organización `ops` con cuatro repositorios vacíos: `incidencias`, para las issues, y `alerting`, `servicio` y `monitoring`, que serán el remoto de los del curso. Por último, en *Configuración → Aplicaciones → Generar token*, un token con permiso de escritura sobre issues.

**2. El receptor, como servicio `tickets`.** El script del [receptor](#receptor-flask-que-abre-y-cierra-incidencias-en-gitea) se guarda tal cual en `/opt/alerting/tickets/app.py`, con este `Dockerfile`:

```dockerfile
FROM python:3-slim
RUN pip install --no-cache-dir flask requests
COPY app.py /app.py
CMD ["python", "/app.py"]
```

y entra en la misma pila que Alertmanager, que le llega por la red del compose como `tickets`:

```yaml
  tickets:
    build: /opt/alerting/tickets
    ports: ["127.0.0.1:25000:25000"]
    env_file: /opt/alerting/secrets/tickets.env
    volumes:
      - tickets_data:/data
```

`gitea_data` y `tickets_data` se declaran en el bloque `volumes` del compose. `secrets/tickets.env`, con permisos 600, lleva `GITEA_URL=http://gitea:23001` (el nombre del servicio, porque el contenedor no lee el `/etc/hosts` de mon01), `GITEA_REPO=ops/incidencias` y `GITEA_TOKEN`. El 25000 solo se publica en `127.0.0.1`, para la prueba siguiente.

**3. El receptor, probado sin Alertmanager.** El JSON de [El JSON del webhook](#el-json-del-webhook) se guarda como `/opt/alerting/tickets/tests/firing.json`, y una copia con `"resolved"` en los dos `status` y un `endsAt` real como `resolved.json`. Se mandan los dos, reiniciando el receptor entre medias:

```bash
cd /opt/alerting/tickets
curl -s -X POST -H 'Content-Type: application/json' \
  --data @tests/firing.json http://127.0.0.1:25000/alertmanager
```

Con el primero aparece en `ops/incidencias` una issue `[critical][app] AppHighErrorRate: ...`; con el segundo, la misma issue recibe el comentario y se cierra. El reinicio demuestra que el receptor no olvida qué issue es de qué alerta.

**4. La conexión con Alertmanager.** Cada uno de los tres receptores de `alertmanager.yml` lleva, además de su correo o su Telegram, un webhook:

```yaml
    webhook_configs:
      - url: http://tickets:25000/alertmanager
        send_resolved: true
```

Va en los tres para que ninguna alarma se quede sin incidencia; una alerta que pasa por dos receptores no duplica su issue, porque su `fingerprint` ya la tiene. Se valida con `amtool check-config` y se recarga Alertmanager.

**5. La prueba con alarmas reales.** Con el bucle de tráfico corriendo se para `db`. A los pocos minutos se abren dos issues: `AppHighErrorRate`, de Prometheus, y `AppLogErrorsBurst`, del ruler de Loki, con `origen: loki`. Se arranca `db` y, en el siguiente `group_interval`, las dos se cierran solas.

**6. El mismo Gitea como remoto.** Cada repositorio del curso (`alerting` y `monitoring` en mon01, `servicio` en app01) recibe su remoto `http://gitea.lab:23001/ops/<repo>.git` y se hace el primer push. Antes del de `alerting`, `git check-ignore secrets/tickets.env` tiene que imprimir la ruta; si no, el token se sube con el resto.

**Lo que hay que llevarse.** La issue se abre con el `firing` y se cierra con el `resolved` sin que nadie la toque. Su título y su cuerpo llevan fecha, origen, criticidad y servicio, que salen de las etiquetas de la regla: la categorización del criterio e se decide al escribir la regla. Y como el cierre depende del `resolved`, una alarma inhibida mientras se resuelve puede dejar su issue abierta: por eso, al verificar, la alarma que inhibe a las demás se provoca la última.

### A2.3 Alertmanager (sesión 14)

<span class="et et-obj">Objetivo</span> **Qué vas a conseguir:** que las alarmas de la A2.2 te lleguen de verdad, al correo o a Telegram según su gravedad, juntas cuando son del mismo problema y calladas cuando sobran.

**Para qué sirve en un trabajo:** una alarma que nadie recibe no sirve, y una que manda cien mensajes por la misma avería hace que se dejen de leer. Decidir quién recibe qué, y callar las alertas durante un mantenimiento, es parte del trabajo diario de guardia.

<span class="et et-pre">Antes de empezar</span>

- Las reglas de la A2.2 cargadas en Prometheus y Loki.
- El bot de Telegram en un grupo de pruebas, con el `chat_id` (el número que identifica el grupo) apuntado y el token en `/opt/alerting/secrets/tg_token`. Si no tienes bot, deja de momento el receptor `critical` así, con correo en lugar de Telegram, y la captura final será la de Mailpit:

    ```yaml
      - name: critical
        email_configs:
          - to: guardia@lab
            send_resolved: true
    ```

- La teoría de hoy: [Alertmanager](#alertmanager), de [El árbol de rutas](#el-arbol-de-rutas) a [Alertas sintéticas](#alertas-sinteticas).

<span class="et et-pas">Pasos</span>

Hoy trabajas en **mon01**. El recorrido: preparas los contenedores (paso 1), cargas la configuración y compruebas las rutas sin mandar nada (paso 2), pones un silencio (paso 3), lanzas alertas de prueba para ver adónde llega cada una (paso 4) y lo guardas en Git (paso 5). El silencio va antes de las pruebas porque una de ellas tiene que esquivarlo.

1. **Prepara la pila.** *En mon01.*

    Alertmanager es el que recibe las alertas de Prometheus y decide a quién avisar. Mailpit es un servidor de correo de pruebas: recoge los correos y los enseña en una web. En el `compose.yml` de `/opt/monitoring`, añade el servicio `mailpit` de [Mailpit como SMTP de pruebas](#mailpit-como-smtp-de-pruebas) y cambia el de `alertmanager` por este, que lee su configuración del repositorio `alerting` y guarda sus datos en un volumen (sin él, cada reinicio borraría los silencios):

    ```yaml
      alertmanager:
        image: prom/alertmanager:v0.28.1
        command:
          - --config.file=/etc/alertmanager/alertmanager.yml
          - --web.listen-address=:29093
          - --storage.path=/alertmanager
        volumes:
          - /opt/alerting/alertmanager:/etc/alertmanager:ro
          - /opt/alerting/secrets:/etc/alertmanager/secrets:ro
          - alertmanager_data:/alertmanager
        ports: ["29093:29093"]
        restart: unless-stopped
    ```

    Declara `alertmanager_data:` en el bloque `volumes` del final del fichero.

    **Sale bien si:** `docker compose config` no da errores.

2. **Carga la configuración y comprueba las rutas.** *En mon01.*

    Las rutas deciden a qué receptor (correo, Telegram) va cada alerta según sus etiquetas. Crea los directorios con `mkdir -p /opt/alerting/alertmanager/templates`. Copia el `alertmanager.yml` de [El árbol de rutas](#el-arbol-de-rutas) en `/opt/alerting/alertmanager/`, cambiando `chat_id` por el de tu grupo, y la plantilla de [Receptores y plantillas](#receptores-y-plantillas) en `alertmanager/templates/lab.tmpl`. Después valida, levanta y pregunta adónde iría una alerta de ejemplo:

    ```bash
    cd /opt/monitoring
    docker compose run --rm --entrypoint amtool alertmanager check-config /etc/alertmanager/alertmanager.yml
    docker compose up -d mailpit alertmanager
    docker compose exec alertmanager amtool config routes test \
      --config.file=/etc/alertmanager/alertmanager.yml severity=warning team=dev
    ```

    Mira a qué receptor llega una `warning` del equipo de desarrollo. Si no te convence, cambia el orden de las rutas o su `continue` (que la alerta siga buscando más rutas después de encajar en una), y apunta la decisión y su porqué en el README.

    **Sale bien si:** `check-config` dice `SUCCESS` y `http://mon01:29093` responde.

3. **Pon un silencio.** *En mon01.*

    Un silencio calla durante un tiempo las alertas que cumplen una condición, como se hace antes de un mantenimiento para no despertar a nadie. Crea uno de una hora sobre `service=db` y déjalo puesto:

    ```bash
    AM="docker compose exec alertmanager amtool --alertmanager.url=http://localhost:29093"
    $AM silence add service=db --duration=1h --author=ops --comment="<motivo del mantenimiento>"
    ```

    **Sale bien si:** `$AM silence query` lo lista y se ve en la pestaña *Silences* de la web.

4. **Prueba con alertas sintéticas.** *En mon01, con Telegram y Mailpit abiertos.*

    Las alertas sintéticas son de mentira: las lanzas a mano y llegan en segundos, sin esperar a una avería. Llevan `service=app` para que el silencio no las tape. Las dos `PruebaGrupo` tienen que llegar juntas; `HostDown` tiene que tapar a `PruebaInhibida` (eso es una inhibición: si el servidor está caído, no hace falta avisar también de lo que corre en él):

    ```bash
    $AM alert add alertname=PruebaGrupo severity=critical service=app instance=r1 --annotation=summary="Prueba de grupo"
    $AM alert add alertname=PruebaGrupo severity=critical service=app instance=r2 --annotation=summary="Prueba de grupo"
    $AM alert add alertname=PruebaAviso severity=warning service=app --annotation=summary="Prueba de la ruta warning"
    $AM alert add alertname=HostDown severity=critical service=node host=app01
    $AM alert add alertname=PruebaInhibida severity=warning service=app host=app01
    $AM alert query -i      # las inhibidas
    ```

    A los 5 minutos se resuelven solas, y el aviso de resuelta llega en el siguiente `group_interval` (cada cuánto se manda la actualización de un grupo ya avisado).

    **Sale bien si:** las dos `PruebaGrupo` llegan a Telegram en un solo mensaje; `PruebaAviso` llega a Mailpit (`http://mon01:28025`) y no a Telegram; `PruebaInhibida` sale en `alert query -i` y no llega a ningún sitio; al rato llega el `[RESOLVED]`.

5. **Guárdalo sin secretos.** *En mon01.*

    Haz commit de `alertmanager/` en `alerting` y del `compose.yml` en `monitoring`. El token de Telegram se queda fuera, en `secrets/`.

    **Sale bien si:** `git status` no muestra nada de `secrets/`.

<span class="et et-com">Comprobación</span>

- [ ] `amtool check-config` sin errores.
- [ ] En Mailpit hay un correo de `PruebaAviso` y ninguno de `PruebaInhibida`, que `alert query -i` da como inhibida.
- [ ] En Telegram, un `[FIRING:2]` con dos alertas y después su `[RESOLVED]`.
- [ ] El silencio se ve en la interfaz de Alertmanager.
- [ ] No hay secretos en el repositorio.

<span class="et et-ent">Para practicar</span>

- **Guarda** `alertmanager/` y el `compose.yml` de mon01 en sus repositorios, y una captura de Telegram con la notificación agrupada en `docs/capturas/` de `alerting`. Los usa la práctica evaluable para los criterios d y e.
- **Piensa:** si se cae app01 de verdad, ¿cuántos mensajes recibirías con esta configuración y cuántos sin la inhibición?
- **Antes de la práctica evaluable**, repasa la [demostración](#demostracion-incidencias-en-gitea): allí añades a estos receptores el webhook que abre incidencias en Gitea, y la repites en tu mon01.

<span class="et et-ext">Si te sobra tiempo</span> Para ver cómo cambia el número de mensajes según cómo se agrupan: cambia en la ruta `critical` el agrupamiento a `group_by: [service]`, recarga Alertmanager y lanza dos sintéticas `critical` con `alertname` distinto y el mismo `service=app`. Compara cuántos mensajes llegan con uno y otro `group_by`, y deja el que prefieras, justificado en el README.

## Sesión 15 · Práctica evaluable

<p class="ut-meta" markdown>24 de noviembre · Práctica evaluable · <span class="dur" tabindex="0" aria-label="Procedimiento de verificación de una alarma · 10 min&#10;Trabajo en la práctica · 100 min" data-dur="Procedimiento de verificación de una alarma · 10 min&#10;Trabajo en la práctica · 100 min">:material-school:<i class="dur-barra" style="--teoria:9%"></i>:material-flask:</span></p>

Cada pieza de la cadena se ha probado ya por separado: las consultas en la A2.1, las reglas en la A2.2, Alertmanager en la A2.3 y el receptor de incidencias en la [demostración](#demostracion-incidencias-en-gitea). Falta montar el receptor en el mon01 de cada uno y seguir alarmas reales de punta a punta, del fallo a la incidencia cerrada, midiendo cuánto tarda cada tramo. Los diez primeros minutos explican el procedimiento con el que se da por buena una alarma; después va el enunciado.

### Procedimiento de verificación de una alarma

**Para qué.** Una alarma escrita no es fiable hasta que se ve que dispara, avisa, deja su incidencia y se cierra sola. Se repite para cada alarma nueva y cada cambio de umbral, `for` o ruta, siempre con los mismos pasos, y el resultado se apunta en el informe.

| Paso | Qué se hace | Qué se comprueba | Dónde |
|---|---|---|---|
| 1 | Se anota la hora y se provoca la condición | El indicador cruza el umbral | Prometheus *Graph* o Grafana |
| 2 | Se espera | La alerta pasa a `pending` | Prometheus *Alerts* |
| 3 | Se espera el `for` | Pasa a `firing`; se anota el tiempo desde el paso 1 | Prometheus *Alerts* y Alertmanager (29093) |
| 4 | Se espera `group_wait` | Llega el aviso a cada canal de la ruta, con su plantilla | Mailpit (28025) y Telegram |
| 5 | Se mira Gitea | Existe la issue con título y fecha correctos | Gitea, ops/incidencias |
| 6 | Se resuelve la causa | El indicador vuelve por debajo del umbral | Prometheus *Graph* |
| 7 | Se espera `group_interval` | Llega el aviso de resolución; se anota el tiempo desde el paso 6 | Los mismos canales |
| 8 | Se mira Gitea | La issue está cerrada con el comentario | Gitea |
| 9 | Se registra | Alarma, categoría, cómo se provoca, tiempos, canales e issue | Informe de verificación |

**Qué se hace con los tiempos.** Se comparan con lo que el servicio tolera. Si la alarma tarda más de lo aceptable, se baja el `for` o la ventana; si ha saltado por un pico sin importancia, se sube el umbral o se alarga el `for` ([Fatiga de alertas](#fatiga-de-alertas)).

### Enunciado

La práctica cierra la unidad en tu mon01: montas el tratamiento de incidencias de la demostración, verificas con él cuatro alarmas reales de orígenes distintos (`HostDown`, `AppHighErrorRate`, `AppLogErrorsBurst` de Loki y `ContainerOOM` de eventos Docker) y entregas el repositorio `alerting` en orden.

Entrega el repositorio `alerting`, comprimido y subido por Aules, con `rules.yml`, `alerts.yml`, `loki-alerts.yml`, `alertmanager.yml` (sin secretos: tokens en ficheros ignorados), el receptor de incidencias con su `Dockerfile` y el `README` que has ido completando. En `docs/` van `umbrales.md` y `cadenas.md` (A2.1) y `verificacion.md`, que escribes hoy: su tabla es a la vez el informe de verificación y la tabla de categorización, con cada alarma, su origen, criticidad y servicio y las fechas de creación y cierre de su issue.

Trabaja en este orden: primero el tratamiento de incidencias, porque sin él la verificación no deja rastro; después las provocaciones, que tardan (una alarma puede llevar un cuarto de hora de punta a punta), lanzando juntas las que no se estorban y dejando para el final la que inhibe a las demás; y al final repasa el repositorio contra la lista de abajo.

1. **Monta el tratamiento de incidencias.** Repite en tu mon01 las cuatro primeras partes de la [demostración](#demostracion-incidencias-en-gitea): Gitea con la organización `ops` y sus cuatro repositorios; el receptor `tickets` con el código, el `Dockerfile` y el compose tal cual; la prueba con `firing.json` y `resolved.json`, con un reinicio del receptor en medio; y el webhook en los tres receptores de `alertmanager.yml`. Es la parte de tratamiento del criterio e. Lo mecánico, para copiar:

    ```bash
    # en tu VM puesto, en mon01 y en app01: el nombre de Gitea va en la línea de mon01
    echo "<IP de mon01> gitea.lab" | sudo tee -a /etc/hosts
    # en mon01, con firing.json ya guardado: la copia resuelta
    cd /opt/alerting/tickets/tests
    sed 's/"firing"/"resolved"/g; s/0001-01-01T00:00:00Z/2026-11-24T10:00:00Z/' firing.json > resolved.json
    ```

    *Pista:* si el receptor responde 500 o no aparece ninguna issue, mira `docker compose logs tickets`: casi siempre es el token o `GITEA_REPO` ([Errores frecuentes](#errores-frecuentes-en-el-laboratorio)).

2. **Prepara la tabla de verificación.** Crea `docs/verificacion.md` con esta cabecera y una fila por alarma. Origen, criticidad y servicio salen de las etiquetas de la regla:

    ```markdown
    | Alarma | Origen | Criticidad | Servicio | Cómo se provoca | Firing (hora y tiempo) | Canales | Issue creada | Resolved (hora y tiempo) | Issue cerrada | Cambios |
    |---|---|---|---|---|---|---|---|---|---|---|
    ```

3. **Lanza primero las provocaciones lentas.** Provoca a la vez `ContainerOOM` y la parada de `db`, que dispara `AppHighErrorRate` desde Prometheus y `AppLogErrorsBurst` desde Loki, anotando la hora de cada una; vuelve a arrancar `db` en cuanto estén abiertas sus dos issues. Son tres grupos distintos en Alertmanager y no se pisan. Mientras esperas los cierres, adelanta los remotos del paso 6.

    *Pista:* el `oom` sale del mismo contenedor de usar y tirar de la A2.1 (en app01, `docker run --rm -m 32m python:3-slim python -c "x = bytearray(200 * 1024 * 1024)"`), y `ContainerOOM` se resuelve sola cuando el evento sale de su ventana de 10 min.

4. **Deja para el final la que inhibe.** Cuando esas tres issues estén cerradas, provoca `HostDown` parando el node_exporter de app01 (`sudo systemctl stop prometheus-node-exporter`) y arráncalo en cuanto llegue el aviso. Va la última por la [inhibición](#inhibicion): mientras está activa, calla las alertas de app01.

5. **Rellena las filas y ajusta lo que piden los tiempos.** Completa la fila de cada alarma siguiendo el [procedimiento](#procedimiento-de-verificacion-de-una-alarma); las horas de firing y resolved están en el cuerpo y el comentario de su issue. Rellena *Cambios* y aplícalos en `alerts.yml`: qué `for` o ventana bajas si tarda demasiado, o qué subes si ha saltado por un pico. Valida, recarga y haz commit de `docs/verificacion.md` y de las reglas que hayas tocado.

6. **Dale un remoto a cada repositorio.** Haz commit de `tickets/` y `alertmanager/` en `alerting`, con los enlaces a las issues en el README en lugar de la nota del remoto pendiente, y del `compose.yml` en `monitoring`. Da a cada copia (`alerting` y `monitoring` en mon01, `servicio` en app01) su remoto y haz el primer push, con el usuario administrador de Gitea; las unidades siguientes trabajan sobre esos remotos:

    ```bash
    git remote add origin http://gitea.lab:23001/ops/<repo>.git    # alerting, monitoring o servicio
    git push -u origin HEAD
    ```

    Antes de comprimir `alerting`, comprueba que no hay secretos ni en el repositorio ni en su historial.

    *Pista:* antes del primer push de `alerting`, `git check-ignore secrets/tickets.env` tiene que imprimir la ruta.

- [ ] `promtool check rules`, `promtool test rules` y `amtool check-config` sin errores (A2.2, pasos 2 y 4, y A2.3, paso 2)
- [ ] Las alertas sobre indicadores calculados (ratios, percentiles, porcentajes) usan su métrica grabada (A2.2, paso 1)
- [ ] Todas las reglas llevan las cinco etiquetas de categorización (A2.2, pasos 1 y 5)
- [ ] El receptor abre y cierra issues con `firing.json` y `resolved.json`, también tras un reinicio, y los tres receptores de Alertmanager llevan el webhook
- [ ] Activación y recuperación probadas para las cuatro alarmas, con issue abierta y cerrada y los tiempos en `docs/verificacion.md`
- [ ] `alerting`, `monitoring` y `servicio` con su último commit en Gitea, y `secrets/` fuera de `alerting`

| Criterio | RA1 | Peso |
|---|---|---|
| Umbrales de contadores y cadenas de eventos definidos desde la documentación (A2.1) | b | 25 % |
| Recording rules de agregación y correlación que generan indicadores nuevos (A2.2) | c | 20 % |
| Eventos integrados en el gestor de alarmas con activación y recuperación probadas y almacenadas fuera (A2.2, A2.3 y la verificación de esta práctica) | d | 30 % |
| Categorización por fecha, origen, criticidad y servicio, con notificación y tratamiento (A2.3, el receptor de incidencias y la tabla de verificación) | e | 25 % |

## Errores frecuentes en el laboratorio

**La división devuelve "no data".** Las etiquetas de los dos lados no coinciden. Se ejecuta cada lado por separado, se comparan las etiquetas y se añade el `ignoring(...)`, `on(...)` o `sum by` que falte. Con cAdvisor, el culpable habitual es `id`.

**La consulta de reinicios da 0 aunque la API se haya reiniciado.** Se está usando `container_start_time_seconds` de cAdvisor, que no cambia al reiniciar, o `increase` en lugar de `changes()`. También pasa si los reinicios van tan seguidos que Prometheus no ve alguno.

**El exporter cae y no salta nada.** `up == 0` no dispara si la serie ha desaparecido. Se añade `or absent(up{job="..."})`.

**La alerta nunca sale de `pending`.** La expresión devuelve series de forma intermitente, normalmente por una ventana demasiado corta, y cada hueco reinicia el `for`. Se amplía la ventana a `[5m]`.

**Las reglas nuevas no aparecen después de recargar Prometheus.** La recarga ha fallado y Prometheus sigue con lo anterior (`prometheus_config_last_reload_successful` vale 0). `docker compose logs --tail 20 prometheus` dice qué fichero y qué línea fallan.

**El ruler de Loki no carga las reglas.** El fichero tiene que quedar en `/loki/rules/fake/` dentro del contenedor; en `/loki/rules/` se ignora sin error.

**El correo no llega a Mailpit.** Falta `smtp_require_tls: false` en `global`, o Alertmanager y Mailpit no están en el mismo compose (`dial tcp: lookup mailpit`).

**La web de Mailpit o la de Gitea no responde en su puerto.** El contenedor se publica en la franja del curso, pero dentro sigue en su puerto de fábrica. Falta `MP_UI_BIND_ADDR` en Mailpit o `GITEA__server__HTTP_PORT` en Gitea.

**Telegram devuelve `Bad Request: chat not found`.** El bot no está en el grupo, o el `chat_id` es el de un usuario y no el del grupo (negativo). Se escribe en el grupo y se consulta `https://api.telegram.org/bot<TOKEN>/getUpdates`.

**La inhibición no inhibe.** Falta `equal` o la etiqueta no tiene el mismo valor en las dos alertas. `amtool alert query -i` dice qué alertas están inhibidas.

**El webhook crea una issue por cada repetición.** El fichero de estado no persiste entre reinicios del contenedor: falta el volumen en `/data`.

**El receptor de incidencias responde 500 y no aparece ninguna issue.** Token sin permiso de escritura sobre issues o `GITEA_REPO` mal escrito; lo dicen los logs de `tickets`. Si es un error de conexión, `GITEA_URL` tiene que ser `http://gitea:23001`, no `gitea.lab`, que el contenedor no resuelve.

Los enlaces para ampliar están en [Para ampliar](../ampliacion.md#ut2-umbrales-agregacion-y-gestion-de-alarmas).
