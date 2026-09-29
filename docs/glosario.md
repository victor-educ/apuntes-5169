# Glosario

Términos de esta asignatura, en una o dos frases, con la unidad donde se tratan a fondo. Toda sigla que se repite en las unidades tiene aquí su entrada, aunque venga de Despliegue; los conceptos de infraestructura que solo se mencionan de paso (hipervisor, DMZ, IaC, pipeline...) están en el [glosario de la asignatura de despliegue](https://victor-educ.github.io/apuntes-5166/glosario/).

**Acta de baja** · Documento que cierra la retirada de un servicio: quién autorizó, lista de comprobación verificada con evidencias, qué se conservó y qué claves se destruyeron. UT8.

**Alertmanager** · Componente que recibe las alertas de Prometheus o Loki, las agrupa, las enruta según etiquetas, las silencia o inhibe y las envía a correo, chat o webhook. UT2.

**Anonimizar** · Transformar datos personales de forma que no se pueda identificar a la persona, conservando su utilidad estadística. UT8.

**API** · Interfaz de programación: el conjunto de rutas HTTP con las que un programa pide datos a otro o se los envía. La del servicio del curso está escrita en Python con Flask y escucha en el 8080; Prometheus, Loki y Alertmanager tienen también la suya. UT1.

**Append-only** · Modo de acceso al destino de las copias en el que el cliente puede añadir datos pero no borrar ni sobrescribir los que ya hay. Es lo que impide que un ransomware que entre por el host copiado destruya también las copias. UT6.

**Basic auth** · Autenticación HTTP con usuario y contraseña en cada petición. Es lo que se pone en los exporters para que solo respondan a Prometheus, y siempre sobre TLS, porque la contraseña viaja en cada petición. UT3.

**bcrypt** · Hash lento pensado para guardar contraseñas. El fichero `--web.config.file` de los exporters guarda así la contraseña de basic auth, nunca en claro. UT3.

**BD** · Base de datos. En el servicio del curso, PostgreSQL con la base `servicio`. UT4, UT5.

**Borrado criptográfico** · Destruir la clave con la que se cifraron unos datos, de modo que quedan irrecuperables aunque los ficheros sigan existiendo. UT8.

**Bucket** · Contenedor con nombre de un almacenamiento de objetos, donde se guardan los ficheros. En el curso, `backups`, donde vive el repositorio restic. UT6.

**Burn rate** · Velocidad a la que se consume el presupuesto de error. Un burn rate de 2 gasta en medio mes lo previsto para uno. UT4.

**CA** · Autoridad de certificación: quien firma los certificados y en quien confían los clientes. La del curso, `Lab 5166 CA`, se crea con `openssl ca` en la A3.2 de Despliegue, vive en `~/ca/` del puesto de administración y firma todo lo que se cifra en las dos asignaturas. UT3.

**cAdvisor** · Exporter que lee los cgroups y expone métricas de CPU, memoria, red y disco por contenedor. UT1.

**Cardinalidad** · Número de series distintas que genera una métrica según los valores que toman sus etiquetas. Una etiqueta que cambia en cada despliegue o en cada petición la dispara. UT1.

**Catálogo de alarmas** · Colección de fichas, una por alarma, con origen, posible fallo, impacto, pasos de análisis, resolución y escalado. UT4.

**cgroups** · Mecanismo del kernel Linux que limita y contabiliza los recursos de un grupo de procesos. Es de donde salen las métricas de recursos de un contenedor. UT1.

**Chunk** · Bloque comprimido de líneas de log en Loki. Loki solo indexa las etiquetas del stream y el tiempo; el texto se busca por fuerza bruta dentro de los chunks que las etiquetas seleccionan. UT1.

**Clear, Purge y Destroy** · Los tres niveles de borrado de soportes de la guía NIST SP 800-88: sobrescritura lógica, borrado que resiste una recuperación de laboratorio y destrucción física del soporte. UT8.

**CN** · Common Name: el campo clásico de nombre de un certificado. Los clientes modernos lo ignoran y miran el SAN; en el curso sirve para distinguir los certificados en `index.txt`. UT3.

**Copy-on-write** · Forma de escribir de ZFS y btrfs que nunca sobrescribe un bloque en su sitio, sino que escribe el nuevo en otro lugar y actualiza los punteros; por eso sobrescribir un fichero no elimina el contenido anterior. UT8.

**Core dump** · Volcado de la memoria de un proceso en el momento de fallar, que se analiza con un depurador. UT5.

**Counter, gauge, histogram, summary** · Los cuatro tipos de métrica de Prometheus: acumulador que solo sube, valor que sube y baja, distribución en cubos, percentiles calculados en el cliente. UT1.

**CPD** · Centro de proceso de datos: la sala o el edificio donde una organización tiene sus servidores. UT2, UT6.

**CPU** · Procesador. Su uso se lee por host con node_exporter y por contenedor con cAdvisor. UT1, UT5.

**CRL** · Lista de certificados revocados que publica una autoridad de certificación. Un certificado revocado sigue valiendo para quien no la consulta, así que publicarla es parte de dar de baja un servicio. UT8.

**CrowdSec** · Alternativa a fail2ban que comparte listas de IP maliciosas entre instalaciones y bloquea mediante bouncers. UT5.

**CVE** · Identificador público de una vulnerabilidad conocida (CVE-2025-12345). UT7.

**CVSS** · Puntuación de 0 a 10 de la gravedad de una vulnerabilidad, con un vector que describe cómo se explota. UT7.

**Dead man's switch** · Alerta que salta cuando deja de llegar una señal periódica; sirve para detectar que una copia de seguridad no se ha ejecutado. UT6.

**Deduplicación** · Técnica de restic y borg que trocea los ficheros en bloques de tamaño variable y solo guarda los que no estaban. Cada copia es completa al restaurar y cuesta lo que una incremental. UT6.

**Digest** · Hash SHA-256 que identifica de forma inequívoca una imagen de contenedor. Las etiquetas se mueven; el digest no. UT7.

**DNAT** · NAT de destino: reescribir la dirección o el puerto de destino de un paquete. Es lo que hace Docker al publicar un puerto, y por eso ese tráfico pasa por la cadena forward y no por input. UT3.

**DNS** · Sistema de nombres: traduce nombres como `grafana.lab` a direcciones IP. En la VPC lo sirve OPNsense desde la A3.1 de Despliegue, con un host override por cada nombre del laboratorio. UT3.

**DOCKER-USER** · Cadena del cortafuegos que Docker crea al principio de `FORWARD` y promete no tocar. Es el único sitio donde una regla del host filtra el tráfico hacia un puerto publicado por un contenedor. UT3, UT5.

**DPO** · Delegado de protección de datos: la persona que supervisa el cumplimiento del RGPD en una organización. En una baja custodia el volcado bloqueado y decide su destrucción. UT8.

**Driver de logs** · Componente de Docker que decide qué se hace con stdout y stderr de un contenedor: fichero JSON, journald, Loki... UT1.

**ENS** · Esquema Nacional de Seguridad: la norma de seguridad obligatoria para el sector público en España. Fija, entre otras cosas, cuánto se conservan los logs y qué nivel de borrado se exige según la categoría del sistema. UT8.

**EPSS** · Exploit Prediction Scoring System: probabilidad, entre 0 y 1, de que una CVE se explote en los próximos 30 días. Complementa a CVSS, que solo mide la gravedad teórica. UT7.

**Exporter** · Programa que expone métricas de algo (el sistema, PostgreSQL, nginx) en formato Prometheus. UT1.

**fail2ban** · Servicio que lee logs, detecta intentos de acceso fallidos repetidos y bloquea la IP en el firewall durante un tiempo. UT5.

**Fatiga de alertas** · Situación en la que el canal de avisos recibe tantas notificaciones que el equipo deja de leerlas, y con ellas la que importaba. Se corrige subiendo el umbral, alargando el `for` o bajando la alarma a `warning`. UT2.

**Fingerprint** · Hash que Alertmanager calcula con las etiquetas de una alerta. Es estable, así que empareja el aviso de firing con su resolved y sirve para saber qué incidencia cerrar. UT2.

**Grype** · Escáner de vulnerabilidades que trabaja sobre imágenes o sobre un SBOM generado por Syft. UT7.

**HTTP** · Protocolo de la web. Con él Prometheus lee `/metrics`, Promtail envía a Loki y se llama a la API del servicio; cifrado con TLS pasa a ser HTTPS. UT1.

**Inhibición** · Regla de Alertmanager que suprime unas alertas mientras otra está activa (si el host está caído, no avisar de cada contenedor de ese host). UT2.

**IP** · Dirección de red de una máquina. Las del laboratorio son fijas y están en la tabla de convenciones de la página del laboratorio. UT1.

**IPAM** · Gestión de direcciones IP: el registro con el que el SDN de Proxmox apunta qué dirección tiene cada VM. Al dar de baja un entorno se comprueba que sus direcciones quedan libres. UT8.

**JSON** · Formato de texto para datos estructurados. Es el de las respuestas de las API, el de los logs estructurados del servicio y el de los informes de k6 y Trivy. UT1.

**JUnit** · Formato XML de resultados de pruebas, con un elemento por caso. pytest y newman lo generan, y Jenkins lo lee para pintar la tendencia de casos correctos y fallidos entre builds. UT4.

**JVM** · Máquina virtual de Java, el runtime que ejecuta las aplicaciones Java. Cuando muere deja su propio informe de fallo (`hs_err_pid<N>.log`) y puede volcar el heap para buscar fugas. UT5.

**k6** · Herramienta de pruebas de carga con scripts en JavaScript, umbrales y salida a Prometheus o JSON. UT4.

**KEV** · Catálogo de CISA de vulnerabilidades explotadas activamente. Si una CVE está ahí, va la primera. UT7.

**KPI** · Indicador clave (key performance indicator): un valor calculado a partir de varias métricas que describe el servicio para alguien que no conoce el código, como la disponibilidad o la latencia p95. UT4.

**Línea base** · Los valores normales de un sistema en un periodo sin incidentes, contra los que se compara cualquier medición posterior. UT5.

**LogQL** · Lenguaje de consulta de Loki: selectores de stream, filtros y funciones que convierten logs en métricas. UT2.

**logrotate** · Servicio que rota, comprime y caduca los ficheros de registro del sistema según una configuración por servicio. UT5.

**Loki** · Sistema de almacenamiento e indexación de logs de Grafana Labs, que indexa etiquetas y no el texto. UT1.

**LOPDGDD** · Ley Orgánica 3/2018 de Protección de Datos Personales y garantía de los derechos digitales, que adapta el RGPD en España. Su artículo 32 regula el bloqueo de datos. UT8.

**LUKS** · Formato estándar de cifrado de discos en Linux. Su cabecera guarda los keyslots, y borrarlos con `cryptsetup luksErase` deja el contenido irrecuperable. UT8.

**LVM** · Gestor de volúmenes lógicos de Linux. Con LVM-thin, el almacenamiento habitual de Proxmox, un snapshot conserva los bloques de un fichero aunque después se sobrescriba. UT8.

**Mailpit** · Servidor de correo de pruebas: acepta cualquier mensaje por SMTP y lo enseña en una web en lugar de entregarlo. UT2.

**mTLS** · TLS mutuo: cliente y servidor presentan certificado. Se usa entre Promtail y Loki. UT3.

**NAS** · Almacenamiento en red al que se llega por SSH o por un protocolo de ficheros. Es el destino habitual de las copias en instalaciones pequeñas y el terreno propio de borg. UT6.

**NAT** · Traducción de direcciones: el cortafuegos o el kernel reescriben el origen o el destino de los paquetes (SNAT para salir, DNAT para publicar). UT3.

**newman** · Ejecutor de línea de comandos de las colecciones de Postman. Es la forma de llevar esas pruebas a un pipeline, con informe JUnit. UT4.

**NIST** · Instituto de estándares de EE. UU. Mantiene la NVD y publica la guía NIST SP 800-88 de borrado de soportes, que ordena los métodos en Clear, Purge y Destroy. UT7, UT8.

**NTP** · Network Time Protocol: el protocolo con el que las máquinas se ponen en hora por la red. En las VM lo usa chrony, y desde la A3.1 de Despliegue toman la hora de OPNsense. UT1.

**NVD** · National Vulnerability Database: la base de datos de vulnerabilidades del NIST, con la ficha completa de cada CVE y su vector CVSS. UT7.

**Object Lock** · Función de S3 y MinIO que impide borrar o modificar un objeto durante un periodo; protege las copias ante ransomware. UT6.

**OCSP** · Protocolo con el que un cliente pregunta en línea a la autoridad de certificación si un certificado concreto está revocado, en lugar de descargarse la CRL entera. UT8.

**OOM** · Out of memory. El kernel mata el proceso que más memoria usa cuando un cgroup o el sistema se queda sin ella; el contenedor sale con código 137. UT1, UT5.

**OSV** · Base de datos abierta de vulnerabilidades por paquete y versión, con API y el escáner `osv-scanner`. Es la fuente que se consulta cuando el hallazgo viene de una dependencia de la aplicación. UT7.

**OWASP** · Fundación abierta de seguridad de aplicaciones web, autora de ZAP. UT4.

**p95** · Percentil 95: el valor que no supera el 95 % de las muestras. Se usa en lugar del máximo para que un pico aislado no distorsione la latencia o la línea base. UT1, UT4, UT5.

**Password spraying** · Ataque de acceso que prueba una contraseña muy común contra muchos usuarios distintos, para no disparar los bloqueos por intentos fallidos de una sola cuenta. UT5.

**Pending y firing** · Estados de una regla de alerta: pending mientras la condición se cumple pero aún no ha pasado el `for`, firing cuando ya se notifica a Alertmanager. UT2.

**photorec** · Herramienta que recupera ficheros borrados buscando sus firmas en el disco. Sirve para demostrar que un borrado no ha sido suficiente. UT8.

**PID** · Identificador de proceso. `docker inspect` da el del proceso principal de un contenedor, y con él se llega a su cgroup en `/proc/<PID>/cgroup`. UT1, UT5.

**PITR** · Point-in-time recovery: restaurar una base de datos al instante exacto anterior al fallo, combinando una copia física con el archivado de WAL. UT6.

**PR** · Pull request: propuesta de cambio sobre un repositorio que alguien revisa antes de fusionarla en la rama principal. Es lo que abre Renovate por cada actualización disponible. UT7.

**Presupuesto de error** · Lo que el servicio puede fallar sin incumplir su SLO: con un objetivo del 99,5 % mensual, unas 3,6 horas. UT4.

**PromQL** · Lenguaje de consulta de Prometheus: selecciona series por sus etiquetas y las combina con funciones como `rate()`, `sum by` o `histogram_quantile()`. UT1, UT2.

**Promtail** · Agente que lee ficheros de log (o el socket de Docker), añade etiquetas y los envía a Loki. UT1.

**promtool, amtool y lokitool** · Utilidades de línea de comandos que validan y prueban los ficheros de Prometheus, Alertmanager y el ruler de Loki antes de recargarlos. UT2.

**Prune** · Operación de restic que reempaqueta el repositorio y elimina los datos que ya no pertenecen a ningún snapshot después de un `forget`. Es lenta, bloquea el repositorio y se programa aparte de la copia. UT6, UT8.

**RAM** · Memoria principal. En el laboratorio se asigna a cada VM al clonarla con `qm set <ID> --memory`, y su reparto está en la página del laboratorio. UT1, UT5.

**Recording rule** · Regla de Prometheus que precalcula una expresión y la guarda como métrica nueva. UT2.

**RED (método)** · Revisión de un servicio por tres medidas: peticiones por segundo, errores por segundo y distribución de la latencia. Mira hacia el usuario, donde USE mira hacia los recursos. UT4.

**Regla 3-2-1** · Tres copias de los datos, en dos soportes distintos, con una de ellas fuera del sitio. UT6.

**Relabel** · Reglas de Prometheus y de Promtail que copian, renombran o descartan etiquetas: sobre los orígenes antes de leerlos (`relabel_configs`) o sobre las series ya leídas, antes de guardarlas (`metric_relabel_configs`). UT1.

**Renovate** · Bot que abre un pull request cuando hay versión nueva de una imagen o dependencia. UT7.

**restic** · Herramienta de copias de seguridad con deduplicación, cifrado y retención integrada, hacia disco, SFTP o S3. UT6.

**RFC** · Documento que fija un estándar de Internet. En el curso aparece el RFC 3339, el formato de fecha (`2026-10-13T09:12:00Z`) de los logs estructurados y de Alertmanager. UT1, UT2.

**RGPD** · Reglamento General de Protección de Datos. Fija los plazos de conservación, el derecho de supresión y la obligación de poder restaurar los datos, que es lo que condiciona dónde vive una copia y cuándo se borra. UT6, UT8.

**Rollback** · Volver a la versión anterior tras una actualización fallida. Exige tener la etiqueta anterior y una copia previa. UT7.

**RPO y RTO** · Recovery Point Objective: cuántos datos se pueden perder (cada cuánto se copia). Recovery Time Objective: cuánto se puede tardar en restaurar. UT6.

**Ruler** · Componente de Loki que evalúa reglas de alerta escritas en LogQL y las envía al mismo Alertmanager que Prometheus. UT2.

**Runbook** · Instrucciones paso a paso para atender una alarma concreta, enlazadas desde la propia alerta. UT4.

**S3 y MinIO** · Almacenamiento de objetos por HTTP: S3 es el servicio de Amazon y MinIO su equivalente instalable, que habla el mismo protocolo. En el laboratorio, el MinIO de 10.10.0.30 guarda el repositorio restic en el bucket `backups`. UT6, UT7, UT8.

**SAN** · Subject Alternative Name: el campo del certificado con los nombres DNS y las IP para los que vale. Es lo que validan Prometheus, Promtail y los navegadores. UT3.

**SARIF** · Static Analysis Results Interchange Format: formato normalizado de resultados de análisis de código, el que Gitea, GitLab y GitHub pintan junto al código. Trivy lo escribe con `--format sarif`. UT7.

**SBOM** · Software Bill of Materials: inventario de todos los componentes de una imagen, en formato CycloneDX o SPDX. UT7.

**Scrape** · Lectura periódica que hace Prometheus del `/metrics` de cada target (cada 15 s en el laboratorio). Si falla, esa muestra no existe y queda un hueco. UT1.

**SDN** · Red definida por software: en Proxmox, las zonas, VNets y subredes que se declaran en el nodo y sobre las que viven las VPC del laboratorio. UT8.

**SEGV** · Señal de violación de segmento: la que el kernel manda a un proceso que accede a memoria que no es suya, y la que deja un core dump. En el laboratorio se provoca con `kill -SEGV`. UT5.

**Señales doradas** · Las cuatro medidas mínimas de un servicio: latencia, tráfico, errores y saturación. UT4.

**Sidecar** · Contenedor auxiliar que se pone al lado de un servicio para darle lo que no tiene sin tocar su código. En el curso, un nginx que añade TLS y basic auth delante de cAdvisor y de las métricas de la API. UT3.

**Silencio** · Supresión temporal de notificaciones de Alertmanager que coinciden con unas etiquetas, típica en mantenimientos. UT2.

**SLI, SLO, SLA** · Indicador de nivel de servicio (la medida), objetivo (el valor comprometido internamente) y acuerdo (el compromiso contractual con consecuencias). UT4.

**SMTP** · Protocolo de envío de correo. Alertmanager lo usa para mandar los avisos, y en el laboratorio los recibe Mailpit. UT2.

**SO** · Sistema operativo. En una imagen, la capa de paquetes de la distribución base, que Trivy escanea por separado de las dependencias del lenguaje. UT7.

**SQL** · Lenguaje de consulta de las bases de datos relacionales como PostgreSQL. UT6, UT8.

**SRE** · Site Reliability Engineering: la forma de operar servicios que Google publicó como libro, de donde salen las señales doradas y el principio de alertar por síntomas. UT2, UT4.

**SSD** · Disco de estado sólido. Su controlador reparte las escrituras entre celdas (wear leveling), así que sobrescribir un fichero no garantiza borrar los datos originales. UT5, UT8.

**SSH** · Protocolo de acceso remoto cifrado a una máquina. En el laboratorio también sirve para los túneles (`ssh -L`) con los que se llega a Prometheus, Alertmanager y Mailpit desde el 3 de diciembre. UT3.

**SSL** · Nombre antiguo de TLS, que sigue apareciendo en mensajes de error y en directivas de nginx (`ssl_protocols`, `ssl_verify_client`). UT3.

**Steal** · Porcentaje de tiempo que una máquina virtual está lista para ejecutar pero el hipervisor no le da CPU. Señal de que el host está sobrevendido. UT5.

**Structured metadata** · Campos que Loki guarda junto a cada línea sin meterlos en el índice. Sirven para filtrar por valores de cardinalidad alta, como `request_id`. UT1.

**Syft** · Generador de SBOM a partir de imágenes, directorios o ficheros de dependencias. UT7.

**SYN** · Primer paquete de una conexión TCP. nmap con `-sS` manda solo el SYN y, si vuelve un SYN-ACK, da el puerto por abierto. UT3, UT5.

**Timer de systemd** · Unit que lanza una unit de servicio a una hora o cada cierto tiempo (`OnCalendar`), con log en el journal y recuperación de ejecuciones perdidas (`Persistent=true`). Es la alternativa a cron. UT5, UT6.

**TLS** · Protocolo que cifra una conexión y permite al cliente comprobar con un certificado quién es el servidor; es la S de HTTPS. En el curso, versión 1.3. UT3.

**Trivy** · Escáner de vulnerabilidades y configuraciones inseguras para imágenes, sistemas de ficheros, código IaC y repositorios. UT7.

**TSDB** · Base de datos de series temporales. Es el almacén local de Prometheus y el formato de índice de Loki, y se administra con su propia API para borrar series. UT1, UT8.

**UID** · Identificador numérico de un usuario en Linux. Los ficheros de un volumen que se restaura tienen que pertenecer al UID con el que corre el contenedor, que no siempre coincide con el del host. UT6.

**Umbral** · Valor a partir del cual una métrica se considera anómala. Se fija desde la documentación del servicio y se ajusta con datos reales. UT2, UT4.

**URL** · Dirección completa de un recurso web, con protocolo, nombre, puerto y ruta (`http://localhost:9090/api/v1/query`). UT2.

**USE (método)** · Revisión de rendimiento que mira, para cada recurso, su utilización, su saturación y sus errores. UT4, UT5.

**VACUUM FULL** · Operación de PostgreSQL que reescribe una tabla eliminando las páginas muertas que DELETE y DROP dejan en el disco. UT8.

**Ventana de mantenimiento** · Franja horaria acordada de antemano en la que se permite actualizar o intervenir sobre un servicio en producción. UT7.

**Versionado semántico** · Convención `MAYOR.MENOR.PARCHE` con la que un proyecto promete qué puede romperse al actualizar. UT7.

**VM** · Máquina virtual. Las del laboratorio las crea Despliegue en Proxmox, cada una con un ID fijo. UT1.

**VPC** · Red privada virtual del laboratorio, dividida en zonas (gestión, front, back, data) detrás de OPNsense. La dev se monta en la UT2 de Despliegue y app01 y mon01 pasan a ella el 27 de noviembre. UT3.

**VU** · Usuario virtual de k6: un bucle que ejecuta el escenario una y otra vez. Las pruebas de carga se dimensionan en VU (10, 50, 100). UT4.

**WAL** · Write-ahead log: diario en el que se anotan las escrituras antes de aplicarlas. PostgreSQL lo usa para poder restaurar a un instante concreto, y el Prometheus de `remote_write` para no perder muestras si el central no responde. UT1, UT6.

**Working set** · Memoria que un contenedor usa de verdad, descontando la caché de página que el kernel puede liberar. Es la que se compara con el límite. UT1.

**XML** · Formato de texto con etiquetas. En el curso, el de los informes JUnit. UT4.

**YAML** · Formato de configuración por sangría que usan Prometheus, Alertmanager, Loki, Promtail y los ficheros compose. UT1.

**ZAP** · OWASP Zed Attack Proxy, escáner de seguridad de aplicaciones web. En modo baseline hace un análisis pasivo rápido. UT4.

**ZFS** · Sistema de ficheros con copy-on-write y snapshots; por eso sobrescribir un fichero en ZFS no borra sus bloques anteriores. UT8.
