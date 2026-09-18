# UT8 · Terminación segura del contenedor

<p class="ut-meta">Módulo 5169 · 10 h · Sesiones 36 a 40 · RA5 CE a, b, c, d</p>

Esta es la última unidad en el centro. Durante el curso habéis levantado el servicio, lo habéis instrumentado, protegido, probado, copiado y actualizado; ahora toca lo contrario: retirarlo sin dejar rastro. El sujeto de la baja es el entorno **pre** que creasteis con OpenTofu en la UT7 para ensayar actualizaciones. Lo vamos a dar de baja de verdad: VM, red, DNS, reglas de firewall, certificados, credenciales, copias en MinIO, logs en Loki, datos en la base de datos y toda referencia en la monitorización. Después de la sesión 41 quedan el examen y la recuperación de abril, y la formación en empresa, donde os pedirán exactamente esto cuando un cliente deje de serlo.

## Introducción

Antes de entrar en las sesiones, tres cosas: lo que tenéis que saber hacer al terminar la unidad, los conceptos y herramientas que van a aparecer, y el plan de las cinco sesiones con lo que se explica y lo que se practica en cada una.

### Qué tienes que saber hacer al terminar

- Inventariar todo lo que un servicio ha dejado en la infraestructura y planificar su baja como un cambio con aprobación, ventana y comunicación (CE a).
- Terminar la aplicación y liberar contenedores, volúmenes, imágenes, VM, IP, DNS, reglas, certificados y credenciales, verificando cada punto con un comando (CE a).
- Eliminar copias de seguridad y logs externos de forma que nadie pueda recuperarlos, eligiendo el método según el soporte (CE b).
- Borrar o anonimizar datos confidenciales en una base de datos que sigue en uso, y saber por qué un DELETE no basta (CE c).
- Retirar targets, reglas, rutas, dashboards y conectividad de la monitorización sin disparar alarmas fantasma (CE d).
- Redactar un acta de baja con evidencias que aguante una auditoría.

### Los conceptos de la unidad

El jueves 12 de marzo, dos días después de que hayáis "dado de baja" el entorno pre parando los contenedores, alguien hace push al repositorio del servicio. El job de pre en Jenkins seguía habilitado, así que construye una imagen nueva con etiqueta `-pre`, la sube al registry e intenta desplegarla en una VM que ya no existe. A la vez, Prometheus lleva dos días con cuatro targets en rojo, Alertmanager ha abierto una incidencia por cada uno y el compañero de guardia ha recibido avisos a las once de la noche por un servicio que todo el mundo sabía que se iba a apagar. Y en MinIO sigue habiendo un repositorio de copias con la base de datos completa de pre, con los correos y teléfonos de los usuarios de prueba, que nadie va a rotar ni a vigilar nunca más. Eso es lo que pasa cuando terminar un servicio se confunde con `docker stop`. Lo que queremos conseguir al final de la unidad es sencillo de decir: que del entorno pre no quede nada que no hayamos decidido conservar, y poder demostrarlo con un acta en la que cada línea lleva su prueba.

| Herramienta o concepto | Qué es, en una frase | Para qué la usamos en esta unidad |
|---|---|---|
| ITIL y el cambio normal | Un catálogo de buenas prácticas para gestionar servicios de TI; un "cambio normal" es el que necesita aprobación, evaluación de riesgo y ventana | Para tratar la baja como un cambio con solicitud, aprobación y acta, no como un apagado |
| RGPD, LOPDGDD y NIST SP 800-88 | Las dos normas de protección de datos que os afectan en España y la guía del instituto de estándares de EE. UU. sobre borrado de soportes | Para decidir qué se conserva, hasta cuándo, y qué método de borrado vale para cada soporte |
| OpenTofu | La herramienta que describe la infraestructura en ficheros y la crea o destruye a partir de ellos (la conocéis de 5166) | `tofu destroy` deshace el entorno pre que `tofu apply` creó, y `tofu state rm` protege lo que era compartido |
| Proxmox: `qm`, `pvesh`, vzdump | Los mandos de línea del hipervisor del aula y su sistema de copias de VM | Destruir VM, snapshots y backups vzdump, y liberar IP en el IPAM del SDN |
| Docker Compose y registry Distribution | Compose gestiona el conjunto de contenedores de un proyecto; el registry es el almacén de imágenes de gitea01 | `compose down -v --rmi all` limpia el host; el borrado por digest y el garbage collect limpian el registry |
| OPNsense, dnsmasq y la CA del curso | El firewall del laboratorio, el DNS del SDN y la autoridad que firmó los certificados de `*.pre.lab` | Quitar reglas y aliases, borrar registros DNS y revocar certificados publicando una CRL |
| Jenkins y Gitea | El servidor de integración continua y el gestor de repositorios de 5166 | Borrar credenciales, tokens, deploy keys y webhooks; deshabilitar el job y archivar el repositorio |
| restic y MinIO | El programa de copias cifradas de la UT6 y el almacén S3 del aula donde guarda el repositorio | Borrado criptográfico (destruir la clave del repositorio) y limpieza de versiones en un bucket versionado |
| Loki y `logcli` | El almacén de logs de la UT1 y su cliente de consulta por terminal | Enviar peticiones de borrado al compactor de Loki y comprobar que `{env="pre"}` no devuelve nada |
| Prometheus, Alertmanager (`amtool`) y Grafana | La pila de monitorización que lleváis usando todo el curso; `amtool` es el cliente de terminal de Alertmanager | Silenciar, retirar targets, reglas, rutas y dashboards, borrar series del TSDB y expirar el silencio al final |
| PostgreSQL | La base de datos del servicio | Entender por qué `DELETE` no borra, usar `VACUUM FULL` y anonimizar con una sal de un solo uso |
| photorec y testdisk | Dos herramientas de recuperación de ficheros y particiones borrados | Demostrar sobre un volumen de pruebas qué se recupera tras `rm`, tras `shred` y con cifrado |

**Cómo está organizada la unidad.** La unidad sigue las cinco sesiones en orden y cada sesión trae primero la teoría que se explica y después su hoja de práctica. En la sesión 36 se planifica la baja como un cambio y se inventaría todo lo que pre ha dejado en la infraestructura; de ahí sale la lista de comprobación. En la 37 se ejecuta esa lista sobre la infraestructura: Docker, registry, VM, red, certificados y credenciales. En la 38 se destruyen copias y logs externos entendiendo por qué borrar no borra, y en la 39 se limpian los datos de la base de datos compartida y la monitorización. La sesión 40 cierra con el acta de baja, que es el entregable evaluable; al final quedan los errores frecuentes como material de consulta.

!!! info "Lo que necesitas de la otra asignatura"
    - El entorno pre que vais a destruir nació con OpenTofu en la [UT5 de 5166](https://victor-educ.github.io/apuntes-5166/ut/ut5-iac/) (diciembre y enero). `tofu destroy` es el inverso exacto de aquel `tofu apply`; tened a mano el repositorio `infra` y el estado.
    - Las credenciales del pipeline que retiráis (registry, SSH de app01-pre, token de Gitea) se crearon en la [UT6 de 5166](https://victor-educ.github.io/apuntes-5166/ut/ut6-ci/), que termina el 26 de febrero, justo al principio de esta unidad. Coordinad con el profesor de 5166 qué credencial es de pre y cuál sigue usando dev.
    - Las reglas de firewall, los aliases y la CA que revocáis vienen de la [UT3 de 5166](https://victor-educ.github.io/apuntes-5166/ut/ut3-seguridad-por-capas/) y de la UT3 de esta asignatura.
    - Mientras haces esta unidad, la [UT7 de 5166](https://victor-educ.github.io/apuntes-5166/ut/ut7-monitorizacion/) (10 a 17 de marzo) monta la pila de monitorización "definitiva". Lo que aquí desconfiguráis (targets, reglas, rutas, dashboards) es el ensayo inverso de esa instalación: cada cosa que retiráis es una que allí tendréis que dar de alta.

### Plan de sesiones

Cada sesión de dos horas empieza con una explicación corta y sigue con laboratorio. La columna "Se explica" es lo que cuento yo al principio (con su duración aproximada); la columna "Se practica" es lo que hacéis vosotros con el material de práctica de esta unidad. Las sesiones marcadas solo como práctica no traen teoría nueva.

| Sesión | Fecha | Tipo | Se explica | Se practica |
|---:|-------|------|------------|-------------|
| [36](#sesion-36-plan-de-baja) | 25 feb | Teoría y práctica | La baja como cambio: aprobación, ventana, qué se conserva; dónde deja rastro un servicio (25 min). | Inventariar todo lo que el servicio ha dejado en el entorno y redactar la lista de comprobación de baja con verificación por punto. |
| [37](#sesion-37-liberar-la-infraestructura) | 9 mar | Práctica | Orden correcto de la baja (10 min). | Baja del entorno pre: compose, imágenes, redes, VM con tofu destroy, DNS, reglas, credenciales y proyecto archivado; verificar cada punto. |
| [38](#sesion-38-copias-y-logs) | 11 mar | Teoría y práctica | Por qué borrar no borra: SSD, copy-on-write, versionado; borrado criptográfico (20 min). | Destruir la clave de restic, borrar versiones en S3, logs rotados y streams de Loki; intentar recuperar con photorec. |
| [39](#sesion-39-datos-y-monitorizacion) | 16 mar | Teoría y práctica | DELETE, DROP y VACUUM FULL; anonimización; desconfigurar targets, reglas y dashboards (15 min). | Anonimizar y borrar con VACUUM FULL; retirar targets, reglas, rutas, dashboards y Promtail; comprobar que no quedan series ni alarmas. |
| [40](#sesion-40-practica-evaluable) | 18 mar | Práctica evaluable | Aclaración del enunciado (10 min). | Cerrar el acta de baja con la lista de comprobación completa y una evidencia por punto. |

## Sesión 36 · Plan de baja

<p class="ut-meta" markdown>25 de febrero · Teoría y práctica · <span class="dur" title="Explicación unos 25 min, práctica unos 95 min">:material-school:<i class="dur-barra" style="--teoria:21%"></i>:material-flask:</span></p>

Al acabar esta sesión tendréis el plan de baja del entorno pre: el inventario de todo lo que ha dejado en la infraestructura y la lista de comprobación con un comando de verificación por punto. Para la hoja hacen falta los dos apartados que explico al principio: por qué la baja se trata como un cambio, con lo que la ley obliga a conservar, y el método para inventariar rastros a partir de las tres cadenas que hay que rastrear.

### La baja es un cambio más

Terminar un contenedor "para siempre" no es `docker stop`. Un servicio deja huella en la infraestructura, en la monitorización, en las copias y en los datos, y cada rastro es un coste o un riesgo: datos personales que siguen existiendo después de que el cliente pidiera su supresión, alarmas HostDown que nadie atiende, una IP reservada que impide reutilizar el rango, un token de Jenkins con permisos sobre un repositorio que ya no existe. Por eso la baja se planifica con una lista de comprobación y se documenta como cualquier otro cambio.

En ITIL (el catálogo de buenas prácticas de gestión de servicios de TI), la baja de un servicio es un **cambio normal**: pasa por una solicitud (RFC), una evaluación de riesgo, la aprobación de quien tiene autoridad sobre el servicio (en una empresa pequeña el responsable técnico y el dueño del negocio; en una grande, el CAB, el comité que aprueba los cambios) y una ventana acordada. No es un cambio estándar (los cambios estándar son los repetitivos y de bajo riesgo, y una baja destruye datos, así que nunca lo es) ni una emergencia. En la práctica eso se traduce en cuatro cosas que tienen que existir antes de tocar nada:

1. **Confirmación escrita** del responsable del servicio con fecha. Un correo vale; una conversación en el pasillo no. Ese correo va al acta.
2. **Ventana de ejecución.** Aunque el servicio ya no dé tráfico, la baja toca sistemas compartidos (el firewall, Prometheus, la base de datos común). Se hace en horario en que alguien pueda deshacer un error.
3. **Comunicación.** A los usuarios que quedasen (aviso previo de cierre, normalmente con semanas), a los equipos que consumen la monitorización (para que no abran incidencias) y al equipo de soporte.
4. **Lista de lo que se conserva y hasta cuándo.** Esto es lo que más se olvida y lo que más problemas legales da en las dos direcciones: conservar de más incumple el principio de limitación del plazo de conservación, y borrar de menos incumple una obligación fiscal.

#### Qué hay que conservar por obligación legal

No sois abogados y yo tampoco, pero un técnico tiene que saber qué preguntar. En España los plazos que os vais a encontrar, de forma orientativa, son estos:

| Tipo de dato | Norma | Plazo habitual |
|---|---|---|
| Facturas y documentación contable | Código de Comercio, art. 30 | 6 años |
| Justificantes con trascendencia tributaria | Ley General Tributaria, art. 66 y siguientes | 4 años desde la prescripción |
| Datos personales de clientes tras el fin de la relación | RGPD art. 5.1.e y 17; LOPDGDD | Bloqueo (art. 32 LOPDGDD) mientras puedan derivarse responsabilidades; después supresión |
| Logs de acceso con IP o usuario | RGPD (son datos personales); ENS (Esquema Nacional de Seguridad) para el sector público | Entre 6 meses y 2 años según finalidad y política; el ENS pide al menos lo que dure el análisis de incidentes |
| Registros de actividades de tratamiento | RGPD art. 30 | Mientras exista el tratamiento, y se actualiza con la baja |

El mecanismo del artículo 32 de la LOPDGDD (bloqueo de datos) es el que da sentido a "qué se conserva por si acaso": los datos se dejan de tratar, se aíslan con acceso restringido al responsable y a la autoridad, y solo se destruyen cuando prescriben las responsabilidades. Técnicamente eso es un dump cifrado en un sitio al que solo llega el DPO (el delegado de protección de datos) o el responsable, con fecha de destrucción apuntada en el acta. Lo que no puede ser es "lo dejamos en el bucket de siempre porque nunca se sabe".

El acta tiene que decir, para cada categoría de datos, una de tres cosas: se destruye ahora (y cómo), se bloquea hasta una fecha (y dónde), o se transfiere a otro servicio que hereda el tratamiento.

### Inventario de rastros

Antes de borrar hay que saber qué hay. Un servicio de un año deja rastros en sitios que nadie recuerda, y el que mejor conoce el sistema se fue hace tres meses. Necesitáis un método, no memoria. El mapa siguiente agrupa los rastros en seis familias; fijaos en que la mitad no están en el servicio sino alrededor de él (red, identidad, copias, monitorización), y esas son las que se olvidan.

```mermaid
mindmap
  root((Servicio pre))
    Cómputo
      Contenedores y volúmenes
      Imágenes locales y en registry
      VM en Proxmox y snapshots
      Plantillas y backups vzdump
    Red
      IP e IPAM del SDN
      Reservas dnsmasq y DNS
      Aliases y reglas OPNsense
      NAT y port forwards
      Certificados en la CA
    Identidad
      Usuarios y tokens Proxmox
      Credenciales Jenkins
      Tokens y deploy keys Gitea
      Secretos en .env y vaults
    Datos
      Base de datos y réplicas
      Dumps sueltos
      Cachés y colas
      WAL archivado
    Copias y logs
      Repositorio restic en MinIO
      Versiones S3 y Object Lock
      Streams en Loki
      Ficheros rotados en hosts
    Monitorización
      Targets Prometheus
      Reglas y silencios
      Rutas Alertmanager
      Dashboards y datasources
      Jobs Promtail
      Runbooks y catálogo
```

El método es buscar el nombre del servicio, su entorno y sus IP en todos los sitios donde pueda estar escrito. Con la etiqueta `env=pre`, el dominio `pre.lab` y el rango de IP que le asignasteis, tenéis tres cadenas que rastrear.

**Repositorios de código y configuración.** Todo lo que está en Git se encuentra con un grep sobre los cuatro repositorios de la asignatura más el de infraestructura de la 5166:

```bash
for r in servicio monitoring alerting operacion infra; do
  echo "== $r"; git -C ~/repos/$r grep -n -i -E 'pre\.lab|env="?pre"?|10\.20\.' -- . ':!*.lock'
done
```

Que salga una línea en `alerting/alerts.yml` que no esperabais es lo normal. Apuntadla.

**Prometheus.** Las series que existen ahora mismo con esa etiqueta, y los targets configurados, aunque estén caídos:

```bash
curl -s 'http://10.10.0.20:9090/api/v1/label/job/values?match[]={env="pre"}' | jq .
curl -s 'http://10.10.0.20:9090/api/v1/targets' | jq '.data.activeTargets[] | select(.labels.env=="pre") | .scrapeUrl'
curl -s 'http://10.10.0.20:9090/api/v1/rules' | jq '.data.groups[].rules[] | select(.query|test("pre")) | .name'
```

**Jenkins.** Las credenciales no se ven con un grep porque están cifradas en `credentials.xml`. Se listan por API o por consola de scripts:

```bash
curl -s -u ops:$TOKEN http://jenkins01.dev.lab:8080/credentials/store/system/domain/_/api/json?depth=1 \
  | jq '.credentials[] | {id, typeName, description}'
curl -s -u ops:$TOKEN 'http://jenkins01.dev.lab:8080/api/json?tree=jobs[name,color]' | jq '.jobs[]|select(.name|test("pre"))'
```

**Gitea.** Tokens de acceso, deploy keys, webhooks y paquetes del registry, por API:

```bash
curl -s -H "Authorization: token $GT" http://gitea01.dev.lab:3000/api/v1/repos/ops/servicio/keys | jq '.[]|{id,title}'
curl -s -H "Authorization: token $GT" http://gitea01.dev.lab:3000/api/v1/repos/ops/servicio/hooks | jq '.[]|{id,config}'
curl -s -H "Authorization: token $GT" 'http://gitea01.dev.lab:3000/api/v1/packages/ops?type=container' | jq '.[]|{name,version}'
```

**DNS y direcciones.** Qué resuelve hoy y qué reservas hay en el IPAM (el registro de qué IP está asignada a quién) del SDN de Proxmox y en las leases de dnsmasq:

```bash
dig +short app01.pre.lab @10.10.0.1; dig +short -x 10.20.2.10 @10.10.0.1
pvesh get /cluster/sdn/ipams/pve/status --output-format json | jq '.[]|select(.vnet=="pre")'
grep -h pre /var/lib/misc/dnsmasq.*.leases 2>/dev/null
```

**Firewall.** En OPNsense, aliases y reglas que mencionen el servicio. Con la API (usuario con clave de API creado en la 5166 UT3):

```bash
curl -s -k -u "$KEY:$SECRET" https://10.10.0.1/api/firewall/alias/searchItem | jq '.rows[]|select(.name|test("pre"))|{uuid,name,content}'
```

Las reglas no tienen aún una API de búsqueda completa en todas las versiones, así que la matriz de reglas que mantenéis en `operacion/` es la fuente. Si no coincide con lo que hay en la interfaz, la matriz estaba mal, y eso también se apunta.

**Certificados.** En la CA del curso, el fichero `index.txt` lista todo lo emitido con su estado (V válido, R revocado, E expirado):

```bash
grep -E 'pre\.lab' /etc/ca/index.txt
```

**Proxmox.** VM, snapshots, backups y plantillas del entorno, y tokens de API creados para OpenTofu:

```bash
qm list | grep pre
for id in $(qm list | awk '/pre/{print $1}'); do qm listsnapshot $id; done
pvesm list local --vmid 210
pveum user token list tofu@pve
```

El resultado de este inventario es una tabla: rastro, dónde, cómo se elimina, cómo se verifica, quién lo hace. Esa tabla es la lista de comprobación de la baja, y es la primera actividad de la unidad.

### A8.1 Plan de baja (sesión 36)

**Objetivo.** `operacion/baja/pre/plan.md` con el inventario de rastros de pre y la lista de comprobación de baja, con un comando de verificación por punto.

**Antes de empezar.**

- SSH a mon01, app01-pre, gitea01, al nodo Proxmox y a la CA; `$TOKEN` (Jenkins), `$GT` (Gitea), `$KEY` y `$SECRET` (OPNsense) exportados en la shell.
- Los cinco repositorios (`servicio`, `monitoring`, `alerting`, `operacion`, `infra`) clonados en `~/repos/` y actualizados.
- Se ha explicado [la baja como cambio](#la-baja-es-un-cambio-mas) y [el inventario de rastros](#inventario-de-rastros); la [lista de comprobación completa](#lista-de-comprobacion-completa) es tu plantilla.

**Pasos.**

1. Crea `operacion/baja/pre/ev/` y `operacion/baja/pre/plan.md` con cuatro secciones: "Inventario de rastros" (tabla rastro, dónde, cómo se elimina, cómo se verifica, quién), "Lista de comprobación", "Qué se conserva" (tabla elemento, motivo, dónde, hasta, responsable) y "Hallazgos no esperados".

2. Rastrea las tres cadenas (`pre.lab`, `env=pre`, el rango `10.20.`) en los cinco repositorios y guarda la salida como primera evidencia:

    ```bash
    for r in servicio monitoring alerting operacion infra; do
      echo "== $r"; git -C ~/repos/$r grep -n -i -E 'pre\.lab|env="?pre"?|10\.20\.' -- . ':!*.lock'
    done | tee ~/repos/operacion/baja/pre/ev/00-grep-repos.txt
    ```

3. Consulta Prometheus: valores de `job` con `env="pre"`, targets activos y reglas que mencionen pre.

    ```bash
    P=http://10.10.0.20:9090
    curl -s "$P/api/v1/label/job/values?match[]={env=\"pre\"}" | jq .
    curl -s "$P/api/v1/targets" | jq '.data.activeTargets[] | select(.labels.env=="pre") | .scrapeUrl'
    curl -s "$P/api/v1/rules" | jq '.data.groups[].rules[] | select(.query|test("pre")) | .name'
    ```

4. Lista credenciales y jobs en Jenkins, y tokens, deploy keys, webhooks y paquetes en Gitea:

    ```bash
    J=http://jenkins01.dev.lab:8080
    curl -s -u ops:$TOKEN "$J/credentials/store/system/domain/_/api/json?depth=1" | jq '.credentials[] | {id, typeName, description}'
    curl -s -u ops:$TOKEN "$J/api/json?tree=jobs[name,color]" | jq '.jobs[]|select(.name|test("pre"))'
    G=http://gitea01.dev.lab:3000/api/v1
    curl -s -H "Authorization: token $GT" $G/repos/ops/servicio/keys | jq '.[]|{id,title}'
    curl -s -H "Authorization: token $GT" $G/repos/ops/servicio/hooks | jq '.[]|{id,config}'
    curl -s -H "Authorization: token $GT" "$G/packages/ops?type=container" | jq '.[]|{name,version}'
    ```

5. En el nodo Proxmox y en el resolver: VM, snapshots, backups vzdump, tokens de API, IPAM, leases y DNS.

    ```bash
    qm list | grep pre
    for id in $(qm list | awk '/pre/{print $1}'); do qm listsnapshot $id; done
    pvesm list local --vmid 210
    pveum user token list tofu@pve
    pvesh get /cluster/sdn/ipams/pve/status --output-format json | jq '.[]|select(.vnet=="pre")'
    grep -h pre /var/lib/misc/dnsmasq.*.leases 2>/dev/null
    dig +short app01.pre.lab @10.10.0.1; dig +short -x 10.20.2.10 @10.10.0.1
    ```

6. Aliases de OPNsense por API (las reglas, comparando la matriz de `operacion/` con la interfaz) y el `index.txt` de la CA:

    ```bash
    curl -s -k -u "$KEY:$SECRET" https://10.10.0.1/api/firewall/alias/searchItem | jq '.rows[]|select(.name|test("pre"))|{uuid,name,content}'
    grep -E 'pre\.lab' /etc/ca/index.txt
    ```

7. Rellena la tabla de inventario con una fila por rastro y las seis familias cubiertas. Lo que haya aparecido y no esperabas (una regla de `alerts.yml`, un alias en uso, un token olvidado) va a "Hallazgos no esperados".
8. Redacta la lista de comprobación en orden de ejecución, partiendo de la [lista completa](#lista-de-comprobacion-completa) y ajustándola a lo encontrado, con una línea por bloque que justifique su posición.
9. Rellena "Qué se conserva" (dump bloqueado, dashboards, repositorio archivado, obligaciones legales de la tabla de conservación) con motivo, dónde y hasta cuándo.
10. Haz commit del plan y de la evidencia del grep.

**Comprobación.** Cuatro secciones rellenas; ninguna fila sin "Cómo se verifica"; la lista empieza por aprobación y silencio y termina por expiración del silencio y acta; al menos un hallazgo no esperado (si no, repasa `alerting` y los aliases).

**Entrega.** `operacion/baja/pre/plan.md` y `operacion/baja/pre/ev/00-grep-repos.txt` en un commit del repositorio `operacion`.

**Si te sobra tiempo.** Redacta el correo de solicitud de baja (RFC) con fecha, ventana y lo que se conserva; guárdalo como `ev/01-aprobacion.txt`.

## Sesión 37 · Liberar la infraestructura

<p class="ut-meta" markdown>9 de marzo · Práctica · <span class="dur" title="Explicación unos 10 min, práctica unos 110 min">:material-school:<i class="dur-barra" style="--teoria:8%"></i>:material-flask:</span></p>

Sesión de práctica casi entera: al terminarla, del entorno pre no quedan contenedores, imágenes, VM, IP, DNS, reglas, certificados válidos ni credenciales, y cada verificación está guardada como evidencia. Al principio explico el orden de la baja, que es lo único que hay que tener claro antes de tocar nada. El detalle de cada capa (Docker, registry, OpenTofu, Proxmox, red, CA y credenciales) está en el apartado de consulta que sigue, para leerlo paso a paso mientras hacéis la hoja.

### El orden importa

Hay una secuencia correcta y no es "borrar de arriba abajo". Si paráis el servicio antes de silenciar las alarmas, Alertmanager abre incidencias, el receptor webhook crea tickets y el equipo de guardia recibe un aviso a las diez de la noche por un servicio que sabíais que ibais a apagar. Si borráis los volúmenes antes de haber verificado que la copia bloqueada existe y se puede restaurar, ya no hay vuelta atrás. Si destruís la VM antes de exportar las evidencias que están dentro (logs de auditoría, por ejemplo), el acta se queda sin pruebas.

```mermaid
flowchart TD
    A[Aprobación escrita<br>y lista de comprobación] --> B[Silenciar alarmas<br>y avisar]
    B --> C[Extraer lo que se conserva:<br>dump bloqueado, evidencias, dashboards a Git]
    C --> D[Parar el servicio<br>compose down]
    D --> E[Periodo de gracia<br>1 a 7 días con todo parado]
    E --> F[Liberar infraestructura:<br>volúmenes, imágenes, VM, red, DNS, firewall]
    F --> G[Revocar certificados<br>y credenciales]
    G --> H[Destruir copias y logs externos]
    H --> I[Borrar o anonimizar<br>datos en la BD compartida]
    I --> J[Desconfigurar monitorización<br>y quitar silencios]
    J --> K[Verificar todo<br>y firmar el acta]
```

El periodo de gracia del paso E es opcional pero muy recomendable: el servicio está parado, nada se ha destruido, y si alguien grita ("el informe mensual tiraba de esa API") se levanta en un minuto. En una empresa una semana es lo habitual; en el laboratorio lo simulamos entre la sesión 38 y la 39.

Los silencios se quitan al final, no antes, y con criterio: si quitáis el silencio con los targets todavía configurados, salta todo. Primero se retiran los targets y las reglas, se comprueba que no hay alertas pendientes, y entonces se expira el silencio.

#### Lista de comprobación completa

Esta es la lista con la que trabajaremos. Cada línea lleva el comando de verificación, porque una casilla marcada sin evidencia no vale nada en un acta.

- [ ] Aprobación escrita archivada (correo con fecha y nombre)
- [ ] Silencio en Alertmanager para `env="pre"` con duración mayor que la ventana: `amtool silence query env=pre`
- [ ] Dump bloqueado cifrado y guardado donde indica el acta; hash SHA-256 apuntado: `sha256sum pre-bloqueo.sql.gpg`
- [ ] Dashboards exportados a Git: `git -C operacion log -1 -- dashboards/pre/`
- [ ] Servicio parado: `docker compose -p servicio-pre ps -a` vacío
- [ ] Volúmenes eliminados: `docker volume ls -q --filter label=com.docker.compose.project=servicio-pre` vacío
- [ ] Imágenes locales eliminadas: `docker images --filter reference='registry.dev.lab/*:pre*'` vacío
- [ ] Imágenes borradas del registry y garbage collect ejecutado: `curl -s registry.dev.lab:5000/v2/ops/api/tags/list`
- [ ] Redes eliminadas: `docker network ls --filter label=com.docker.compose.project=servicio-pre` vacío
- [ ] VM destruidas: `qm list | grep pre` vacío; `tofu state list` vacío
- [ ] Snapshots y backups vzdump del entorno eliminados: `pvesm list local --vmid 210` vacío
- [ ] IP liberadas en IPAM y reservas dnsmasq: `pvesh get /cluster/sdn/ipams/pve/status` sin la vnet
- [ ] DNS no resuelve: `dig +short app01.pre.lab @10.10.0.1` vacío
- [ ] Aliases y reglas de firewall eliminados; matriz actualizada; `nmap -Pn 10.20.1.10 -p 80,443,8080` sin respuesta
- [ ] Certificados revocados y CRL publicada: `openssl crl -in crl.pem -noout -text | grep -A2 'Serial Number: 1A'`
- [ ] Tokens y usuarios de Proxmox, Jenkins y Gitea eliminados: listados vacíos por API
- [ ] Proyecto en Jenkins deshabilitado y repositorio en Gitea archivado
- [ ] Clave del repositorio restic destruida: `restic -r s3:... snapshots` falla con "wrong password or no key found"
- [ ] Versiones y marcadores de borrado en S3 eliminados: `mc ls --versions --recursive s3/backups/pre/` vacío
- [ ] Streams de Loki borrados: `logcli query '{env="pre"}' --since=8760h` sin resultados
- [ ] Ficheros de log rotados eliminados en web01, app01, db01 y en mon01
- [ ] Tablas del servicio eliminadas y `VACUUM FULL` ejecutado; usuarios anonimizados: consulta de comprobación
- [ ] Réplicas, dumps sueltos, caché y colas revisados
- [ ] Targets, reglas, rutas, receptores, datasources y jobs Promtail retirados; `count({env="pre"})` sin datos tras la retención o tras `delete_series`
- [ ] Runbooks y catálogo de alarmas marcados como retirados con fecha
- [ ] Silencio expirado y sin alertas: `amtool alert query env=pre` vacío
- [ ] Acta firmada y guardada con las incidencias del servicio

### Liberar los recursos de la infraestructura

*Material de consulta: no se explica en clase; lo necesitas para la hoja de práctica de esta sesión.*

Aquí empieza la parte de manos en el teclado: vamos a devolver a la infraestructura todo lo que el entorno pre tenía asignado, capa por capa, desde los contenedores hasta los certificados y las credenciales. Cada recurso tiene un comando para liberarlo y otro para comprobar que ya no está, y ese segundo es el que va al acta. La tabla resume el qué; el cómo y el por qué van debajo, por subapartados.

| Recurso | Cómo se libera | Cómo se verifica |
|---|---|---|
| Contenedores | `docker compose down` | `docker ps -a` vacío del proyecto |
| Volúmenes | `docker compose down -v`; `docker volume rm` | `docker volume ls`; espacio en `df -h` |
| Imágenes | `docker image rm`; `docker image prune -a` (con cuidado); borrar del registry | `docker images`; API del registry |
| Redes | `docker network rm` | `docker network ls` |
| Máquinas virtuales | `tofu destroy` del entorno ([5166 UT5](https://victor-educ.github.io/apuntes-5166/ut/ut5-iac/)) o `qm destroy` | Proxmox sin la VM; estado de OpenTofu vacío |
| IP, DNS, DHCP | Quitar reserva en dnsmasq / IPAM del SDN; borrar registros | `dig` no resuelve; reserva libre |
| Reglas de firewall y NAT | Eliminar reglas y aliases del servicio | Matriz de reglas actualizada; escaneo |
| Certificados | Revocar en la CA | CRL y respuesta OCSP |
| Credenciales | Borrar usuarios de servicio, tokens, credenciales en Jenkins, secretos | Inventario de credenciales |
| Pipeline y repositorio | Archivar el proyecto en Jenkins/Gitea (no borrar el código) | Lista de proyectos |

#### Docker en app01 de pre

`docker compose down` para y elimina contenedores y redes del proyecto, pero deja los volúmenes con nombre y las imágenes. Con `-v` elimina también los volúmenes declarados en el fichero y los anónimos; con `--rmi all` borra las imágenes que usaba el proyecto. Es la forma limpia de terminar un despliegue de Compose porque usa las etiquetas `com.docker.compose.project` para saber qué es suyo y no toca nada de otros proyectos del mismo host:

```bash
cd /opt/servicio && docker compose -p servicio-pre down -v --rmi all --remove-orphans
docker volume ls -q --filter label=com.docker.compose.project=servicio-pre   # vacío
docker network ls --filter label=com.docker.compose.project=servicio-pre     # vacío
docker ps -a --filter label=com.docker.compose.project=servicio-pre          # vacío
```

`--remove-orphans` limpia contenedores del proyecto que ya no aparecen en el fichero (por ejemplo, un sidecar que quitasteis en la UT7 y que seguía parado). Lo que `down -v` no ve son los volúmenes creados a mano con `docker volume create` y montados como `external: true`. Esos hay que borrarlos con `docker volume rm` y, antes, mirar quién más los usa: `docker ps -a --filter volume=pgdata-pre`.

`docker image prune -a` borra todas las imágenes sin contenedor asociado del host entero, no solo las del proyecto. En app01 de pre, donde solo vive este servicio, es aceptable. En un host compartido no lo ejecutéis: usad `docker image rm` con la referencia concreta. `docker system prune --volumes` es todavía más agresivo (contenedores parados, redes sin usar, imágenes colgantes, caché de build y volúmenes anónimos) y es lo que dejaréis ejecutado justo antes de destruir la VM, porque a esas alturas ya da igual.

El volumen borrado con `docker volume rm` es un `rm -rf` de `/var/lib/docker/volumes/<nombre>/_data`. Los bloques siguen en el disco de la VM. Lo veremos en el apartado de copias: la baja del volumen se completa cuando se destruye el disco de la VM, y de verdad cuando el disco estaba cifrado o cuando se sobrescribe.

#### Borrar del registry

Borrar una imagen del host no la quita del registry en gitea01, y ese es el rastro que más se olvida: la imagen `servicio:1.4.2-pre` con el `.env` de pruebas dentro de una capa sigue ahí para quien la pida. La API del registry de Distribution (la imagen `registry:2` que usáis) no borra por etiqueta, borra por **digest** del manifiesto (la huella SHA-256 que identifica su contenido), y solo si el registry arrancó con borrado habilitado (`REGISTRY_STORAGE_DELETE_ENABLED=true`).

```bash
R=http://registry.dev.lab:5000
# 1. Obtener el digest del manifiesto (hay que pedir el media type correcto)
D=$(curl -sI -H 'Accept: application/vnd.oci.image.manifest.v1+json, application/vnd.docker.distribution.manifest.v2+json' \
     $R/v2/ops/api/manifests/1.4.2-pre | awk -F': ' '/[Dd]ocker-Content-Digest/{print $2}' | tr -d '\r')
echo $D    # sha256:9f2c...
# 2. Borrar el manifiesto por digest (borra todas las etiquetas que apunten a él)
curl -s -o /dev/null -w '%{http_code}\n' -X DELETE $R/v2/ops/api/manifests/$D   # 202
# 3. Comprobar
curl -s $R/v2/ops/api/tags/list
```

El paso 2 desvincula el manifiesto, pero las capas (blobs) siguen ocupando disco hasta que se ejecuta el recolector de basura, que necesita el registry parado o en modo solo lectura:

```bash
docker compose -f /opt/registry/compose.yml stop registry
docker compose -f /opt/registry/compose.yml run --rm registry garbage-collect --delete-untagged /etc/docker/registry/config.yml
docker compose -f /opt/registry/compose.yml start registry
```

Si en vez de `registry:2` usáis el registro de paquetes integrado en Gitea, la operación es `DELETE /api/v1/packages/ops/container/api/1.4.2-pre` y Gitea limpia los blobs en su tarea periódica de mantenimiento. Y si la imagen se subió a un registro público (Docker Hub, ghcr.io) para alguna prueba, hay que borrarla allí también; los registries públicos guardan el manifiesto en cachés intermedias durante horas.

#### OpenTofu y el estado

El entorno pre nació con `tofu apply` sobre el código de la 5166 UT5, así que muere con `tofu destroy`. Antes de ejecutarlo, mirad el plan: `tofu plan -destroy` os enseña todo lo que va a desaparecer, y ahí es donde descubrís que el módulo de red compartía la vnet con dev o que el estado incluye el usuario de API de Proxmox que otros entornos usan.

```bash
cd infra/envs/pre
tofu plan -destroy -out=destroy.plan
# revisar: solo recursos de pre; si hay algo compartido, sacarlo del estado sin destruirlo
tofu state rm module.red.proxmox_virtual_environment_network_linux_bridge.vmbr_shared
tofu apply destroy.plan
tofu state list          # vacío
```

`tofu state rm` quita un recurso del estado sin tocarlo en Proxmox; es la herramienta para "esto no es de pre, no lo destruyas". Cuando el estado queda vacío, el fichero `terraform.tfstate` (o el workspace en el backend remoto) todavía contiene el historial y, en el caso del provider de Proxmox, puede contener datos sensibles como la contraseña de cloud-init en claro. El estado se borra también: `tofu workspace delete pre` si usáis workspaces, o el fichero y sus copias `.backup` si es local. El estado va al mismo tratamiento que un dump de base de datos.

#### Proxmox: qm destroy y lo que arrastra

Si OpenTofu no puede (el estado se perdió, la VM se creó a mano), se hace con `qm`:

```bash
qm stop 210 --timeout 60
qm listsnapshot 210
qm destroy 210 --purge --destroy-unreferenced-disks 1
```

`--purge` quita la VM de los trabajos de backup, de la replicación y de la configuración de HA; sin él, el job de vzdump (las copias de VM de Proxmox) del domingo fallará con "VM 210 not found" cada semana hasta que alguien lo edite. `--destroy-unreferenced-disks` borra también los discos que quedaron huérfanos en el storage (por ejemplo, un `vm-210-disk-1` que desvinculasteis para probar algo). Los snapshots se destruyen con la VM. Lo que **no** se destruye son los backups vzdump ya hechos ni las plantillas clonadas de esa VM:

```bash
pvesm list local --vmid 210       # backups vzdump
pvesm free local:backup/vzdump-qemu-210-2027_02_28-02_00_01.vma.zst
qm list | awk '$1>=9000'          # plantillas; comprobar si alguna nació de pre
```

En Proxmox Backup Server, si lo usáis, los snapshots del grupo `vm/210` se eliminan con `proxmox-backup-client forget` o desde la interfaz, y el espacio se libera en el siguiente garbage collect del datastore, que respeta un periodo de 24 horas y 5 minutos sobre los chunks. Hasta entonces, los datos están ahí.

#### Direcciones, DNS y firewall

Con el SDN de Proxmox y el IPAM integrado (`pve`), las IP se asignan por vnet y las reservas se guardan en el IPAM y en la configuración de dnsmasq. Al destruir la vnet de pre con OpenTofu se liberan las dos cosas; si la vnet sigue porque es compartida, se liberan las IP una a una:

```bash
pvesh delete /cluster/sdn/vnets/pre/ips --zone lab --vnet pre --ip 10.20.2.10
pvesh set /cluster/sdn                       # aplicar
pvesh get /cluster/sdn/ipams/pve/status --output-format json | jq '.[]|select(.ip|startswith("10.20."))'
```

Los registros DNS del laboratorio los sirve Unbound (el resolutor DNS integrado en OPNsense) mediante host overrides o el propio dnsmasq del SDN según cómo lo montarais en la 5166 UT2. En cualquiera de los dos, la verificación es la misma: `dig` con `+short` contra el resolver del laboratorio no devuelve nada, ni en directa ni en inversa. Un DNS que sigue resolviendo un nombre a una IP libre es un problema de seguridad futuro: cuando esa IP se reasigne, `app01.pre.lab` apuntará a una máquina que no es.

En OPNsense, el orden es reglas primero y aliases después, porque un alias en uso no se puede borrar. Las reglas que permitían a mon01 llegar a los exporters de pre (las de la UT3 de esta asignatura) y las de NAT hacia web01 de pre desaparecen; los aliases `pre_front`, `pre_back`, `pre_data` también. Luego `Apply changes` o, por API, `firewall/filter/apply` y `firewall/alias/reconfigure`. La verificación no es mirar la interfaz: es un escaneo desde fuera y desde mon01 que ya no llega a nada, y la matriz de reglas de `operacion/` actualizada en un commit.

#### Certificados: revocar en la CA propia

Los certificados de `*.pre.lab` que emitió la CA del curso siguen siendo válidos hasta su fecha de caducidad aunque la máquina haya desaparecido. Si alguien conserva la clave privada (estaba en un volumen, en un backup, en el estado de OpenTofu), puede suplantar el servicio. Revocar es lo único que lo impide, y solo funciona si los clientes consultan la CRL (la lista de certificados revocados que publica la CA) o un respondedor OCSP (el servicio que contesta en línea si un certificado concreto está revocado).

```bash
cd /etc/ca
openssl ca -config ca.cnf -revoke certs/app01.pre.lab.pem -crl_reason cessationOfOperation
openssl ca -config ca.cnf -gencrl -out crl/ca.crl.pem
openssl crl -in crl/ca.crl.pem -noout -text | grep -B1 -A3 'cessationOfOperation'
cp crl/ca.crl.pem /var/www/ca/ca.crl      # donde apunta el crlDistributionPoints de los certificados
```

El motivo `cessationOfOperation` es el correcto para una baja (no `keyCompromise`, salvo que sepáis que la clave se filtró). Para OCSP con OpenSSL basta un respondedor mínimo sobre el mismo `index.txt`:

```bash
openssl ocsp -index index.txt -CA ca.pem -rsigner ocsp.pem -rkey ocsp.key -port 2560 -text &
openssl ocsp -issuer ca.pem -cert certs/app01.pre.lab.pem -url http://ca.dev.lab:2560 -resp_text | grep 'Cert Status'
# Cert Status: revoked
```

Si vuestra CA es step-ca, es `step ca revoke --cert app01.pre.lab.pem --key app01.pre.lab.key --reason cessationOfOperation` y el propio step-ca responde OCSP. El certificado de la CA no se toca; lo que se revoca son las hojas. Y la clave privada del servicio se destruye con el resto del entorno, que es lo que hace innecesaria la revocación en la práctica pero no en la teoría: la revocación cubre las copias de la clave que no sabéis que existen.

#### Credenciales y tokens

Cada sistema tiene las suyas y todas hay que tocarlas. Un token que sobrevive a su servicio es una credencial huérfana: nadie lo rota, nadie lo audita y sigue siendo válido.

```bash
# Proxmox: token de API que usaba OpenTofu para pre, y el usuario si era exclusivo
pveum user token remove tofu@pve pre
pveum user delete tofu-pre@pve
pveum user token list tofu@pve

# Gitea: token de acceso del pipeline, deploy key y webhook del repositorio
curl -X DELETE -H "Authorization: token $GT" http://gitea01.dev.lab:3000/api/v1/users/jenkins/tokens/pre-deploy
curl -X DELETE -H "Authorization: token $GT" http://gitea01.dev.lab:3000/api/v1/repos/ops/servicio/keys/7
curl -X DELETE -H "Authorization: token $GT" http://gitea01.dev.lab:3000/api/v1/repos/ops/servicio/hooks/3

# Jenkins: credencial del entorno pre (registry, SSH de app01-pre, token de Gitea)
curl -X POST -u ops:$TOKEN http://jenkins01.dev.lab:8080/credentials/store/system/domain/_/credential/pre-ssh-app01/doDelete
```

Además de lo que está en los sistemas, están los secretos que viajaron: el `.env` de pre que alguien tiene en su portátil, la contraseña de cloud-init en el estado de OpenTofu, el `pass` de restic en `/etc/restic/` de app01-pre. La destrucción de la VM se lleva los de dentro; los de fuera se piden por escrito y se anotan como "solicitada destrucción a X, confirmada el día Y".

#### Archivar, no borrar el código

El repositorio del servicio y el Jenkinsfile no se borran: son el registro de cómo se hizo y pueden hacer falta para una auditoría o para resucitar el servicio. En Gitea, el repositorio se archiva (Settings, Danger Zone, Archive, o `PATCH /api/v1/repos/ops/servicio` con `{"archived": true}`): queda en solo lectura, no acepta pushes ni issues, y se ve en la lista con la marca de archivado. En Jenkins no hay "archivar": el job de pre se deshabilita (`Disable Project`, o `POST /job/servicio-pre/disable`) y se mueve a una carpeta `archivo/`; el Jenkinsfile sigue en el repositorio. Borrar el job elimina también el historial de builds y sus artefactos, que a veces son la única evidencia de qué versión se desplegó cuándo.

### A8.2 Liberar la infraestructura (sesión 37)

**Objetivo.** El entorno pre sin contenedores, imágenes, VM, IP, DNS, reglas, certificados válidos ni credenciales, con la salida de cada verificación en `operacion/baja/pre/ev/`.

**Antes de empezar.**

- El plan de A8.1 aprobado (nota del profesor con fecha en `ev/01-aprobacion.txt`).
- `infra/envs/pre` con su estado, `tofu` instalado y el token de Proxmox del provider exportado.
- Acceso a app01-pre, gitea01, nodo Proxmox, CA y OPNsense; `$TOKEN`, `$GT`, `$KEY`, `$SECRET` y `$GRAFANA_TOKEN` en la shell.
- Se ha explicado [el orden de la baja](#el-orden-importa); el detalle de cada capa está en [Liberar los recursos de la infraestructura](#liberar-los-recursos-de-la-infraestructura).

**Pasos.**

1. Silencia antes de tocar nada; el silencio dura más que la ventana (hasta la sesión 39):

    ```bash
    A=http://10.10.0.20:9093
    amtool --alertmanager.url=$A silence add env=pre --duration=10d --author="$USER" --comment="Baja servicio-pre, RFC en operacion/baja/pre"
    amtool --alertmanager.url=$A silence query env=pre | tee ~/repos/operacion/baja/pre/ev/02-silencio.txt
    ```

2. Extrae lo que se conserva: dump completo cifrado (bloqueo) y dashboards a Git.

    ```bash
    ssh db01-pre 'pg_dump -U app servicio' | gpg -e -r dpo@lab -o pre-bloqueo.sql.gpg
    sha256sum pre-bloqueo.sql.gpg | tee ~/repos/operacion/baja/pre/ev/03-dump-bloqueo.txt
    G=http://10.10.0.20:3000; H="Authorization: Bearer $GRAFANA_TOKEN"
    mkdir -p ~/repos/operacion/dashboards/pre
    for uid in $(curl -s -H "$H" "$G/api/search?tag=pre" | jq -r '.[].uid'); do
      curl -s -H "$H" $G/api/dashboards/uid/$uid | jq '.dashboard' > ~/repos/operacion/dashboards/pre/$uid.json
    done
    git -C ~/repos/operacion add dashboards/pre && git -C ~/repos/operacion commit -m "UT8: dashboards de pre archivados antes de la baja"
    ```

    El `.gpg` se entrega al profesor (hace de DPO) y no se sube al repositorio.

3. Deshabilita el job de pre en Jenkins antes de tocar el registry, para que un push no reconstruya imágenes:

    ```bash
    curl -s -o /dev/null -w '%{http_code}\n' -X POST -u ops:$TOKEN http://jenkins01.dev.lab:8080/job/servicio-pre/disable
    ```

4. Para el servicio y limpia el host en app01-pre:

    ```bash
    cd /opt/servicio && docker compose -p servicio-pre down -v --rmi all --remove-orphans
    docker volume rm pgdata-pre                      # los volúmenes external, a mano
    { docker ps -a --filter label=com.docker.compose.project=servicio-pre
      docker volume ls -q --filter label=com.docker.compose.project=servicio-pre
      docker network ls --filter label=com.docker.compose.project=servicio-pre
      docker images --filter reference='registry.dev.lab/*:pre*'; } | tee ev/04-docker.txt
    docker system prune -af --volumes
    ```

5. Borra las imágenes `-pre` del registry por digest y ejecuta el garbage collect en gitea01:

    ```bash
    R=http://registry.dev.lab:5000
    for t in $(curl -s $R/v2/ops/api/tags/list | jq -r '.tags[]|select(test("pre"))'); do
      D=$(curl -sI -H 'Accept: application/vnd.oci.image.manifest.v1+json, application/vnd.docker.distribution.manifest.v2+json' \
           $R/v2/ops/api/manifests/$t | awk -F': ' '/[Dd]ocker-Content-Digest/{print $2}' | tr -d '\r')
      echo "$t $D"; curl -s -o /dev/null -w '%{http_code}\n' -X DELETE $R/v2/ops/api/manifests/$D   # 202
    done
    docker compose -f /opt/registry/compose.yml stop registry
    docker compose -f /opt/registry/compose.yml run --rm registry garbage-collect --delete-untagged /etc/docker/registry/config.yml
    docker compose -f /opt/registry/compose.yml start registry
    curl -s $R/v2/ops/api/tags/list | tee ev/05-registry.txt
    ```

6. Destruye las VM con OpenTofu, revisando el plan y sacando del estado lo compartido:

    ```bash
    cd ~/repos/infra/envs/pre
    tofu plan -destroy -out=destroy.plan | tee ~/repos/operacion/baja/pre/ev/06-tofu-plan.txt
    # si el plan lista algo que no es de pre (bridge compartido, usuario de API):
    tofu state rm module.red.proxmox_virtual_environment_network_linux_bridge.vmbr_shared
    tofu apply destroy.plan
    tofu state list | tee ~/repos/operacion/baja/pre/ev/07-tofu-state.txt     # vacío
    ```

    Si el estado no sirve: `qm stop 210 --timeout 60 && qm destroy 210 --purge --destroy-unreferenced-disks 1`, y lo mismo para 211 a 213.

7. Backups vzdump y plantillas nacidas de pre:

    ```bash
    pvesm list local --vmid 210
    pvesm free local:backup/vzdump-qemu-210-2027_02_28-02_00_01.vma.zst
    qm list | awk '$1>=9000'
    { qm list | grep pre; pvesm list local --vmid 210; } | tee ev/08-proxmox.txt
    ```

8. IP y DNS. Si la vnet se destruyó con OpenTofu, solo verifica; si era compartida, libera las IP una a una y borra los host overrides de Unbound en OPNsense:

    ```bash
    pvesh delete /cluster/sdn/vnets/pre/ips --zone lab --vnet pre --ip 10.20.2.10
    pvesh set /cluster/sdn
    { pvesh get /cluster/sdn/ipams/pve/status --output-format json | jq '.[]|select(.ip|startswith("10.20."))'
      dig +short app01.pre.lab @10.10.0.1; dig +short -x 10.20.2.10 @10.10.0.1; } | tee ev/09-red-dns.txt
    ```

9. En OPNsense, reglas primero (mon01 hacia los exporters de pre, NAT hacia web01-pre) y aliases después (`pre_front`, `pre_back`, `pre_data`); aplica, actualiza la matriz de reglas en `operacion/` y haz commit. Verifica desde fuera y desde mon01:

    ```bash
    nmap -Pn 10.20.1.10 -p 80,443,8080 | tee ev/10-firewall.txt
    ssh mon01 'nc -zv -w2 10.20.2.10 9100' 2>&1 | tee -a ev/10-firewall.txt
    ```

10. Revoca los certificados en la CA y publica la CRL:

    ```bash
    cd /etc/ca
    for c in certs/*.pre.lab.pem; do openssl ca -config ca.cnf -revoke $c -crl_reason cessationOfOperation; done
    openssl ca -config ca.cnf -gencrl -out crl/ca.crl.pem
    cp crl/ca.crl.pem /var/www/ca/ca.crl
    openssl crl -in crl/ca.crl.pem -noout -text | grep -B1 -A3 'cessationOfOperation' | tee ~/repos/operacion/baja/pre/ev/11-crl.txt
    ```

11. Credenciales y tokens en Proxmox, Gitea y Jenkins (ajusta los identificadores a los que listaste en A8.1) y vuelve a ejecutar los listados del paso 4 de A8.1 en `ev/12-credenciales.txt`:

    ```bash
    pveum user token remove tofu@pve pre; pveum user delete tofu-pre@pve 2>/dev/null
    G=http://gitea01.dev.lab:3000/api/v1
    curl -X DELETE -H "Authorization: token $GT" $G/users/jenkins/tokens/pre-deploy
    curl -X DELETE -H "Authorization: token $GT" $G/repos/ops/servicio/keys/7
    curl -X DELETE -H "Authorization: token $GT" $G/repos/ops/servicio/hooks/3
    curl -X POST -u ops:$TOKEN http://jenkins01.dev.lab:8080/credentials/store/system/domain/_/credential/pre-ssh-app01/doDelete
    ```

12. Archiva el repositorio (no lo borres) y comprueba el estado del job:

    ```bash
    curl -s -X PATCH -H "Authorization: token $GT" -H 'Content-Type: application/json' -d '{"archived": true}' $G/repos/ops/servicio | jq '.archived'
    curl -s -u ops:$TOKEN "http://jenkins01.dev.lab:8080/api/json?tree=jobs[name,color]" | jq '.jobs[]|select(.name|test("pre"))' | tee ev/13-archivado.txt
    ```

13. Marca en `plan.md` cada punto con su evidencia y apunta en "Incidencias" cualquier desviación. Commit.

**Comprobación.** Listados de `ev/04` a `ev/13` vacíos o con el estado esperado (`202`, `archived: true`, job `disabled`); `dig` sin respuesta; `tofu state list` vacío; la CRL con todos los `*.pre.lab`.

**Entrega.** Commit en `operacion` con `plan.md`, `ev/02` a `ev/13` y la matriz de reglas corregida; el `pre-bloqueo.sql.gpg` entregado al profesor fuera del repositorio.

**Si te sobra tiempo.** Levanta el respondedor OCSP mínimo del apartado de certificados y guarda `Cert Status: revoked` en `ev/11b-ocsp.txt`.

## Sesión 38 · Copias y logs

<p class="ut-meta" markdown>11 de marzo · Teoría y práctica · <span class="dur" title="Explicación unos 20 min, práctica unos 100 min">:material-school:<i class="dur-barra" style="--teoria:17%"></i>:material-flask:</span></p>

En esta sesión cambiamos de escala: ya no quitamos cosas de sistemas que obedecen a un comando, sino que garantizamos que unos datos no se puedan recuperar. Explico por qué borrar no borra (SSD, copy-on-write, versionado, snapshots) y de ahí sale el orden de opciones, con el borrado criptográfico primero; los comandos de restic, S3 y Loki están en los subapartados. El apartado de photorec es material de consulta para los tres escenarios de recuperación de la hoja.

### Por qué borrar no borra

Este apartado cambia de escala. Hasta ahora hemos quitado cosas de sistemas que obedecen a un comando; ahora hay que garantizar que unos datos no se puedan recuperar, y para eso hace falta entender qué ocurre en el disco cuando borráis un fichero. La idea con la que tenéis que salir es que el borrado fiable solo se consigue si los datos nunca estuvieron en claro, y por eso la lista de opciones empieza por el cifrado y no por sobrescribir.

`rm` desvincula el nombre del fichero de sus bloques y marca los bloques como libres. Los datos siguen ahí hasta que otra escritura los pise, y un `photorec` (herramienta de recuperación de ficheros borrados) los recupera en minutos. Sobre eso se apilan cuatro mecanismos modernos que hacen que ni siquiera sobrescribir garantice nada:

- **SSD y wear leveling.** El controlador del SSD reparte las escrituras entre celdas para que se desgasten por igual. Cuando `shred` escribe tres veces sobre el mismo LBA (la dirección lógica del bloque, lo que el sistema operativo cree que es "el mismo sitio"), el controlador escribe en tres páginas físicas distintas y deja la original marcada como inválida, legible con acceso al chip. TRIM (`fstrim`, `blkdiscard`) le dice al controlador que puede borrar esas páginas, pero "puede" no es "debe", y no todos los firmwares lo hacen de inmediato.
- **Copy-on-write en ZFS y btrfs.** Estos sistemas de ficheros nunca sobrescriben un bloque en su sitio: escriben el nuevo en otro lugar y actualizan los punteros. `shred` en ZFS crea tres copias nuevas y deja la original intacta. Y si hay un snapshot, el bloque original está referenciado y ni siquiera se marca libre. En Proxmox con storage ZFS (`local-zfs`) el disco de la VM es un zvol (un volumen de bloques dentro de ZFS) con esta propiedad.
- **Versionado en el almacenamiento de objetos.** Un bucket S3 o MinIO con versionado activado no borra nunca: `DELETE` crea un marcador de borrado y la versión anterior sigue ahí. Con Object Lock en modo compliance, ni el administrador puede borrarla antes de que venza la retención.
- **Snapshots del hipervisor y backups.** El snapshot de "antes de la actualización" de la UT7 contiene el disco completo con los datos que acabáis de borrar dentro de la VM. Lo mismo el vzdump de cada domingo.

La consecuencia práctica es que el borrado fiable de un soporte que no controláis físicamente (una VM sobre ZFS, un bucket en un proveedor, un disco de un servidor alquilado) solo se consigue de una manera: que los datos nunca hayan estado en claro. De ahí el orden de opciones.

#### Opciones de más a menos fiable

**1. Borrado criptográfico.** Si las copias estaban cifradas (restic, borg, gpg, LUKS, S3 con SSE-KMS), se destruye la clave y los datos se vuelven ruido aunque el fichero exista, aunque haya versiones, aunque haya snapshots. Es el método correcto y por eso las copias se cifran desde el principio ([UT6](ut6-copias-seguridad.md)). NIST SP 800-88 lo reconoce como técnica de *Purge* (borrado que resiste incluso una recuperación de laboratorio) siempre que la clave se haya gestionado bien (nunca almacenada junto a los datos, nunca copiada a sitios que no controláis).

En restic, el repositorio tiene una clave maestra que se guarda cifrada con cada contraseña en `keys/`. Destruir todos los ficheros de `keys/` y la contraseña equivale a destruir la clave maestra:

```bash
export RESTIC_REPOSITORY=s3:http://10.10.0.30:9000/backups/pre
restic key list                                           # apuntar los IDs en el acta
mc rm --recursive --force --versions s3/backups/pre/keys/  # todas las versiones de las claves
shred -u /etc/restic/pass-pre                              # y la contraseña, en todos los hosts que la tenían
restic snapshots
# Fatal: wrong password or no key found
```

La comprobación final es esa: restic no puede abrir el repositorio con la contraseña correcta porque ya no hay clave que abrir. Si además queréis liberar el espacio, `mc rm --recursive --force --versions s3/backups/pre/` después; pero la irrecuperabilidad ya estaba garantizada antes de eso.

Con borg, la clave está en el repositorio (modo `repokey`) o en `~/.config/borg/keys/` (modo `keyfile`): `borg key export` os dice cuál y dónde, y se destruyen las copias de la clave y la passphrase. Con LUKS, `cryptsetup luksErase /dev/sdb1` borra todos los keyslots de la cabecera (queda la cabecera sin claves; `cryptsetup luksDump` muestra los slots vacíos) y la copia de la cabecera que guardasteis con `luksHeaderBackup` hay que destruirla también. Con KMS (el servicio gestor de claves del proveedor) en la nube, `aws kms schedule-key-deletion --key-id <id> --pending-window-in-days 7`: el borrado es diferido a propósito y en esos siete días cualquier objeto cifrado con esa clave sigue siendo legible.

**2. Borrado en el sistema de copias.** Si el repositorio se conserva porque tiene otros clientes, se borran los snapshots del servicio y se reescribe: `restic forget --tag pre --prune` (o `--host app01-pre`) marca los snapshots y `prune` reempaqueta los packs que los contenían. En borg, `borg delete --glob-archives 'pre-*' repo` y luego `borg compact repo`; sin `compact` borg solo marca los segmentos y el espacio, y los datos, siguen. En los dos casos, los packs antiguos se han borrado del backend, lo cual sobre S3 con versionado significa que hay que ir al punto 3.

**3. Almacenamiento de objetos.** Listar todas las versiones y marcadores del prefijo, borrarlos explícitamente por `VersionId`, quitar las reglas de ciclo de vida y comprobar que no hay replicación a otro bucket:

```bash
aws --endpoint-url http://10.10.0.30:9000 s3api list-object-versions --bucket backups --prefix pre/ \
  --query '{Objects: [Versions[].{Key:Key,VersionId:VersionId}, DeleteMarkers[].{Key:Key,VersionId:VersionId}][]}' \
  --output json > versiones.json
aws --endpoint-url http://10.10.0.30:9000 s3api delete-objects --bucket backups --delete file://versiones.json
aws --endpoint-url http://10.10.0.30:9000 s3api list-object-versions --bucket backups --prefix pre/   # vacío
mc replicate ls s3/backups                                                                            # sin réplicas
```

`delete-objects` admite 1000 claves por llamada; con más, se trocea. Si el bucket tiene Object Lock en modo *governance*, el borrado necesita `--bypass-governance-retention` y el permiso correspondiente; en modo *compliance* no hay forma de borrar antes de la fecha de retención, ni siquiera siendo root del proveedor. Eso es exactamente lo que se buscaba al activarlo para protegerse del ransomware, y por eso la retención de Object Lock se decide contando con la baja: una retención de 90 días significa que el acta de baja tendrá una fecha de destrucción 90 días posterior a la ejecución, y alguien tiene que volver a verificarla entonces. Con la clave de restic destruida, mientras tanto, esos objetos son ruido.

**4. Sobrescritura y descarte.** En discos magnéticos dedicados, `shred -n 3 -z -v /dev/sdb` (tres pasadas aleatorias y una de ceros) o `nwipe` (el sucesor de DBAN, con verificación y registro) valen como *Clear* de NIST. En SSD y NVMe, la herramienta correcta no es `shred` sino el borrado seguro del propio firmware: `nvme format /dev/nvme0n1 --ses=1` (o `--ses=2` para borrado criptográfico si el disco lo soporta) y `hdparm --security-erase` en SATA; `blkdiscard /dev/nvme0n1` descarta todo el dispositivo pero, por lo dicho antes, sin garantía de que las celdas se borren. En cinta, el borrado lo hace el software de la librería. En una VM de Proxmox no tenéis acceso a nada de esto: el disco virtual es un fichero o un zvol, y "sobrescribir" desde dentro de la VM no llega al soporte físico.

**5. Destrucción física.** Desmagnetizado, trituración o perforación, cuando lo exige la normativa (ENS categoría alta, datos de salud, contratos que lo especifican). Se hace con proveedor certificado y certificado de destrucción por número de serie. NIST SP 800-88 rev. 1 es la referencia que os van a citar en cualquier pliego: define *Clear* (sobrescritura lógica, protege contra recuperación con herramientas de software), *Purge* (borrado criptográfico, secure erase, desmagnetizado; protege contra recuperación de laboratorio) y *Destroy*, y tiene un apéndice con el método adecuado por tipo de soporte.

#### Logs externos

Los logs del servicio están en tres sitios: en Loki, en los ficheros rotados de cada host y en cualquier sitio al que se reenviaran (un SIEM, que es el sistema que centraliza eventos de seguridad, o el correo de las alarmas). En Loki 3.x el borrado por consulta lo hace el **compactor**, y hay que tenerlo configurado para ello:

```yaml
compactor:
  working_directory: /loki/compactor
  retention_enabled: true
  delete_request_store: filesystem
limits_config:
  deletion_mode: filter-and-delete
  retention_period: 168h
  retention_stream:
    - selector: '{env="pre"}'
      priority: 1
      period: 24h
```

Con eso, la petición de borrado se envía a la API del compactor y se ejecuta en el siguiente ciclo, después del periodo de cancelación (24 horas por defecto, `delete_request_cancel_period`) durante el cual se puede anular. No es inmediato, y eso hay que reflejarlo en el acta:

```bash
curl -s -X POST -G 'http://10.10.0.20:3100/loki/api/v1/delete' \
  --data-urlencode 'query={env="pre"}' --data-urlencode 'start=1700000000' --data-urlencode "end=$(date +%s)"
curl -s 'http://10.10.0.20:3100/loki/api/v1/delete' | jq .        # estado: received, processed
logcli --addr=http://10.10.0.20:3100 query '{env="pre"}' --since=8760h --limit=1   # vacío tras el procesado
logcli --addr=http://10.10.0.20:3100 series '{env="pre"}'                          # vacío
```

La `retention_stream` con `period: 24h` para `{env="pre"}` es el cinturón además de los tirantes: aunque llegue algún log rezagado (un Promtail que no se retiró), desaparece en un día. Los chunks borrados del filesystem o del bucket de Loki están sujetos a lo mismo que todo lo demás: versionado, snapshots, bloques libres.

En los hosts, los ficheros rotados: `/var/log/nginx/*.gz` en web01, los JSON del driver de logs de Docker en `/var/lib/docker/containers/*/` en app01, `/var/log/postgresql/` en db01, y el `positions.yaml` de Promtail. Y en mon01, si en la UT2 configurasteis Alertmanager para enviar correo a Mailpit, la bandeja de Mailpit contiene el texto de las alertas con etiquetas e IP. Se limpia con `journalctl --vacuum-time=1s` para el journal si el servicio escribía ahí, `rm` para los rotados, y la destrucción de la VM para todos a la vez.

La verificación tiene dos niveles: una consulta que no devuelve nada, y, sobre un soporte al que tengáis acceso, una herramienta de recuperación que no encuentra nada legible. Ese segundo nivel es la actividad A8.3.

### Verificar con photorec sobre un volumen de pruebas

*Material de consulta: no se explica en clase; lo necesitas para la hoja de práctica de esta sesión.*

Para que el "irrecuperable" del acta no sea un acto de fe, la actividad A8.3 lo pone a prueba sobre un volumen que controláis del todo. Se crea un fichero imagen, se formatea, se escribe un fichero reconocible, se borra con cada método y se intenta recuperar:

```bash
dd if=/dev/zero of=prueba.img bs=1M count=256 && mkfs.ext4 -q prueba.img
mkdir -p /mnt/prueba && sudo mount -o loop prueba.img /mnt/prueba
sudo pg_dump -U app servicio > /mnt/prueba/dump.sql        # o cualquier fichero de texto con datos reconocibles
sudo rm /mnt/prueba/dump.sql && sudo umount /mnt/prueba
photorec /d recuperado /cmd prueba.img search             # modo no interactivo
grep -rl 'INSERT INTO usuarios' recuperado/ | head
```

Qué esperar: tras un `rm` en ext4, photorec recupera el dump entero (encuentra la firma de texto y sigue los bloques contiguos). Tras `shred -u` sobre el fichero antes del `umount`, photorec no encuentra nada legible en ext4 sobre un fichero imagen; repetid la prueba con la imagen en un dataset ZFS con un snapshot previo y veréis que el `shred` no ha servido para nada: `zfs diff` y un `photorec` sobre el snapshot lo devuelven íntegro. Y si el volumen estaba en LUKS y se han destruido los keyslots, photorec sobre el dispositivo en bruto encuentra solo ruido: ninguna firma de fichero.

`testdisk` es el hermano de photorec para recuperar particiones y tablas de ficheros enteras; sirve para demostrar que una tabla de particiones borrada con `wipefs` sigue siendo reconstruible. Ninguna de las dos herramientas hace nada contra un cifrado con clave destruida, y esa es la conclusión que tiene que aparecer en el informe de la actividad, con las capturas.

### A8.3 Copias y logs (sesión 38)

**Objetivo.** Repositorio restic de pre irrecuperable, prefijo `pre/` del bucket sin versiones, Loki sin nada para `{env="pre"}` e informe de photorec con tres escenarios.

**Antes de empezar.**

- A8.2 terminada: las VM de pre no existen. Queda lo de fuera: MinIO (10.10.0.30), Loki y Mailpit en mon01, y la contraseña de restic en los hosts que la tenían.
- `mc` con el alias `s3` hacia MinIO, `aws` CLI, `restic`, `logcli`, `photorec` (paquete `testdisk`) y una máquina con ZFS (el nodo Proxmox con `local-zfs` o una VM).
- Se ha explicado [por qué borrar no borra](#por-que-borrar-no-borra); los comandos están en [Opciones de más a menos fiable](#opciones-de-mas-a-menos-fiable), [Logs externos](#logs-externos) y [Verificar con photorec](#verificar-con-photorec-sobre-un-volumen-de-pruebas).

**Pasos.**

1. Borrado criptográfico del repositorio restic: lista las claves (los IDs van al acta), destruye todas sus versiones y la contraseña:

    ```bash
    export RESTIC_REPOSITORY=s3:http://10.10.0.30:9000/backups/pre
    export RESTIC_PASSWORD_FILE=/etc/restic/pass-pre
    restic key list | tee ~/repos/operacion/baja/pre/ev/14-restic-keys.txt
    mc ls --versions s3/backups/pre/keys/
    mc rm --recursive --force --versions s3/backups/pre/keys/
    mc ls --versions s3/backups/pre/keys/                       # vacío
    restic snapshots 2>&1 | tee ~/repos/operacion/baja/pre/ev/15-restic-nokey.txt   # Fatal: wrong password or no key found
    shred -u /etc/restic/pass-pre                               # en cada host que la tenía
    rm -rf ~/.cache/restic
    ```

2. Versiones y marcadores del prefijo `pre/` en el bucket, replicación y ciclo de vida:

    ```bash
    E="--endpoint-url http://10.10.0.30:9000"
    aws $E s3api list-object-versions --bucket backups --prefix pre/ \
      --query '{Objects: [Versions[].{Key:Key,VersionId:VersionId}, DeleteMarkers[].{Key:Key,VersionId:VersionId}][]}' \
      --output json > versiones.json
    aws $E s3api delete-objects --bucket backups --delete file://versiones.json   # máximo 1000 claves por llamada
    aws $E s3api list-object-versions --bucket backups --prefix pre/ | tee ~/repos/operacion/baja/pre/ev/16-s3-versions.json
    mc replicate ls s3/backups | tee -a ~/repos/operacion/baja/pre/ev/16-s3-versions.json
    ```

    Con Object Lock en modo compliance, `delete-objects` fallará: apunta la fecha de retención en "Pendientes con fecha".

3. Comprueba que el compactor de Loki en mon01 admite borrados (`retention_enabled: true`, `deletion_mode: filter-and-delete`; si no, corrígelo y reinicia Loki). Envía la petición y comprueba el estado:

    ```bash
    L=http://10.10.0.20:3100
    curl -s -X POST -G "$L/loki/api/v1/delete" \
      --data-urlencode 'query={env="pre"}' --data-urlencode 'start=1700000000' --data-urlencode "end=$(date +%s)"
    curl -s "$L/loki/api/v1/delete" | jq . | tee ~/repos/operacion/baja/pre/ev/17-loki-delete.json   # status: received
    ```

    El compactor la procesa tras el periodo de cancelación (24 h por defecto); la verificación con `logcli` se hace al empezar la sesión 39.

4. Añade la `retention_stream` de 24 h para `{env="pre"}` en la configuración de Loki del repositorio `monitoring`, commit y despliegue.

5. Logs fuera de las VM destruidas, en mon01: la bandeja de Mailpit y el journal:

    ```bash
    ssh mon01 'curl -s -X DELETE http://localhost:8025/api/v1/messages; sudo journalctl --vacuum-time=1s'
    ssh mon01 'curl -s http://localhost:8025/api/v1/messages | jq .total' | tee ~/repos/operacion/baja/pre/ev/18-mailpit.txt   # 0
    ```

6. Escenario 1 de recuperación (`rm` en ext4), con un volumen de pruebas y un fichero reconocible:

    ```bash
    mkdir -p ~/photorec && cd ~/photorec
    dd if=/dev/zero of=prueba.img bs=1M count=256 && mkfs.ext4 -q prueba.img
    mkdir -p /mnt/prueba && sudo mount -o loop prueba.img /mnt/prueba
    for i in $(seq 1 500); do echo "INSERT INTO usuarios VALUES ($i, 'usuario$i@pre.lab', '6000000$i');"; done | sudo tee /mnt/prueba/dump.sql >/dev/null
    sudo rm /mnt/prueba/dump.sql && sudo umount /mnt/prueba
    photorec /d rec-rm /cmd prueba.img search
    grep -rl 'INSERT INTO usuarios' rec-rm/ | head | tee ev-photorec-rm.txt
    ```

7. Escenario 2 (`shred` en ext4): repite el paso 6 con `sudo shred -u -n 3 /mnt/prueba/dump.sql` en lugar de `rm`, recupera en `rec-shred/` y guarda el `grep` en `ev-photorec-shred.txt`.

8. Escenario 3 (ZFS con snapshot):

    ```bash
    sudo zfs create rpool/prueba
    sudo cp dump.sql /rpool/prueba/ && sudo zfs snapshot rpool/prueba@antes
    sudo shred -u -n 3 /rpool/prueba/dump.sql
    sudo zfs diff rpool/prueba@antes
    grep -c 'INSERT INTO usuarios' /rpool/prueba/.zfs/snapshot/antes/dump.sql | tee ev-photorec-zfs.txt
    sudo zfs destroy -r rpool/prueba
    ```

9. Redacta `operacion/baja/pre/informe-photorec.md` con una tabla (escenario, método, qué recuperó photorec, por qué) y la conclusión: qué método garantiza la irrecuperabilidad en un soporte que no controlas. Copia los tres `ev-photorec-*.txt` a `ev/19-photorec/`.

**Comprobación.** `restic snapshots` falla con "wrong password or no key found"; `list-object-versions` de `pre/` sin `Versions` ni `DeleteMarkers`; petición de Loki en `received` o `processed`; escenario 1 recupera el dump entero, el 2 nada y el 3 conserva las 500 líneas en el snapshot.

**Entrega.** Commit en `operacion` con `ev/14` a `ev/19` e `informe-photorec.md`; commit en `monitoring` con la `retention_stream`.

**Si te sobra tiempo.** Cuarto escenario: volumen LUKS, `cryptsetup luksErase` y photorec sobre la imagen en bruto (cero ficheros).

## Sesión 39 · Datos y monitorización

<p class="ut-meta" markdown>16 de marzo · Teoría y práctica · <span class="dur" title="Explicación unos 15 min, práctica unos 105 min">:material-school:<i class="dur-barra" style="--teoria:13%"></i>:material-flask:</span></p>

Última sesión de ejecución: al acabar, la base de datos de dev no conserva datos personales del tenant pre y la monitorización no guarda ninguna referencia al entorno, con el silencio expirado y sin alertas. Explico dos cosas: qué hace de verdad un DELETE en PostgreSQL y cómo se anonimiza cuando hay que conservar estadísticas, y en qué orden se desconfiguran targets, reglas, rutas, dashboards y Promtail para no disparar alarmas fantasma.

### Datos confidenciales en la base de datos interna

En el laboratorio la base de datos de pre muere con db01-pre. Pero el caso habitual en una empresa es que la base de datos se conserva porque otros servicios la usan, y lo que hay que borrar son las tablas y los datos de un servicio concreto. Ahí es donde `DELETE` engaña.

PostgreSQL usa MVCC (control de concurrencia por versiones: cada modificación crea una versión nueva de la fila en vez de tocar la vieja): `DELETE` no borra la fila, marca la tupla como muerta y la deja en la página hasta que `VACUUM` la recicla. `VACUUM` normal libera el espacio para que la propia tabla lo reutilice, pero no lo devuelve al sistema operativo ni sobrescribe nada: los bytes siguen en el fichero de la tabla, legibles con un editor hexadecimal por cualquiera con acceso al directorio de datos o a un backup del mismo. `DROP TABLE` desvincula los ficheros del segmento, con lo que aplica lo del apartado anterior.

```sql
-- Antes: ver cuánto hay muerto
SELECT relname, n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname LIKE 'pre_%';
DELETE FROM pre_sesiones;   -- deja n_dead_tup = filas
VACUUM FULL pre_sesiones;   -- reescribe la tabla en un fichero nuevo y desvincula el viejo
DROP TABLE pre_sesiones, pre_pedidos CASCADE;
```

`VACUUM FULL` bloquea la tabla en exclusiva mientras la reescribe, lo que en una tabla de 50 GB en producción puede ser una hora sin servicio. `pg_repack` hace lo mismo sin bloqueo largo (crea una copia, la sincroniza con triggers y cambia los ficheros al final): `pg_repack -d servicio -t usuarios`. En los dos casos el fichero antiguo se desvincula, no se sobrescribe. Hay tres sitios más donde viven los datos borrados y que la gente olvida: el **WAL** (el registro de escrituras de PostgreSQL: cada fila insertada o actualizada está en los segmentos de WAL y en el archivo de WAL si hay PITR configurado (recuperación a un punto en el tiempo), hasta que la retención los recicle), las **réplicas** (la réplica en streaming aplica el DELETE, pero su base backup inicial y sus propios WAL archivados no) y los **dumps sueltos** (el `pg_dump` en `/tmp` que alguien hizo para probar la migración, el fichero en el portátil de desarrollo, el que dejó el pipeline en un workspace de Jenkins).

#### Anonimizar cuando hay que conservar estadísticas

Si el negocio necesita conservar los pedidos por mes pero no a quién se hicieron, no se borra: se anonimiza. El RGPD (considerando 26) distingue **seudonimización**, que sigue siendo dato personal porque con información adicional se puede volver a identificar, de **anonimización**, que ya no lo es. Un `md5(email)` sin sal es seudonimización: cualquiera con una lista de correos puede calcular el hash y cruzar. Para que sea anonimización el hash lleva una sal aleatoria que se destruye al acabar, o directamente se sustituye por valores sintéticos que no derivan del original:

```sql
BEGIN;
-- Sal de un solo uso; no se guarda en ningún sitio
CREATE TEMP TABLE sal AS SELECT gen_random_uuid()::text AS s;
UPDATE usuarios u SET
  email    = encode(sha256(convert_to(u.email || (SELECT s FROM sal), 'UTF8')), 'hex') || '@anon.invalid',
  nombre   = 'Usuario ' || u.id,
  telefono = NULL,
  direccion = NULL,
  ip_alta  = NULL,
  fecha_nac = date_trunc('year', u.fecha_nac)   -- se conserva el año para estadísticas de edad
WHERE u.tenant = 'pre';
-- Tablas satélite con texto libre: no se pueden anonimizar, se borran
DELETE FROM comentarios WHERE usuario_id IN (SELECT id FROM usuarios WHERE tenant = 'pre');
COMMIT;
VACUUM FULL usuarios, comentarios;
-- Comprobación
SELECT count(*) FROM usuarios WHERE tenant='pre' AND (email NOT LIKE '%@anon.invalid' OR telefono IS NOT NULL);
```

Los campos de texto libre (comentarios, notas del pedido, motivo de la devolución) no se pueden anonimizar de forma fiable: la gente escribe su teléfono en el comentario. Se borran. Y las columnas cuasi-identificadoras (código postal completo, fecha de nacimiento exacta, sexo) combinadas identifican a una persona aunque no haya nombre, así que se generalizan (año en vez de fecha, provincia en vez de código postal). Si os piden "anonimizar la base de datos" sin más detalle, esa es la conversación que hay que tener antes de escribir el UPDATE.

#### Si la base de datos se elimina entera

Se para el servicio, se borra el volumen o el disco y se aplica al soporte lo del apartado de copias. Con el volumen cifrado con LUKS, se destruyen las claves y listo. Sin cifrado, la única garantía en una VM es la destrucción del disco virtual más la confianza en que el storage del hipervisor no lo conserve en snapshots, que es una confianza que conviene verificar con `zfs list -t snapshot | grep vm-210`.

Quedan las **cachés y colas**: Redis guarda en memoria y, si tiene persistencia, en `dump.rdb` y en el AOF (su fichero de registro de escrituras); `FLUSHDB` y borrar los ficheros. RabbitMQ o similar: purgar la cola (`rabbitmqctl purge_queue pedidos-pre`) y borrar el vhost. Un mensaje encolado con datos de un cliente puede sobrevivir años en una cola muerta que nadie consume.

### Desconfigurar la monitorización y las alarmas

Un servicio dado de baja que sigue en la monitorización genera alarmas eternas, ocupa retención y, lo peor, puede resucitar por error: alguien ve el target en rojo, piensa que se cayó y lo levanta desde un snapshot.

**Prometheus.** Se quitan los targets del `prometheus.yml` o del fichero de descubrimiento (`file_sd`), se valida y se recarga sin reiniciar:

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://10.10.0.20:9090/-/reload      # necesita --web.enable-lifecycle
curl -s http://10.10.0.20:9090/api/v1/targets | jq '[.data.activeTargets[]|select(.labels.env=="pre")]|length'   # 0
```

Quitar el target no borra las series: las muestras siguen en el TSDB hasta que la retención las expulse (15 días por defecto); el TSDB es la base de datos de series temporales de Prometheus, sus ficheros en `/prometheus`. `count({env="pre"})` devuelve datos hasta entonces. Si el acta necesita que desaparezcan ya, la API de administración lo hace, si Prometheus arrancó con `--web.enable-admin-api`:

```bash
curl -X POST -G http://10.10.0.20:9090/api/v1/admin/tsdb/delete_series --data-urlencode 'match[]={env="pre"}'
curl -X POST http://10.10.0.20:9090/api/v1/admin/tsdb/clean_tombstones
curl -s -G http://10.10.0.20:9090/api/v1/query --data-urlencode 'query=count({env="pre"})' | jq '.data.result'   # []
```

`delete_series` marca tombstones; `clean_tombstones` reescribe los bloques para que las muestras desaparezcan del disco. Sin el segundo, las muestras siguen en los ficheros de bloque aunque las consultas no las devuelvan.

**Reglas.** Las recording rules `app:...` y las alertas de `alerts.yml` que filtren por `env="pre"` se retiran del repositorio `alerting` con un commit; el pipeline las despliega y recarga. Antes de retirarlas, el silencio que pusisteis al principio sigue activo, así que el vaivén de la recarga no dispara nada. Comprobación: `curl -s :9090/api/v1/rules | jq '.data.groups[].rules[]|select(.query|test("pre"))'` vacío.

**Alertmanager.** Rutas con `match` sobre `env: pre`, receptores exclusivos (el webhook al gestor de incidencias de pre, el correo del equipo) y las plantillas que los referencian. `amtool check-config alertmanager.yml` y recarga con `curl -X POST :9093/-/reload`. Las incidencias abiertas de pre en el gestor se cierran con el motivo "servicio dado de baja" y referencia al acta. Al final, se expira el silencio y se comprueba que no hay nada:

```bash
amtool --alertmanager.url=http://10.10.0.20:9093 silence query env=pre         # apuntar el ID
amtool --alertmanager.url=http://10.10.0.20:9093 silence expire <id>
amtool --alertmanager.url=http://10.10.0.20:9093 alert query env=pre            # vacío
```

**Grafana.** Los dashboards se exportan a Git antes de borrarlos, porque un dashboard bien hecho vale para el siguiente servicio y porque en una auditoría os pueden pedir "qué se vigilaba":

```bash
G=http://10.10.0.20:3000; H="Authorization: Bearer $GRAFANA_TOKEN"
for uid in $(curl -s -H "$H" "$G/api/search?tag=pre" | jq -r '.[].uid'); do
  curl -s -H "$H" $G/api/dashboards/uid/$uid | jq '.dashboard' > operacion/dashboards/pre/$uid.json
  curl -s -X DELETE -H "$H" $G/api/dashboards/uid/$uid
done
curl -s -X DELETE -H "$H" $G/api/datasources/uid/loki-pre   # solo si la fuente era exclusiva de pre
git -C operacion add dashboards/pre && git -C operacion commit -m "UT8: dashboards de pre archivados antes de la baja"
```

Si la fuente de datos era la compartida (el Prometheus y el Loki de mon01), no se toca: solo se borra la que apuntaba en exclusiva a pre. Las reglas de alerta propias de Grafana, si las usasteis, se listan en `/api/v1/provisioning/alert-rules` y se borran igual.

**Promtail.** El job con `env: pre` se quita del `promtail.yml` de cada host antes de destruir las VM; si las VM ya no existen, no hay nada que quitar, pero el fichero de configuración en el repositorio `monitoring` sí, y ese commit es la evidencia.

**Firewall y runbooks.** Las reglas de la UT3 que permitían a mon01 llegar a los exporters de pre (9100, 8080, 9187, 9102) se quitan con las demás del apartado de infraestructura, y se comprueba desde mon01 que `nc -zv 10.20.2.10 9100` no conecta. En `operacion/`, la ficha de cada alarma de pre y su runbook se marcan como retirados con fecha y enlace al acta; no se borran, porque el catálogo de alarmas es historia del servicio.

### A8.4 Datos y monitorización (sesión 39)

**Objetivo.** La base de datos de dev sin datos personales del tenant `pre`, y la monitorización sin referencias a pre: cero targets, reglas, series, dashboards, silencios y alertas.

**Antes de empezar.**

- Acceso a db01 de dev con el usuario `app`, a mon01 y a jenkins01.
- Prometheus arrancado con `--web.enable-lifecycle` y `--web.enable-admin-api` (si falta, añádelo al compose de mon01 y reinicia antes de empezar).
- Los repositorios `alerting`, `monitoring` y `operacion` actualizados; los dashboards de pre ya están en Git desde A8.2.
- Se ha explicado [DELETE y VACUUM FULL](#datos-confidenciales-en-la-base-de-datos-interna), [la anonimización](#anonimizar-cuando-hay-que-conservar-estadisticas) y [la desconfiguración de la monitorización](#desconfigurar-la-monitorizacion-y-las-alarmas).

**Pasos.**

1. Cierra el pendiente de la sesión anterior: la petición de borrado de Loki debe estar en `processed`.

    ```bash
    L=http://10.10.0.20:3100
    curl -s "$L/loki/api/v1/delete" | jq '.[]|{status,query}'
    logcli --addr=$L query '{env="pre"}' --since=8760h --limit=1 | tee ~/repos/operacion/baja/pre/ev/20-loki-vacio.txt
    logcli --addr=$L series '{env="pre"}' | tee -a ~/repos/operacion/baja/pre/ev/20-loki-vacio.txt
    ```

2. En db01 de dev, mide antes de tocar nada: la consulta de `pg_stat_user_tables` del paso 5 y `SELECT count(*) FROM usuarios WHERE tenant='pre';`.

3. Anonimiza los usuarios del tenant `pre` con sal de un solo uso y borra las tablas satélite con texto libre, en una transacción:

    ```sql
    BEGIN;
    CREATE TEMP TABLE sal AS SELECT gen_random_uuid()::text AS s;
    UPDATE usuarios u SET
      email    = encode(sha256(convert_to(u.email || (SELECT s FROM sal), 'UTF8')), 'hex') || '@anon.invalid',
      nombre   = 'Usuario ' || u.id,
      telefono = NULL,
      direccion = NULL,
      ip_alta  = NULL,
      fecha_nac = date_trunc('year', u.fecha_nac)
    WHERE u.tenant = 'pre';
    DELETE FROM comentarios WHERE usuario_id IN (SELECT id FROM usuarios WHERE tenant = 'pre');
    COMMIT;
    ```

4. Borra las tablas exclusivas del servicio y reescribe las que se conservan:

    ```sql
    DROP TABLE pre_sesiones, pre_pedidos CASCADE;
    VACUUM FULL usuarios, comentarios;
    ```

    Si `VACUUM FULL` falla por espacio (necesita el doble del tamaño de la tabla), haz primero los `DROP`.

5. Consulta de control y estadísticas; la salida va al acta:

    ```bash
    psql -U app -d servicio -c "SELECT count(*) AS restos FROM usuarios WHERE tenant='pre' AND (email NOT LIKE '%@anon.invalid' OR telefono IS NOT NULL OR direccion IS NOT NULL OR ip_alta IS NOT NULL);" \
         -c "SELECT relname, n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname IN ('usuarios','comentarios') OR relname LIKE 'pre_%';" \
      | tee ~/repos/operacion/baja/pre/ev/21-db-control.txt
    ```

6. Revisa dónde más pueden vivir esos datos: réplica (`SELECT * FROM pg_stat_replication;`), dumps sueltos, WAL archivado y cachés:

    ```bash
    ssh db01 'sudo find /tmp /var/tmp /home -name "*.sql*" -o -name "*.dump" 2>/dev/null; ls /var/lib/postgresql/archive 2>/dev/null | tail -3'
    ssh jenkins01 'sudo find /var/lib/jenkins/workspace -name "*.sql*" 2>/dev/null'
    redis-cli -h 10.20.2.11 --scan --pattern 'pre:*' | head 2>/dev/null   # si sobrevive un Redis compartido
    ```

    Lo que aparezca se borra con `shred -u`; el WAL archivado va a "Pendientes con fecha".

7. Prometheus: quita los targets de pre en `monitoring` (`prometheus.yml` o `file_sd`), commit y despliegue; valida, recarga y borra las series:

    ```bash
    P=http://10.10.0.20:9090
    promtool check config /etc/prometheus/prometheus.yml
    curl -X POST $P/-/reload
    curl -s $P/api/v1/targets | jq '[.data.activeTargets[]|select(.labels.env=="pre")]|length'      # 0
    curl -X POST -G $P/api/v1/admin/tsdb/delete_series --data-urlencode 'match[]={env="pre"}'
    curl -X POST $P/api/v1/admin/tsdb/clean_tombstones
    curl -s -G $P/api/v1/query --data-urlencode 'query=count({env="pre"})' | jq '.data.result' | tee ~/repos/operacion/baja/pre/ev/22-prometheus.txt   # []
    ```

8. Reglas: retira de `alerting/alerts.yml` y de las recording rules lo que filtre por `env="pre"`; commit y despliegue. El silencio sigue activo, así que la recarga no dispara nada.

    ```bash
    curl -s $P/api/v1/rules | jq '.data.groups[].rules[]|select(.query|test("pre"))' | tee ~/repos/operacion/baja/pre/ev/23-reglas.txt   # vacío
    ```

9. Alertmanager: quita rutas con `env: pre`, receptores exclusivos y sus plantillas en `alertmanager.yml` de `alerting`; valida y recarga:

    ```bash
    amtool check-config alertmanager.yml
    curl -X POST http://10.10.0.20:9093/-/reload
    ```

10. Grafana y Promtail: borra los dashboards de pre (ya exportados) y la datasource exclusiva si la había; quita el job `env: pre` de `promtail.yml` en `monitoring`, commit.

    ```bash
    G=http://10.10.0.20:3000; H="Authorization: Bearer $GRAFANA_TOKEN"
    for uid in $(curl -s -H "$H" "$G/api/search?tag=pre" | jq -r '.[].uid'); do curl -s -X DELETE -H "$H" $G/api/dashboards/uid/$uid; done
    curl -s -X DELETE -H "$H" $G/api/datasources/uid/loki-pre      # solo si era exclusiva de pre
    curl -s -H "$H" "$G/api/search?tag=pre" | tee ~/repos/operacion/baja/pre/ev/24-grafana.txt   # []
    ```

11. Comprueba que no hay alertas pendientes y solo entonces expira el silencio:

    ```bash
    A=http://10.10.0.20:9093
    amtool --alertmanager.url=$A alert query env=pre                       # vacío antes de expirar
    amtool --alertmanager.url=$A silence expire $(amtool --alertmanager.url=$A silence query -q env=pre)
    sleep 120; amtool --alertmanager.url=$A alert query env=pre | tee ~/repos/operacion/baja/pre/ev/25-alertas.txt   # vacío
    ```

12. En `operacion/`, marca las fichas de alarma y runbooks de pre como retirados con fecha y enlace al acta; no los borres. Commit.

**Comprobación.** `restos = 0`; `n_dead_tup` de `usuarios` y `comentarios` a 0 y sin tablas `pre_*`; `count({env="pre"})` devuelve `[]`; targets, reglas y dashboards de pre a cero; `amtool alert query env=pre` vacío dos minutos después de expirar el silencio.

**Entrega.** Commit en `operacion` con `ev/20` a `ev/25` y los runbooks marcados; commits en `alerting` (reglas y rutas) y `monitoring` (targets y Promtail).

**Si te sobra tiempo.** `hexdump -C` sobre el fichero de `usuarios` (`pg_relation_filepath`) antes y después del `VACUUM FULL`: `DELETE` deja los bytes.

## Sesión 40 · Práctica evaluable

<p class="ut-meta" markdown>18 de marzo · Práctica evaluable · <span class="dur" title="Explicación unos 10 min, práctica unos 110 min">:material-school:<i class="dur-barra" style="--teoria:8%"></i>:material-flask:</span></p>

La práctica evaluable cierra la unidad con el acta de baja: el documento que recoge la aprobación, lo conservado, lo destruido con su método, la lista de comprobación con una evidencia por punto y los pendientes con fecha. Al principio aclaro el enunciado y repaso la plantilla del acta, que es el apartado que sigue; el resto de la sesión es para completarla y entregar.

### Acta de baja

El acta es el entregable de la unidad y el documento que cierra la vida del servicio. Se guarda con las incidencias del servicio (en `operacion/`, en el gestor de incidencias, en el sistema documental de la empresa) y es lo que se enseña cuando dentro de dos años alguien pregunta "¿qué pasó con los datos de pre?".

```markdown
# Acta de baja · servicio-pre (entorno pre del servicio del curso)

## Identificación
- Servicio: API del curso, entorno pre (pre.lab, VLAN/vnet pre, VMs 210 a 213)
- Responsable del servicio: <nombre, cargo>
- Ejecuta: <nombre>, con revisión de <nombre>
- Solicitud: RFC-2027-014, aprobada el 2027-03-04 por <nombre> (correo adjunto)
- Ventana: 2027-03-11 16:00 a 2027-03-23 18:00
- Comunicación: aviso a usuarios el 2027-02-25; aviso a soporte y guardia el 2027-03-09

## Qué se conserva
| Elemento | Motivo | Dónde | Hasta | Responsable de destruirlo |
|---|---|---|---|---|
| Dump de pedidos anonimizado | Estadística | operacion/datos/pre-pedidos-anon.sql.gpg | Indefinido (sin datos personales) | n/a |
| Dump completo cifrado (bloqueo LOPDGDD art. 32) | Posibles reclamaciones | Caja fuerte documental, solo DPO | 2030-03-23 | DPO |
| Repositorio de código (archivado) | Trazabilidad | Gitea ops/servicio | Indefinido | n/a |
| Dashboards y catálogo de alarmas | Reutilización | operacion/dashboards/pre | Indefinido | n/a |

## Qué se destruye y cómo
| Elemento | Método | Fecha | Evidencia |
|---|---|---|---|
| Repositorio restic pre | Borrado criptográfico: claves 3f9a…, b21c… y contraseña destruidas | 2027-03-16 | ev/03-restic-nokey.txt |
| Versiones S3 backups/pre/ | delete-objects por VersionId; sin réplicas | 2027-03-16 | ev/04-s3-versions.json |
| ...

## Lista de comprobación
| # | Punto | Verificación | Resultado | Evidencia |
|---|---|---|---|---|
| 1 | Silencio activo | amtool silence query env=pre | OK, id 7c1e… | ev/01.txt |
| ...

## Incidencias durante la baja
- 2027-03-11: el alias pre_back estaba en uso por una regla no inventariada (matriz desactualizada). Corregida la matriz, commit a1b2c3.

## Pendientes con fecha
- 2027-06-14: verificar expiración de Object Lock en backups/pre/ y borrar (responsable: <nombre>)
- 2030-03-23: destruir el dump bloqueado (responsable: DPO)

## Firmas
Ejecuta: ______  Revisa: ______  Responsable del servicio: ______  Fecha: ______
```

La sección de pendientes con fecha es la que distingue un acta útil de una que se archiva y se olvida: todo lo que no se pudo destruir en la ventana (Object Lock, bloqueo legal, KMS diferido) tiene fecha y nombre, y entra en el calendario del equipo.

### Enunciado

Entrega el **acta de baja** del entorno pre siguiendo la plantilla del apartado correspondiente, con la lista de comprobación completa y una evidencia por punto (salida de comando o captura, en `operacion/baja/pre/ev/`), incluida la prueba de que las copias no son restaurables (restic sin clave y listado de versiones vacío), el informe de recuperación con photorec sobre el volumen de pruebas, la consulta de control sobre la base de datos anonimizada y la demostración de que la monitorización no conserva referencias (targets, reglas, series, silencios y alertas a cero). El acta incluye la aprobación, lo conservado con fecha de destrucción, las incidencias durante la baja y los pendientes con fecha y responsable.

Entregables:

- [ ] `operacion/baja/pre/acta.md` con todas las secciones de la plantilla y firmas
- [ ] `operacion/baja/pre/ev/` con una evidencia numerada por punto de la lista
- [ ] Informe de photorec con los tres escenarios y conclusión
- [ ] Commits en `alerting`, `monitoring` y `operacion` que retiran reglas, targets, jobs y marcan los runbooks

| Criterio | RA5 | Peso |
|---|---|---|
| Aplicación terminada y todos los recursos de infraestructura liberados y verificados | a | 30 % |
| Copias de seguridad y logs externos eliminados de forma irrecuperable | b | 30 % |
| Datos confidenciales borrados en la base de datos interna | c | 15 % |
| Conectividad y referencias en monitorización y alarmas eliminadas | d | 25 % |

## Errores frecuentes en el laboratorio

**El pipeline vuelve a desplegar pre.** Alguien hace push al repositorio del servicio, el job de pre seguía habilitado, y Jenkins intenta desplegar contra una VM que no existe. Falla, pero antes ha creado imágenes nuevas en el registry con etiqueta `-pre`. Orden correcto: deshabilitar el job antes de borrar del registry, no después.

**`tofu destroy` quiere borrar el usuario de API de Proxmox que usan todos los entornos.** El recurso `proxmox_virtual_environment_user` se definió en el módulo compartido y entró en el estado de pre. Diagnóstico: `tofu plan -destroy` lo lista. Solución: `tofu state rm` de ese recurso antes de destruir, y mover su definición a un estado propio para el futuro.

**`docker compose down -v` no borra el volumen de la base de datos.** Estaba declarado `external: true`. `docker volume ls` lo muestra y `docker volume rm` lo quita. La lección es que `down -v` solo gestiona lo que Compose creó.

**Alertmanager dispara HostDown a los dos minutos de parar el servicio.** El silencio se puso con `env=pre` pero la alerta lleva la etiqueta `env="pre-lab"` porque en la UT2 se etiquetó distinto en el job de node_exporter. Diagnóstico: `amtool alert query` muestra las etiquetas reales. Se corrige el matcher del silencio; y se apunta en la matriz de etiquetas.

**La petición de borrado de Loki queda en "received" para siempre.** El compactor no tiene `retention_enabled: true` o el `deletion_mode` sigue en `disabled`. Los logs de Loki lo dicen: `delete requests are disabled`. Se corrige la configuración, se reinicia Loki y la petición se procesa en el siguiente ciclo.

**`count({env="pre"})` sigue devolviendo series después de quitar los targets.** Es lo esperado: quitar el target detiene el scrape, no borra el TSDB. O esperáis la retención o usáis `delete_series` más `clean_tombstones`. Si `delete_series` responde 405, Prometheus no arrancó con `--web.enable-admin-api`.

**El registry responde 405 al DELETE del manifiesto.** No arrancó con `REGISTRY_STORAGE_DELETE_ENABLED=true`. Se añade la variable al compose del registry y se reinicia. Y si responde 404 con un digest que acabáis de leer, es que pedisteis el manifiesto sin la cabecera `Accept` correcta y el registry os devolvió el digest de un manifiesto convertido, no del real.

**Después de destruir la VM, `dig` sigue resolviendo `app01.pre.lab`.** El registro estaba en un host override de Unbound puesto a mano en la 5166, no en el dnsmasq del SDN. El inventario con grep no lo encontró porque la configuración de OPNsense no está en Git. Moraleja para el plan: incluir OPNsense en el inventario aunque cueste hacerlo a mano.

**`restic snapshots` sigue funcionando después de borrar `keys/`.** La caché local de restic (`~/.cache/restic/`) todavía tiene el índice, pero no la clave; funciona solo hasta que necesita leer un pack. O bien había otra copia de la clave: `restic key list` antes de borrar os habría dicho cuántas claves había. Volved a listar el prefijo `keys/` con `--versions`: casi siempre queda una versión antigua sin borrar.

**`VACUUM FULL` falla por falta de espacio.** Necesita el doble del tamaño de la tabla porque escribe la copia entera antes de borrar la original. Con `df -h` se ve. `pg_repack` tiene el mismo requisito. Solución en el laboratorio: borrar primero las tablas grandes con `DROP` y hacer `VACUUM FULL` solo en las que se conservan.

Los enlaces para ampliar y los apartados que van más allá de lo que se hace en clase están en [Para ampliar](../ampliacion.md#ut8-terminacion-segura-del-contenedor).
