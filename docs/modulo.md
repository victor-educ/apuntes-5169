# Presentación y evaluación

<p class="ut-meta">Módulo 5169 · Mantenimiento del sistema de contenedores desplegado · 110 h · Clases martes y jueves, 1 h 50 min por sesión</p>

## De qué va la asignatura

Desplegar un servicio es un día; mantenerlo son años. Esta asignatura se ocupa de todo lo que pasa después de que un servicio llega a producción: vigilarlo, avisar cuando se degrada, probarlo, protegerlo, actualizarlo, copiarlo y, al final de su vida, retirarlo con limpieza. Es el trabajo diario de quien opera una plataforma de contenedores.

La asignatura hermana, [Despliegue de plataformas de contenedores](https://victor-educ.github.io/apuntes-5166/) (módulo 5166), se cursa el mismo año. Las dos comparten laboratorio y el mismo servicio de referencia: la plataforma se monta allí y se mantiene aquí, y la pila de monitorización de mon01 se monta aquí desde octubre. Qué asignatura explica a fondo cada herramienta que usan las dos está en la tabla de [herramientas compartidas](#herramientas-compartidas-con-despliegue).

## Resultados de aprendizaje

El currículo define cinco resultados de aprendizaje. Las tablas siguientes los recogen en lenguaje llano e indican en qué unidad se trabaja cada criterio; el texto oficial está en el real decreto del curso de especialización.

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
| Prácticas evaluables UT1, UT2 y UT3 | Informe de observabilidad, repositorio de alarmas e informe de seguridad de la monitorización | 40 % de la 1ª evaluación |
| Examen 1ª evaluación (10 dic 2026) | UT1 a UT3, prueba práctica en el laboratorio | 60 % de la 1ª evaluación |
| Prácticas evaluables UT4, UT7 y UT8 | Dossier de operación; informe de vulnerabilidades y actualización; acta de baja, que se entrega antes del examen | 40 % de la 2ª evaluación |
| Examen 2ª evaluación (23 mar 2027) | UT4, UT7 y UT8, prueba práctica en el laboratorio | 60 % de la 2ª evaluación |
| Formación en empresa | UT5 y UT6 con ficha de evidencias firmada por el tutor | Según el plan de FE del centro |

Los exámenes son las sesiones 19 (10 de diciembre de 2026) y 40 (23 de marzo de 2027), integradas en el calendario de cada evaluación: la teoría y los exámenes terminan antes de Pascua. Cada práctica evaluable lleva su tabla de criterios y pesos al final de la unidad. La de la UT8 es la única sin sesión propia: el acta de baja se va construyendo en las hojas A8.1 a A8.5 y se entrega por Aules antes del examen del 23 de marzo. Una entrega fuera de plazo sin causa justificada se corrige sobre el 50 %.

!!! examen "Lo que no se admite"
    Informes sin evidencias (capturas o salidas de comandos con fecha), repositorios con secretos en el historial, datos reales de la empresa sin anonimizar en las fichas de evidencias, y "copias de seguridad" que nunca se han restaurado. En todos los casos la práctica vuelve al alumno sin nota hasta que lo corrija.

## Herramientas de la asignatura

| Capa | Herramienta | Alternativas que se mencionan |
|------|-------------|-------------------------------|
| Métricas | Prometheus, cAdvisor, node_exporter, postgres_exporter | Zabbix, Datadog |
| Logs y eventos | Loki, Promtail, driver json-file | Fluent Bit, Vector, ELK / OpenSearch, Graylog |
| Visualización y alarmas | Grafana, Alertmanager | Grafana Alerting, PagerDuty |
| Incidencias | Issues de Gitea | GitLab, Zammad, GLPI, Jira |
| Pruebas | pytest, k6, OWASP ZAP | newman, JMeter, Locust, Nikto |
| Accesos | fail2ban | CrowdSec |
| Copias | restic, pg_dump, systemd timers, MinIO | borg, Proxmox Backup Server |
| Versiones y vulnerabilidades | Renovate, Syft, Trivy, Grype | Dependabot, Docker Scout, osv-scanner |
| Borrado seguro | restic forget, VACUUM FULL, shred, photorec | nwipe, borrado criptográfico con LUKS |

### Herramientas compartidas con Despliegue

Varias herramientas salen en las dos asignaturas. Cada una se explica a fondo en una sola sesión, la de la tabla; cuando vuelve a aparecer en la otra asignatura, el apartado abre con un repaso que enlaza a esa sesión y dedica sus minutos solo a lo nuevo.

| Herramienta | Se explica a fondo en | Se instala en |
|-------------|-----------------------|---------------|
| nmap y tcpdump | [Despliegue, sesión 10](https://victor-educ.github.io/apuntes-5166/ut/ut2-vpc/#sesion-10-comunicacion-entre-zonas) (6 nov) | El puesto de administración y la VM que indique cada hoja |
| nftables en el propio host | [Sesión 17](ut/ut3-seguridad-monitorizacion.md#sesion-17-red-firewall-y-tls) (1 dic) | Cada host que se cierra |
| Trivy | [Sesión 25](ut/ut4-kpi-pruebas.md#sesion-25-estres-y-seguridad) (21 ene) | El puesto de administración, en la A4.6 |
| gitleaks y pre-commit | [Despliegue, sesión 28](https://victor-educ.github.io/apuntes-5166/ut/ut5-iac/#sesion-28-correccion-de-hallazgos) (27 ene) | El puesto de administración |
| Seguridad de la pila de monitorización | [UT3](ut/ut3-seguridad-monitorizacion.md) (26 nov a 3 dic) | mon01 |
| PromQL, reglas, Alertmanager, Grafana y KPI | [UT1](ut/ut1-observabilidad.md), [UT2](ut/ut2-alarmas.md) y [UT4](ut/ut4-kpi-pruebas.md) | mon01 |

La UT7 de Despliegue trabaja en marzo sobre esta misma pila de mon01 y en el mismo repositorio `monitoring`, con ficheros propios: repasa lo que aquí se da en la UT1, la UT2, la UT3 y la UT4 y evalúa lo que aquí no se toca (métricas del orquestador protegidas, targets generados con Ansible y secretos fuera del repositorio).

## Sobre el material

Este material lo ha escrito Víctor Sellés para la asignatura, con el apoyo de Claude (el asistente de IA de Anthropic) en la redacción, ampliación y revisión, a partir de apuntes propios y de la planificación del curso. La revisión final es del autor, igual que los errores que queden, que se corrigen durante el curso.

## Metodología

Cada sesión de 110 minutos tiene una parte corta de explicación (entre 10 y 30 minutos, según la sesión) y una parte larga de laboratorio sobre el contenedor de referencia. El [calendario](calendario.md) dice, sesión a sesión, qué se explica y qué se practica, y de qué tipo es cada sesión: teoría y práctica, solo práctica, práctica evaluable, examen o proyecto. Cada unidad está ordenada por sesiones: un par de párrafos sobre lo que se busca y, después, sesión a sesión, los apartados de teoría que se explican ese día seguidos de la hoja de práctica, con objetivo, requisitos previos, pasos, comprobación y un último apartado, «Para practicar». Las hojas no se entregan: lo que se entrega es la práctica evaluable que cierra cada unidad. Como el servicio es el mismo durante todo el curso, lo que se configura en una unidad sigue funcionando en la siguiente: las alarmas de la UT2 usan las métricas de la UT1, las pruebas de la UT4 verifican la actualización de la UT7 y la UT8 da de baja el entorno pre que la UT7 actualizó, con todo lo que dejó en la monitorización, las copias y el resto del laboratorio. Por eso conviene no dejar nada a medias.

Todo lo que se hace se documenta en el momento. Al final de cada unidad esa documentación es la práctica evaluable.

Después de Pascua solo queda la sesión del 8 de abril, que se dedica a un proyecto conjunto con el resto de asignaturas del curso y se plantea aparte.
