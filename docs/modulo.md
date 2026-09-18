# Presentación y evaluación

<p class="ut-meta">Módulo 5169 · Mantenimiento del sistema de contenedores desplegado · 110 h · Clases martes y jueves, 2 h por sesión</p>

## De qué va la asignatura

Desplegar un servicio es un día; mantenerlo son años. Esta asignatura se ocupa de todo lo que pasa después de que el pipeline de la asignatura de despliegue ponga el servicio en producción: vigilarlo, avisar cuando se degrada, probarlo, protegerlo, actualizarlo, copiarlo y, al final de su vida, retirarlo con limpieza. Es el trabajo diario de quien opera una plataforma de contenedores.

La asignatura hermana, [Despliegue de plataformas de contenedores](https://victor-educ.github.io/apuntes-5166/) (módulo 5166), se cursa el mismo año. Compartimos laboratorio y el mismo servicio de referencia; lo que se monta allí es lo que se mantiene aquí.

## Resultados de aprendizaje

Cinco resultados de aprendizaje. Los resumo con mis palabras y digo en qué unidad se trabaja cada criterio; el texto oficial está en el real decreto del curso de especialización.

**RA1. Monitoriza el sistema de contenedores: integra los datos, gestiona las alarmas y protege las comunicaciones.**

| CE | Qué se pide | Unidad |
|----|-------------|--------|
| a | Integrar métricas, logs y eventos del contenedor en el sistema de monitorización y verificar comunicación, integridad y almacenamiento | UT1 |
| b | Definir umbrales sobre contadores y cadenas en eventos a partir de la documentación | UT2 |
| c | Agregar y correlar contadores para generar indicadores nuevos | UT2 |
| d | Integrar los eventos en el gestor de alarmas, con activación y recuperación probadas | UT2 |
| e | Categorizar las alarmas por fecha, origen, criticidad y servicio, notificarlas y tratarlas | UT2 |
| f | Auditar las comunicaciones de la monitorización: solo protocolos y puertos necesarios | UT3 |
| g | Configurar y documentar las reglas de protección entre el contenedor y la monitorización | UT3 |

**RA2. Define indicadores del servicio y lo prueba.**

| CE | Qué se pide | Unidad |
|----|-------------|--------|
| a | Documentar las métricas con contador, referencia y categoría (capacidad, rendimiento, calidad) | UT4 |
| b | Implementar indicadores con fórmulas y umbrales por tipo | UT4 |
| c | Elaborar el catálogo de alarmas con origen, fallo, impacto y pasos de resolución | UT4 |
| d | Ejecutar pruebas funcionales, de calidad de servicio, rendimiento, seguridad y estrés | UT4 |
| e | Documentar las pruebas con evidencias, indicadores y registros | UT4 |
| f | Revisar periódicamente los indicadores frente a sus umbrales | UT4 |

**RA3. Explota los registros del sistema: logs, accesos, fallos y rendimiento.**

| CE | Qué se pide | Unidad |
|----|-------------|--------|
| a | Revisar los archivos de registro de forma periódica y reportar fallos | UT5 (empresa) |
| b | Monitorizar los accesos y detectar intentos de fuerza bruta | UT5 (empresa) |
| c | Analizar fallos y reinicios a partir de crashdumps y registros de error | UT5 (empresa) |
| d | Evaluar el rendimiento del equipo frente a una línea base | UT5 (empresa) |

**RA4. Mantiene el sistema: copias de seguridad, actualizaciones y vulnerabilidades.**

| CE | Qué se pide | Unidad |
|----|-------------|--------|
| a | Programar copias de seguridad y verificar su ejecución | UT6 (empresa) |
| b | Exportar las copias a un medio externo cumpliendo las políticas de almacenamiento, rotación y limpieza | UT6 (empresa) |
| c | Restaurar periódicamente en una plataforma de pruebas | UT6 (empresa) |
| d | Mantener el software de base: seguir versiones e instalarlas | UT7 |
| e | Examinar vulnerabilidades en código, binarios y librerías y proponer soluciones | UT7 |
| f | Actualizar desde repositorio con verificación de datos | UT7 |
| g | Analizar y resolver los problemas de actualización en plazo y reportarlos a desarrollo | UT7 |
| h | Comprobar funcionamiento e integridad en entorno previo y producción | UT7 |
| i | Documentar y trazar las actualizaciones en el repositorio de incidencias | UT7 |

**RA5. Termina el contenedor de forma segura.**

| CE | Qué se pide | Unidad |
|----|-------------|--------|
| a | Terminar la aplicación y liberar todos los recursos de infraestructura | UT8 |
| b | Eliminar copias de seguridad y logs externos de forma irrecuperable | UT8 |
| c | Borrar los datos confidenciales de la base de datos interna | UT8 |
| d | Eliminar la conectividad y las referencias en la monitorización y las alarmas | UT8 |

## Cómo se evalúa

| Evaluación | Qué entra | Peso |
|------------|-----------|-----:|
| Prácticas evaluables UT1, UT2, UT3, UT4 | Informes, repositorios y el dossier de operación | 40 % de la 1ª evaluación |
| Examen 1ª evaluación (28 ene 2027) | UT1 a UT4, prueba práctica en el laboratorio | 60 % de la 1ª evaluación |
| Prácticas evaluables UT7, UT8 | Informe de vulnerabilidades y actualización; acta de baja | 40 % de la 2ª evaluación |
| Examen 2ª evaluación (23 mar 2027) | UT7 y UT8, prueba práctica en el laboratorio | 60 % de la 2ª evaluación |
| Formación en empresa | UT5 y UT6 con ficha de evidencias firmada por el tutor | Según el plan de FE del centro |

Los exámenes son las sesiones 28 (28 de enero de 2027) y 41 (23 de marzo de 2027), integradas en el calendario de cada evaluación. Cada práctica evaluable lleva su tabla de criterios y pesos al final de la unidad. Una entrega fuera de plazo sin causa justificada se corrige sobre el 50 %.

!!! warning "Lo que no se admite"
    Informes sin evidencias (capturas o salidas de comandos con fecha), repositorios con secretos en el historial, datos reales de la empresa sin anonimizar en las fichas de evidencias, y "copias de seguridad" que nunca se han restaurado. En todos los casos la práctica vuelve al alumno sin nota hasta que lo corrija.

## Herramientas de la asignatura

| Capa | Herramienta | Alternativas que se mencionan |
|------|-------------|-------------------------------|
| Métricas | Prometheus, cAdvisor, node_exporter, postgres_exporter | Zabbix, Datadog |
| Logs y eventos | Loki, Promtail, driver json-file | Fluent Bit, Vector, ELK / OpenSearch, Graylog |
| Visualización y alarmas | Grafana, Alertmanager | Grafana Alerting, PagerDuty |
| Incidencias | Issues de Gitea / GitLab | Zammad, GLPI, Jira |
| Pruebas | k6, newman / pytest, OWASP ZAP | JMeter, Locust, Nikto |
| Accesos | fail2ban | CrowdSec |
| Copias | restic, pg_dump, systemd timers, MinIO | borg, Proxmox Backup Server |
| Versiones y vulnerabilidades | Renovate, Syft, Trivy, Grype | Dependabot, Docker Scout, osv-scanner |
| Borrado seguro | restic forget, VACUUM FULL, shred, photorec | nwipe, borrado criptográfico con LUKS |

## Sobre el material

Estos apuntes los he escrito yo apoyándome en Claude, el asistente de IA de Anthropic: partí de mis apuntes en Word y de la planificación de sesiones, y usé la herramienta para redactar, ampliar y revisar cada unidad. Lo digo porque no quiero que haya dudas sobre cómo se ha hecho. La revisión final y los errores que queden son míos, y los iré corrigiendo durante el curso.

## Metodología

Cada sesión de dos horas tiene una parte corta de explicación (entre 10 y 30 minutos, según la sesión) y una parte larga de laboratorio sobre el contenedor de referencia. El [calendario](calendario.md) y el plan de sesiones de cada unidad dicen, sesión a sesión, qué se explica y qué se practica, y de qué tipo es cada sesión: teoría y práctica, solo práctica, práctica evaluable o examen. Los apartados de contenido de cada unidad son lo que explico en clase y sirven de consulta; la sección de material de práctica tiene una hoja por sesión con objetivo, requisitos previos, pasos, comprobación y entrega. Como el servicio es el mismo durante todo el curso, lo que se configura en una unidad sigue funcionando en la siguiente: las alarmas de la UT2 usan las métricas de la UT1, las pruebas de la UT4 verifican la actualización de la UT7 y la UT8 desmonta todo lo anterior. Por eso conviene no dejar nada a medias.

Todo lo que se hace se documenta en el momento. Al final de cada unidad esa documentación es la práctica evaluable.
