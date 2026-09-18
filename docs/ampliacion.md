# Para ampliar

Esta página recoge, unidad por unidad, dos cosas que no caben en las sesiones: los apartados que van más allá de lo que se hace en clase (no se explican ni los necesita ninguna hoja de práctica, pero son lo que os vais a encontrar en una empresa) y los enlaces para seguir por vuestra cuenta. Las unidades quedan así con lo que se da en cada sesión, y esto está aquí para cuando queráis ir más lejos o cuando algo de la formación en empresa os suene y queráis situarlo.

## UT1 · Observabilidad de contenedores: métricas, logs y eventos

Enlaces para ampliar de la [UT1](ut/ut1-observabilidad.md): la documentación de referencia de cgroups, cAdvisor, Prometheus, las librerías cliente, Promtail, Loki, remote_write y chrony. En esta unidad todos los apartados de teoría se explican en clase o los necesita alguna hoja de práctica, así que no hay apartados movidos.

### Enlaces

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

## UT2 · Umbrales, agregación y gestión de alarmas

Apartados y enlaces de la unidad de alarmas que van más allá de lo que se hace en clase; vienen de [UT2](ut/ut2-alarmas.md).

#### Grafana Alerting como alternativa

![Grafana](img/grafana-logo.svg){ .logo-inline } Grafana 12 trae su propio motor de alertas: reglas que se definen desde la interfaz sobre cualquier fuente de datos (Prometheus, Loki, PostgreSQL, InfluxDB), con periodo de evaluación y pendiente equivalentes a `interval` y `for`, puntos de contacto que son los receptores, y políticas de notificación que son el árbol de rutas. Por debajo lleva embebido un Alertmanager, y puede configurarse para enviar a uno externo (el nuestro en mon01) en lugar de al suyo.

<figure markdown="span">
  ![Dashboard de Grafana con paneles de series temporales](img/grafana-dashboard.png){ width="640" }
  <figcaption>Un dashboard de Grafana. Las reglas de Grafana Alerting se crean desde el mismo panel que muestra la serie. Fuente: Joel Kennedy, dominio público, vía Wikimedia Commons.</figcaption>
</figure>

Cuándo sí: cuando la fuente de datos no tiene ruler propio (una base de datos SQL, un Elasticsearch), cuando el equipo que define las alertas trabaja sólo en Grafana y no toca ficheros YAML, o cuando quieres una alerta puntual sobre un panel sin pasar por el repositorio. Cuándo no: cuando las reglas deben vivir en Git con revisión y `promtool test` (Grafana permite provisionarlas desde ficheros, pero el flujo es más incómodo), cuando ya tienes el ruler de Loki y Prometheus mandando al mismo Alertmanager (tener reglas en dos motores es tener dos sitios donde buscar), y cuando la disponibilidad de las alertas no puede depender de la base de datos de Grafana. En el laboratorio la norma es: reglas en ficheros, Prometheus y Loki como motores, Grafana para mirar.

### Enlaces

- [Monitoring Distributed Systems, libro SRE de Google](https://sre.google/sre-book/monitoring-distributed-systems/): el capítulo del que sale la filosofía de síntomas frente a causas y las cuatro señales de oro. Gratuito y corto.
- [Alerting rules, documentación de Prometheus](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/): sintaxis completa de las reglas, `for`, `keep_firing_for` y las variables de plantilla.
- [Recording rules: best practices](https://prometheus.io/docs/practices/rules/): la convención `nivel:métrica:operaciones` con ejemplos.
- [Alerting: best practices](https://prometheus.io/docs/practices/alerting/): página breve de Prometheus que resume qué alertar y qué no.
- [Querying basics y functions](https://prometheus.io/docs/prometheus/latest/querying/functions/): referencia de `rate`, `increase`, `changes`, `absent`, `histogram_quantile` y el resto.
- [Alertmanager configuration](https://prometheus.io/docs/alerting/latest/configuration/): todas las opciones de rutas, inhibición, intervalos de silencio y cada receptor.
- [Notification template reference](https://prometheus.io/docs/alerting/latest/notifications/): los objetos y funciones disponibles en las plantillas.
- [LogQL: log queries](https://grafana.com/docs/loki/latest/query/log_queries/) y [metric queries](https://grafana.com/docs/loki/latest/query/metric_queries/): selectores, filtros, parsers y funciones de rango.
- [Loki ruler](https://grafana.com/docs/loki/latest/alert/): configuración del ruler, alertas y recording rules sobre logs.
- [Grafana Alerting](https://grafana.com/docs/grafana/latest/alerting/): reglas, puntos de contacto y políticas de notificación en Grafana 12.
- [Uso de la API de Gitea](https://docs.gitea.com/usage/api-usage): cómo generar el token y dónde está el Swagger con los endpoints de issues que usa el receptor webhook.
- [Mailpit](https://mailpit.axllent.org/docs/): instalación, opciones y API del SMTP de pruebas.

## UT3 · Seguridad de las comunicaciones de monitorización

Lista de enlaces de referencia de la [UT3](ut/ut3-seguridad-monitorizacion.md), la unidad que audita, cierra y cifra las comunicaciones de la pila de monitorización; todos los apartados de esa unidad se explican en clase o los necesita alguna hoja de práctica, así que aquí solo queda la lista de enlaces.

### Enlaces

- [Packet filtering and firewalls (documentación de Docker)](https://docs.docker.com/engine/network/packet-filtering-firewalls/): explica la cadena DOCKER-USER, por qué las reglas de INPUT no ven los puertos publicados y cómo restringir el acceso externo.
- [Exporter toolkit: web configuration](https://github.com/prometheus/exporter-toolkit/blob/master/docs/web-configuration.md): referencia completa del `web.config.file` (TLS, mTLS, basic auth, cabeceras HTTP) que comparten node_exporter y el resto de exporters oficiales.
- [Configuración de Prometheus: scrape_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config): sintaxis exacta de `scheme`, `tls_config`, `basic_auth` y `password_file` en los jobs.
- [Prometheus security model](https://prometheus.io/docs/operating/security/): lo que Prometheus asume sobre quién llega a su API y por qué no trae autenticación de serie.
- [Loki: configuración del servidor](https://grafana.com/docs/loki/latest/configure/): bloque `server` con `http_tls_config` y `client_auth_type` para el mTLS.
- [Promtail: pipeline stages, replace](https://grafana.com/docs/loki/latest/send-data/promtail/stages/replace/): comportamiento exacto de la etapa `replace` con y sin grupos de captura, con ejemplos.
- [Grafana: configure security](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/): acceso anónimo, cookies seguras, proxy inverso y roles.
- [Wiki de nftables: Quick reference](https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes): sintaxis de tablas, cadenas base, conjuntos y expresiones `ct`.
- [Manual de OPNsense: reglas de cortafuegos](https://docs.opnsense.org/manual/firewall.html): orden de evaluación por interfaz, alias y registro.
- [nmap reference guide: port scanning basics](https://nmap.org/book/man-port-scanning-basics.html): definición oficial de los estados open, closed y filtered y de cada tipo de escaneo.
- [Aqua Security: 300.000 servidores Prometheus y exporters expuestos](https://www.aquasec.com/blog/300000-prometheus-servers-and-exporters-exposed-to-dos-attacks/): el informe de diciembre de 2024 con lo que se encontró en instancias abiertas a Internet.

## UT4 · Indicadores, KPI y pruebas del servicio

Enlaces de ampliación de la unidad [UT4](ut/ut4-kpi-pruebas.md): las referencias originales de las señales doradas, USE y RED, el burn rate, los histogramas de Prometheus y la documentación de k6, ZAP, trivy, newman y Jenkins. Ningún apartado de teoría de la unidad se ha movido aquí: todos se explican en clase o los necesita alguna hoja de práctica.

### Enlaces

- [Monitoring Distributed Systems (Google SRE Book)](https://sre.google/sre-book/monitoring-distributed-systems/): el capítulo de donde salen las cuatro señales doradas y el principio de alertar por síntomas.
- [Alerting on SLOs (Google SRE Workbook)](https://sre.google/workbook/alerting-on-slos/): burn rate, ventanas múltiples y la tabla de umbrales; es la referencia para las alarmas de calidad.
- [The USE Method, Brendan Gregg](https://www.brendangregg.com/usemethod.html): el método para recursos, con la lista de comprobación por componente de un servidor Linux.
- [Histograms and summaries (Prometheus)](https://prometheus.io/docs/practices/histograms/): cómo funciona `histogram_quantile`, el error de interpolación y por qué los cubos deben elegirse alrededor del SLO.
- [Query functions (Prometheus)](https://prometheus.io/docs/prometheus/latest/querying/functions/): referencia de `rate`, `increase`, `predict_linear`, `deriv` y `histogram_quantile` con sus condiciones de uso.
- [Documentación de k6](https://grafana.com/docs/k6/latest/): opciones, escenarios y ejecutores, umbrales, salidas (JSON, Prometheus remote write) y la referencia de métricas internas.
- [Thresholds (k6)](https://grafana.com/docs/k6/latest/using-k6/thresholds/): sintaxis completa de umbrales, umbrales por etiqueta y `abortOnFail`.
- [ZAP Baseline Scan](https://www.zaproxy.org/docs/docker/baseline-scan/): parámetros, códigos de salida y formato del fichero de reglas.
- [Trivy](https://trivy.dev/latest/docs/): escaneo de imágenes, filtros de severidad, `--ignore-unfixed`, formatos de salida y modo servidor.
- [Running collections with Newman](https://learning.postman.com/docs/collections/using-newman-cli/command-line-integration-with-newman/): ejecución, variables de entorno y reporters, incluido el JUnit.
- [Pipeline step junit (Jenkins)](https://www.jenkins.io/doc/pipeline/steps/junit/): cómo publica Jenkins los informes y qué significa UNSTABLE.
- [Reporting (Grafana)](https://grafana.com/docs/grafana/latest/dashboards/create-reports/): informes programados; conviene leer el aviso de edición antes de contar con ello.

## UT5 · Explotación de logs, accesos y rendimiento

Enlaces de referencia de la unidad de logs, accesos y rendimiento que se cursa en la empresa: [UT5](ut/ut5-logs-accesos-rendimiento.md).

### Enlaces

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

## UT6 · Copias de seguridad y restauración

Enlaces de ampliación de la [UT6](ut/ut6-copias-seguridad.md), la unidad de copias de seguridad y restauración que se cursa en la empresa. No hay apartados movidos: todos los de la unidad los necesita alguna de las tres actividades.

### Enlaces

- [Documentación de restic](https://restic.readthedocs.io/en/stable/): referencia completa de comandos, backends, variables de entorno y el diseño interno del repositorio (capítulo "References"), que explica por qué deduplica y cifra como lo hace.
- [BorgBackup, guía rápida](https://borgbackup.readthedocs.io/en/stable/quickstart.html): para comparar con restic y entender el modo `--append-only` del servidor.
- [PostgreSQL 17, capítulo Backup and Restore](https://www.postgresql.org/docs/17/backup.html): dump lógico, copia física y PITR con archivado de WAL, que es el siguiente paso cuando el RPO baja de una hora.
- [pg_dump, página de referencia](https://www.postgresql.org/docs/17/app-pgdump.html): todas las opciones de formato y selección, y las limitaciones (no vuelca roles ni tablespaces).
- [mysqldump, manual de MySQL 8.4](https://dev.mysql.com/doc/refman/8.4/en/mysqldump.html): el detalle de `--single-transaction` y cuándo no basta.
- [systemd.timer(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html) y [systemd.time(7)](https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html): sintaxis de `OnCalendar`, `Persistent`, `RandomizedDelaySec` y ejemplos de expresiones de calendario.
- [crontab(5) en man7.org](https://man7.org/linux/man-pages/man5/crontab.5.html): formato de las entradas de cron y las cadenas especiales (`@daily`).
- [node_exporter, textfile collector](https://github.com/prometheus/node_exporter#textfile-collector): cómo exponer métricas propias desde un fichero, con el detalle del `mv` atómico.
- [MinIO, Object Lock y retención](https://min.io/docs/minio/linux/administration/object-management/object-retention.html): modos *governance* y *compliance*, retención por defecto del bucket y su relación con el versionado.
- [Healthchecks.io, documentación](https://healthchecks.io/docs/): el patrón *dead man's switch* con ejemplos de integración desde bash, cron y systemd.
- [Proxmox VE, Backup and Restore](https://pve.proxmox.com/wiki/Backup_and_Restore): modos de copia de VM (`stop`, `suspend`, `snapshot`) y por qué el snapshot no sustituye a la copia de aplicación.
- [Reglamento (UE) 2016/679, RGPD](https://eur-lex.europa.eu/eli/reg/2016/679/oj): artículo 32 (seguridad del tratamiento, que incluye la capacidad de restaurar) y capítulo V (transferencias a terceros países), que condicionan dónde puede estar la copia.

## UT7 · Actualización y gestión de vulnerabilidades

Enlaces para ampliar de la unidad [UT7](ut/ut7-actualizacion-vulnerabilidades.md), Actualización y gestión de vulnerabilidades; ningún apartado de teoría ha salido de la unidad.

### Enlaces

- [Semantic Versioning 2.0.0](https://semver.org/lang/es/): la especificación completa, corta, y la que citan todos los proyectos que dicen seguirla.
- [Documentación de Renovate](https://docs.renovatebot.com/): referencia de todas las opciones de `renovate.json`, con la lista de gestores que detecta y los presets como `config:recommended`.
- [Trivy, documentación oficial](https://trivy.dev/latest/docs/): modos de escaneo, filtros, formatos de salida e integración en CI, con ejemplos para Jenkins y GitLab.
- [Especificación CVSS v4.0 de FIRST](https://www.first.org/cvss/v4.0/specification-document): qué mide cada métrica del vector y cómo se calcula la puntuación; incluye la calculadora.
- [EPSS, de FIRST](https://www.first.org/epss/): el modelo, los datos diarios descargables y por qué complementa a CVSS.
- [Catálogo KEV de CISA](https://www.cisa.gov/known-exploited-vulnerabilities-catalog): la lista de vulnerabilidades con explotación confirmada, descargable en JSON y CSV para cruzarla con vuestro SBOM.
- [OSV.dev](https://osv.dev/): base de datos abierta de vulnerabilidades por paquete y versión, con API y el proyecto `osv-scanner`.
- [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/): el formato del CHANGELOG, con las categorías y las razones de cada regla.
- [Guía de actualización de PostgreSQL](https://www.postgresql.org/docs/current/upgrading.html): diferencia entre versión menor y mayor, y las tres formas de hacer una mayor.
- [Documentación de la imagen oficial de Python en Docker Hub](https://hub.docker.com/_/python): descripción de cada variante (`slim`, `alpine`, `bookworm`, `trixie`) y sus limitaciones.

## UT8 · Terminación segura del contenedor

Enlaces para ampliar de la [UT8](ut/ut8-terminacion-segura.md). En esta unidad todos los apartados de teoría se explican en clase o los necesita alguna hoja de práctica, así que aquí solo están las referencias para seguir por vuestra cuenta.

### Enlaces

- [NIST SP 800-88 rev. 1, Guidelines for Media Sanitization](https://csrc.nist.gov/pubs/sp/800/88/r1/final): la referencia para Clear, Purge y Destroy por tipo de soporte; es lo que citan los pliegos.
- [Reglamento (UE) 2016/679 (RGPD)](https://eur-lex.europa.eu/eli/reg/2016/679/oj): artículos 5, 17 y 30 y considerando 26 para entender conservación, supresión y anonimización.
- [LOPDGDD, Ley Orgánica 3/2018](https://www.boe.es/buscar/act.php?id=BOE-A-2018-16673): el artículo 32 (bloqueo de datos) es el que da forma técnica a "qué se conserva".
- [Docker Compose: `docker compose down`](https://docs.docker.com/reference/cli/docker/compose/down/): qué borra cada opción y qué deja.
- [Distribution registry: API HTTP](https://distribution.github.io/distribution/spec/api/) y [garbage collection](https://distribution.github.io/distribution/about/garbage-collection/): borrado por digest y recolección de blobs.
- [OpenTofu: `tofu destroy`](https://opentofu.org/docs/cli/commands/destroy/) y [`tofu state rm`](https://opentofu.org/docs/cli/commands/state/rm/): destruir sin llevarse lo compartido.
- [Proxmox VE: página de manual de `qm`](https://pve.proxmox.com/pve-docs/qm.1.html): opciones `--purge` y `--destroy-unreferenced-disks` de `qm destroy`.
- [Grafana Loki: borrado de logs](https://grafana.com/docs/loki/latest/operations/storage/logs-deletion/) y [retención](https://grafana.com/docs/loki/latest/operations/storage/retention/): API del compactor, modos de borrado y `retention_stream`.
- [Prometheus: API de administración del TSDB](https://prometheus.io/docs/prometheus/latest/querying/api/#tsdb-admin-apis): `delete_series` y `clean_tombstones`.
- [PostgreSQL 17: VACUUM](https://www.postgresql.org/docs/17/sql-vacuum.html): qué hace VACUUM y qué hace VACUUM FULL con el espacio.
- [restic: gestión de claves](https://restic.readthedocs.io/en/stable/045_working_with_repos.html) y [forget y prune](https://restic.readthedocs.io/en/stable/060_forget.html): cómo se protege la clave maestra y qué reescribe prune.
- [Amazon S3: borrar versiones de objetos](https://docs.aws.amazon.com/AmazonS3/latest/userguide/DeletingObjectVersions.html): por qué un DELETE en un bucket versionado no borra.
- [PhotoRec, CGSecurity](https://www.cgsecurity.org/wiki/PhotoRec): cómo recupera ficheros por firma y por qué no le afecta el sistema de ficheros.
- [man shred(1)](https://man7.org/linux/man-pages/man1/shred.1.html) y [man blkdiscard(8)](https://man7.org/linux/man-pages/man8/blkdiscard.8.html): las advertencias sobre SSD y sistemas copy-on-write están en la propia página de manual.

