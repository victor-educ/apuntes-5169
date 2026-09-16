# Glosario

Términos de esta asignatura, en una o dos frases, con la unidad donde se tratan a fondo. Los de infraestructura (hipervisor, VPC, DMZ, IaC, pipeline...) están en el [glosario de la asignatura de despliegue](https://victor-educ.github.io/apuntes-5166/glosario/).

**Acta de baja** · Documento que cierra la retirada de un servicio: quién autorizó, lista de comprobación verificada con evidencias, qué se conservó y qué claves se destruyeron. UT8.

**Alertmanager** · Componente que recibe las alertas de Prometheus o Loki, las agrupa, las enruta según etiquetas, las silencia o inhibe y las envía a correo, chat o webhook. UT2.

**Anonimizar** · Transformar datos personales de forma que no se pueda identificar a la persona, conservando su utilidad estadística. UT8.

**Borrado criptográfico** · Destruir la clave con la que se cifraron unos datos, de modo que quedan irrecuperables aunque los ficheros sigan existiendo. UT8.

**Burn rate** · Velocidad a la que se consume el presupuesto de error. Un burn rate de 2 gasta en medio mes lo previsto para uno. UT4.

**cAdvisor** · Exporter que lee los cgroups y expone métricas de CPU, memoria, red y disco por contenedor. UT1.

**Catálogo de alarmas** · Colección de fichas, una por alarma, con origen, posible fallo, impacto, pasos de análisis, resolución y escalado. UT4.

**cgroups** · Mecanismo del kernel Linux que limita y contabiliza los recursos de un grupo de procesos. Es de donde salen las métricas de recursos de un contenedor. UT1.

**Core dump** · Volcado de la memoria de un proceso en el momento de fallar, que se analiza con un depurador. UT5.

**Counter, gauge, histogram, summary** · Los cuatro tipos de métrica de Prometheus: acumulador que solo sube, valor que sube y baja, distribución en cubos, percentiles calculados en el cliente. UT1.

**CrowdSec** · Alternativa a fail2ban que comparte listas de IP maliciosas entre instalaciones y bloquea mediante bouncers. UT5.

**CVE** · Identificador público de una vulnerabilidad conocida (CVE-2025-12345). UT7.

**CVSS** · Puntuación de 0 a 10 de la gravedad de una vulnerabilidad, con un vector que describe cómo se explota. UT7.

**Dead man's switch** · Alerta que salta cuando deja de llegar una señal periódica; sirve para detectar que una copia de seguridad no se ha ejecutado. UT6.

**Digest** · Hash SHA-256 que identifica de forma inequívoca una imagen de contenedor. Las etiquetas se mueven; el digest no. UT7.

**Driver de logs** · Componente de Docker que decide qué se hace con stdout y stderr de un contenedor: fichero JSON, journald, Loki... UT1.

**Exporter** · Programa que expone métricas de algo (el sistema, PostgreSQL, nginx) en formato Prometheus. UT1.

**fail2ban** · Servicio que lee logs, detecta intentos de acceso fallidos repetidos y bloquea la IP en el firewall durante un tiempo. UT5.

**Grype** · Escáner de vulnerabilidades que trabaja sobre imágenes o sobre un SBOM generado por Syft. UT7.

**Inhibición** · Regla de Alertmanager que suprime unas alertas mientras otra está activa (si el host está caído, no avisar de cada contenedor de ese host). UT2.

**k6** · Herramienta de pruebas de carga con scripts en JavaScript, umbrales y salida a Prometheus o JSON. UT4.

**KEV** · Catálogo de CISA de vulnerabilidades explotadas activamente. Si una CVE está ahí, va la primera. UT7.

**Línea base** · Los valores normales de un sistema en un periodo sin incidentes, contra los que se compara cualquier medición posterior. UT5.

**LogQL** · Lenguaje de consulta de Loki: selectores de stream, filtros y funciones que convierten logs en métricas. UT2.

**Loki** · Sistema de almacenamiento e indexación de logs de Grafana Labs, que indexa etiquetas y no el texto. UT1.

**mTLS** · TLS mutuo: cliente y servidor presentan certificado. Se usa entre Promtail y Loki. UT3.

**Object Lock** · Función de S3 y MinIO que impide borrar o modificar un objeto durante un periodo; protege las copias ante ransomware. UT6.

**OOM** · Out of memory. El kernel mata el proceso que más memoria usa cuando un cgroup o el sistema se queda sin ella; el contenedor sale con código 137. UT1, UT5.

**Presupuesto de error** · Lo que el servicio puede fallar sin incumplir su SLO: con un objetivo del 99,5 % mensual, unas 3,6 horas. UT4.

**Promtail** · Agente que lee ficheros de log (o el socket de Docker), añade etiquetas y los envía a Loki. UT1.

**Recording rule** · Regla de Prometheus que precalcula una expresión y la guarda como métrica nueva. UT2.

**Renovate** · Bot que abre un merge request cuando hay versión nueva de una imagen o dependencia. UT7.

**restic** · Herramienta de copias de seguridad con deduplicación, cifrado y retención integrada, hacia disco, SFTP o S3. UT6.

**Rollback** · Volver a la versión anterior tras una actualización fallida. Exige tener la etiqueta anterior y una copia previa. UT7.

**RPO y RTO** · Recovery Point Objective: cuántos datos se pueden perder (cada cuánto se copia). Recovery Time Objective: cuánto se puede tardar en restaurar. UT6.

**Runbook** · Instrucciones paso a paso para atender una alarma concreta, enlazadas desde la propia alerta. UT4.

**SBOM** · Software Bill of Materials: inventario de todos los componentes de una imagen, en formato CycloneDX o SPDX. UT7.

**Silencio** · Supresión temporal de notificaciones de Alertmanager que coinciden con unas etiquetas, típica en mantenimientos. UT2.

**SLI, SLO, SLA** · Indicador de nivel de servicio (la medida), objetivo (el valor comprometido internamente) y acuerdo (el compromiso contractual con consecuencias). UT4.

**Syft** · Generador de SBOM a partir de imágenes, directorios o ficheros de dependencias. UT7.

**Trivy** · Escáner de vulnerabilidades y configuraciones inseguras para imágenes, sistemas de ficheros, código IaC y repositorios. UT7.

**Umbral** · Valor a partir del cual una métrica se considera anómala. Se fija desde la documentación del servicio y se ajusta con datos reales. UT2, UT4.

**VACUUM FULL** · Operación de PostgreSQL que reescribe una tabla eliminando las páginas muertas que DELETE y DROP dejan en el disco. UT8.

**Working set** · Memoria que un contenedor usa de verdad, descontando la caché de página que el kernel puede liberar. Es la que se compara con el límite. UT1.

**ZAP** · OWASP Zed Attack Proxy, escáner de seguridad de aplicaciones web. En modo baseline hace un análisis pasivo rápido. UT4.
