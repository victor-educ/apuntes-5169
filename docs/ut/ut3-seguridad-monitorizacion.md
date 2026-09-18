# UT3 · Seguridad de las comunicaciones de monitorización

<p class="ut-meta">Módulo 5169 · 8 h · Sesiones 16 a 19 · RA1 CE f, g</p>

En la UT1 conectasteis el servicio del curso a la pila de mon01 (node_exporter, cAdvisor, postgres_exporter, el `/metrics` de la API, Promtail y Loki) y en la UT2 le pusisteis alarmas encima. Todo eso funciona, pero funciona en claro: cada exporter es un servidor HTTP sin autenticación, Promtail manda los logs a Loki por HTTP y Grafana está en el 3000 con la contraseña que le pusisteis el primer día. En esta unidad damos la vuelta a la pila y la miramos como la miraría alguien que ha entrado en la red: qué puertos hay abiertos, quién puede llegar a ellos y qué se lleva si llega. Después cerramos lo que sobra por capas (red Docker, firewall del host, OPNsense), ciframos y autenticamos lo que queda, y lo documentamos como una política que se pueda auditar. Lo que se hace aquí se apoya directamente en la [UT3 de la asignatura de despliegue](https://victor-educ.github.io/apuntes-5166/ut/ut3-seguridad-por-capas/): mismas zonas, mismo OPNsense, mismas herramientas (nmap, tcpdump, matriz de pruebas). En la UT4 volveremos a la explotación (KPI y pruebas) sobre una pila que ya no filtra nada.

## Qué tienes que saber hacer al terminar

- Inventariar lo que escucha en cada host (`ss`, `docker ps`) y contrastarlo desde fuera con nmap y tcpdump, dejando evidencias con fecha (CE f).
- Rellenar una matriz de exposición origen/destino/puerto/esperado/obtenido y justificar cada puerto que queda abierto (CE f).
- Explicar por qué un puerto publicado por Docker se salta el firewall del host y resolverlo de tres maneras distintas (CE g).
- Escribir un fichero nftables persistente por host y la regla equivalente en OPNsense (CE g).
- Activar TLS y basic auth en los exporters, mTLS entre Promtail y Loki y TLS con roles en Grafana, y demostrar con curl, openssl y tcpdump que el tráfico va cifrado (CE g).
- Redactar secretos en los logs, fijar retención y permisos, y escribir el documento de política de protección entre el contenedor y la monitorización (CE g).

## Antes de entrar en detalle

Imagina que un compañero de otro grupo, desde su VM en la zona front, lanza un `curl` contra app01 por el puerto 9100 y le vuelve la lista completa de lo que corre en tu máquina: versión del kernel, servicios activos, IPs, hasta la hora del último reinicio. Nadie se entera, porque nada lo registra. Ahora cambia "compañero" por alguien que ha entrado en la red a través del proxy de la DMZ, y añade que puede leer todos los logs en Loki, silenciar las alarmas de Alertmanager y entrar en Grafana con la contraseña del primer día. Ese es el problema de la unidad: la pila que montaste para vigilar el servicio se ha convertido en la puerta más fácil para atacarlo. Lo que queremos al terminar cabe en una frase: que a la monitorización solo llegue quien tiene que llegar, que todo lo que viaje por la red vaya cifrado y con credenciales, y que puedas demostrarlo con pruebas fechadas.

| Herramienta o concepto | Qué es, en una frase | Para qué la usamos en esta unidad |
|------------------------|----------------------|-----------------------------------|
| `ss` y `docker ps` | Comandos que listan desde dentro del host qué puertos escuchan y cuáles publicó Docker | Inventario inicial: saber qué cree el host que expone |
| nmap | Un escáner de puertos: pregunta a una máquina remota, puerto a puerto, si alguien responde | Comprobar desde cada zona qué se ve de verdad, antes y después de cerrar |
| tcpdump | Un grabador de tráfico de red: muestra los paquetes que pasan por una interfaz | Ver quién usa cada puerto y demostrar que el tráfico va cifrado |
| nftables | El firewall del kernel Linux (sucesor de iptables), con reglas escritas en un fichero de texto | Cerrar en cada host todo lo que no justifique la matriz |
| Docker y su NAT | Docker publica puertos reescribiendo el destino de los paquetes en el kernel, no abriendo un socket normal | Entender por qué un puerto publicado ignora las reglas del firewall y cómo evitarlo |
| OPNsense | El cortafuegos central de la VPC que montaste en la 5166, con reglas por interfaz | Segunda capa: bloquear entre zonas lo que los hosts ya bloquean |
| TLS y la CA del curso | TLS es el cifrado de HTTPS; la CA es la autoridad que firma certificados en la que confían todos | Cifrar el tráfico de métricas y logs con certificados propios, sin comprar ninguno |
| Basic auth y bcrypt | Usuario y contraseña en cada petición HTTP; bcrypt es un hash lento pensado para guardar contraseñas | Que los exporters solo respondan a Prometheus |
| mTLS | TLS mutuo: también el cliente presenta certificado y el servidor lo valida | Que Loki solo acepte logs de los Promtail legítimos |
| nginx como proxy inverso | Un servidor web puesto delante de otro para aportarle lo que no tiene (TLS, autenticación) | Proteger Grafana, cAdvisor y el `/metrics` de la API sin tocarlos |
| Etapa `replace` de Promtail | Un filtro que reescribe cada línea de log antes de enviarla | Borrar contraseñas y tokens de los logs antes de que lleguen a Loki |

Cómo está organizada la unidad: sigue el orden de un trabajo real de seguridad, primero se mide, luego se cierra, luego se cifra y al final se documenta. Empieza por la superficie de exposición, para que sepas qué se filtra y por qué importa. Sigue la auditoría, con la matriz de exposición como resultado, porque la práctica pide el antes y el después. Antes de tocar el firewall hay un apartado sobre Docker y NAT, porque sin él las reglas que escribas parecerán no funcionar. Después vienen las dos capas de red (nftables por host y OPNsense), el cifrado y la autenticación (certificados, exporters, Loki, Grafana), los datos en sí (redacción, retención, permisos) y la verificación. Cierra la política de protección, que resume las reglas en un documento que un auditor pueda comprobar.

!!! info "Lo que necesitas de la otra asignatura"
    Esta unidad (24 nov a 3 dic) va en paralelo con la [UT3 de Despliegue, seguridad por capas con OPNsense](https://victor-educ.github.io/apuntes-5166/ut/ut3-seguridad-por-capas/) (18 nov a 4 dic): nmap, tcpdump, nftables, la CA del curso y las reglas de OPNsense se explican allí desde cero esa misma quincena, y aquí se dan por conocidos y se aplican a los puertos de la monitorización.
    Hasta ahora app01 y mon01 vivían en el entorno provisional del bridge del aula (vmbr0). Con la UT2 de la 5166 terminada (13 nov) ya existe la VPC dev, y durante su UT3 se le pone el cortafuegos: esta unidad es el momento de mover las VM a la VPC, detrás de OPNsense, y todas las IP 10.10.x.x de los ejemplos suponen que ya están allí.
    La matriz de reglas del cortafuegos que hiciste en la 5166 es la que aquí se amplía con los puertos de los exporters y de Loki: no empieces una nueva.

## Plan de sesiones

Cada sesión de dos horas empieza con una explicación corta y sigue con laboratorio. La columna "Se explica" es lo que cuento yo al principio (con su duración aproximada); la columna "Se practica" es lo que hacéis vosotros con el material de práctica de esta unidad. Las sesiones marcadas solo como práctica no traen teoría nueva.

| Sesión | Fecha | Tipo | Se explica | Se practica |
|---:|-------|------|------------|-------------|
| [16](#a31-auditoria-inicial-sesion-16) | 24 nov | Teoría y práctica | Superficie de exposición de la monitorización; ss, nmap y tcpdump aplicados a exporters (20 min). | Inventario de puertos en app01, db01 y mon01, escaneo desde otras subredes, matriz de exposición con lo que no debería verse. |
| [17](#a32-red-y-firewall-sesion-17) | 26 nov | Teoría y práctica | Por qué un puerto publicado en Docker salta el firewall del host y cómo se corrige; red dedicada y nftables (20 min). | Mover exporters a la red monitoring o a la IP de gestión, reglas nftables por host y en OPNsense; repetir el escaneo. Trasladar app01 y mon01 a la VPC dev. |
| [18](#a33-tls-y-autenticacion-sesion-18) | 1 dic | Teoría y práctica | web.config.file en exporters, bcrypt, mTLS entre Promtail y Loki (20 min). | Certificados de la CA del curso, TLS y basic auth en exporters, mTLS Promtail-Loki; Prometheus sigue en UP y curl sin certificado falla. |
| [19](#practica-evaluable-ut3-sesion-19) | 3 dic | Práctica evaluable | Aclaración del enunciado (10 min). | Cerrar la matriz de puertos antes y después, reglas, configuración TLS, evidencias y el documento de política. |

## Superficie de exposición de la monitorización

*Se explica en la sesión 16 (unos 10 min). El resto del apartado es material de consulta para la práctica.*

Cada exporter es un servidor HTTP más en cada máquina. Cuando montasteis la pila en la UT1 el objetivo era que Prometheus llegase a todo (el scrape: la lectura periódica de cada `/metrics`), y la forma rápida de conseguirlo fue publicar puertos en el host: `9100:9100`, `8080:8080`, `9187:9187`. El resultado es que la monitorización abre más puertos que la propia aplicación. El servicio del curso, bien desplegado, expone un único 443 en web01; la monitorización, sin control, expone cuatro puertos en app01, uno en db01 y cinco en mon01, ninguno con contraseña.

<figure markdown="span">
  ![Arquitectura de Prometheus](../img/prometheus-arquitectura.svg){ width="640" }
  <figcaption>Arquitectura de Prometheus: cada flecha de scrape es una conexión HTTP que hay que proteger. Fuente: Proyecto Prometheus, Apache 2.0.</figcaption>
</figure>

### Qué se ve en un /metrics abierto

La idea de que las métricas son "solo números" no aguanta un `curl`. Esto es lo que devuelve un node_exporter recién instalado a cualquiera que llegue al 9100:

```text
node_uname_info{domainname="(none)",machine="x86_64",nodename="app01",release="6.12.22-amd64",sysname="Linux",version="#1 SMP PREEMPT_DYNAMIC Debian 6.12.22-1"} 1
node_os_info{id="debian",name="Debian GNU/Linux",version_id="13"} 1
node_network_address_info{address="10.10.2.10",device="ens18",netmask="24"} 1
node_filesystem_size_bytes{device="/dev/sda1",mountpoint="/var/lib/docker"} 2.1e+10
node_systemd_unit_state{name="ssh.service",state="active"} 1
node_systemd_unit_state{name="postgresql.service",state="inactive"} 1
node_boot_time_seconds 1.763979e+09
```

Con eso un atacante sabe la versión exacta del kernel (y por tanto qué CVE, vulnerabilidades publicadas con identificador, aplican), el nombre y la IP de la máquina, qué servicios están corriendo y cuándo se reinició por última vez (una máquina con 400 días de uptime no se parchea). cAdvisor es peor: `container_last_seen` lleva las etiquetas `image` y `name` de cada contenedor, así que se ve `registry.dev.lab/servicio/api:1.4.2`, y cualquier etiqueta del contenedor aparece como `container_label_*`. Si alguien puso una URL con credenciales en una etiqueta del compose, está ahí. postgres_exporter publica los nombres de las bases de datos, el número de conexiones por usuario y, si se activó la colección de `pg_settings`, la configuración completa del servidor. El `/metrics` de la API del curso enumera todas las rutas que existen (`http_requests_total{path="/admin/export"}`), incluidas las que no están enlazadas desde ningún sitio.

Los propios servidores de la pila filtran todavía más. La API de Prometheus responde en `/api/v1/targets` con la lista de todo lo que se monitoriza (un mapa de la infraestructura, con IPs y puertos), y `/api/v1/status/config` devuelve la configuración cargada. Prometheus enmascara los campos marcados como secreto (`password: <secret>`), pero no puede saber que la URL de un `remote_write` o un `relabel_config` (dos opciones de configuración de Prometheus) lleva un token dentro. Si Prometheus arranca con `--web.enable-admin-api`, cualquiera puede borrar series con un `POST` a `/api/v1/admin/tsdb/delete_series`, y con `--web.enable-lifecycle` puede pararlo con un `POST` a `/-/quit`. Alertmanager acepta silencios sin autenticación por su API: un atacante silencia la alarma y después ataca. Loki sin autenticación es el caso más grave: cualquiera lee todos los logs por `/loki/api/v1/query_range` (y en los logs de la API están las peticiones completas, con IPs de clientes y a veces con parámetros que nadie debería haber registrado) y cualquiera inyecta líneas falsas por `/loki/api/v1/push`, lo que sirve para envenenar una investigación o disparar alarmas falsas hasta que el equipo deje de mirarlas.

### Casos reales

Esto no es teórico. En diciembre de 2024 el equipo de investigación de Aqua Security publicó un recuento hecho con Shodan y Censys (buscadores que indexan máquinas conectadas a Internet): alrededor de 296.000 instancias de node_exporter y 40.000 servidores Prometheus accesibles desde Internet sin ninguna autenticación, muchos de ellos con `/api/v1/status/config` mostrando credenciales de servicios en URLs y con la API de administración activa. Ya en 2021 JFrog había hecho un análisis parecido y había encontrado servidores Prometheus expuestos de empresas grandes cuya configuración incluía contraseñas de bases de datos y claves de proveedores de nube. Con Grafana el caso más conocido es la vulnerabilidad CVE-2021-43798 (versiones 8.0 a 8.3): un recorrido de directorios por la ruta de los plugins, sin autenticación, que permitía leer cualquier fichero del contenedor, incluido `grafana.db` con las credenciales cifradas de todos los datasources (las fuentes de datos conectadas), y el patrón de ataque era buscar Grafanas expuestos con Shodan y recorrer la lista. Y el error más repetido de todos: Grafana con `admin/admin` de fábrica, alcanzable desde fuera, sin cambiar. En un laboratorio pequeño uno piensa que a nadie le importa; en la empresa donde vais a hacer prácticas la pila de monitorización suele ser el servidor más viejo, con menos parches y con más información sobre el resto.

### Puertos del entorno del curso y quién debe llegar

| Componente | Puerto | Dónde corre | Quién debe llegar |
|------------|-------:|-------------|-------------------|
| node_exporter | 9100 | app01, db01, mon01 | Solo mon01 |
| cAdvisor | 8080 | app01 | Solo mon01 |
| postgres_exporter | 9187 | db01 | Solo mon01 |
| /metrics de la app | 9102 | app01 | Solo mon01 |
| Promtail | 9080 | app01, db01 | Nadie (solo inicia salida hacia Loki) |
| Loki | 3100 | mon01 | Promtail de cada host, Grafana |
| Prometheus | 9090 | mon01 | Grafana, administradores |
| Alertmanager | 9093 | mon01 | Prometheus, administradores |
| Grafana | 3000 (443 tras proxy) | mon01 | Usuarios |

Todo lo demás, cerrado. Y de estos, los administradores entran por la red de gestión (10.10.0.0/24) con usuario nominal, nunca desde front ni desde back. Fíjate en que los tres servidores de mon01 (Prometheus, Alertmanager, Loki) hablan entre sí dentro de la misma máquina: no hay ningún motivo para que sus puertos se publiquen en la IP 10.10.0.20 salvo el de Grafana, y este detrás de nginx.

## Auditar: saber qué hay antes de tocar nada

*Se explica en la sesión 16 (unos 10 min). El resto del apartado es material de consulta para la práctica.*

La auditoría es la parte del criterio f y se hace antes de proteger nada, porque la práctica evaluable pide la matriz antes y después. El procedimiento tiene cinco pasos y se repite en app01, db01 y mon01.

### Inventario desde dentro

Empieza en el propio host. `ss -tlnup` lista los sockets TCP y UDP en escucha con el proceso que los tiene abiertos (necesita root para ver el proceso):

```bash
sudo ss -tlnup
# Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
# tcp   LISTEN 0      4096         0.0.0.0:9100       0.0.0.0:*     users:(("node_exporter",pid=812,fd=3))
# tcp   LISTEN 0      4096         0.0.0.0:8080       0.0.0.0:*     users:(("docker-proxy",pid=1544,fd=4))
# tcp   LISTEN 0      4096         0.0.0.0:9102       0.0.0.0:*     users:(("docker-proxy",pid=1590,fd=4))
# tcp   LISTEN 0      128        127.0.0.1:9080       0.0.0.0:*     users:(("promtail",pid=790,fd=7))
# tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*     users:(("sshd",pid=610,fd=3))
```

La columna de dirección local es la que importa: `0.0.0.0` significa todas las interfaces; `127.0.0.1` significa solo local (Promtail está bien); `10.10.0.11` significaría solo la interfaz de gestión. Los puertos que aparecen con `docker-proxy` son puertos publicados por Docker, y ahí hay una trampa que veremos en el apartado siguiente: Docker puede publicar un puerto sin que aparezca ningún proceso en `ss` si `userland-proxy` está desactivado, porque entonces la publicación se hace solo con NAT. Por eso el segundo comando es obligatorio:

```bash
docker ps --format '{{.Names}}\t{{.Ports}}'
# api          0.0.0.0:9102->9102/tcp, :::9102->9102/tcp
# cadvisor     0.0.0.0:8080->8080/tcp, :::8080->8080/tcp
# promtail     9080/tcp
```

`0.0.0.0:9102->9102/tcp` es un puerto abierto a todo el mundo, en IPv4 y en IPv6 (`:::9102`). `9080/tcp` sin flecha es un puerto que el contenedor expone (`EXPOSE` en su imagen) pero que no está publicado: solo es alcanzable desde la red Docker.

### Escaneo desde fuera

El inventario dice lo que el host cree que expone; el escaneo dice lo que de verdad se ve desde cada zona, que es lo que un atacante vería. Se hace desde tres orígenes como mínimo: mon01 (que debe ver los exporters), una máquina de front como web01 (que no debe verlos) y el puesto de administración de la 5166 (10.10.0.50). nmap tiene varios tipos de escaneo y conviene saber cuál usar:

- `-sS` (SYN scan, necesita root): envía un SYN (el primer paquete de una conexión TCP) y mira la respuesta sin completar la conexión. Es el rápido y el que no deja una conexión en el log de la aplicación.
- `-sT` (connect scan): completa el three-way handshake (SYN, SYN-ACK, ACK: la conexión entera). Es el único posible sin root y el que usarás desde un contenedor sin capacidades.
- `-sU`: UDP. Lento y ambiguo (sin respuesta puede ser open o filtered). Aquí solo interesa para comprobar que no hay nada en UDP.
- `-sV`: tras encontrar un puerto abierto, habla con él para identificar el servicio. Es el que te dirá si hay TLS (el cifrado de HTTPS) o no después de la sesión 18.

```bash
sudo nmap -sS -p- --reason -T4 10.10.2.10        # desde web01, todos los puertos TCP
sudo nmap -sS -p 22,8080,9080,9100,9102 --reason 10.10.2.10   # desde mon01, solo los esperados
```

La lectura de los estados es lo que se pide en la práctica y lo que la gente confunde:

| Estado nmap | Qué ha pasado en la red | Qué significa aquí |
|-------------|-------------------------|--------------------|
| `open` | Llegó un SYN-ACK | Hay un servicio escuchando y nada lo filtra |
| `closed` | Llegó un RST | El paquete llegó al host pero nadie escucha en ese puerto; un firewall con `reject` también da esto |
| `filtered` | No llegó nada, o llegó un ICMP unreachable | Un firewall con `drop` lo ha tirado por el camino (OPNsense o nftables) |

Antes de la unidad, desde web01 el 9100 de app01 sale `open`. Después de la sesión 17 tiene que salir `filtered` (si usas `drop`) y desde mon01 tiene que seguir `open`. Guarda la salida completa con `-oN escaneo-app01-desde-web01-$(date +%F).txt`: la fecha en el nombre y la cabecera con la hora que nmap pone son la evidencia. Con `-T4 -p-` el escaneo de un host de la VPC tarda entre 10 y 60 segundos si responde con RST y varios minutos si todo está filtrado, porque nmap tiene que esperar el timeout de cada puerto; es normal y también es una pista de que el firewall está haciendo `drop`.

### Protocolos y autenticación

El tercer paso comprueba qué protocolo habla cada puerto abierto y si pide credenciales. Con `curl -v` se ve todo el diálogo:

```bash
curl -sv http://10.10.2.10:9100/metrics 2>&1 | head -20
```

Si desde web01 esto devuelve `HTTP/1.1 200 OK` y a continuación las métricas, tienes las tres cosas mal a la vez: alcanzable desde donde no toca, en claro y sin autenticación. Apúntalo así, en tres columnas, porque las tres se arreglan en sitios distintos: el firewall, TLS y basic auth (usuario y contraseña en cada petición HTTP).

### Tráfico real con tcpdump

El escaneo dice qué puertos aceptan conexiones; tcpdump dice quién las está usando de verdad. En app01, mirando el tráfico de Promtail hacia Loki:

```bash
sudo tcpdump -i ens18 -nn port 3100 -c 20
# 10:42:17.301 IP 10.10.2.10.48122 > 10.10.0.20.3100: Flags [P.], seq ..., length 1834
# 10:42:17.303 IP 10.10.0.20.3100 > 10.10.2.10.48122: Flags [P.], seq ..., length 89
```

Lo que buscas: que el único destino sea 10.10.0.20 y que sea app01 quien inicia (puertos altos de origen, 3100 de destino). Y en mon01, `sudo tcpdump -i ens18 -nn 'dst port 9100 or dst port 8080 or dst port 9187 or dst port 9102'` debe mostrar solo salidas hacia los tres hosts cada 15 segundos (el `scrape_interval`), sin nada entrante. Si añades `-A` verás el contenido en ASCII: antes de la sesión 18 se leen las líneas `GET /metrics HTTP/1.1` y las métricas; después no se lee nada. Esa pareja de capturas, antes y después, es la evidencia de cifrado que pide la práctica. El nombre de la interfaz en las VM de Proxmox con virtio es `ens18`; compruébalo con `ip -br link`.

### La matriz de exposición

Todo lo anterior se registra en una matriz como la matriz de pruebas de la [UT3 de despliegue](https://victor-educ.github.io/apuntes-5166/ut/ut3-seguridad-por-capas/), pero orientada a puertos de monitorización. Una fila por combinación origen/destino/puerto que merezca la pena comprobar:

| Origen | Destino | Puerto | Esperado | Obtenido (antes) | Evidencia |
|--------|---------|-------:|----------|------------------|-----------|
| mon01 10.10.0.20 | app01 10.10.2.10 | 9100 | open | open | escaneo-app01-desde-mon01-2026-11-24.txt |
| web01 10.10.1.10 | app01 10.10.2.10 | 9100 | filtered | open | escaneo-app01-desde-web01-2026-11-24.txt |
| web01 10.10.1.10 | app01 10.10.2.10 | 8080 | filtered | open | ídem |
| app01 10.10.2.10 | mon01 10.10.0.20 | 3100 | open | open | tcpdump-app01-3100-2026-11-24.txt |
| web01 10.10.1.10 | mon01 10.10.0.20 | 3100 | filtered | open | escaneo-mon01-desde-web01-2026-11-24.txt |
| web01 10.10.1.10 | mon01 10.10.0.20 | 9090 | filtered | open | ídem |
| admin 10.10.0.50 | mon01 10.10.0.20 | 443 | open | closed (aún no hay proxy) | ídem |
| otra VPC | mon01 10.10.0.20 | 3000 | filtered | open | escaneo-mon01-desde-pre-2026-11-24.txt |

Las filas donde esperado y obtenido no coinciden se marcan en rojo y son la lista de trabajo de las sesiones 17 y 18. La columna "obtenido (después)" se añade al final de la unidad con la fecha nueva.

```mermaid
flowchart LR
    subgraph antes[Antes: quién llega a app01]
        W1[web01 front] -->|9100 8080 9102 open| A1[app01]
        M1[mon01] -->|9100 8080 9102 open| A1
        P1[VPC pre] -->|9100 8080 9102 open| A1
        I1[Internet vía proxy comprometido] -->|9100 8080 9102 open| A1
    end
    subgraph despues[Después]
        M2[mon01 10.10.0.20] -->|9100 8080 9102 TLS y basic auth| A2[app01]
        W2[web01 front] -.->|filtered| A2
        P2[VPC pre] -.->|filtered| A2
        I2[Internet] -.->|filtered| A2
    end
```

## Por qué Docker se salta el firewall del host

*Se explica en la sesión 17 (unos 10 min). El resto del apartado es material de consulta para la práctica.*

Este es el punto donde cae casi todo el mundo, y merece entender el mecanismo porque os lo vais a encontrar en cualquier empresa que tenga Docker en máquinas con firewall propio. Alguien pone reglas en nftables (el firewall del kernel Linux) o en ufw (un frontal simplificado de iptables) para cerrar el 8080, comprueba con `nft list ruleset` que están, y desde otra máquina el 8080 sigue abierto. La regla no está mal: es que el paquete nunca pasa por ella.

Para seguir el mecanismo hay que conocer tres piezas de Netfilter, el filtro de paquetes del kernel: las tablas (`nat` para reescribir direcciones, `filter` para aceptar o tirar), las cadenas por las que pasa un paquete según su camino (`PREROUTING` al entrar, `INPUT` si va al propio host, `FORWARD` si el host lo reenvía) y DNAT, la reescritura de la dirección de destino.

Cuando publicas un puerto con `-p 8080:8080`, Docker no abre un socket en el host y reenvía (eso solo lo hace `docker-proxy` como apoyo para el tráfico local). Lo que hace es escribir reglas en las tablas del kernel: una regla DNAT en la cadena `PREROUTING` de la tabla `nat` que cambia el destino `10.10.2.10:8080` por `172.18.0.3:8080` (la IP del contenedor en su red bridge), y reglas en la cadena `FORWARD` de la tabla `filter` que aceptan ese tráfico hacia el bridge. Un paquete que llega de fuera con destino al contenedor entra por `PREROUTING`, se le cambia el destino, y como el nuevo destino no es una IP local del host, el kernel lo enruta: pasa por `FORWARD`, no por `INPUT`. Tus reglas de `INPUT` (que es donde todo el mundo pone las reglas de "este host solo acepta X") no lo ven jamás. Con `-p 9100:9100` de un node_exporter en contenedor pasa exactamente lo mismo.

```mermaid
flowchart LR
    IN[Paquete a 10.10.2.10:8080] --> PRE[nat PREROUTING<br>DNAT Docker: a 172.18.0.3:8080]
    PRE --> DEC{destino local?}
    DEC -->|no, es 172.18.0.3| FWD[filter FORWARD]
    DEC -->|sí| INPUT[filter INPUT<br>tus reglas de host]
    FWD --> DU[DOCKER-USER<br>tus reglas]
    DU --> DK[DOCKER<br>reglas de Docker: accept]
    DK --> CT[Contenedor]
```

Docker sabe que esto es un problema y por eso crea la cadena `DOCKER-USER` al principio de `FORWARD`: es la única cadena que Docker promete no tocar, y todo el tráfico hacia contenedores pasa por ella antes que por las reglas de Docker. En Debian 13 con Docker Engine 28 las reglas de Docker se escriben con la API de iptables, que por debajo es `iptables-nft`, así que conviven con tu ruleset de nftables en tablas distintas (`ip filter`, `ip nat` de Docker frente a tu `inet fw`). Las versiones recientes de Docker Engine 28 traen además un backend nftables nativo experimental; en clase usamos el de iptables, que es el que viene activo por defecto. Un matiz de nftables que aquí importa: cuando hay varias tablas con cadenas base en el mismo hook, el kernel las evalúa todas. Un `accept` en la cadena de Docker no salva al paquete de un `drop` en la tuya; un `drop` en cualquiera es definitivo. Por eso una cadena `forward` en tu tabla con reglas de `drop` explícitas funciona aunque Docker acepte el tráfico en la suya.

Hay tres soluciones, y en el laboratorio usaremos las tres según el caso:

1. **Publicar en una IP concreta.** `-p 10.10.0.11:9100:9100` (o `"10.10.0.11:9100:9100"` en el compose) hace que la regla DNAT solo se aplique a paquetes cuyo destino original es esa IP. Si app01 tiene una interfaz en la red de gestión con la 10.10.0.11 y Prometheus la usa como target, nadie desde front ni desde back llega al puerto, porque en esas interfaces no hay DNAT. Es la solución mínima y vale para los exporters que corren como servicio del sistema (node_exporter con `--web.listen-address=10.10.0.11:9100`), que no tienen nada que ver con Docker pero sufren el mismo problema de "escuchar en todas partes". Se puede fijar como valor por defecto en `/etc/docker/daemon.json` con `"ip": "10.10.0.11"`, y entonces cualquier `-p 8080:8080` que a alguien se le olvide restringir se publica en la IP de gestión y no en `0.0.0.0`.
2. **No publicar: red interna.** Si Prometheus corre en un contenedor de la misma máquina, no hace falta publicar nada: los dos contenedores comparten una red Docker y se hablan por nombre. Es lo que haremos en mon01 con Prometheus, Alertmanager, Loki y Grafana. Para app01, donde Prometheus está en otra máquina, la red Docker no cruza hosts, así que se combina con la solución 1 o la 3 (apartado siguiente).
3. **Reglas en DOCKER-USER o en la cadena forward de nftables.** Es la solución que protege aunque alguien publique mal un puerto, y por eso es la que exige la política. Con iptables:

```bash
iptables -I DOCKER-USER -i ens18 -p tcp -m conntrack --ctorigdstport 8080 ! -s 10.10.0.20 -j DROP
```

El `--ctorigdstport` es necesario porque en `DOCKER-USER` el paquete ya ha pasado por el DNAT y su puerto de destino es el del contenedor, que no siempre coincide con el publicado; conntrack (el seguimiento de conexiones del kernel) recuerda el destino original. Con nftables la misma idea va en la cadena `forward` de tu tabla y la veremos en el fichero completo más abajo.

!!! warning "ufw y Docker"
    `ufw` es un frontal de iptables que solo escribe en `INPUT`, así que con Docker no sirve para nada respecto a los puertos publicados, y hay años de hilos en foros de gente sorprendida. Si en la empresa os encontráis ufw en una máquina con Docker, asumid que los puertos publicados están abiertos y comprobadlo con nmap desde fuera.

## Reducir: red Docker dedicada y firewall por host

*Se explica en la sesión 17 (unos 10 min). El resto del apartado es material de consulta para la práctica.*

Cerramos puertos. El objetivo es que, al terminar, un escaneo desde cualquier sitio que no sea mon01 devuelva `filtered` para todos los puertos de la monitorización, y que ese resultado se deba a dos capas independientes: la forma de publicar los puertos y el firewall de cada host, por un lado, y OPNsense en el centro de la VPC, por otro. Si una de las dos falla o alguien la desconfigura, la otra sigue cerrando.

<figure markdown="span">
  ![DMZ con un cortafuegos](../img/dmz-un-firewall.svg){ width="560" }
  <figcaption>La monitorización cruza zonas (gestión hacia back y data) y por eso necesita regla en el cortafuegos central, además de en cada host. Fuente: Pbroks13, dominio público, vía Wikimedia Commons.</figcaption>
</figure>

### La red monitoring en cada host

En app01 el compose del servicio gana una red más. Los exporters que van en contenedor (cAdvisor y el propio contenedor de la API con su 9102) dejan de publicar puertos en `0.0.0.0` y se conectan a una red `monitoring`; lo que se publica se publica solo en la IP por la que llega mon01:

```yaml
services:
  api:
    image: registry.dev.lab/servicio/api:1.4.2
    networks: [backend, monitoring]
    ports:
      - "10.10.0.11:9102:9102"
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.52.1
    networks: [monitoring]
    ports:
      - "10.10.0.11:8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

networks:
  backend:
  monitoring:
    internal: false
```

Prometheus llega desde mon01 a `10.10.0.11:8080` y `10.10.0.11:9102`; desde la red back o desde front esos puertos no existen. La red `monitoring` aquí sirve para dos cosas: separar los exporters de la red `backend` de la aplicación (cAdvisor no tiene por qué poder hablar con la base de datos) y preparar el terreno para el sidecar TLS de cAdvisor (un contenedor auxiliar que se pone al lado del servicio para darle lo que le falta) que veremos después. Si en un host futuro Prometheus corriese en la misma máquina, se marcaría `internal: true` y no se publicaría nada.

En mon01 la pila entera va en una red interna y solo Grafana (detrás de nginx) publica el 443. Prometheus habla con `alertmanager:9093`, Grafana con `prometheus:9090` y `loki:3100`, todo por nombre dentro de la red, y Loki publica el 3100 únicamente en `10.10.0.20` para que lleguen los Promtail de app01 y db01. Al hacer `docker ps` en mon01 después de este cambio solo deben aparecer dos flechas: `10.10.0.20:443->443` y `10.10.0.20:3100->3100`.

### nftables por host: el fichero completo

Cada host lleva su propio firewall aunque OPNsense ya filtre entre zonas, porque OPNsense no ve el tráfico dentro de una misma subred (una máquina comprometida en back llegaría a app01 sin pasar por él) y porque dos capas de fabricantes distintos son lo que pide la defensa en profundidad. El fichero va en `/etc/nftables.conf`, que es el que carga `nftables.service` en Debian, y se activa con `systemctl enable --now nftables`. Este es el de app01 completo. Fíjate en las dos cadenas: `input` es para lo que escucha el propio host (node_exporter, SSH) y `forward` para los puertos que publica Docker, que no pasan por `input`:

```text
#!/usr/sbin/nft -f
flush table inet fw

table inet fw {
    set mon_ports {
        type inet_service
        elements = { 9100, 8080, 9102 }
    }

    chain input {
        type filter hook input priority filter; policy drop;

        iif lo accept
        ct state established,related accept
        ct state invalid drop
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept

        # gestión: SSH solo desde la red de gestión
        ip saddr 10.10.0.0/24 tcp dport 22 accept

        # servicio: la API la consume web01 a través del cortafuegos
        ip saddr 10.10.1.10 tcp dport 8080 accept

        # monitorización: exporters como servicio (node_exporter) solo desde mon01
        ip saddr 10.10.0.20 tcp dport @mon_ports accept
        tcp dport @mon_ports log prefix "mon-denegado: " drop
    }

    chain forward {
        type filter hook forward priority filter; policy accept;

        # puertos publicados por Docker: mismo criterio, aunque el DNAT ya haya ocurrido
        iifname "ens18" ip saddr != 10.10.0.20 ct original proto-dst @mon_ports log prefix "mon-docker-denegado: " drop
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

Varias decisiones que conviene entender. La cadena `input` tiene política `drop` y solo abre lo que la matriz justifica; la regla de `log ... drop` para los puertos de monitorización va explícita, aunque la política ya los tiraría, para que quede rastro en `journalctl -k` de quién lo intenta. La cadena `forward` no tiene política `drop`: si la tuviera, rompería el tráfico entre contenedores y la salida a Internet de los contenedores (que también pasa por `forward`), y tendríamos que reescribir todas las reglas de Docker a mano. En vez de eso se deja `accept` y se ponen `drop` explícitos para lo que nos importa, apoyándonos en que un `drop` en nuestra tabla es definitivo aunque la tabla de Docker acepte. El `ct original proto-dst` es el equivalente nftables del `--ctorigdstport`: el puerto de destino tal como llegó, antes del DNAT. `flush table inet fw` al principio hace el fichero idempotente (se puede recargar con `nft -f /etc/nftables.conf` sin duplicar reglas) y a la vez no toca las tablas de Docker, que es lo que pasaría con un `flush ruleset` completo: Docker no las regenera hasta que se reinicia el servicio, y os quedaríais con contenedores publicados sin NAT.

El fichero de db01 es el mismo cambiando el puerto de servicio (5432 desde 10.10.2.10) y el conjunto de puertos (`{ 9100, 9187 }`). El de mon01 abre 9100 desde sí mismo (Prometheus dentro de la máquina llega a node_exporter por la IP del host), 3100 desde app01 y db01, 443 desde la red de gestión y nada más. Después de cargar, la comprobación es la de siempre: `nft list ruleset`, y el escaneo desde web01 pasando de `open` a `filtered`.

### Segunda capa en OPNsense

Con los hosts cerrados, OPNsense debe decir lo mismo desde el centro. En la 5166 dejasteis el cortafuegos con política de denegación por defecto entre zonas y reglas explícitas para el servicio (front hacia back:8080, back hacia data:5432, gestión hacia todo por 22). La monitorización añade tres reglas, y se escriben con alias para que la matriz y el firewall usen los mismos nombres:

| Interfaz | Acción | Origen | Destino | Puertos | Log | Descripción |
|----------|--------|--------|---------|---------|-----|-------------|
| GESTION | pass | 10.10.0.20 (alias `mon01`) | 10.10.2.10 (alias `app01`) | alias `mon_ports_app`: 9100, 8080, 9102 | sí | scrape a app01 |
| GESTION | pass | `mon01` | 10.10.3.10 (`db01`) | `mon_ports_db`: 9100, 9187 | sí | scrape a db01 |
| BACK y DATA | pass | `app01`, `db01` | `mon01` | 3100 | sí | Promtail hacia Loki |

Las reglas se ponen en la interfaz por la que entra el tráfico al cortafuegos (la de gestión para los scrapes, back y data para Loki), que es como OPNsense evalúa. Cualquier otro origen hacia esos puertos cae en la denegación por defecto, que también registra. Con esto, un escaneo desde web01 hacia app01:9100 muestra `filtered` por dos motivos independientes, y en el registro en vivo de OPNsense (Firewall, Log Files, Live View) aparece el bloqueo con la regla que lo decidió. Guardad una captura de esa vista: es evidencia de la segunda capa. Si app01 tiene además la interfaz de gestión 10.10.0.11 y publicáis ahí los exporters, el scrape de mon01 no cruza OPNsense (misma subred) y la primera regla deja de tener tráfico, pero se mantiene por si algún día el exporter vuelve a la IP de back.

## Proteger: cifrado y autenticación

*Se explica en la sesión 18 (unos 20 min). El resto del apartado es material de consulta para la práctica.*

Con el firewall, alguien de front ya no llega a los exporters. Pero cualquiera con acceso a la red de gestión (un portátil de un administrador, una VM mal colocada, o la propia mon01 si la comprometen) sigue leyendo las métricas y los logs en claro. TLS resuelve la confidencialidad y la autenticación del servidor; basic auth o mTLS (TLS mutuo: también el cliente presenta certificado) resuelven quién puede pedir.

### Certificados con la CA del curso

La CA del curso es la que creasteis en la [UT3 de despliegue](https://victor-educ.github.io/apuntes-5166/ut/ut3-seguridad-por-capas/) con openssl (`Lab 5166 CA`, clave EC P-256, `ca.crt` y `ca.key`). Guardad `ca.key` en el puesto de administración, nunca en mon01 ni en los hosts. Por cada servicio que va a hablar TLS se emite un certificado de servidor con su nombre DNS en el SAN (Subject Alternative Name, la lista de nombres e IP para los que vale el certificado), porque Prometheus y Promtail validan el nombre, no el CN (el campo clásico de nombre del certificado):

```bash
# en el puesto de administración, para node_exporter de app01
openssl req -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
  -keyout app01-node.key -out app01-node.csr -subj "/CN=app01.dev.lab"
openssl x509 -req -in app01-node.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -days 365 -out app01-node.crt \
  -extfile <(printf "subjectAltName=DNS:app01.dev.lab,IP:10.10.0.11\nextendedKeyUsage=serverAuth")
```

Para los certificados de cliente de Promtail (uno por host) el `extendedKeyUsage` es `clientAuth` y el CN puede ser `promtail-app01`. Los ficheros se copian al host con `scp` y se dejan con propietario el usuario del servicio y permisos 600 en la clave. Un certificado de un año en un laboratorio está bien; en una empresa lo normal es una CA interna que emite por ACME (el protocolo automático que usa Let's Encrypt), por ejemplo step-ca, con certificados de días, y es lo que vais a ver cuando llegue la práctica.

### TLS y basic auth en los exporters

node_exporter, postgres_exporter y el resto de exporters oficiales usan la misma librería (`exporter-toolkit`) y admiten `--web.config.file`. El fichero tiene dos bloques, TLS de servidor y usuarios de basic auth:

```yaml
# /etc/node_exporter/web.yml
tls_server_config:
  cert_file: /etc/node_exporter/app01-node.crt
  key_file: /etc/node_exporter/app01-node.key
  min_version: TLS13
basic_auth_users:
  prometheus: "$2y$10$Wb3l0jwVfM1xC5eYq7p0Ee2q1HkzFq4t1g1hA5m9Xn3uQ2Xv3q5Ki"
```

La contraseña va en bcrypt (un hash lento hecho a propósito para contraseñas), no en claro. Se genera con `htpasswd` (paquete `apache2-utils`) o con Python si no queréis instalar nada:

```bash
htpasswd -nBC 10 prometheus          # pide la contraseña y escribe usuario:hash
python3 -c 'import bcrypt,getpass; print(bcrypt.hashpw(getpass.getpass().encode(), bcrypt.gensalt(10)).decode())'
```

El coste 10 es suficiente: el exporter comprueba el hash en cada scrape (cada 15 segundos), y un coste 14 haría que cada petición tardase medio segundo de CPU. El servicio arranca con `node_exporter --web.config.file=/etc/node_exporter/web.yml --web.listen-address=10.10.0.11:9100` y en Prometheus el job cambia de esquema y gana credenciales:

```yaml
scrape_configs:
  - job_name: node
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/ca.crt
      server_name: app01.dev.lab
    basic_auth:
      username: prometheus
      password_file: /etc/prometheus/secrets/node_exporter.pass
    static_configs:
      - targets: ["10.10.0.11:9100"]
        labels: { host: app01 }
```

`password_file` en lugar de `password` para que la contraseña no esté en un `prometheus.yml` que va a un repositorio de Gitea (el servidor Git del laboratorio); el fichero de secretos se monta en el contenedor de Prometheus y queda fuera del repo. `server_name` es necesario cuando el target es una IP y el certificado lleva el nombre DNS. Si el target del job tiene varios hosts con certificados distintos, cada uno debe llevar su IP en el SAN, que es lo que hace el `IP:10.10.0.11` de arriba, y entonces `server_name` sobra.

```mermaid
sequenceDiagram
    participant P as Prometheus (mon01)
    participant N as node_exporter (app01:9100)
    P->>N: ClientHello (TLS 1.3, SNI app01.dev.lab)
    N->>P: ServerHello + certificado firmado por Lab 5166 CA
    P->>P: valida cadena contra ca.crt y el nombre del SAN
    P->>N: GET /metrics + Authorization: Basic (cifrado)
    N->>N: bcrypt.compare(contraseña, hash) del usuario prometheus
    alt credenciales válidas
        N->>P: 200 OK + métricas (cifradas)
    else sin cabecera o hash no coincide
        N->>P: 401 Unauthorized
    end
```

cAdvisor no usa exporter-toolkit y no tiene TLS ni autenticación propias. Las opciones son dos: dejarlo sin publicar y confiar en el firewall (aceptable, y es lo que hace mucha gente), o ponerle delante un contenedor nginx en la misma red `monitoring` que termine TLS y pida basic auth, y publicar solo ese nginx. En clase haremos lo segundo para cAdvisor y para el 9102 de la API, con un único nginx sidecar que atiende dos `server` (uno por puerto) y reenvía a `cadvisor:8080` y `api:9102` por la red interna; el fichero de configuración es el mismo que el del proxy de Grafana con `auth_basic` añadido, así que no lo repito. El job de Prometheus para ellos es idéntico al de node_exporter. Si la API se hubiera instrumentado con una librería que sí soporta TLS (el cliente Python o el de Go lo permiten con unas líneas), se podría proteger el 9102 en la propia aplicación, pero la solución del sidecar tiene la ventaja de que no toca el código del servicio.

### mTLS entre Promtail y Loki

En el sentido de los logs la relación se invierte: son los hosts los que hablan con mon01, y Loki tiene que saber que quien le envía logs es un Promtail legítimo y no cualquiera con acceso al 3100. Basic auth valdría, pero Loki no la implementa de forma nativa (habría que ponerle nginx delante) y con TLS ya en marcha lo natural es autenticación mutua: Loki presenta su certificado de servidor y exige a cada cliente uno firmado por la misma CA. En el lado de Loki:

```yaml
# loki-config.yml (fragmento)
server:
  http_listen_port: 3100
  http_tls_config:
    cert_file: /etc/loki/tls/mon01-loki.crt
    key_file: /etc/loki/tls/mon01-loki.key
    client_auth_type: RequireAndVerifyClientCert
    client_ca_file: /etc/loki/tls/ca.crt
```

Y en cada Promtail, el cliente lleva su certificado:

```yaml
# promtail-config.yml (fragmento) en app01
clients:
  - url: https://mon01.dev.lab:3100/loki/api/v1/push
    tls_config:
      ca_file: /etc/promtail/tls/ca.crt
      cert_file: /etc/promtail/tls/promtail-app01.crt
      key_file: /etc/promtail/tls/promtail-app01.key
      server_name: mon01.dev.lab
```

Con `RequireAndVerifyClientCert`, un `curl https://mon01.dev.lab:3100/ready --cacert ca.crt` sin certificado de cliente ya no devuelve nada: el handshake termina con `alert certificate required` antes de que exista una petición HTTP. Eso tiene una consecuencia que hay que prever: Grafana también es cliente de Loki, así que su datasource necesita el certificado de cliente. En el provisioning de Grafana (los ficheros YAML con los que Grafana crea datasources y dashboards al arrancar) se declara con `jsonData: { tlsAuth: true, tlsAuthWithCACert: true }` y las claves en `secureJsonData` (`tlsCACert`, `tlsClientCert`, `tlsClientKey`), o bien se emite un certificado `grafana` con `clientAuth` y se monta. Lo mismo para cualquier `promtool` o `logcli` (las herramientas de línea de comandos de Prometheus y de Loki) que uséis desde el puesto: `logcli --ca-cert --cert --key`. Promtail está en modo mantenimiento desde Loki 3 y Grafana recomienda Alloy como sustituto; la configuración TLS de Alloy es equivalente (`loki.write` con bloque `tls_config`), así que lo que aprendéis aquí se traslada tal cual.

### Grafana detrás de nginx con TLS y roles

Grafana no debe escuchar directamente en 10.10.0.20:3000. En mon01 se añade un nginx al compose, en la red interna, que publica `10.10.0.20:443` y reenvía a `grafana:3000`:

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name mon01.dev.lab;
    ssl_certificate     /etc/nginx/tls/mon01.crt;
    ssl_certificate_key /etc/nginx/tls/mon01.key;
    ssl_protocols TLSv1.3;

    location / {
        proxy_pass http://grafana:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        # Grafana Live usa WebSocket
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Y en la configuración de Grafana (por variables de entorno en el compose, que en Grafana equivalen a las secciones de `grafana.ini`) se cierra lo que viene abierto de fábrica:

```yaml
environment:
  GF_SERVER_ROOT_URL: https://mon01.dev.lab/
  GF_SECURITY_ADMIN_USER: admin
  GF_SECURITY_ADMIN_PASSWORD__FILE: /run/secrets/grafana_admin
  GF_SECURITY_COOKIE_SECURE: "true"
  GF_AUTH_ANONYMOUS_ENABLED: "false"
  GF_USERS_ALLOW_SIGN_UP: "false"
  GF_USERS_AUTO_ASSIGN_ORG_ROLE: Viewer
  GF_AUTH_BASIC_ENABLED: "false"
```

Los roles de Grafana son cuatro por organización: Admin (todo), Editor (crea y modifica dashboards y alertas), Viewer (solo mira) y, desde Grafana 10, No basic role combinado con permisos por recurso. La regla de la política es sencilla: cada persona con su usuario nominal, Viewer por defecto, Editor para quien mantiene dashboards y un solo Admin (el de la cuenta de servicio de provisioning, no una persona). `auto_assign_org_role: Viewer` garantiza que un usuario nuevo no pueda tocar nada hasta que alguien le suba el rol. `auth.basic` desactivado evita que la API de Grafana acepte usuario y contraseña en cada petición (los tokens de cuenta de servicio son la forma correcta de automatizar, y se pueden revocar). Prometheus y Alertmanager no se publican: quien necesite su interfaz entra por SSH a mon01 con un túnel (`ssh -L 9090:localhost:9090 ops@mon01`) o lo mira desde Grafana, que para eso está el datasource.

## Datos: redacción, retención y permisos

*No se explica en clase: es material de consulta para las prácticas y para ampliar.*

Cifrar el canal no sirve de nada si el contenido ya está mal. Los logs de la API del curso pasaron a Loki en la UT1 tal como salían de la aplicación, y las aplicaciones registran cosas que no deberían: parámetros de la petición con contraseñas, cabeceras `Authorization`, tokens en URLs de callback. La primera línea de defensa es la aplicación: registrar de forma estructurada (JSON con campos elegidos), nunca volcar la petición entera, y tener una lista de campos que el logger sustituye antes de escribir. Como no siempre controlas el código, la segunda línea está en Promtail, que puede reescribir cada línea antes de enviarla:

```yaml
scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
    pipeline_stages:
      - replace:
          expression: '(?i)password=(\S+)'
          replace: '***'
      - replace:
          expression: 'Authorization: (?:Basic|Bearer) (\S+)'
          replace: '***'
      - replace:
          expression: '\b(\d{4})\d{8}(\d{4})\b'
          replace: '${1}********${2}'
```

En la etapa `replace`, si la expresión tiene grupos de captura, se sustituye el contenido de cada grupo (no la línea entera), de modo que `password=hunter2` queda como `password=***` y la clave del campo se conserva para poder seguir buscando. El tercer ejemplo enmascara el centro de un número de 16 dígitos (una tarjeta) dejando los cuatro primeros y los cuatro últimos. Se comprueba con `promtail --dry-run --config.file=...` sobre un fichero de muestra antes de desplegar, y después con una consulta en Grafana: `{service="api"} |= "password="` no debe devolver nada que no lleve asteriscos. La redacción en Promtail tiene un límite: actúa sobre lo que ya salió de la aplicación, así que ese secreto estuvo unos milisegundos en `journald` o en el fichero JSON del driver de Docker. Por eso la política pide las dos cosas.

La retención también es una decisión de seguridad: cuanto más tiempo guardas, más hay que robar y más cuesta cumplir una petición de borrado. En Loki se fija en `limits_config: retention_period: 168h` con el compactor activo (`compactor: { retention_enabled: true, delete_request_store: filesystem }`); sin el compactor, la retención no se aplica y el disco crece hasta llenarse. En Prometheus es `--storage.tsdb.retention.time=15d`. Y los volúmenes donde vive todo esto (`/var/lib/monitoring/prometheus`, `/var/lib/monitoring/loki`, `/var/lib/monitoring/grafana`) van con permisos 700 y propietario el UID con el que corre cada contenedor (65534 para Prometheus, 10001 para Loki, 472 para Grafana), de forma que un usuario sin privilegios de mon01 no pueda leer la base de datos de Grafana, que contiene las credenciales de los datasources, ni los chunks (los ficheros de datos) de Loki con los logs de todos.

## Verificar que de verdad está protegido

*No se explica en clase: es material de consulta para las prácticas y para ampliar.*

Lo que vale para la práctica no es el fichero de configuración sino la prueba de que hace lo que dice. Estas son las comprobaciones, con el resultado esperado:

```bash
# 1. sin CA: el certificado no se puede validar
curl https://10.10.0.11:9100/metrics
# curl: (60) SSL certificate problem: unable to get local issuer certificate

# 2. con CA pero sin credenciales
curl --cacert ca.crt https://app01.dev.lab:9100/metrics
# 401 Unauthorized

# 3. con CA y credenciales: las métricas
curl --cacert ca.crt -u prometheus https://app01.dev.lab:9100/metrics | head -3

# 4. el handshake visto por openssl: versión, cifrado, cadena y verificación
openssl s_client -connect 10.10.0.11:9100 -servername app01.dev.lab -CAfile ca.crt </dev/null
#   Protocol: TLSv1.3 / Cipher: TLS_AES_128_GCM_SHA256
#   Verify return code: 0 (ok)

# 5. mTLS en Loki: sin certificado de cliente no hay ni HTTP
curl --cacert ca.crt https://mon01.dev.lab:3100/ready
# curl: (56) ... alert certificate required
curl --cacert ca.crt --cert promtail-app01.crt --key promtail-app01.key https://mon01.dev.lab:3100/ready
# ready

# 6. Prometheus sigue viendo los targets UP
curl -s localhost:9090/api/v1/targets | jq -r '.data.activeTargets[] | "\(.labels.job) \(.scrapeUrl) \(.health)"'
```

La evidencia de cifrado en la red es tcpdump con `-A` sobre el puerto del exporter durante un scrape: antes se leía `GET /metrics HTTP/1.1` y las métricas; ahora se ven los bytes del ClientHello (`16 03 01` al principio del primer paquete de la conexión, TLS con versión de registro 1.0 que después negocia 1.3) y a partir de ahí datos que no significan nada. `nmap -sV -p 9100 10.10.0.11` desde mon01 lo resume en una línea: `9100/tcp open ssl/http Prometheus node_exporter`. Ese `ssl/` es la diferencia entre la matriz de antes y la de después.

## La política de protección entre contenedor y monitorización

*Se explica en la sesión 19 (unos 10 min). El resto del apartado es material de consulta para la práctica.*

El criterio g no pide solo configurar sino documentar. El documento de política que se entrega es corto (una o dos páginas), vive en el repositorio `monitoring` junto a la configuración, y dice lo que se cumple y cómo se comprueba. El del curso tiene cinco reglas:

1. **Los exporters solo son alcanzables desde mon01.** Publicados en la IP de gestión o en red interna, con regla en nftables del host y en OPNsense. Comprobación: escaneo desde front, back y otra VPC con resultado `filtered`, y desde mon01 `open`.
2. **Todo tráfico de monitorización va cifrado y autenticado.** TLS 1.3 con certificados de la CA del curso; basic auth en exporters; mTLS entre Promtail y Loki; Grafana tras nginx con TLS y usuarios nominales. Comprobación: curl sin CA, sin credenciales y sin certificado de cliente fallan; tcpdump no muestra HTTP en claro.
3. **Nada de la monitorización es alcanzable desde la DMZ externa.** web01 no tiene ninguna regla hacia mon01, y mon01 no publica más que 443 y 3100 en la red de gestión. Comprobación: escaneo desde web01 a mon01 con todos los puertos `filtered`.
4. **Los administradores acceden por la red de gestión, con usuario nominal.** Grafana sin acceso anónimo ni registro; Prometheus y Alertmanager solo por túnel SSH. Comprobación: lista de usuarios de Grafana con roles, `docker ps` de mon01.
5. **Cualquier puerto nuevo requiere actualizar la matriz y las reglas.** Un exporter nuevo entra por un cambio en el repositorio que toca a la vez el compose, `nftables.conf`, las reglas de OPNsense y la matriz, y se prueba con el mismo escaneo. Sin eso no se despliega.

Junto a cada regla van la fecha de la última comprobación y el nombre del fichero de evidencia. Es el mismo formato de procedimiento de cambios de la 5166, y es lo que un auditor externo (o el tutor de empresa) va a pedir en la primera reunión.

## Errores frecuentes en el laboratorio

**El puerto sigue abierto después de poner la regla en nftables.** Casi siempre es un puerto publicado por Docker y la regla está en `input`. Míralo con `docker ps` (si tiene flecha con `0.0.0.0`, es esto) y pon la regla en `forward` con `ct original proto-dst`, o publica en la IP concreta.

**Después de `nft flush ruleset` los contenedores no tienen red.** El `flush ruleset` ha borrado las tablas `ip nat` e `ip filter` de Docker. `systemctl restart docker` las regenera. En el fichero usa `flush table inet fw` y nunca `flush ruleset`.

**Prometheus marca el target como DOWN con `x509: certificate signed by unknown authority`.** Prometheus no encuentra `ca.crt` (ruta dentro del contenedor distinta de la del host) o el certificado se emitió con otra CA. `docker exec prometheus ls -l /etc/prometheus/ca.crt` y `openssl verify -CAfile ca.crt app01-node.crt`.

**`x509: certificate is valid for app01.dev.lab, not 10.10.0.11`.** El target es una IP y el certificado no la lleva en el SAN. Añade `server_name` al `tls_config` o reemite el certificado con `IP:10.10.0.11`.

**El target da `401 Unauthorized` con la contraseña correcta.** El hash bcrypt tiene un `$` que se ha comido el shell o YAML. En el `web.yml` va entre comillas dobles; si lo generaste con `echo` sin comillas simples, se han perdido caracteres. Regenera con `htpasswd -nBC 10` y pega el hash tal cual.

**Promtail registra `certificate required` o `bad certificate` contra Loki.** Falta el certificado de cliente o se emitió sin `extendedKeyUsage=clientAuth`. Verifica con `openssl x509 -in promtail-app01.crt -noout -ext extendedKeyUsage`.

**Grafana dice que Loki no responde después de activar mTLS.** El datasource no tiene certificado de cliente. Añádelo en el provisioning (`tlsAuth`) o emite uno para Grafana.

**Grafana redirige a `http://` o pierde la sesión al entrar por nginx.** Falta `GF_SERVER_ROOT_URL` con `https://` o la cabecera `X-Forwarded-Proto`. Sin ella Grafana cree que está en HTTP y las cookies marcadas `secure` no se envían.

**nmap desde web01 tarda diez minutos y no acaba.** Es lo esperado con todo filtrado: cada puerto espera su timeout. Escanea solo los puertos de la matriz (`-p 22,8080,9100,9102,3100,9090,3000`) para las evidencias y guarda un `-p-` solo una vez.

**Alerta `NodeExporterDown` de la UT2 saltando cada 15 segundos tras el cambio.** Cambiaste el exporter a TLS antes que el job de Prometheus, o al revés. Cambia las dos cosas en la misma ventana y recarga Prometheus con `docker compose kill -s HUP prometheus`; si tienes que hacerlo por partes, pon un silencio de 30 minutos en Alertmanager antes.

## Material de práctica

### A3.1 Auditoría inicial (sesión 16)

**Sesión 16 · 24 de noviembre · Teoría y práctica · unos 100 min de práctica**

**Objetivo.** Tener la matriz de exposición de la pila de monitorización tal como está hoy, con una evidencia fechada por fila y las filas que no deberían estar así marcadas en rojo.

**Antes de empezar.**

- app01, db01 y mon01 encendidas con la pila de la UT2 funcionando (Prometheus con todos los targets en UP).
- Acceso por SSH a las tres máquinas, a web01 y al puesto de administración (10.10.0.50), que hace de "otra VPC" si pre no está levantada.
- nmap y tcpdump instalados donde vayas a lanzarlos.
- Un directorio `monitoring/audit/antes/` en el repositorio `monitoring` para las salidas.
- Se ha explicado al principio de la sesión [qué se ve en un /metrics abierto](#superficie-de-exposicion-de-la-monitorizacion) y los [cinco pasos de la auditoría](#auditar-saber-que-hay-antes-de-tocar-nada); los estados de nmap se explican en la [UT3 de despliegue](https://victor-educ.github.io/apuntes-5166/ut/ut3-seguridad-por-capas/).

**Pasos.**

1. Inventario desde dentro, en cada uno de los tres hosts. Guarda las dos salidas en un fichero por host:

    ```bash
    H=$(hostname); F=inventario-$H-$(date +%F).txt
    { echo "== ss =="; sudo ss -tlnup; echo "== docker ps =="; docker ps --format '{{.Names}}\t{{.Ports}}'; } | tee $F
    ```

    Anota por cada puerto la dirección local (`0.0.0.0`, `127.0.0.1` o una IP concreta) y si lo publica Docker (`docker-proxy` o flecha en `docker ps`).

2. Escaneo desde mon01 hacia app01, db01 y el propio mon01, solo con los puertos esperados. Desde mon01 todo lo de la tabla de puertos debe salir `open`:

    ```bash
    sudo nmap -sS -p 22,8080,9080,9100,9102 --reason 10.10.2.10 -oN escaneo-app01-desde-mon01-$(date +%F).txt
    sudo nmap -sS -p 22,9080,9100,9187 --reason 10.10.3.10 -oN escaneo-db01-desde-mon01-$(date +%F).txt
    ```

3. Escaneo completo desde web01 (front) hacia app01 y mon01. Es el escaneo que hace de atacante; con `-p-` tarda entre 10 y 60 segundos mientras los puertos responden con RST:

    ```bash
    sudo nmap -sS -p- --reason -T4 10.10.2.10 -oN escaneo-app01-desde-web01-$(date +%F).txt
    sudo nmap -sS -p- --reason -T4 10.10.0.20 -oN escaneo-mon01-desde-web01-$(date +%F).txt
    ```

4. Repite el paso 3 desde pre o desde el puesto de administración, con `-desde-pre-` o `-desde-admin-` en el nombre. Sin root en el origen, usa `-sT`.

5. Protocolo y autenticación de cada puerto que salió `open` desde web01. Un `200 OK` con métricas es alcanzable, en claro y sin credenciales:

    ```bash
    for p in 9100 8080 9102; do echo "== $p =="; curl -sv http://10.10.2.10:$p/metrics 2>&1 | head -20; done | tee curl-app01-desde-web01-$(date +%F).txt
    curl -sv http://10.10.0.20:3100/ready 2>&1 | head -20 | tee curl-mon01-3100-desde-web01-$(date +%F).txt
    ```

6. Tráfico real del 3100 en app01. Comprueba antes el nombre de la interfaz con `ip -br link` (en Proxmox con virtio es `ens18`) y captura 20 paquetes de Promtail hacia Loki, con `-A` para que se lea el contenido en claro:

    ```bash
    sudo tcpdump -i ens18 -nn -A port 3100 -c 20 | tee tcpdump-app01-3100-$(date +%F).txt
    ```

7. Copia todos los ficheros al puesto (`scp usuario@host:'*-2026-11-24.txt' monitoring/audit/antes/`) y rellena `monitoring/audit/matriz.md` con las columnas origen, destino, puerto, esperado, obtenido (antes) y evidencia, como en [La matriz de exposición](#la-matriz-de-exposicion), con al menos las ocho filas del ejemplo. La columna "esperado" sale de la tabla [Puertos del entorno del curso](#puertos-del-entorno-del-curso-y-quien-debe-llegar).

8. Marca en rojo (negrita o una columna "estado") las filas donde esperado y obtenido no coinciden. Esa lista es el trabajo de las sesiones 17 y 18.

**Comprobación.** Un fichero de inventario por host, al menos cuatro escaneos con fecha en el nombre y en la cabecera de nmap, un fichero de curl y una captura de tcpdump donde se lee `POST /loki/api/v1/push` en claro. En la matriz, las filas con origen web01 hacia puertos de monitorización están en rojo (`open` donde se esperaba `filtered`) y las filas desde mon01 no.

**Entrega.** Nada que entregar todavía: `monitoring/audit/antes/` y `matriz.md` forman la parte "antes" de la práctica evaluable de la sesión 19. Haz commit en el repositorio `monitoring` antes de salir.

**Si te sobra tiempo.** Consulta `http://10.10.0.20:9090/api/v1/targets` y `/api/v1/status/config` desde web01 y anota qué se lee de la infraestructura sin ninguna credencial.

### A3.2 Red y firewall (sesión 17)

**Sesión 17 · 26 de noviembre · Teoría y práctica · unos 100 min de práctica**

**Objetivo.** Que un escaneo desde web01 hacia app01 y mon01 devuelva `filtered` en todos los puertos de monitorización, que desde mon01 sigan `open`, y que el cierre se deba a dos capas (nftables en cada host y OPNsense) que sobreviven a un reinicio.

**Antes de empezar.**

- La matriz "antes" de la A3.1 con sus filas en rojo: es la lista de trabajo.
- app01 y mon01 trasladadas a la VPC dev detrás de OPNsense, con su interfaz en la red de gestión (app01 10.10.0.11, db01 10.10.0.12, mon01 10.10.0.20). Si todavía están en vmbr0, el primer paso es moverlas, como en la UT3 de la 5166.
- Acceso a la interfaz web de OPNsense con la matriz de reglas que dejaste en la 5166.
- Se ha explicado al principio de la sesión [por qué Docker se salta el firewall del host](#por-que-docker-se-salta-el-firewall-del-host) y las [tres soluciones](#reducir-red-docker-dedicada-y-firewall-por-host). Ten a mano el [fichero nftables completo](#nftables-por-host-el-fichero-completo) y la [tabla de reglas de OPNsense](#segunda-capa-en-opnsense).

**Pasos.**

1. Si no lo has hecho ya, mueve app01, db01 y mon01 a la VPC dev y comprueba en mon01 que Prometheus ve los targets en UP con sus IP nuevas. No sigas hasta que todo esté en UP.

2. En app01, edita el compose del servicio: añade la red `monitoring`, conecta a ella `api` y `cadvisor`, y cambia sus `ports` para publicar solo en la IP de gestión, tal como aparece en [La red monitoring en cada host](#la-red-monitoring-en-cada-host):

    ```yaml
    ports:
      - "10.10.0.11:9102:9102"     # api
      - "10.10.0.11:8080:8080"     # cadvisor
    ```

    Aplica con `docker compose up -d` y comprueba con `docker ps --format '{{.Names}}\t{{.Ports}}'` que las flechas ya no empiezan por `0.0.0.0`.

3. En app01, db01 y mon01, edita la unidad de node_exporter (`systemctl edit node_exporter`) y añade a `ExecStart` `--web.listen-address=10.10.0.11:9100` (la IP de gestión de cada host). `systemctl daemon-reload && systemctl restart node_exporter`, y `ss -tlnp | grep 9100` debe mostrar esa IP, no `0.0.0.0`. En db01 haz lo mismo con postgres_exporter (`10.10.0.12:9187`).

4. En mon01, edita el compose de la pila: Prometheus y Alertmanager dejan de publicar puertos, Loki publica solo `10.10.0.20:3100:3100` y Grafana, de momento, `10.10.0.20:3000:3000` (el 443 llega en la sesión 18). `docker compose up -d` y `docker ps` debe mostrar solo esas dos flechas.

5. En Prometheus, cambia los targets de los jobs a las IP de gestión (`10.10.0.11:9100`, `10.10.0.11:8080`, `10.10.0.11:9102`, `10.10.0.12:9100`, `10.10.0.12:9187`) y recarga con `docker compose kill -s HUP prometheus`. Todos los targets deben volver a UP.

6. Escribe `/etc/nftables.conf` en app01 copiando el [fichero completo](#nftables-por-host-el-fichero-completo) del apartado (cadenas `input` y `forward`, conjunto `mon_ports`). En db01 cambia la regla de servicio por `ip saddr 10.10.2.10 tcp dport 5432 accept` y el conjunto por `{ 9100, 9187 }`. En mon01, el conjunto es `{ 9100 }` con origen la propia máquina, más `ip saddr { 10.10.2.10, 10.10.3.10 } tcp dport 3100 accept` y `ip saddr 10.10.0.0/24 tcp dport { 443, 3000 } accept` (el 3000 se retira en la sesión 18). Antes de activarlo, valida la sintaxis sin cargarlo:

    ```bash
    sudo nft -c -f /etc/nftables.conf
    ```

7. Activa el firewall en cada host desde una sesión SSH que entre por la red de gestión (si entras por otra interfaz, la política `drop` de `input` te cierra la puerta):

    ```bash
    sudo systemctl enable --now nftables
    sudo nft list ruleset | head -40
    ```

    Comprueba que Prometheus sigue en UP y que los contenedores de app01 siguen saliendo a Internet; si no, has puesto política `drop` en `forward`.

8. Reinicia app01 y comprueba al volver que `nft list ruleset` muestra la tabla `inet fw` y que `docker ps` sigue publicando en la IP de gestión.

9. En OPNsense, crea los alias `mon01`, `app01`, `db01`, `mon_ports_app` y `mon_ports_db` en Firewall, Aliases, y después las tres reglas de la [tabla del apartado](#segunda-capa-en-opnsense) en su interfaz, con registro activado. Aplica los cambios.

10. Repite los escaneos de la A3.1 desde web01, desde pre (o el puesto) y desde mon01, con la fecha nueva, en `monitoring/audit/despues/`. Escanea solo los puertos de la matriz para no esperar diez minutos por host:

    ```bash
    sudo nmap -sS -p 22,8080,9080,9100,9102 --reason 10.10.2.10 -oN escaneo-app01-desde-web01-$(date +%F).txt
    sudo nmap -sS -p 22,443,3000,3100,9090,9093,9100 --reason 10.10.0.20 -oN escaneo-mon01-desde-web01-$(date +%F).txt
    ```

11. Comprueba que los intentos denegados dejan rastro en las dos capas: en app01, `sudo journalctl -k | grep mon-` debe mostrar líneas `mon-denegado:` con la IP de web01; en OPNsense, Firewall, Log Files, Live View, filtra por el 9100 y captura la pantalla.

**Comprobación.** Desde web01 y desde pre, los puertos de monitorización de app01 y mon01 salen `filtered`; desde mon01 siguen `open`. Prometheus tiene todos los targets en UP, Grafana sigue mostrando los dashboards de la UT2 y los contenedores tienen salida a Internet. Tras el reinicio de app01, todo lo anterior sigue igual.

**Entrega.** Nada que entregar todavía. Deja en el repositorio `monitoring` los tres `nftables.conf` (en `monitoring/seguridad/nftables/`), la captura del Live View de OPNsense y los escaneos en `monitoring/audit/despues/`; la matriz gana la columna "obtenido (después)" para las filas de red, y el resto se completa en la sesión 18.

**Si te sobra tiempo.** Desactiva temporalmente la regla de OPNsense y repite el escaneo desde web01: debe seguir saliendo `filtered` gracias a nftables, que es lo que significa tener dos capas.

### A3.3 TLS y autenticación (sesión 18)

**Sesión 18 · 1 de diciembre · Teoría y práctica · unos 100 min de práctica**

**Objetivo.** Que todo el tráfico de monitorización vaya cifrado y con credenciales: exporters con TLS y basic auth, Loki con mTLS, Grafana detrás de nginx en el 443, y Prometheus con todos los targets en UP a través de ese cifrado.

**Antes de empezar.**

- La red y el firewall de la A3.2 funcionando (targets en UP, escaneo desde web01 `filtered`).
- `ca.crt` y `ca.key` de la CA del curso (`Lab 5166 CA`) en el puesto de administración, y solo ahí.
- `htpasswd` (paquete `apache2-utils`) o Python con `bcrypt` en el puesto.
- Resolución de nombres para `app01.dev.lab`, `db01.dev.lab` y `mon01.dev.lab` (DNS del laboratorio o `/etc/hosts` en mon01 y en el puesto).
- Se ha explicado al principio de la sesión el `web.config.file`, bcrypt y el mTLS. Ten a mano [Certificados con la CA del curso](#certificados-con-la-ca-del-curso), [TLS y basic auth en los exporters](#tls-y-basic-auth-en-los-exporters), [mTLS entre Promtail y Loki](#mtls-entre-promtail-y-loki) y [Grafana detrás de nginx](#grafana-detras-de-nginx-con-tls-y-roles). Sigue el orden de los pasos y no cambies dos cosas a la vez.

**Pasos.**

1. En el puesto, emite los certificados de servidor. Repite el par de comandos del apartado de certificados para cada uno, cambiando nombre, SAN e IP: `app01-node` (DNS app01.dev.lab, IP 10.10.0.11), `db01-node` y `db01-postgres` (db01.dev.lab, 10.10.0.12), `mon01-node`, `mon01-loki` y `mon01` para nginx (mon01.dev.lab, 10.10.0.20). Todos con `extendedKeyUsage=serverAuth`. Comprueba cada uno con `openssl x509 -in app01-node.crt -noout -subject -ext subjectAltName`.

2. Emite los certificados de cliente: `promtail-app01`, `promtail-db01` y `grafana`, con `extendedKeyUsage=clientAuth` y sin SAN:

    ```bash
    openssl req -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
      -keyout promtail-app01.key -out promtail-app01.csr -subj "/CN=promtail-app01"
    openssl x509 -req -in promtail-app01.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
      -days 365 -out promtail-app01.crt -extfile <(printf "extendedKeyUsage=clientAuth")
    ```

3. Genera el hash bcrypt del usuario `prometheus` y guarda la contraseña en claro en el puesto, porque la necesitarás en el paso 6:

    ```bash
    htpasswd -nBC 10 prometheus
    ```

4. Copia a cada host su certificado, su clave y `ca.crt` (nunca `ca.key`) con `scp`, en `/etc/node_exporter/` (y `/etc/postgres_exporter/` en db01), con propietario el usuario del servicio y la clave en 600. Escribe `/etc/node_exporter/web.yml` con el bloque `tls_server_config` y `basic_auth_users` del apartado, pegando el hash entre comillas dobles. Añade `--web.config.file=/etc/node_exporter/web.yml` al `ExecStart` de la unidad y reinicia. Verifica en el mismo host:

    ```bash
    curl --cacert /etc/node_exporter/ca.crt -u prometheus https://app01.dev.lab:9100/metrics | head -3
    ```

5. En app01, añade al compose un contenedor nginx en la red `monitoring` que termine TLS y pida basic auth para cAdvisor y la API: dos bloques `server` (8080 y 9102) con el certificado de app01, `auth_basic` con un fichero `htpasswd` del usuario `prometheus`, y `proxy_pass` a `cadvisor:8080` y `api:9102`. Los `ports` pasan a nginx y `cadvisor` y `api` dejan de publicar; `docker ps` debe mostrar solo nginx.

6. En mon01, crea `/etc/prometheus/secrets/node_exporter.pass` con la contraseña en claro (600, fuera del repositorio) y móntalo junto a `ca.crt` en el contenedor de Prometheus. Cambia todos los jobs al esquema `https` con `tls_config` y `basic_auth` como en el ejemplo del apartado. Recarga con `docker compose kill -s HUP prometheus`: si algún target está DOWN, su mensaje de error está en [Errores frecuentes](#errores-frecuentes-en-el-laboratorio).

7. Activa mTLS en Loki: copia `mon01-loki.crt`, su clave y `ca.crt` a `/var/lib/monitoring/loki/tls/`, monta la carpeta como `/etc/loki/tls` y añade el bloque `http_tls_config` con `client_auth_type: RequireAndVerifyClientCert`. Reinicia Loki. En ese momento los Promtail dejan de poder enviar: es esperado, y se arregla en el paso siguiente.

8. En app01 y db01, copia el certificado de cliente de Promtail y `ca.crt` a `/etc/promtail/tls/`, cambia `clients` a `https://mon01.dev.lab:3100/loki/api/v1/push` con el `tls_config` del apartado y reinicia Promtail. `docker logs promtail` (o `journalctl -u promtail`) debe dejar de mostrar errores de envío. Aprovecha para añadir las tres etapas `replace` de [Datos: redacción, retención y permisos](#datos-redaccion-retencion-y-permisos) al `pipeline_stages`, y pruébalas antes con `promtail --dry-run` sobre un fichero con una línea `password=hunter2`.

9. En mon01, da a Grafana su certificado de cliente para Loki en el provisioning del datasource: `jsonData: { tlsAuth: true, tlsAuthWithCACert: true }` y `secureJsonData` con `tlsCACert`, `tlsClientCert` y `tlsClientKey`. Reinicia Grafana y comprueba en Explore que Loki devuelve logs.

10. Pon Grafana detrás de nginx: añade el contenedor nginx al compose de mon01 con la configuración del apartado, publica `10.10.0.20:443:443`, quita el `3000` de Grafana y añade las variables `GF_*` de seguridad. Retira el 3000 de `nftables.conf` de mon01 y recarga con `nft -f`. Entra en `https://mon01.dev.lab/` desde el puesto (con `ca.crt` importada en el navegador) y crea un usuario nominal para cada miembro del grupo.

11. Fija la retención y los permisos: en Loki `limits_config: retention_period: 168h` con `compactor: { retention_enabled: true, delete_request_store: filesystem }`; en Prometheus `--storage.tsdb.retention.time=15d`; y en mon01 `chmod 700` y `chown` de `/var/lib/monitoring/{prometheus,loki,grafana}` a los UID 65534, 10001 y 472.

12. Verifica desde mon01 con las seis comprobaciones del apartado [Verificar que de verdad está protegido](#verificar-que-de-verdad-esta-protegido) y guarda las salidas en `monitoring/audit/despues/`. Captura además un scrape cifrado desde app01 y un `nmap -sV` desde mon01:

    ```bash
    sudo tcpdump -i ens18 -nn -A 'dst port 9100' -c 20 | tee tcpdump-app01-9100-tls-$(date +%F).txt
    sudo nmap -sV -p 9100,8080,9102 10.10.0.11 -oN nmap-sv-app01-$(date +%F).txt
    ```

**Comprobación.** Todos los targets en UP con `scrapeUrl` que empieza por `https://`. `curl` sin CA da error 60, con CA y sin credenciales da 401, con las dos devuelve métricas. `curl --cacert ca.crt https://mon01.dev.lab:3100/ready` sin certificado de cliente falla con `certificate required` y con él responde `ready`. En tcpdump del 9100 ya no se lee `GET /metrics`; `nmap -sV` muestra `ssl/http`. Grafana abre en el 443 con certificado válido, sin acceso anónimo, y muestra logs de Loki. En Grafana, `{service="api"} |= "password="` no devuelve nada sin asteriscos.

**Entrega.** Nada que entregar todavía. Deja en `monitoring/seguridad/` los `web.yml` (hash sustituido por `<bcrypt>`), la configuración de nginx, los fragmentos de Loki y Promtail y las variables de Grafana; las comprobaciones van en `monitoring/audit/despues/`. Completa la columna "obtenido (después)" de la matriz.

**Si te sobra tiempo.** Emite un certificado con otra CA improvisada, ponlo en un node_exporter y observa el error exacto que da Prometheus; vuelve a dejar el bueno.

## Práctica evaluable

### Práctica evaluable UT3 (sesión 19)

**Sesión 19 · 3 de diciembre · Práctica evaluable · unos 110 min de práctica**

Toma la pila de monitorización de tu entorno tal como quedó en la UT2 y déjala conforme a la política de protección de esta unidad, con evidencias de cada paso. La sesión 19 se dedica a cerrar lo que falte, a repetir la auditoría completa sobre el estado final y a redactar el documento de política. Se entrega por Aules como un directorio `monitoring/seguridad/` en el repositorio `monitoring`, con un `README.md` que enlace cada evidencia.

Entregables:

- [ ] Matriz de exposición antes y después, con las mismas filas y las columnas obtenido (antes), obtenido (después) y evidencia.
- [ ] `nftables.conf` de app01, db01 y mon01, y captura de las reglas y alias de OPNsense.
- [ ] Configuración de TLS y autenticación: `web.yml` de los exporters (sin el hash real, sustituido por `<bcrypt>`), jobs de Prometheus, fragmentos de Loki y Promtail con mTLS, nginx y variables de Grafana, `pipeline_stages` de redacción.
- [ ] Evidencias de escaneo desde mon01, front, back y otra VPC (ficheros `-oN` con fecha) que demuestran que solo mon01 llega a los exporters.
- [ ] Evidencias de tcpdump antes (HTTP legible) y después (TLS), y salidas de curl y openssl s_client con y sin certificado.
- [ ] Documento de política de protección entre contenedor y monitorización adaptado a tu entorno, con las cinco reglas, su comprobación y la fecha de la última verificación.

| Criterio | RA1 | Peso |
|----------|-----|-----:|
| Auditoría de comunicaciones: solo protocolos y puertos requeridos habilitados, con evidencias | f | 50 % |
| Reglas de protección configuradas (red, firewall, TLS, autenticación, redacción) y documentadas | g | 50 % |

Para el criterio f se valora que la matriz sea completa (todos los orígenes relevantes, no solo mon01), que los estados de nmap estén bien interpretados y que cada fila tenga evidencia con fecha. Para el criterio g se valora que las reglas estén en las dos capas, que la configuración sea persistente (sobrevive a reiniciar host y contenedores) y que el documento de política se corresponda con lo que de verdad hay configurado; una política que dice "mTLS" con un Loki que acepta HTTP se corrige sobre la mitad de ese criterio.

## Para ampliar

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
