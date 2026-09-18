# Glosario

Términos de esta asignatura, en una o dos frases, con la unidad donde se tratan a fondo. Los de infraestructura (hipervisor, VPC, DMZ, IaC, pipeline...) están en el [glosario de la asignatura de despliegue](https://victor-educ.github.io/apuntes-5166/glosario/).

**Acta de baja** · Documento que cierra la retirada de un servicio: quién autorizó, lista de comprobación verificada con evidencias, qué se conservó y qué claves se destruyeron. UT8.

**Alertmanager** · Componente que recibe las alertas de Prometheus o Loki, las agrupa, las enruta según etiquetas, las silencia o inhibe y las envía a correo, chat o webhook. UT2.

**Anonimizar** · Transformar datos personales de forma que no se pueda identificar a la persona, conservando su utilidad estadística. UT8.

**Append-only** · Modo de acceso al destino de las copias en el que el cliente puede añadir datos pero no borrar ni sobrescribir los que ya hay. Es lo que impide que un ransomware que entre por el host copiado destruya también las copias. UT6.

**Borrado criptográfico** · Destruir la clave con la que se cifraron unos datos, de modo que quedan irrecuperables aunque los ficheros sigan existiendo. UT8.

**Burn rate** · Velocidad a la que se consume el presupuesto de error. Un burn rate de 2 gasta en medio mes lo previsto para uno. UT4.

**cAdvisor** · Exporter que lee los cgroups y expone métricas de CPU, memoria, red y disco por contenedor. UT1.

**Catálogo de alarmas** · Colección de fichas, una por alarma, con origen, posible fallo, impacto, pasos de análisis, resolución y escalado. UT4.

**cgroups** · Mecanismo del kernel Linux que limita y contabiliza los recursos de un grupo de procesos. Es de donde salen las métricas de recursos de un contenedor. UT1.

**Chunk** · Bloque comprimido de líneas de log en Loki. Loki solo indexa las etiquetas del stream y el tiempo; el texto se busca por fuerza bruta dentro de los chunks que las etiquetas seleccionan. UT1.

**Clear, Purge y Destroy** · Los tres niveles de borrado de soportes de la guía NIST SP 800-88: sobrescritura lógica, borrado que resiste una recuperación de laboratorio y destrucción física del soporte. UT8.

**Copy-on-write** · Forma de escribir de ZFS y btrfs que nunca sobrescribe un bloque en su sitio, sino que escribe el nuevo en otro lugar y actualiza los punteros; por eso sobrescribir un fichero no elimina el contenido anterior. UT8.

**Core dump** · Volcado de la memoria de un proceso en el momento de fallar, que se analiza con un depurador. UT5.

**Counter, gauge, histogram, summary** · Los cuatro tipos de métrica de Prometheus: acumulador que solo sube, valor que sube y baja, distribución en cubos, percentiles calculados en el cliente. UT1.

**CRL** · Lista de certificados revocados que publica una autoridad de certificación. Un certificado revocado sigue valiendo para quien no la consulta, así que publicarla es parte de dar de baja un servicio. UT8.

**CrowdSec** · Alternativa a fail2ban que comparte listas de IP maliciosas entre instalaciones y bloquea mediante bouncers. UT5.

**CVE** · Identificador público de una vulnerabilidad conocida (CVE-2025-12345). UT7.

**CVSS** · Puntuación de 0 a 10 de la gravedad de una vulnerabilidad, con un vector que describe cómo se explota. UT7.

**Dead man's switch** · Alerta que salta cuando deja de llegar una señal periódica; sirve para detectar que una copia de seguridad no se ha ejecutado. UT6.

**Deduplicación** · Técnica de restic y borg que trocea los ficheros en bloques de tamaño variable y solo guarda los que no estaban. Cada copia es completa al restaurar y cuesta lo que una incremental. UT6.

**Digest** · Hash SHA-256 que identifica de forma inequívoca una imagen de contenedor. Las etiquetas se mueven; el digest no. UT7.

**Driver de logs** · Componente de Docker que decide qué se hace con stdout y stderr de un contenedor: fichero JSON, journald, Loki... UT1.

**EPSS** · Exploit Prediction Scoring System: probabilidad, entre 0 y 1, de que una CVE se explote en los próximos 30 días. Complementa a CVSS, que solo mide la gravedad teórica. UT7.

**Exporter** · Programa que expone métricas de algo (el sistema, PostgreSQL, nginx) en formato Prometheus. UT1.

**fail2ban** · Servicio que lee logs, detecta intentos de acceso fallidos repetidos y bloquea la IP en el firewall durante un tiempo. UT5.

**Fatiga de alertas** · Situación en la que el canal de avisos recibe tantas notificaciones que el equipo deja de leerlas, y con ellas la que importaba. Se corrige subiendo el umbral, alargando el `for` o bajando la alarma a `warning`. UT2.

**Grype** · Escáner de vulnerabilidades que trabaja sobre imágenes o sobre un SBOM generado por Syft. UT7.

**Inhibición** · Regla de Alertmanager que suprime unas alertas mientras otra está activa (si el host está caído, no avisar de cada contenedor de ese host). UT2.

**k6** · Herramienta de pruebas de carga con scripts en JavaScript, umbrales y salida a Prometheus o JSON. UT4.

**KEV** · Catálogo de CISA de vulnerabilidades explotadas activamente. Si una CVE está ahí, va la primera. UT7.

**Línea base** · Los valores normales de un sistema en un periodo sin incidentes, contra los que se compara cualquier medición posterior. UT5.

**LogQL** · Lenguaje de consulta de Loki: selectores de stream, filtros y funciones que convierten logs en métricas. UT2.

**logrotate** · Servicio que rota, comprime y caduca los ficheros de registro del sistema según una configuración por servicio. UT5.

**Loki** · Sistema de almacenamiento e indexación de logs de Grafana Labs, que indexa etiquetas y no el texto. UT1.

**LUKS** · Formato estándar de cifrado de discos en Linux. Su cabecera guarda los keyslots, y borrarlos con `cryptsetup luksErase` deja el contenido irrecuperable. UT8.

**mTLS** · TLS mutuo: cliente y servidor presentan certificado. Se usa entre Promtail y Loki. UT3.

**NAS** · Almacenamiento en red al que se llega por SSH o por un protocolo de ficheros. Es el destino habitual de las copias en instalaciones pequeñas y el terreno propio de borg. UT6.

**newman** · Ejecutor de línea de comandos de las colecciones de Postman. Es la forma de llevar esas pruebas a un pipeline, con informe JUnit. UT4.

**NVD** · National Vulnerability Database: la base de datos de vulnerabilidades del NIST, con la ficha completa de cada CVE y su vector CVSS. UT7.

**Object Lock** · Función de S3 y MinIO que impide borrar o modificar un objeto durante un periodo; protege las copias ante ransomware. UT6.

**OCSP** · Protocolo con el que un cliente pregunta en línea a la autoridad de certificación si un certificado concreto está revocado, en lugar de descargarse la CRL entera. UT8.

**OOM** · Out of memory. El kernel mata el proceso que más memoria usa cuando un cgroup o el sistema se queda sin ella; el contenedor sale con código 137. UT1, UT5.

**OSV** · Base de datos abierta de vulnerabilidades por paquete y versión, con API y el escáner `osv-scanner`. Es la fuente que se consulta cuando el hallazgo viene de una dependencia de la aplicación. UT7.

**Password spraying** · Ataque de acceso que prueba una contraseña muy común contra muchos usuarios distintos, para no disparar los bloqueos por intentos fallidos de una sola cuenta. UT5.

**photorec** · Herramienta que recupera ficheros borrados buscando sus firmas en el disco. Sirve para demostrar que un borrado no ha sido suficiente. UT8.

**PITR** · Point-in-time recovery: restaurar una base de datos al instante exacto anterior al fallo, combinando una copia física con el archivado de WAL. UT6.

**PR** · Pull request: propuesta de cambio sobre un repositorio que alguien revisa antes de fusionarla en la rama principal. Es lo que abre Renovate por cada actualización disponible. UT7.

**Presupuesto de error** · Lo que el servicio puede fallar sin incumplir su SLO: con un objetivo del 99,5 % mensual, unas 3,6 horas. UT4.

**Promtail** · Agente que lee ficheros de log (o el socket de Docker), añade etiquetas y los envía a Loki. UT1.

**Recording rule** · Regla de Prometheus que precalcula una expresión y la guarda como métrica nueva. UT2.

**RED (método)** · Revisión de un servicio por tres medidas: peticiones por segundo, errores por segundo y distribución de la latencia. Mira hacia el usuario, donde USE mira hacia los recursos. UT4.

**Regla 3-2-1** · Tres copias de los datos, en dos soportes distintos, con una de ellas fuera del sitio. UT6.

**Renovate** · Bot que abre un merge request cuando hay versión nueva de una imagen o dependencia. UT7.

**restic** · Herramienta de copias de seguridad con deduplicación, cifrado y retención integrada, hacia disco, SFTP o S3. UT6.

**RGPD** · Reglamento General de Protección de Datos. Fija los plazos de conservación, el derecho de supresión y la obligación de poder restaurar los datos, que es lo que condiciona dónde vive una copia y cuándo se borra. UT6, UT8.

**Rollback** · Volver a la versión anterior tras una actualización fallida. Exige tener la etiqueta anterior y una copia previa. UT7.

**RPO y RTO** · Recovery Point Objective: cuántos datos se pueden perder (cada cuánto se copia). Recovery Time Objective: cuánto se puede tardar en restaurar. UT6.

**Runbook** · Instrucciones paso a paso para atender una alarma concreta, enlazadas desde la propia alerta. UT4.

**SBOM** · Software Bill of Materials: inventario de todos los componentes de una imagen, en formato CycloneDX o SPDX. UT7.

**Señales doradas** · Las cuatro medidas mínimas de un servicio: latencia, tráfico, errores y saturación. UT4.

**Silencio** · Supresión temporal de notificaciones de Alertmanager que coinciden con unas etiquetas, típica en mantenimientos. UT2.

**SLI, SLO, SLA** · Indicador de nivel de servicio (la medida), objetivo (el valor comprometido internamente) y acuerdo (el compromiso contractual con consecuencias). UT4.

**SRE** · Site Reliability Engineering: la forma de operar servicios que Google publicó como libro, de donde salen las señales doradas y el principio de alertar por síntomas. UT2, UT4.

**Steal** · Porcentaje de tiempo que una máquina virtual está lista para ejecutar pero el hipervisor no le da CPU. Señal de que el host está sobrevendido. UT5.

**Syft** · Generador de SBOM a partir de imágenes, directorios o ficheros de dependencias. UT7.

**Trivy** · Escáner de vulnerabilidades y configuraciones inseguras para imágenes, sistemas de ficheros, código IaC y repositorios. UT7.

**TSDB** · Base de datos de series temporales. Es el almacén local de Prometheus y el formato de índice de Loki, y se administra con su propia API para borrar series. UT1, UT8.

**Umbral** · Valor a partir del cual una métrica se considera anómala. Se fija desde la documentación del servicio y se ajusta con datos reales. UT2, UT4.

**USE (método)** · Revisión de rendimiento que mira, para cada recurso, su utilización, su saturación y sus errores. UT4, UT5.

**VACUUM FULL** · Operación de PostgreSQL que reescribe una tabla eliminando las páginas muertas que DELETE y DROP dejan en el disco. UT8.

**Ventana de mantenimiento** · Franja horaria acordada de antemano en la que se permite actualizar o intervenir sobre un servicio en producción. UT7.

**Versionado semántico** · Convención `MAYOR.MENOR.PARCHE` con la que un proyecto promete qué puede romperse al actualizar. UT7.

**WAL** · Write-ahead log: diario en el que se anotan las escrituras antes de aplicarlas. PostgreSQL lo usa para poder restaurar a un instante concreto, y el Prometheus de `remote_write` para no perder muestras si el central no responde. UT1, UT6.

**Working set** · Memoria que un contenedor usa de verdad, descontando la caché de página que el kernel puede liberar. Es la que se compara con el límite. UT1.

**ZAP** · OWASP Zed Attack Proxy, escáner de seguridad de aplicaciones web. En modo baseline hace un análisis pasivo rápido. UT4.
