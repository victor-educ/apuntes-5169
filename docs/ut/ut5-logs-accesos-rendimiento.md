# UT5 · Explotación de logs, accesos y rendimiento

<p class="ut-meta">Módulo 5169 · 14 h · Formación en empresa (29 mar a 9 jun 2027) · RA3 CE a, b, c, d</p>

Hasta aquí todo lo habéis hecho sobre el laboratorio: la pila de observabilidad de mon01 (UT1), las alarmas (UT2), la monitorización de seguridad (UT3) y los KPI y las pruebas de carga (UT4). Esta unidad y la siguiente (UT6, copias de seguridad) se cursan en la empresa, entre el 19 de abril y el 9 de junio de 2027, sobre un sistema real que os asigne el tutor. No hay sesiones numeradas ni laboratorio compartido: hay cuatro actividades (A5.1 a A5.4), una actividad de cierre y una ficha de evidencias que firma el tutor de empresa. Este documento es la guía de referencia para hacer ese trabajo con criterio, y al mismo tiempo la lista de lo que tenéis que traer de vuelta. Cuando volváis, la UT7 (actualización y vulnerabilidades) y la UT8 (terminación segura) cierran el módulo.

Una regla desde el principio: todo dato de la empresa que aparezca en las evidencias se anonimiza. Nombres de host, IPs públicas, nombres de usuario, dominios, rutas con nombre de cliente. Sustituidlos por equivalentes (`web01`, `203.0.113.45`, `usuario_a`, `empresa.example`) antes de pegar nada en el informe. Las redes 192.0.2.0/24, 198.51.100.0/24 y 203.0.113.0/24 están reservadas por la RFC 5737 justo para esto, para documentación, y el dominio `example` por la RFC 2606. Si dudáis de si algo se puede incluir, preguntad al tutor antes de incluirlo, no después.

## Qué tienes que saber hacer al terminar

- Revisar los registros de un servicio de forma periódica y sistemática, saber qué buscar en ellos y reportar los fallos encontrados como una incidencia útil (CE 3a).
- Monitorizar los accesos a un sistema (SSH, proxy inverso, aplicación), reconocer patrones de fuerza bruta y password spraying, y configurar o revisar el mecanismo de bloqueo (fail2ban, CrowdSec o el que use la empresa) (CE 3b).
- Analizar un fallo o un reinicio a partir del código de salida, los volcados de memoria y los registros de error, llegar a una hipótesis, corregir el origen y verificar que no se repite (CE 3c).
- Tomar una línea base de rendimiento de CPU, memoria, disco y red, compararla con un periodo de carga y proponer una acción justificada (CE 3d).

## Antes de entrar en detalle

Un caso que ya os ha pasado en el laboratorio: el jueves a las tres de la tarde la API de `app01` empieza a devolver algún 502. Nadie mira nada porque "funciona casi siempre". El lunes se descubre que el contenedor lleva cuatro días reiniciándose cada veinte minutos por falta de memoria, que una IP de fuera probó ochenta usuarios distintos por SSH y que el disco está al 94 % porque nadie configuró la rotación de logs. Nada era grave el jueves; el lunes son tres incidencias a la vez. Lo que queremos conseguir cabe en una frase: sentaros diez minutos cada mañana delante de un sistema real, ver lo que va mal antes de que se note, y contarlo de forma que otra persona pueda actuar.

| Herramienta o concepto | Qué es, en una frase | Para qué la usamos en esta unidad |
|---|---|---|
| journald y `journalctl` | El registro central de systemd con lo que escriben los servicios y el kernel, y el comando para leerlo | Buscar errores por prioridad y servicio, y vigilar el disco que ocupa |
| `docker logs` | El comando que muestra lo que un contenedor ha escrito por pantalla | Revisar los errores de las últimas 24 h contenedor por contenedor |
| Loki y LogQL | La base de datos de logs de UT1 y su lenguaje de consulta, parecido al de Prometheus | Guardar las búsquedas en un panel de Grafana y convertirlas en alertas |
| logrotate | Un programa que cada noche comprime y borra logs antiguos para que el disco no se llene | Comprobar que los logs de nginx y de la aplicación tienen retención |
| fail2ban | Un vigilante que lee los logs, cuenta los fallos de cada IP y la bloquea un rato en el firewall | Cortar la fuerza bruta contra SSH, nginx y la aplicación |
| CrowdSec | Lo mismo que fail2ban, con una lista de IPs maliciosas compartida entre quienes lo usan | Alternativa con varios hosts o cuando se bloquea en OPNsense |
| nftables | El firewall del kernel de Linux, el mismo de UT3 | Ver con vuestros ojos que la IP bloqueada lo está de verdad |
| cgroups y OOM killer | El mecanismo del kernel que limita la memoria de un contenedor, y el proceso que mata al que se pasa | Entender por qué un contenedor muere con código 137 |
| Core dumps (gdb, py-spy, jmap) | Una copia de la memoria de un proceso al morir, y las herramientas que la leen según el lenguaje | Saber en qué función ha reventado un servicio en vez de adivinarlo |
| Prometheus, node_exporter, cAdvisor y `sar` | La pila de métricas de mon01 (recolector y agentes de host y contenedores), y el grabador a fichero que la sustituye si no hay servidor | Tomar la línea base de CPU, memoria, disco y red y compararla con un día de carga |
| Método USE | Tres preguntas por recurso (uso, saturación, errores) para no dejarse nada | Recorrer CPU, memoria, disco y red con el mismo guion |

Cómo está organizada la unidad: primero cómo trabajar en la empresa, porque marca qué podéis tocar y qué no. Los cuatro apartados técnicos van en el orden de los criterios de evaluación, que es también el de la rutina diaria. Primero los logs (CE 3a), porque ahí aparece todo lo demás. Después los accesos (CE 3b), un tipo concreto de log con su patrón y su herramienta de respuesta. Después los fallos y reinicios (CE 3c), que juntan lo que dice el log con lo que dice el kernel. Y por último el rendimiento (CE 3d), que necesita una semana de datos: empezad a grabarlos el primer día aunque sea lo último que analicéis.

!!! info "Lo que necesitas de la otra asignatura"
    Mientras hacéis esta unidad estáis también en la empresa con la UT4 de 5166, Nube pública
    ([https://victor-educ.github.io/apuntes-5166/ut/ut4-nube-publica/](https://victor-educ.github.io/apuntes-5166/ut/ut4-nube-publica/)).
    Son las mismas semanas y, casi seguro, los mismos sistemas: la nube de la empresa donde en 5166 desplegáis es
    donde aquí revisáis logs, accesos y rendimiento, así que acordad con el tutor un único sistema para las dos
    asignaturas y reutilizad las evidencias que sirvan para ambas (una captura de Grafana anonimizada vale en las dos).
    De 5166 os hacen falta además dos cosas anteriores: el firewall nftables de la UT3, para entender dónde mete
    fail2ban su tabla, y la pila de monitorización de la UT7 (marzo), que es la versión definitiva del mon01 sobre
    el que se construyen los paneles de línea base.

## Cómo trabajar en la empresa

Cada empresa tiene sus herramientas. Unas tendrán Loki y Grafana como en mon01, otras tendrán Elastic y Kibana, Graylog, Datadog, o simplemente ficheros en `/var/log` y un `grep`. Lo que evaluamos no es la herramienta, es el método: qué buscáis, cómo lo justificáis y qué hacéis con lo que encontráis. Por eso cada apartado de esta unidad da primero el método y después los comandos para el caso más habitual (Linux con Docker, journald y nginx), con su equivalente en Loki cuando existe. Si en la empresa hay otra cosa, adaptad el comando y anotad en la evidencia cuál habéis usado.

Lo segundo que hay que tener claro es el alcance. Vais a tocar sistemas en producción o cerca de producción. Todo lo que sea lectura (leer logs, consultar métricas, listar IPs bloqueadas) lo podéis hacer con la autorización general del tutor. Todo lo que cambie algo (activar una jail, cambiar un límite de memoria, reiniciar un servicio) se acuerda antes con el tutor, se hace en el horario que él diga y se apunta. Una evidencia que diga "reinicié el contenedor para comprobar" sin que el tutor lo supiera es una evidencia que no vale, aunque el resultado sea correcto.

```mermaid
flowchart LR
    D[Diario 10 min] --> D1[Errores de las ultimas 24 h]
    D --> D2[Accesos fallidos y baneos]
    D --> D3[Reinicios y OOM]
    D --> D4[Panel de rendimiento vs linea base]
    S[Semanal 45 min] --> S1[Tendencias 7 dias]
    S --> S2[Incidencias abiertas y cerradas]
    S --> S3[Espacio en disco y rotacion]
    S --> S4[Revisar ignoreip y listas de baneo]
    D1 --> I[Incidencia si procede]
    D2 --> I
    D3 --> I
    S1 --> R[Informe semanal al tutor]
```

Ese esquema es lo que al final de la estancia tenéis que entregar convertido en un procedimiento de dos páginas con vuestros comandos. Guardad desde el primer día lo que ejecutáis: un fichero de texto con fecha, comando y una línea de resultado os ahorrará la mitad del trabajo del cierre.

## Revisar los archivos de registro (CE 3a)

Los logs se leen de forma periódica y sistemática, no solo cuando algo falla. La diferencia entre un administrador que "mira los logs cuando pasa algo" y uno que los revisa cada mañana es que el segundo detecta el problema una semana antes, cuando todavía es una línea rara y no una caída. En UT1 montasteis Loki para tener los logs centralizados; en la empresa el sitio donde están los logs os lo dirán, y puede ser cualquiera de estos: ficheros en `/var/log`, el journal de systemd (el registro central donde systemd guarda lo que escriben los servicios), `docker logs` o `docker compose logs`, o una plataforma central.

### Qué buscar

- Niveles ERROR y FATAL, y excepciones no controladas: en Java o Python se reconocen por la traza de pila (varias líneas que empiezan por `at ...` o `File "..."`), en Go por `panic:` seguido de `goroutine`. Una excepción no controlada que se repite cada pocos minutos es un bug que alguien tiene que mirar aunque el servicio siga respondiendo.
- Salidas inesperadas: reinicios del servicio, `timeout`, `connection refused`, `connection reset by peer`, `out of memory`, `too many open files`, `no space left on device`, códigos HTTP 5xx en el proxy. Cada uno apunta a una capa distinta: `connection refused` es que no había nadie escuchando (el backend estaba caído o reiniciando), `timeout` es que había alguien pero no respondió a tiempo, `502` en nginx es el primero visto desde el proxy, `504` es el segundo.
- Cambios de volumen. El doble de líneas de lo habitual suele ser un bucle de reintentos o un escáner; el silencio total es peor, porque significa que el servicio no está escribiendo (colgado, disco lleno, o el agente de logs muerto). Loki lo mide con `count_over_time` y Prometheus con la métrica `promtail_sent_entries_total` o su equivalente en Alloy (Promtail y Alloy son los agentes que recogen los logs del host y los envían a Loki); sin nada de eso, `wc -l` sobre el fichero de hoy comparado con el de ayer.
- Avisos que anuncian un fallo futuro: `deprecated`, certificados que caducan (`certificate will expire`), `disk usage above 85%`, reconexiones a la base de datos, `slow query` en PostgreSQL si tiene activado `log_min_duration_statement`.

### Comandos para la revisión diaria

Con Docker, la ventana de tiempo es el parámetro que más se olvida. `docker logs app` sin más os vuelca todo lo que el contenedor ha escrito desde que existe, y en un contenedor con semanas de vida eso son cientos de miles de líneas.

```bash
# Errores de las últimas 24 h en un contenedor
docker logs --since 24h app 2>&1 | grep -E "ERROR|FATAL|Exception|panic:" | tail -50

# Cuántos por tipo (agrupa por la parte del mensaje tras el nivel)
docker logs --since 24h app 2>&1 | grep -oE "(ERROR|FATAL) [A-Za-z._]+" | sort | uniq -c | sort -rn | head

# Servicio de systemd: solo prioridad err o superior desde ayer
journalctl -u nginx --since yesterday -p err --no-pager

# Todo el host, prioridad crit o superior, con explicación de cada mensaje
journalctl -p crit -x --since "-24h"

# Ficheros clásicos: nginx, últimas 24 h de 5xx
awk -v d="$(date -d '-1 day' +%d/%b/%Y)" '$4 ~ d && $9 ~ /^5/' /var/log/nginx/access.log | wc -l
```

`journalctl -p` acepta el nombre o el número de prioridad syslog (0 emerg a 7 debug); `-p err` equivale a `-p 0..3`. Docker escribe stdout y stderr por separado y `docker logs` los devuelve en los dos descriptores, de ahí el `2>&1` antes del `grep`.

En Loki, las mismas búsquedas con LogQL, su lenguaje de consulta: el selector entre llaves elige el flujo de logs y lo que va detrás lo filtra o lo cuenta. La ventaja no es la sintaxis, es que la consulta se guarda en un panel de Grafana y no depende de que os acordéis del comando.

```text
{job="docker", container="app"} |~ "ERROR|FATAL|Exception"

sum by (level) (count_over_time({job="docker", container="app"} | json | level =~ "error|fatal" [24h]))

# Volumen de líneas por hora, para ver el "doble de lo habitual" o el silencio
sum(count_over_time({job="docker", container="app"}[1h]))

# 5xx en nginx en la última hora
count_over_time({job="nginx"} | pattern `<ip> - - [<_>] "<method> <uri> <_>" <status> <_>` | status >= 500 [1h])
```

En Grafana 12 estas consultas van a un dashboard "Revisión diaria" con cuatro paneles (errores por nivel, volumen por hora, 5xx, accesos fallidos) y una variable `$container`. Es el panel que abrís cada mañana, y una captura suya con fecha vale como evidencia de la revisión. Guardad también las consultas en la pestaña Explore con la estrella de favoritos (query library en Grafana 12), así el siguiente que llegue las hereda.

### Un script diario que resuma errores

Cuando no hay plataforma central, o como complemento a ella, un script en cron (el planificador de tareas de Linux) que resuma los errores por tipo y lo envíe por correo o a un canal de chat es lo que mantiene la rutina viva los días que no os acordáis. Este es un ejemplo mínimo que podéis llevar a la empresa y adaptar; lo importante es que agrupa, no que vuelque.

```bash
#!/usr/bin/env bash
# /usr/local/sbin/resumen-logs.sh : resumen diario de errores por contenedor
set -euo pipefail
DESDE="24h"
DEST="ops@empresa.example"
TMP=$(mktemp)
{
  echo "Resumen de errores $(hostname) $(date +%F) (últimas $DESDE)"
  echo
  for c in $(docker ps --format '{{.Names}}'); do
    n=$(docker logs --since "$DESDE" "$c" 2>&1 | grep -cE "ERROR|FATAL|panic:" || true)
    echo "== $c: $n líneas de error"
    if [ "$n" -gt 0 ]; then
      docker logs --since "$DESDE" "$c" 2>&1 \
        | grep -E "ERROR|FATAL|panic:" \
        | sed -E 's/^[0-9T:.-]+Z? //; s/[0-9a-f]{8,}/<id>/g; s/[0-9]{2,}/<n>/g' \
        | sort | uniq -c | sort -rn | head -10
    fi
    echo
  done
  echo "== Reinicios de contenedores en $DESDE"
  docker events --since "$DESDE" --until 0s --filter event=die --filter event=oom \
    --format '{{.Time}} {{.Actor.Attributes.name}} {{.Action}} exit={{.Actor.Attributes.exitCode}}' 2>/dev/null || true
  echo
  echo "== journal, prioridad err o superior"
  journalctl -p err --since "-$DESDE" --no-pager -q | tail -20
} > "$TMP"
mail -s "Resumen de errores $(hostname) $(date +%F)" "$DEST" < "$TMP"
rm -f "$TMP"
```

El `sed` del medio es el truco que hace útil el resumen: sustituye identificadores y números por marcadores para que "timeout en petición 48213" y "timeout en petición 48214" cuenten como el mismo error. Sin eso, el `uniq -c` no agrupa nada. Se programa en `/etc/cron.d/resumen-logs` a las 07:30 (`30 7 * * * root /usr/local/sbin/resumen-logs.sh`), o como timer de systemd si la empresa los prefiere. `docker events --until 0s` es la forma de que el comando termine en lugar de quedarse escuchando.

### Retención: logrotate y journald

Un sistema en el que nadie ha pensado en la retención acaba con el disco lleno, y un disco lleno en un host de contenedores es una caída completa (Docker no puede escribir su estado, PostgreSQL deja de aceptar escrituras). Tres sitios a revisar la primera semana en la empresa.

Los logs de Docker con el driver `json-file` (el que hay por defecto) no rotan si nadie lo configura. Lo normal es fijarlo en `/etc/docker/daemon.json` para todos los contenedores:

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "5" }
}
```

Eso limita cada contenedor a 250 MB de logs. Solo afecta a contenedores creados después del cambio; los que ya existen hay que recrearlos (`docker compose up -d --force-recreate`). Para ver cuánto ocupa ahora mismo: `du -sh /var/lib/docker/containers/*/*-json.log | sort -rh | head`. Alternativa: driver `journald`, que mete los logs del contenedor en el journal del host y hereda su retención; entonces se leen con `journalctl CONTAINER_NAME=app`.

journald tiene su propia retención en `/etc/systemd/journald.conf`: `SystemMaxUse=2G` limita el total en disco, `MaxRetentionSec=1month` la antigüedad, y `Storage=persistent` asegura que sobreviva al reinicio (con `auto`, si no existe `/var/log/journal` se queda en RAM y se pierde al reiniciar; en Debian 13 el directorio existe por defecto). Para ver y recortar:

```bash
journalctl --disk-usage
journalctl --vacuum-size=1G          # deja como mucho 1 GB
journalctl --vacuum-time=30d         # elimina lo anterior a 30 días
journalctl --verify                  # comprueba la integridad de los ficheros
```

`vacuum` solo borra ficheros de archivo, nunca el activo; si el uso no baja, primero `journalctl --rotate` y después el vacuum.

logrotate se encarga de los ficheros clásicos (`/var/log/nginx/*.log`, `/var/log/auth.log`, los de la aplicación si escribe a fichero). Cada servicio deja su configuración en `/etc/logrotate.d/`, y un ejemplo típico para una aplicación en `/srv/app/logs`:

```text
/srv/app/logs/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
```

`copytruncate` copia el fichero y vacía el original en lugar de renombrarlo; es lo que hace falta cuando el proceso mantiene el descriptor abierto y no sabe recibir un `SIGHUP` (la mayoría de aplicaciones dentro de contenedores que escriben a un volumen). El precio es que las líneas escritas entre la copia y el truncado se pierden. Con `logrotate -d /etc/logrotate.d/app` se simula sin tocar nada, y `logrotate -f` fuerza una rotación para comprobar que funciona. Documentación: la página de manual de logrotate en [man7.org](https://man7.org/linux/man-pages/man8/logrotate.8.html) y la de [journald.conf](https://man7.org/linux/man-pages/man5/journald.conf.5.html).

### Cómo se redacta una incidencia útil

Todo fallo encontrado se reporta. Reportar no es mandar un mensaje diciendo "he visto errores en app"; es abrir una incidencia con lo que alguien necesita para decidir si es urgente y por dónde empezar. Los ingredientes son siempre los mismos y os conviene tener la plantilla a mano en la herramienta que use la empresa (Jira, GitLab issues, un Gitea como el del laboratorio, o un correo estructurado).

```text
Título: [servicio] resumen de una línea del síntoma
Fecha y hora de detección: 2027-05-04 08:10 (revisión diaria)
Servicio / host: api (contenedor app en app01)
Mensaje exacto (una línea, tal cual aparece):
  ERROR sqlalchemy.pool: QueuePool limit of size 5 overflow 10 reached, connection timed out
Frecuencia: 143 apariciones entre 02:00 y 02:40; ninguna fuera de esa ventana
Impacto observado: 37 respuestas 502 en nginx en el mismo intervalo; sin quejas de usuarios
Contexto: coincide con la copia de seguridad de la BD (02:00) según el cron de db01
Fragmento de log (10 líneas alrededor, anonimizado): adjunto
Hipótesis (opcional, marcada como tal): el pool se agota porque las consultas
  se ralentizan durante el pg_dump
Gravedad propuesta: media (recurrente, sin pérdida de servicio visible)
```

El mensaje exacto es lo más valioso: es lo que se busca en Google, en la documentación y en el histórico de incidencias. Copiadlo literal, no lo parafraseéis. La frecuencia y la ventana temporal son lo segundo, porque convierten "hay errores" en "hay errores a las dos de la mañana", que ya es media investigación hecha. La hipótesis va aparte y marcada, para que nadie la confunda con un hecho.

## Monitorizar los accesos (CE 3b)

Un sistema expuesto a Internet recibe intentos de acceso desde el minuto uno. Un servidor con SSH en el puerto 22 abierto ve entre cientos y varios miles de intentos fallidos al día sin que nadie lo esté atacando en particular; son botnets que prueban credenciales por defecto contra todo lo que responde. Monitorizar los accesos es distinguir ese ruido de fondo de un ataque dirigido, y asegurarse de que el mecanismo de bloqueo hace su trabajo.

### Dónde se registran

| Origen | Dónde mirar | Qué contiene |
|---|---|---|
| SSH | `/var/log/auth.log` (Debian con rsyslog) o `journalctl _COMM=sshd` | Intentos fallidos, logins correctos, usuario inválido, clave aceptada |
| sudo, su, PAM | mismo fichero o journal, `_COMM=sudo` | Quién escaló privilegios y qué ejecutó |
| Proxy inverso (nginx) | `/var/log/nginx/access.log` y `error.log` | Intentos a rutas de administración, 401/403, escáneres |
| Aplicación | su log (stdout del contenedor) | Login fallido, usuario bloqueado, cambio de contraseña, tokens rechazados |
| Panel del hipervisor | Proxmox: `/var/log/pve/tasks/`, `journalctl -u pvedaemon`, `-u pveproxy` | Accesos al panel web, acciones sobre VM |
| VPN | WireGuard no registra handshakes por defecto; OpenVPN en su log; OPNsense en Sistema > Registro | Conexiones, orígenes, fallos de autenticación |
| Firewall | OPNsense, `nft monitor`, `journalctl -k` con reglas `log` | Conexiones rechazadas, escaneos de puertos |

En Debian 13 sin rsyslog instalado (rsyslog es el servicio clásico que reparte los mensajes del sistema en ficheros de `/var/log`), `auth.log` no existe y todo está en el journal; `journalctl _COMM=sshd --since today` es el equivalente. Si la empresa centraliza en Loki con Alloy o Promtail, el job suele llamarse `auth` o `syslog` y ahí van todas las búsquedas de abajo con `|=`.

### Qué se busca: patrones de fuerza bruta y password spraying

Fuerza bruta clásica: una misma IP prueba muchas contraseñas contra uno o pocos usuarios. En `auth.log` se ve así (anonimizado):

```text
May  4 03:12:41 web01 sshd[18422]: Failed password for root from 203.0.113.45 port 51204 ssh2
May  4 03:12:44 web01 sshd[18422]: Failed password for root from 203.0.113.45 port 51204 ssh2
May  4 03:12:47 web01 sshd[18422]: Failed password for root from 203.0.113.45 port 51204 ssh2
May  4 03:12:49 web01 sshd[18422]: Failed password for root from 203.0.113.45 port 51204 ssh2
May  4 03:12:50 web01 sshd[18422]: Failed password for root from 203.0.113.45 port 51204 ssh2
May  4 03:12:50 web01 sshd[18422]: Disconnecting authenticating user root 203.0.113.45 port 51204: Too many authentication failures [preauth]
May  4 03:12:53 web01 sshd[18431]: Invalid user admin from 203.0.113.45 port 51290
May  4 03:12:55 web01 sshd[18431]: Failed password for invalid user admin from 203.0.113.45 port 51290 ssh2
```

Password spraying: una IP (o varias coordinadas) prueba una o dos contraseñas contra muchos usuarios distintos. Está pensado justo para esquivar el `maxretry` por usuario y por IP: pocos intentos por cuenta, repartidos en el tiempo. Se reconoce por la variedad de usuarios, no por el volumen:

```text
May  4 04:01:10 web01 sshd[19002]: Invalid user oracle from 198.51.100.7 port 40122
May  4 04:03:52 web01 sshd[19017]: Invalid user postgres from 198.51.100.7 port 40310
May  4 04:06:31 web01 sshd[19033]: Invalid user jenkins from 198.51.100.7 port 40488
May  4 04:09:14 web01 sshd[19051]: Invalid user deploy from 198.51.100.7 port 40615
May  4 04:11:58 web01 sshd[19068]: Invalid user ubuntu from 198.51.100.7 port 40790
```

Un intento cada tres minutos no dispara un `findtime = 10m` con `maxretry = 5` (los dos parámetros con los que fail2ban decide cuándo bloquear, que se explican más abajo: cuántos fallos y en qué ventana de tiempo). Por eso la detección de spraying se hace contando usuarios distintos por IP en ventanas largas, no intentos.

```bash
# Fuerza bruta: intentos fallidos por IP, hoy
journalctl _COMM=sshd --since today --no-pager \
  | grep -oE "Failed password for (invalid user )?\S+ from \S+" \
  | awk '{print $NF}' | sort | uniq -c | sort -rn | head

# Spraying: usuarios distintos por IP en las últimas 24 h
journalctl _COMM=sshd --since -24h --no-pager \
  | grep -oE "Invalid user \S+ from \S+" \
  | awk '{print $5, $3}' | sort -u | awk '{print $1}' | uniq -c | sort -rn | head

# Logins correctos, para ver quién ha entrado y desde dónde (y cuándo)
journalctl _COMM=sshd --since -7d --no-pager | grep "Accepted"

# nginx: IPs que más 401/403/404 generan (escáneres y rutas de administración)
awk '$9 ~ /^(401|403|404)$/ {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
awk '$7 ~ /(wp-login|wp-admin|\.env|phpmyadmin|\/admin|\.git)/ {print $1, $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
```

Otras dos señales que no son fuerza bruta pero se revisan en el mismo pase: accesos correctos fuera del horario habitual (un `Accepted publickey` a las 03:40 de una cuenta que siempre entra de 8 a 18) y orígenes geográficos raros. Para lo segundo, `geoiplookup` (paquete `geoip-bin`) o la base de datos GeoLite2 de MaxMind (gratuita con registro) dan el país de una IP; en Loki, Alloy tiene una etapa `geoip` que añade `geoip_country_code` como etiqueta, y con eso el panel de Grafana puede mostrar un mapa de orígenes. Un país nuevo en la lista de logins correctos es motivo de llamada, no de incidencia.

### Detección en Loki y alerta

La regla que se apuntó en el original, ampliada. En Loki 3.x las reglas de alerta van en el ruler, el componente que evalúa consultas cada cierto tiempo igual que hace Prometheus con sus reglas (`ruler.yaml` o un fichero en el directorio de reglas), con la misma sintaxis que Prometheus, y disparan a Alertmanager como las de UT2.

```yaml
groups:
  - name: accesos
    rules:
      - alert: FuerzaBrutaSSH
        expr: |
          sum by (host) (count_over_time({job="auth"} |= "Failed password" [5m])) > 20
        for: 0m
        labels: { severity: warning }
        annotations:
          summary: "Más de 20 fallos SSH en 5 min en {{ $labels.host }}"
      - alert: PasswordSprayingSSH
        expr: |
          count by (host, ip) (
            sum by (host, ip, user) (
              count_over_time({job="auth"}
                | regexp `Invalid user (?P<user>\S+) from (?P<ip>\S+)` [1h])
          )) > 8
        for: 0m
        labels: { severity: warning }
        annotations:
          summary: "IP {{ $labels.ip }} ha probado más de 8 usuarios distintos en 1 h"
```

La segunda es la de spraying: extrae usuario e IP con `regexp`, agrupa por los tres, y cuenta cuántos usuarios distintos tiene cada IP. Un umbral de 8 usuarios distintos en una hora no lo alcanza nadie legítimo. Ojo con `count_over_time` sin agregación en logs con muchas etiquetas: devuelve una serie por stream y la alerta se dispara por cada una. La agregación `sum by` es la que la deja en una por host.

### fail2ban a fondo

![Logo de fail2ban](../img/fail2ban-logo.png){ .logo-inline }

fail2ban lee ficheros de log (o el journal), aplica expresiones regulares (filtros) y, cuando una IP supera `maxretry` fallos dentro de `findtime`, ejecuta una acción de bloqueo durante `bantime`. Cada combinación de filtro más acción es una jail. La configuración de fábrica está en `/etc/fail2ban/jail.conf` y no se toca; lo vuestro va en `/etc/fail2ban/jail.local` o en ficheros bajo `/etc/fail2ban/jail.d/`, que se cargan encima. El `jail.local` de abajo es uno completo para un host con SSH y nginx; las tres líneas que más problemas evitan o causan son `ignoreip`, `backend` y `bantime.increment`, y son las que hay que entender antes de copiarlo.

```ini
# /etc/fail2ban/jail.local
[DEFAULT]
# Redes que nunca se bloquean: loopback, oficina, monitorización, bastión
ignoreip = 127.0.0.1/8 ::1 10.10.0.0/24 203.0.113.0/28
bantime  = 1h
findtime = 10m
maxretry = 5
# Bantime incremental: cada reincidencia dobla el tiempo, hasta 4 semanas
bantime.increment = true
bantime.factor    = 2
bantime.maxtime   = 4w
# Debian 13: sin rsyslog, el backend systemd lee el journal
backend = systemd
banaction = nftables-multiport
banaction_allports = nftables-allports

[sshd]
enabled = true
mode    = aggressive

[nginx-http-auth]
enabled  = true
logpath  = /var/log/nginx/error.log

[nginx-botsearch]
enabled  = true
logpath  = /var/log/nginx/access.log
maxretry = 2

[recidive]
enabled  = true
bantime  = 1w
findtime = 1d
maxretry = 3
```

Puntos a entender de esa configuración. `ignoreip` es obligatorio pensarlo antes de activar nada: si os bloqueáis a vosotros mismos o a la sonda de monitorización, tenéis un problema autoinfligido que en una empresa se nota. Ahí van la red de gestión, la IP pública de la oficina y la del bastión desde el que se administra. `bantime.increment` es la forma moderna de castigar reincidentes: una IP que vuelve tras el primer baneo de 1 h recibe 2 h, luego 4 h, y así hasta `bantime.maxtime`; fail2ban guarda el historial en su base de datos SQLite (`/var/lib/fail2ban/fail2ban.sqlite3`), así que sobrevive a reinicios. La jail `recidive` es el mecanismo antiguo para lo mismo (lee el propio log de fail2ban y banea en todos los puertos a quien ha sido baneado tres veces en un día); se pueden usar las dos. El modo `aggressive` del filtro sshd incluye también los intentos que se quedan en preauth y los de usuario inválido.

Un filtro propio para la aplicación. Si la API del curso escribe `WARN auth: login failed for user=victor ip=203.0.113.45` cuando alguien falla la contraseña, el filtro es una regex con el marcador `<HOST>` (o `<ADDR>` en versiones recientes) en el sitio de la IP:

```ini
# /etc/fail2ban/filter.d/app-login.conf
[Definition]
failregex = ^.*auth: login failed for user=\S+ ip=<HOST>\s*$
ignoreregex =
```

```ini
# /etc/fail2ban/jail.d/app-login.conf
[app-login]
enabled  = true
filter   = app-login
logpath  = /srv/app/logs/app.log
port     = http,https
maxretry = 8
findtime = 15m
bantime  = 2h
```

Aquí hay un detalle de contenedores: fail2ban corre en el host y necesita leer el log de la aplicación. Si la aplicación escribe a stdout, el fichero está en `/var/lib/docker/containers/<id>/<id>-json.log` (cada línea envuelta en JSON, la regex tiene que contemplarlo) y cambia de nombre al recrear el contenedor; es más limpio que la aplicación escriba a un volumen montado, o usar `backend = systemd` con el driver de logs `journald` y `journalmatch = CONTAINER_NAME=app` en la jail. Y la IP que ve la aplicación detrás de nginx es la del proxy, no la del cliente, salvo que nginx pase `X-Forwarded-For` o `X-Real-IP` (cabeceras HTTP en las que el proxy pone la IP original del cliente) y la aplicación lo registre. Sin eso, el filtro banearía a nginx.

Para probar un filtro antes de activarlo se usa `fail2ban-regex`, que dice cuántas líneas casan y cuáles no:

```bash
fail2ban-regex /srv/app/logs/app.log /etc/fail2ban/filter.d/app-login.conf --print-all-missed | tail -20
fail2ban-regex "$(journalctl _COMM=sshd --since -1h -o short-iso | tail -200)" sshd
```

Operación con `fail2ban-client`:

```bash
fail2ban-client status                       # jails activas
fail2ban-client status sshd                  # fallos, baneos actuales e IPs
fail2ban-client set sshd unbanip 203.0.113.45
fail2ban-client set sshd banip 198.51.100.7  # baneo manual
fail2ban-client get sshd bantime
fail2ban-client reload                       # recarga sin perder baneos
fail2ban-client banned                       # todas las IPs baneadas por jail
```

El bloqueo lo aplica la acción. Con `banaction = nftables-multiport`, fail2ban crea una tabla `inet f2b-table` (el nombre exacto depende de la versión) con un set por jail y una regla que descarta el tráfico de las IPs del set en los puertos de la jail. Se comprueba directamente con nftables (el firewall del kernel que ya manejasteis en UT3), que es donde de verdad se ve si el bloqueo existe:

```bash
nft list table inet f2b-table
nft list set inet f2b-table addr-set-sshd
```

Si en el host ya hay un firewall nftables propio (como el de 5166 UT3, [https://victor-educ.github.io/apuntes-5166/ut/ut3-seguridad-por-capas/](https://victor-educ.github.io/apuntes-5166/ut/ut3-seguridad-por-capas/)), fail2ban añade su tabla aparte y las dos conviven; el orden de evaluación entre tablas depende de la prioridad de las cadenas, y la de fail2ban usa `priority -1` para ir antes que `filter` (0). Con Docker de por medio, el tráfico a puertos publicados por contenedores pasa por la cadena `DOCKER-USER` de iptables antes que por `INPUT`; fail2ban tiene la acción `iptables-multiport` con `chain = DOCKER-USER` para ese caso, o se banea en `forward` con una acción nftables personalizada. Es uno de los motivos por los que en el laboratorio el bloqueo lo hacemos en el proxy o en OPNsense y no en app01.

Comprobar un bloqueo de verdad, que es lo que pide la actividad A5.2: desde una máquina que no esté en `ignoreip` (una VM de pruebas, el móvil con datos), fallar la contraseña SSH seis veces, ver la IP en `fail2ban-client status sshd`, ver la regla en nftables, comprobar que la conexión ahora da `Connection refused` o se queda en timeout, y desbanear. Capturas de esos cuatro pasos son la evidencia. Documentación oficial: [https://github.com/fail2ban/fail2ban/wiki](https://github.com/fail2ban/fail2ban/wiki) y las páginas de manual `jail.conf(5)`.

### CrowdSec como alternativa

CrowdSec hace lo mismo que fail2ban con dos diferencias de diseño. La primera es que separa la detección (el agente, que lee logs con "parsers" y "scenarios" en YAML descargados desde su hub) del bloqueo (los "bouncers": un componente para nftables, otro para nginx, otro para OPNsense, otro para Traefik, que consultan la API local del agente y aplican las decisiones). Así el agente puede estar en un host y el bouncer en el firewall. La segunda es que es colaborativo: cada instalación que reporta ataques a la red de CrowdSec recibe a cambio una lista de IPs maliciosas vistas por la comunidad, de modo que bloqueáis a un atacante antes de que os toque a vosotros. Esa lista es opcional y en algunas empresas no gusta enviar datos fuera; se puede desactivar (`crowdsec` en modo sin API central) y sigue funcionando como fail2ban.

```bash
cscli metrics                  # qué logs lee y qué escenarios disparan
cscli decisions list           # IPs bloqueadas, motivo, duración
cscli decisions add --ip 203.0.113.45 --duration 4h --reason "prueba manual"
cscli decisions delete --ip 203.0.113.45
cscli alerts list              # histórico de detecciones
cscli hub list                 # colecciones instaladas (nginx, sshd, linux)
cscli bouncers list            # bouncers registrados y cuándo consultaron
```

Cuándo elegir cuál: fail2ban si el sistema es uno o dos hosts, ya hay experiencia con él y se quiere control fino con regex propias; CrowdSec si hay varios hosts, se quiere bloquear en un punto central (OPNsense tiene plugin oficial) o se valora la lista comunitaria. En el laboratorio del curso, con OPNsense delante de la VPC, CrowdSec con el bouncer en OPNsense es la arquitectura más limpia; en la empresa, lo que haya. Documentación: [https://docs.crowdsec.net/](https://docs.crowdsec.net/).

## Fallos y reinicios: crashdumps y registros de error (CE 3c)

Un contenedor que se reinicia solo y vuelve a funcionar es el fallo más fácil de ignorar y el que más caro sale ignorar. Con `restart: unless-stopped` en el Compose, el servicio está caído unos segundos, el proxy devuelve unos 502, y nadie se entera hasta que el reinicio pasa a ser cada diez minutos. El análisis siempre sigue el mismo camino.

```mermaid
flowchart TD
    A[Sintoma: reinicio, 5xx, alerta] --> B[Codigo de salida y OOMKilled]
    B --> C[Logs de los 5 min previos]
    C --> D[Metricas del momento: memoria, CPU, conexiones]
    D --> E{Hay dump?}
    E -- si --> F[Analizar dump: gdb, py-spy, jmap]
    E -- no --> G[Habilitar dumps para la proxima]
    F --> H[Hipotesis]
    G --> H
    H --> I[Reproducir en pruebas]
    I --> J{Se reproduce?}
    J -- no --> H
    J -- si --> K[Corregir el origen]
    K --> L[Verificar: no se repite en N dias]
    L --> M[Cerrar incidencia con informe]
```

### Códigos de salida

Lo primero que se mira, porque acota el problema a una de tres familias: lo mató el kernel, lo mató Docker, o se murió solo.

| Código | Qué significa | Causa habitual |
|---|---|---|
| 0 | Salida normal | El proceso principal terminó (un script que acaba, un `CMD` mal pensado) |
| 1 | Error genérico de la aplicación | Excepción no controlada, fichero de configuración que falta, variable de entorno vacía |
| 2 | Uso incorrecto | Argumentos erróneos al binario, típico en shell |
| 125 | Fallo del propio `docker run` | Opción inválida, imagen inexistente |
| 126 | El comando no es ejecutable | Permisos del entrypoint, falta el bit `x` |
| 127 | Comando no encontrado | Ruta errónea en `CMD`, binario que no está en la imagen |
| 134 | 128 + 6 (SIGABRT) | `abort()`: aserción fallida, error de memoria detectado por la propia biblioteca (glibc) |
| 137 | 128 + 9 (SIGKILL) | OOM killer (comprobar `OOMKilled`), o `docker stop` que agotó el periodo de gracia, o alguien con `kill -9` |
| 139 | 128 + 11 (SIGSEGV) | Fallo de segmentación: bug en código nativo, biblioteca incompatible, corrupción de memoria |
| 143 | 128 + 15 (SIGTERM) | Parada ordenada: `docker stop`, `compose down`, reinicio del host; el proceso capturó la señal y salió |
| 255 | Fuera de rango | Muchos programas lo usan como "error no especificado"; SSH lo devuelve al fallar la conexión |

La regla es: código mayor de 128 es señal (código menos 128 da el número de señal, `kill -l N` da el nombre). `137` sin `OOMKilled=true` es que a alguien se le acabó la paciencia con un `docker stop` (10 s de gracia por defecto, `stop_grace_period` en Compose) o que lo mató un `kill -9`; 137 con `OOMKilled=true` es memoria. Con `143` el proceso salió bien pero alguien le pidió salir, y lo que hay que averiguar es quién: un `docker events` o el journal de dockerd lo dicen.

```bash
docker inspect app --format '{{.RestartCount}} {{.State.ExitCode}} {{.State.OOMKilled}} {{.State.FinishedAt}} {{.State.Error}}'
docker events --since 24h --until 0s --filter container=app --filter event=die --filter event=oom --filter event=restart
docker compose ps -a         # muestra "Exited (137) 2 hours ago"
```

`RestartCount` se pone a cero cuando el contenedor se recrea, así que un `0` no es garantía de nada si alguien hizo `compose up` esta mañana. Las políticas de reinicio de Docker: `no`, `on-failure[:N]` (solo si sale con código distinto de 0, como mucho N veces), `always` y `unless-stopped` (como `always`, pero si se paró a mano no arranca con el demonio). En producción lo normal es `unless-stopped`; `on-failure:5` tiene sentido en trabajos por lotes que no deben reintentar sin fin. Docker aplica un retardo creciente entre reinicios (100 ms, 200 ms, 400 ms...) que se reinicia si el contenedor aguanta 10 s; un contenedor en bucle de arranque se reconoce por `RestartCount` alto y `docker ps` mostrando "Restarting (1) 3 seconds ago".

### OOM: el kernel y los cgroups

Cuando un contenedor tiene `mem_limit` (Compose) o `--memory`, Docker lo traduce a `memory.max` en el cgroup v2 del contenedor (los cgroups son el mecanismo del kernel con el que Docker acota la CPU y la memoria de cada contenedor). Si el uso llega al límite, el kernel invoca al OOM killer dentro de ese cgroup y mata el proceso que más memoria consume (lo normal, el principal), y el contenedor sale con 137. Si no hay límite, el contenedor puede consumir toda la RAM del host y el OOM killer global mata lo que le parezca, que puede ser otro contenedor o PostgreSQL. Por eso todo contenedor en producción lleva límite: mejor que muera él solo que el host entero.

```bash
# Mensajes del OOM killer, con marca de tiempo legible
dmesg -T | grep -iE "oom|killed process" | tail
journalctl -k --since -24h | grep -iE "oom-kill|out of memory"
# Ejemplo de lo que aparece:
#   Memory cgroup out of memory: Killed process 21877 (python3) total-vm:1893204kB, anon-rss:1044120kB ...
#   oom-kill:constraint=CONSTRAINT_MEMCG,...,task=python3,pid=21877,uid=1000

# Límite y uso del cgroup del contenedor (cgroup v2, Debian 13 y Docker 28)
CG=$(docker inspect app --format '{{.Id}}')
cat /sys/fs/cgroup/system.slice/docker-$CG.scope/memory.max
cat /sys/fs/cgroup/system.slice/docker-$CG.scope/memory.current
cat /sys/fs/cgroup/system.slice/docker-$CG.scope/memory.events    # oom_kill N
cat /sys/fs/cgroup/system.slice/docker-$CG.scope/memory.stat | head -20

docker stats --no-stream app
```

`memory.events` guarda un contador `oom_kill` que no se pierde entre reinicios del proceso (sí del contenedor), y es la forma más limpia de saber si hubo OOM aunque el mensaje del kernel ya no esté. `anon-rss` en el mensaje del kernel es la memoria realmente usada por el proceso; `total-vm` es la reservada y no dice mucho. En Prometheus, `container_memory_working_set_bytes` de cAdvisor (el exporter que publica las métricas de cada contenedor) es lo que compara Docker contra el límite (excluye la caché de ficheros reclamable), y `container_oom_events_total` cuenta los OOM; un panel con el working set y el límite `container_spec_memory_limit_bytes` superpuestos muestra la fuga como una rampa que sube hasta tocar la línea y cae en vertical.

Una fuga de memoria y un pico de carga se ven distinto en ese panel: la fuga es una rampa constante, día tras día, hasta el OOM; el pico es un escalón que coincide con un evento (la copia de seguridad, un informe pesado, un ataque). La corrección es diferente: la fuga se arregla en el código (o se mitiga con reinicios programados mientras se arregla), el pico con más límite o con menos carga.

### Core dumps en contenedores

Un core dump es la imagen de la memoria del proceso en el momento de morir por señal (SIGSEGV, SIGABRT). Por defecto en un host Linux con systemd los captura `systemd-coredump` y se listan con `coredumpctl`. En un contenedor hay dos obstáculos: `core_pattern` (el parámetro del kernel que dice dónde y cómo se escribe el dump) es del kernel y por tanto del host, no del contenedor (un `|/usr/lib/systemd/systemd-coredump ...` del host se ejecuta en el espacio de nombres del host y a veces no puede leer el binario del contenedor), y el `ulimit -c` del contenedor suele ser 0.

```bash
# En el host
cat /proc/sys/kernel/core_pattern
# Si empieza por | es systemd-coredump; si es una ruta, escribe fichero
coredumpctl list
coredumpctl info 21877
coredumpctl debug 21877          # abre gdb sobre el dump

# Alternativa para contenedores: patrón de fichero en un directorio compartido
echo '/var/crash/core.%e.%p.%t' > /proc/sys/kernel/core_pattern
sysctl -w fs.suid_dumpable=2
```

```yaml
# docker-compose.yml: permitir dumps y montar el directorio
services:
  app:
    ulimits:
      core: -1
    volumes:
      - /var/crash:/var/crash
```

El directorio `/var/crash` tiene que existir dentro del contenedor en la misma ruta que indica `core_pattern`, porque el kernel escribe la ruta interpretada desde el espacio de nombres de montaje del proceso que murió. Y hay que vigilar el tamaño: un dump de una JVM con 4 GB de heap ocupa 4 GB.

Análisis según el runtime, con una herramienta por lenguaje: gdb es el depurador clásico de binarios nativos (C, C++, Go, Rust) y es el que lee el dump; py-spy inspecciona un proceso Python vivo sin pararlo; jcmd, jmap y jstack son las utilidades de la JVM para volcar memoria e hilos. En el bloque, fijaos en que gdb se ejecuta dentro de la misma imagen que murió y en que Java no necesita el dump del kernel porque genera el suyo.

```bash
# C/C++/Go/Rust: gdb básico sobre el dump, con el mismo binario y bibliotecas
docker run --rm -it -v /var/crash:/var/crash --entrypoint bash app-imagen:1.4.2
gdb /usr/local/bin/servidor /var/crash/core.servidor.17.1746340361
(gdb) bt            # traza de pila del hilo que falló
(gdb) info threads  # todos los hilos
(gdb) thread apply all bt   # traza de todos
(gdb) frame 3       # saltar a un marco concreto
(gdb) info locals   # variables locales de ese marco

# Python: py-spy no necesita dump, inspecciona el proceso vivo (colgado, no muerto)
pip install py-spy
py-spy dump --pid $(docker inspect app --format '{{.State.Pid}}')
py-spy top --pid <pid>          # qué funciones consumen CPU ahora mismo
# Un proceso Python que ha muerto por segfault viene de una extensión en C: gdb + python-gdb.py

# Java: heap dump para fugas, thread dump para cuelgues
docker exec app jcmd 1 GC.heap_info
docker exec app jmap -dump:live,format=b,file=/var/crash/heap.hprof 1
docker exec app jstack 1 > /var/crash/threads.txt
# O que la JVM lo haga sola al morir por OOM:
# JAVA_TOOL_OPTIONS="-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/crash"
```

Para gdb el detalle importante es que el binario y las bibliotecas tienen que ser exactamente los del contenedor que murió; por eso se abre gdb dentro de la misma imagen (y versión). Sin símbolos de depuración la traza da direcciones y nombres de función, que suele bastar para saber en qué biblioteca ha reventado. Los `.hprof` de Java se analizan con Eclipse MAT o VisualVM en vuestro portátil, no en el servidor. El PID `1` en los `docker exec` es el proceso principal del contenedor; si la aplicación arranca a través de un script, buscad el PID real con `docker top app`.

### Registros de error de la aplicación

Aparte de stdout, muchos runtimes dejan su propio informe de fallo: la JVM escribe `hs_err_pid<N>.log` en el directorio de trabajo cuando muere por un error fatal (con la traza, los hilos y el estado de memoria; es lo primero que se busca en un 134 de Java); Node deja la traza en stderr y con `--report-on-fatalerror` genera un JSON de diagnóstico; PostgreSQL escribe en `/var/log/postgresql/postgresql-17-main.log` (o stdout en el contenedor oficial) y ahí está el `FATAL: too many connections` o el `server process was terminated by signal 9` que explica un 137 del contenedor de la aplicación media hora después. Para un front en el navegador, los errores están en el cliente y no en el servidor: sin un servicio tipo Sentry o GlitchTip (plataformas que reciben los errores del navegador y del servidor y los agrupan) que los recoja, solo se ven en la consola del usuario, y la evidencia es la captura que os manden.

### El método y la plantilla de informe

Lo que reúne el diagrama de arriba, con el orden que importa: primero se recopila, después se piensa. Reunir el dump o el log de error, las líneas de log de los cinco minutos previos al fallo (`docker logs --until "2027-05-04T02:41:00" --since "2027-05-04T02:36:00" app`), y las métricas de ese instante (memoria, CPU, conexiones, latencia; en Grafana, el intervalo de tiempo fijado a esos diez minutos y una captura). Con eso se formula una hipótesis, se intenta reproducir en un entorno previo de la empresa (nunca en producción), se corrige el origen y no el síntoma (subir el límite de memoria para una fuga es un parche, arreglar la fuga es la corrección; poner `restart: always` a algo que muere cada hora no es corregir nada), y se verifica durante días que el reinicio no se repite. Todo va a la incidencia.

El informe de A5.3 es una página y sigue esta estructura:

```text
1. Síntoma y detección: qué se vio, cuándo, cómo se detectó (alerta, revisión, aviso)
2. Datos del fallo: contenedor, imagen y versión, código de salida, OOMKilled,
   RestartCount, mensaje del kernel o del runtime (literal)
3. Contexto: log de los 5 min previos (extracto anonimizado), métricas del momento
   (captura o tabla: memoria usada/límite, CPU, conexiones, latencia p95)
4. Análisis del dump o del registro de error: traza, hilo, función, o "no había dump,
   se ha habilitado para la próxima"
5. Hipótesis (una principal, alternativas descartadas y por qué)
6. Reproducción: dónde, cómo, resultado
7. Corrección aplicada: qué se cambió (fichero, valor antes y después), quién lo aprobó
8. Verificación: métrica o comando que demuestra que no se repite, y durante cuánto tiempo
9. Acciones pendientes: lo que queda para el equipo de desarrollo o para más adelante
```

## Rendimiento del equipo (CE 3d)

El rendimiento no se evalúa con un número, se evalúa contra una línea base: los mismos indicadores en una semana normal. "CPU al 60 %" no dice nada; "CPU al 60 % cuando la línea base de ese día y esa hora es 25 %" sí. Sin línea base, la mitad de las alertas de rendimiento son falsas y la otra mitad llegan tarde.

### Cómo se toma y se guarda la línea base

La toma más útil es la de Prometheus, porque ya está: si `node_exporter` y cAdvisor llevan semanas en marcha, la línea base son las últimas dos o cuatro semanas, y en Grafana se superpone con la función "time shift" de un panel (mostrar la misma serie desplazada 7 días, en gris, debajo de la actual) o con una consulta `offset 1w`. La línea base no es un promedio plano: tiene forma (el pico de las 9:00, el valle de la noche, la copia de las 02:00), y lo que se compara es la forma.

```text
# Uso de CPU del host ahora frente a hace una semana
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m] offset 1w)))

# Percentil 95 de la última semana, como número de referencia
quantile_over_time(0.95, (100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))))[7d:5m])
```

Si no hay Prometheus, la línea base se toma con `sar` (paquete `sysstat`), que graba cada 10 minutos en `/var/log/sysstat/` y guarda un mes por defecto: `sar -u -f /var/log/sysstat/sa04` da la CPU de todo el día 4, `sar -r` la memoria, `sar -d` el disco, `sar -n DEV` la red. Activadlo la primera semana en la empresa si no está (`ENABLED="true"` en `/etc/default/sysstat`), porque la semana de datos es el requisito de A5.4 y no se puede recuperar hacia atrás.

Se guarda como tabla, con lo mínimo que hace falta para comparar después: por recurso, valor medio, p95 (el valor que el 95 % de las muestras no supera, para que un pico aislado no distorsione) y máximo, en horario laboral y fuera de él, más las tres o cuatro horas concretas que definen la forma (pico de la mañana, batch nocturno). Y se guarda con fecha y con la versión de la aplicación desplegada en ese momento, porque una línea base tomada con la versión 1.4 no sirve para juzgar la 1.6 si el despliegue cambió el consumo.

| Recurso | Métrica | Media laboral | p95 laboral | Máximo | Media nocturna | Hora del pico |
|---|---|---|---|---|---|---|
| CPU host | % uso | 22 | 48 | 71 | 6 | 09:15 |
| CPU contenedor app | % de un núcleo | 35 | 80 | 140 | 4 | 09:15 |
| Memoria app | working set / límite | 410 MB / 1 GB | 520 MB | 610 MB | 380 MB | 17:40 |
| Disco vdb (datos) | %util, await ms | 12, 1.8 | 35, 4.1 | 88, 22 | 45 (02:00) | 02:10 |
| Red ens18 | Mbit/s entrada/salida | 3 / 11 | 9 / 30 | 24 / 85 | 0.4 / 1 | 12:30 |
| Peticiones | req/s, p95 latencia ms | 18, 120 | 45, 260 | 70, 410 | 2, 90 | 09:15 |

### USE por recurso: comandos y métricas

El método USE (Utilization, Saturation, Errors, de Brendan Gregg) es la lista de comprobación: para cada recurso, cuánto se usa, cuánta cola hay esperando por él, y cuántos errores da. La tabla del original, ampliada con las tres preguntas y con la métrica de Prometheus equivalente para que en la empresa podáis usar lo uno o lo otro.

| Recurso | Comando | Métrica Prometheus | Utilización | Saturación | Errores / señal de problema |
|---|---|---|---|---|---|
| CPU | `top`, `mpstat -P ALL 1`, `docker stats` | `node_cpu_seconds_total`, `container_cpu_usage_seconds_total`, `container_cpu_cfs_throttled_periods_total` | % no idle > 80 % sostenido | load average > número de núcleos (`node_load1`), `%r` de `vmstat` alto, throttling de CFS en contenedores con límite de CPU | `%steal` alto en VM (ver abajo) |
| Memoria | `free -h`, `vmstat 1` (columnas `si`/`so`) | `node_memory_MemAvailable_bytes`, `container_memory_working_set_bytes` frente a `container_spec_memory_limit_bytes` | working set cerca del límite | swap activo (`si`/`so` > 0 de forma continua), `node_vmstat_pswpin` | OOM kills (`container_oom_events_total`) |
| Disco | `iostat -xz 1`, `df -h`, `df -i` | `node_disk_io_time_seconds_total`, `node_disk_read_time_seconds_total`, `node_filesystem_avail_bytes` | `%util` > 80 % | `await` > 10 ms en SSD o > 30 ms en disco rotacional, `aqu-sz` creciente | errores en `dmesg` (`I/O error`, `reset`), espacio < 15 %, inodos < 10 % |
| Red | `ss -s`, `ss -tan state established \| wc -l`, `iftop -i ens18`, `ip -s link` | `node_network_receive_bytes_total`, `node_network_transmit_drop_total`, `node_netstat_Tcp_RetransSegs` | Mbit/s frente a la capacidad del enlace | cola de conexiones (`ss -ltn` columna Recv-Q en escucha), `SYN` en `ss -s` | retransmisiones, `drop` y `errors` en `ip -s link`, `overruns` |
| Procesos | `ps aux --sort=-%mem \| head`, `pidstat -r -u 5`, `docker top app` | `container_memory_working_set_bytes` por contenedor, `process_resident_memory_bytes` de la propia aplicación | un proceso con % desproporcionado | número de hilos creciente (`ps -o nlwp`), descriptores de fichero (`ls /proc/<pid>/fd \| wc -l`) frente a `ulimit -n` | un proceso que crece sin parar (fuga), zombis (`Z` en `ps`) |

Dos comandos que merecen explicación aparte. `vmstat 1 5` da cinco muestras de un segundo; las columnas que importan son `r` (procesos esperando CPU; si es mayor que los núcleos de forma continua, hay saturación de CPU), `b` (bloqueados en E/S), `si`/`so` (swap in/out en KB/s; cualquier valor sostenido distinto de cero es un problema) y `wa` (% de CPU esperando E/S; alto quiere decir que el disco es el cuello de botella, no la CPU). `iostat -xz 1` con `-x` da las columnas extendidas y `-z` oculta los discos sin actividad; `%util` es el porcentaje de tiempo con al menos una petición en curso, `await` la latencia media de cada petición en milisegundos, y `aqu-sz` la longitud media de la cola. En un SSD virtual de Proxmox, `await` por encima de 10 ms de forma continua indica que el almacenamiento del host está saturado, y eso no se arregla desde la VM.

### El steal en máquinas virtuales

`%st` en `top` o `mpstat` es el tiempo en que la VM quería ejecutar y el hipervisor le dio la CPU a otra. Es la única columna que apunta fuera de la máquina que estáis mirando. Un `steal` por encima del 5 al 10 % sostenido significa que el host de Proxmox (o el de la nube) está sobresuscrito: hay más vCPU asignadas de las que el host puede atender a la vez. Desde dentro de la VM la aplicación va lenta sin que ninguna métrica interna lo explique (CPU al 40 %, latencia el doble). En Prometheus es `rate(node_cpu_seconds_total{mode="steal"}[5m])`, y una alerta razonable es steal > 10 % durante 15 minutos. La acción no es de la VM: es hablar con quien administra el hipervisor (en la empresa, el equipo de sistemas; en el laboratorio, mirar en el nodo de Proxmox qué otra VM se está comiendo los núcleos, como en 5166 UT1, [https://victor-educ.github.io/apuntes-5166/ut/ut1-virtualizacion/](https://victor-educ.github.io/apuntes-5166/ut/ut1-virtualizacion/)).

### Señales de problema y acciones

La comparación con la línea base termina en una propuesta. Las propuestas tipo, para que no os quedéis en "la CPU está alta":

- CPU del contenedor limitada por CFS, el planificador del kernel que frena al contenedor cuando agota su cuota `cpus:` (`throttled_periods` creciente con uso por debajo del 100 % del host): subir `cpus:` en el Compose o quitar el límite si el host tiene margen. Caso muy frecuente con límites puestos "por si acaso" a 0.5 CPU.
- Memoria del contenedor con rampa hasta el límite y OOM: fuga; pedir corrección a desarrollo y, mientras, reinicio programado en la ventana de menor carga, con `restart` y monitorización del contador de OOM. Si no es rampa sino escalón, subir el límite y justificarlo con el nuevo consumo.
- Disco con `await` alto y `wa` alto en la ventana de la copia de seguridad: mover la copia, limitar su E/S con `ionice -c3` o `pg_dump` con `--jobs` menor, o pedir almacenamiento más rápido para la BD.
- Disco lleno o cerca: revisar la retención de logs (apartado anterior), `docker system df` y `docker system prune` de imágenes huérfanas, dumps olvidados en `/var/crash`.
- Red con retransmisiones: mirar primero el enlace físico o virtual (`ip -s link`, errores en la interfaz), después la MTU (el tamaño máximo de paquete de la interfaz) si hay túneles (WireGuard o VXLAN restan cabecera; 1420 o 1450 son valores habituales), después el firewall que pueda estar descartando.
- Latencia p95 alta con todos los recursos del host tranquilos: el cuello está fuera (base de datos, servicio externo, DNS) o en el pool de conexiones de la aplicación; se ve en el log de la aplicación y en las métricas de la BD (`pg_stat_activity`, consultas lentas), no en `top`.
- Carga muy por encima de la línea base sin ninguna causa interna: mirar los accesos. Un escáner o un ataque a la API se ve antes en `req/s` por IP que en la CPU.

Se documenta con una tabla antes/después (línea base, periodo de carga, diferencia, umbral) o con una captura del panel con la línea base superpuesta, y se propone la acción con su coste: ajustar límites (gratis, inmediato), escalar (dinero o recursos del hipervisor), optimizar una consulta (tiempo de desarrollo), limpiar disco (gratis, pero recurrente si no se arregla la retención).

## Errores frecuentes en el laboratorio

Aunque esta unidad se hace en la empresa, estos son los tropiezos que se repiten y que os conviene conocer antes de llegar.

- `docker logs` sin `--since` tarda minutos y llena el terminal. Siempre con ventana de tiempo, y con `--tail 200` para un vistazo rápido.
- `grep ERROR` no encuentra nada porque la aplicación escribe en JSON con `"level":"error"` en minúsculas. Mirad primero cinco líneas del log para ver el formato y usad `grep -i` o `jq`: `docker logs --since 1h app | jq -r 'select(.level=="error") | .msg'`.
- fail2ban activo pero `status sshd` da cero fallos: el backend lee un `auth.log` que no existe porque el sistema no tiene rsyslog. `backend = systemd` en `jail.local` y reiniciar fail2ban. Se comprueba con `fail2ban-client get sshd logpath` o mirando `/var/log/fail2ban.log`.
- Baneada la IP de la propia oficina o de la sonda de Blackbox de mon01. `ignoreip` antes de `enabled = true`, y si ya ha pasado, `fail2ban-client set sshd unbanip`.
- fail2ban banea en `INPUT` pero el puerto publicado por Docker sigue accesible: el tráfico a contenedores pasa por `DOCKER-USER`/`forward`, no por `input`. Bloquear en el proxy, en OPNsense, o usar la acción con `chain = DOCKER-USER`.
- El filtro propio banea al proxy nginx (10.10.1.10) en lugar de al cliente: la aplicación registra la IP del proxy. Configurar `X-Forwarded-For` en nginx y que la aplicación lo registre (con cuidado: solo confiar en esa cabecera cuando viene del proxy).
- `OOMKilled=false` con código 137: no era memoria, era un `docker stop` sin tiempo de gracia suficiente. La aplicación no maneja `SIGTERM` y Docker la mata a los 10 s; arreglar la aplicación o subir `stop_grace_period`.
- No hay core dump aunque `ulimit -c` es ilimitado en el contenedor: el `core_pattern` del host apunta a un directorio que no existe dentro del contenedor, o a `systemd-coredump` que no encuentra el binario. Montar `/var/crash` en la misma ruta y usar un patrón de fichero.
- `RestartCount` a cero después de un `compose up -d` que recreó el contenedor. Los reinicios anteriores están en `docker events` o en el journal de dockerd (`journalctl -u docker | grep -i oom`), no en `inspect`.
- Línea base tomada un festivo o en la semana de vacaciones de medio equipo. La línea base es una semana representativa; si no lo es, anotadlo y tomad otra.
- `iostat` muestra `%util` al 100 % en un volumen virtual y se concluye que el disco está saturado, cuando en dispositivos con colas paralelas (NVMe, almacenamiento en red) `%util` deja de ser fiable a partir de cierto punto. En esos casos manda `await`.
- La captura de Grafana en la evidencia lleva el nombre real de la empresa en el título del dashboard o en la etiqueta `instance`. Revisar cada captura antes de adjuntarla; recortar o pixelar lo que haga falta.

## Actividades

Las cuatro actividades se hacen sobre sistemas reales de la empresa, en el orden que el tutor considere, dentro del periodo del 19 de abril al 9 de junio de 2027. Cada una produce una evidencia concreta que se adjunta a la ficha. Anonimizad todo.

### A5.1 Revisión de logs (CE 3a)

Durante al menos cinco días laborables, revisad diariamente los logs de un servicio que os asigne el tutor. Llevad un registro con estas columnas: fecha, servicio, líneas o intervalo revisado, comando o consulta usada, errores encontrados (tipo y recuento), incidencias abiertas. Al menos uno de los hallazgos se reporta como incidencia siguiendo la plantilla de esta unidad, en la herramienta de la empresa o, si no procede abrirla ahí, en el formato de la plantilla. Evidencia: el registro de los cinco días y la incidencia (anonimizada). Valdrá más una incidencia sobre un aviso pequeño bien documentado que una sobre un fallo grande descrito a medias.

### A5.2 Accesos y fuerza bruta (CE 3b)

Revisad los accesos de un sistema (SSH, proxy inverso o aplicación) durante al menos 24 h de log. Identificad los intentos fallidos, contad por IP y por usuario, y clasificad lo que veáis: ruido de fondo, fuerza bruta, spraying, o nada (también es un resultado, si el sistema no está expuesto). Configurad o revisad fail2ban, CrowdSec o el mecanismo que use la empresa: jails activas, `ignoreip`, tiempos, y si hay un filtro propio para la aplicación. Comprobad un bloqueo real de acuerdo con el tutor (desde una IP de pruebas) y desbanead después. Evidencia: extracto de log con el patrón identificado y explicado, configuración (anonimizada), y lista de IPs bloqueadas con la prueba del bloqueo (salida de `fail2ban-client status`, `cscli decisions list` o equivalente, y la regla en el firewall).

### A5.3 Fallos y reinicios (CE 3c)

Analizad un reinicio o fallo real del periodo (o, si no ocurre ninguno, uno reproducido en un entorno de pruebas de la empresa; una forma sencilla es un contenedor con `mem_limit: 64m` ejecutando un proceso que reserva memoria, o un `kill -SEGV` sobre el proceso principal con dumps habilitados). Recoged el código de salida, `OOMKilled`, el dump o el registro de error, el log previo y las métricas del momento; formulad la hipótesis, aplicad o proponed la corrección y verificad. Evidencia: informe de análisis de una página con la estructura de nueve puntos de esta unidad.

### A5.4 Rendimiento (CE 3d)

Tomad la línea base de CPU, memoria, disco y red de un equipo durante una semana (Prometheus si lo hay, `sar` si no) y comparadla con un periodo de carga: el cierre de mes, una campaña, una prueba de carga acordada con el tutor como las de UT4, o simplemente el día de más tráfico de la semana siguiente. Aplicad USE a cada recurso. Evidencia: tabla comparativa con el formato de esta unidad (o captura del panel con la línea base superpuesta) y una propuesta de acción justificada, con su coste y con lo que esperáis que cambie en la métrica si se aplica.

### Actividad de cierre

Procedimiento de dos páginas: "Rutina de revisión diaria y semanal de logs, accesos, fallos y rendimiento en la empresa", con los comandos o consultas concretos que habéis usado, la herramienta de cada paso, el tiempo estimado, qué se considera normal y qué dispara una incidencia. Tiene que poder seguirlo alguien que llegue nuevo al puesto. El diagrama de rutina del principio de la unidad es el esqueleto; vosotros ponéis los comandos.

### Ficha de evidencias

La ficha la rellena el tutor de empresa conforme se van entregando las evidencias, y se entrega firmada al final de la estancia junto con el conjunto de evidencias.

| Actividad | CE | Fecha | Evidencia adjunta | Observaciones del tutor | Firma |
|---|---|---|---|---|---|
| A5.1 Logs | 3a | | | | |
| A5.2 Accesos | 3b | | | | |
| A5.3 Fallos | 3c | | | | |
| A5.4 Rendimiento | 3d | | | | |
| Cierre | (todos) | | | | |

## Práctica evaluable

En esta unidad no hay una práctica de laboratorio aparte: la práctica evaluable es el conjunto de las cuatro evidencias más el procedimiento de cierre, con la ficha firmada por el tutor de empresa. Se entrega en el repositorio de Gitea del módulo (carpeta `ut5/`, un fichero Markdown o PDF por actividad, más `rutina.md` y la ficha escaneada) antes del 11 de junio de 2027, dos días después de terminar la estancia. Todo anonimizado; una evidencia con datos identificables de la empresa se devuelve sin corregir hasta que se anonimice.

Entregables:

- [ ] `a5.1-logs.md`: registro de cinco días e incidencia reportada
- [ ] `a5.2-accesos.md`: extracto de log clasificado, configuración y prueba de bloqueo
- [ ] `a5.3-fallos.md`: informe de análisis de una página
- [ ] `a5.4-rendimiento.md`: tabla de línea base, comparación y propuesta
- [ ] `rutina.md`: procedimiento diario y semanal de dos páginas
- [ ] `ficha-evidencias.pdf`: firmada por el tutor

| Criterio | CE | Peso | Qué se valora |
|---|---|---|---|
| A5.1 Revisión de logs | 3a | 20 % | Registro completo y constante; incidencia con mensaje literal, frecuencia, impacto y fragmento; comandos o consultas adecuados al sistema |
| A5.2 Accesos y fuerza bruta | 3b | 20 % | Patrón correctamente identificado y explicado; configuración coherente (`ignoreip`, tiempos, filtro si procede); bloqueo comprobado de extremo a extremo |
| A5.3 Fallos y reinicios | 3c | 20 % | Datos completos del fallo; hipótesis razonada y alternativas descartadas; corrección del origen y verificación |
| A5.4 Rendimiento | 3d | 20 % | Línea base representativa y bien guardada; USE aplicado; propuesta justificada con coste y efecto esperado |
| Procedimiento de cierre | a, b, c, d | 15 % | Reproducible por otra persona; comandos reales; criterios de normalidad y de escalado |
| Forma y anonimización | | 5 % | Evidencias legibles, con fecha, sin datos identificables; ficha firmada |

Las observaciones del tutor de empresa en la ficha se tienen en cuenta en cada apartado. Una actividad sin firma del tutor no puntúa.

## Para ampliar

- [journalctl(1) en man7.org](https://man7.org/linux/man-pages/man1/journalctl.1.html): todas las opciones de filtrado (`-u`, `-p`, `_COMM`, `--since`, `-o json`) y las de vacuum; es la referencia que se consulta cada semana.
- [logrotate(8) en man7.org](https://man7.org/linux/man-pages/man8/logrotate.8.html): directivas, orden de evaluación y el detalle de `copytruncate` frente a `create`.
- [Configuración de logging en Docker](https://docs.docker.com/engine/logging/configure/): drivers, `max-size`, `max-file`, y el driver `journald` con sus campos.
- [LogQL en la documentación de Loki](https://grafana.com/docs/loki/latest/query/): filtros, parsers (`json`, `logfmt`, `pattern`, `regexp`) y las funciones de agregación sobre rangos que usan las alertas de esta unidad.
- [Wiki de fail2ban en GitHub](https://github.com/fail2ban/fail2ban/wiki): manual de uso, escritura de filtros y acciones, y ejemplos de jails.
- [Documentación de CrowdSec](https://docs.crowdsec.net/): conceptos de agente, escenarios y bouncers, e instalación del bouncer para nftables y para OPNsense.
- [Wiki de nftables](https://wiki.nftables.org/): tablas, sets y prioridades de cadenas, necesario para entender cómo convive el bloqueo de fail2ban con un firewall propio.
- [core(5) en man7.org](https://man7.org/linux/man-pages/man5/core.5.html): los especificadores de `core_pattern`, el comportamiento con tuberías y el efecto de `fs.suid_dumpable`.
- [The USE Method, de Brendan Gregg](https://www.brendangregg.com/usemethod.html): el método de la sección de rendimiento con su lista de comprobación por recurso para Linux.
- [Documentación de node_exporter](https://github.com/prometheus/node_exporter): colectores y nombres de métricas de CPU, memoria, disco y red que aparecen en la tabla USE.
- [Apuntes de la 5166, UT7 Monitorización](https://victor-educ.github.io/apuntes-5166/ut/ut7-monitorizacion/): la pila de mon01 sobre la que se construyen los paneles de esta unidad.
