# Bibliografía y enlaces

Cada unidad tiene su sección "Para ampliar" con enlaces concretos. Aquí está lo transversal. La documentación de Proxmox, OpenTofu, Ansible, Jenkins y las plataformas de nube está en la [bibliografía de la asignatura de despliegue](https://victor-educ.github.io/apuntes-5166/recursos/).

## Documentación oficial

| Herramienta | Enlace | Qué mirar |
|-------------|--------|-----------|
| Prometheus | [prometheus.io/docs](https://prometheus.io/docs/introduction/overview/) | PromQL, reglas, Alertmanager, buenas prácticas de instrumentación |
| Alertmanager | [prometheus.io/docs/alerting](https://prometheus.io/docs/alerting/latest/overview/) | Rutas, inhibición, receptores, plantillas |
| Loki | [grafana.com/docs/loki](https://grafana.com/docs/loki/latest/) | LogQL, Promtail, retención, ruler |
| Grafana | [grafana.com/docs/grafana](https://grafana.com/docs/grafana/latest/) | Dashboards, alerting, provisioning |
| cAdvisor | [github.com/google/cadvisor](https://github.com/google/cadvisor) | Métricas expuestas y despliegue |
| Docker | [docs.docker.com/engine/logging](https://docs.docker.com/engine/logging/) | Drivers de logs, eventos, límites de recursos |
| cgroups v2 | [kernel.org cgroup-v2](https://docs.kernel.org/admin-guide/cgroup-v2.html) | De dónde salen las métricas de recursos |
| nftables | [wiki.nftables.org](https://wiki.nftables.org/) | Filtrado por host |
| k6 | [grafana.com/docs/k6](https://grafana.com/docs/k6/latest/) | Scripts, umbrales, escenarios, salidas |
| OWASP ZAP | [zaproxy.org/docs](https://www.zaproxy.org/docs/) | Baseline scan, informes |
| Postman / newman | [learning.postman.com](https://learning.postman.com/docs/collections/using-newman-cli/command-line-integration-with-newman/) | Ejecución de colecciones en CI |
| fail2ban | [github.com/fail2ban/fail2ban](https://github.com/fail2ban/fail2ban) | Jails y filtros |
| CrowdSec | [docs.crowdsec.net](https://docs.crowdsec.net/) | Alternativa colaborativa |
| restic | [restic.readthedocs.io](https://restic.readthedocs.io/) | Copias, retención, restauración |
| MinIO | [min.io/docs](https://min.io/docs/minio/linux/index.html) | Versionado, Object Lock, políticas |
| Trivy | [trivy.dev/docs](https://trivy.dev/latest/docs/) | Escaneo de imágenes, ficheros y configuración |
| Syft y Grype | [github.com/anchore](https://github.com/anchore) | SBOM y escaneo |
| Renovate | [docs.renovatebot.com](https://docs.renovatebot.com/) | Configuración y reglas de paquetes |
| NVD, OSV, KEV | [nvd.nist.gov](https://nvd.nist.gov/), [osv.dev](https://osv.dev/), [cisa.gov/kev](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) | Fichas de vulnerabilidades y prioridad |
| PostgreSQL | [postgresql.org/docs](https://www.postgresql.org/docs/current/) | pg_dump, VACUUM, pg_stat |
| NIST SP 800-88 | [csrc.nist.gov](https://csrc.nist.gov/pubs/sp/800/88/r2/final) | Borrado seguro de soportes |

## Libros

- Betsy Beyer y otros, *Site Reliability Engineering* y *The Site Reliability Workbook* (Google). Gratis en [sre.google/books](https://sre.google/books/). Los capítulos de monitorización, alertas, SLO y gestión de incidentes son la base de las UT2 y UT4.
- Brian Brazil, *Prometheus: Up & Running* (O'Reilly, 2ª ed.). El libro de referencia de Prometheus, escrito por uno de sus desarrolladores.
- Charity Majors y otros, *Observability Engineering* (O'Reilly). Por qué los logs estructurados y la correlación cambian la forma de depurar.
- W. Curtis Preston, *Modern Data Protection* (O'Reilly). Copias de seguridad explicadas por alguien que lleva treinta años restaurándolas.
- Liz Rice, *Container Security* (O'Reilly). Qué hay dentro de una imagen y por dónde se ataca.

## Para practicar fuera del aula

- [Prometheus playground de PromLabs](https://promlabs.com/promql-cheat-sheet/) y su [chuleta de PromQL](https://promlabs.com/promql-cheat-sheet/).
- [Grafana Play](https://play.grafana.org/) para explorar dashboards, Loki y alertas sin instalar nada.
- [k6 examples](https://grafana.com/docs/k6/latest/examples/) con scripts de todos los tipos de prueba.
- [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/) para probar ZAP contra una aplicación hecha para ser atacada.
- [Awesome Prometheus alerts](https://samber.github.io/awesome-prometheus-alerts/) con reglas de alerta para casi cualquier exporter, como punto de partida.

## Comunidades y noticias

- [Blog de Grafana Labs](https://grafana.com/blog/) y [blog de Prometheus](https://prometheus.io/blog/).
- [Aqua Security blog](https://www.aquasec.com/blog/) sobre Trivy y seguridad de contenedores.
- [oss-security](https://www.openwall.com/lists/oss-security/) para avisos de vulnerabilidades en software libre.
- [r/sysadmin](https://www.reddit.com/r/sysadmin/) y [r/devops](https://www.reddit.com/r/devops/).

## Normativa

- Real decreto del curso de especialización: búscalo en el [BOE](https://www.boe.es/). El resumen de RA y CE está en la [página de presentación](modulo.md).
- [RGPD](https://eur-lex.europa.eu/eli/reg/2016/679/oj?locale=es) y [LOPDGDD](https://www.boe.es/buscar/act.php?id=BOE-A-2018-16673) para plazos de conservación y borrado de datos personales (UT6 y UT8).

## Créditos de las imágenes

| Fichero | Autor y licencia | Origen |
|---------|------------------|--------|
| prometheus-arquitectura.svg | Proyecto Prometheus, Apache 2.0 | github.com/prometheus/prometheus |
| grafana-dashboard.png | Joel Kennedy, dominio público | Wikimedia Commons |
| cvss4.svg | FIRST, CC BY-SA 4.0 | Wikimedia Commons |
| semver.svg | Surjit Bains, CC BY-SA 4.0 | Wikimedia Commons |
| fail2ban-logo.png | Proyecto fail2ban, GFDL | Wikimedia Commons |
| dmz-un-firewall.svg | Pbroks13, dominio público | Wikimedia Commons |
| Logos de Prometheus, Grafana, Loki y Jenkins | Marcas de sus titulares, uso nominativo | Wikimedia Commons, jenkins.io |
