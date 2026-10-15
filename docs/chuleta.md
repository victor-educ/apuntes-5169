# Chuleta de comandos

Lo que se teclea una y otra vez en esta asignatura. Los comandos de Proxmox, OpenTofu, Ansible, Jenkins y certificados están en la [chuleta de la asignatura de despliegue](https://victor-educ.github.io/apuntes-5166/chuleta/).

Tres reglas para que los comandos funcionen tal cual durante todo el curso:

- El servicio se maneja siempre por su compose (`/opt/servicio/compose.yml`, proyecto `servicio`), nunca por el nombre del contenedor escrito a mano.
- Desde el 3 de diciembre Prometheus, Alertmanager y Mailpit solo escuchan en el `127.0.0.1` de mon01: sus bloques se lanzan desde una shell en mon01, la que deja abierta el túnel de [Cómo se llega a la pila](laboratorio.md#como-se-llega-a-la-pila-desde-el-3-de-diciembre), y `logcli` desde el puesto de administración con su certificado de cliente.
- `app01`, `db01` y `mon01` son los nombres cortos de las VM: hasta el 27 de noviembre, los del `/etc/hosts` de la A1.4; desde el traslado, sus IP de la VPC (10.10.2.10, 10.10.3.10 y 10.10.0.20) o sus nombres `app01.dev.lab`, `db01.dev.lab` y `mon01.lab`.

## Docker: observar un contenedor

```bash
C=/opt/servicio/compose.yml                       # el compose del servicio, en app01 y en app01-pre
APP=$(docker compose -f $C ps -q app)             # el contenedor de la API, sin escribir su nombre
docker stats --no-stream
docker compose -f $C logs -f --since 10m app
docker compose -f $C logs --no-log-prefix app | grep '"level": "error"' | tail -50
docker events --filter type=container --format '{{json .}}'
docker inspect $APP --format '{{.RestartCount}} {{.State.ExitCode}} {{.State.OOMKilled}}'
docker inspect $APP --format '{{.State.Health.Status}}'
docker compose -f $C ps && docker compose -f $C top
docker system df -v
ssh ops@app01 docker compose -f /opt/servicio/compose.yml restart app   # desde el puesto, sin entrar
# Límites del driver de logs (/etc/docker/daemon.json)
# { "log-driver": "json-file", "log-opts": { "max-size": "50m", "max-file": "5" } }
sudo systemctl restart docker
# cgroups v2 del contenedor
cat /sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' $APP).scope/memory.current
```

## Prometheus y PromQL

```bash
# En una shell de mon01
cd /opt/monitoring && docker compose exec prometheus promtool check config /etc/prometheus/prometheus.yml
cd /opt/alerting/prometheus                       # las reglas viven en el repositorio alerting
docker run --rm -v "$PWD:/w" --entrypoint promtool prom/prometheus check rules /w/rules.yml /w/alerts.yml
docker run --rm -v "$PWD:/w" -w /w/tests --entrypoint promtool prom/prometheus test rules alerts_test.yml
curl -s -X POST http://localhost:29090/-/reload
curl -s 'http://localhost:29090/api/v1/query?query=up' | jq .
curl -s http://app01:29102/metrics | grep -E '^app_'
```

```text
sum(rate(app_http_requests_total{status=~"5.."}[5m])) / sum(rate(app_http_requests_total[5m]))
histogram_quantile(0.95, sum by (le) (rate(app_http_request_duration_seconds_bucket[5m])))
container_memory_working_set_bytes{service="app"} / container_spec_memory_limit_bytes{service="app"}
changes(process_start_time_seconds{job="app"}[1h])    # es un gauge: changes, no increase
sum(pg_stat_activity_count) / sum(pg_settings_max_connections)
predict_linear(node_filesystem_avail_bytes{mountpoint="/data"}[7d], 30*86400) < 0   # /data: el disco de datos de db01
absent(up{job="app"})
count({job="app"})
```

## Loki y LogQL

```bash
# Desde el 3 de diciembre, en el puesto de administración y con certificado; antes basta LOKI_ADDR=http://mon01:23100
export LOKI_ADDR=https://10.10.0.20:23100
export LOKI_CA_CERT_PATH=$HOME/tls-monitorizacion/ca.crt
export LOKI_CLIENT_CERT_PATH=$HOME/tls-monitorizacion/logcli.crt LOKI_CLIENT_KEY_PATH=$HOME/tls-monitorizacion/logcli.key
logcli labels
logcli query '{service="app"}' --since=1h --limit=50
logcli query 'sum(count_over_time({service="app", level="error"} [5m]))'
# Borrar streams de un intervalo (compactor con retention_enabled y delete_request_store)
curl -G -X POST --cacert $LOKI_CA_CERT_PATH --cert $LOKI_CLIENT_CERT_PATH --key $LOKI_CLIENT_KEY_PATH \
  $LOKI_ADDR/loki/api/v1/delete --data-urlencode 'query={service="app"}' --data-urlencode start=<unix> --data-urlencode end=<unix>
# En mon01: las métricas del propio Promtail de app01
curl -s http://app01:29080/metrics | grep promtail_dropped
```

```text
{service="app", level="error"}
{service="app", level="error"} | json | msg=~"timeout.*"
sum(count_over_time({service="app", level="error"} [5m]))
count_over_time({service="docker-events", action="oom"} [10m]) > 0
sum(rate({service="nginx"} | json | status >= 500 [1m]))
```

## Alertmanager

```bash
# En una shell de mon01, en /opt/monitoring: amtool viene dentro del contenedor de Alertmanager
AM="docker compose exec alertmanager amtool --alertmanager.url=http://localhost:29093"
docker compose exec alertmanager amtool check-config /etc/alertmanager/alertmanager.yml
$AM alert query
$AM alert add alertname=Prueba severity=warning service=app
$AM silence add alertname=AppRestarting --duration=2h --comment "mantenimiento"
$AM silence query
$AM silence expire <id>
curl -s http://localhost:29093/api/v2/alerts | jq '.[] | {alertname: .labels.alertname, state: .status.state}'
curl -s http://localhost:28025/api/v1/messages | jq .total     # correos que ha recibido Mailpit
```

## Red y firewall del host

```bash
ss -tlnup
docker ps --format '{{.Names}} {{.Ports}}'
nmap -sS -p- -T4 app01
nmap -sV -p 29100,28081,29102,29080 app01
tcpdump -i ens18 -n port 23100
# nftables del host: el fichero completo es el de la UT3 (tabla inet fw, conjunto mon_ports)
sudo nft -c -f /etc/nftables.conf && sudo systemctl reload nftables
sudo nft list table inet fw
journalctl -k -g 'mon-denegado|mon-docker-denegado'     # quién ha chocado con el cortafuegos del host
# TLS
openssl s_client -connect 10.10.2.10:29100 -CAfile ca.crt </dev/null 2>/dev/null | openssl x509 -noout -subject -dates
curl -s --cacert ca.crt -u prometheus https://10.10.2.10:29100/metrics | head
htpasswd -nBC 10 prometheus          # hash bcrypt para web.config.file
```

## Pruebas

```bash
# En el puesto de administración, contra dev: la UT4 no toca pre
export REQUESTS_CA_BUNDLE=~/ca/ca.crt              # requests no lee el almacén del sistema: sin esto, SSLError
# Funcionales
API_URL=https://api.dev.lab python3 -m pytest tests/functional -v --junitxml=reports/pytest.xml
docker run --rm -v "$PWD/tests/postman:/etc/newman" -v ~/ca/ca.crt:/ca.crt:ro -e NODE_EXTRA_CA_CERTS=/ca.crt \
  postman/newman run api-curso.json --env-var base=https://api.dev.lab --reporters cli,junit
# Carga y estrés
k6 run tests/k6/carga.js
k6 run --vus 50 --duration 5m tests/k6/carga.js
k6 run --summary-export=tests/evidence/1.4.2/k6-resumen.json tests/k6/carga.js
k6 run --out experimental-prometheus-rw tests/k6/carga.js   # K6_PROMETHEUS_RW_SERVER_URL por un túnel al 29090 de mon01
# Seguridad: ZAP en mon01, Trivy en el puesto de administración
docker run --rm --network host -v "$PWD/zap:/zap/wrk:rw" ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t https://api.dev.lab -r zap.html -I   # -I: los WARN no dan error
ssh ops@10.10.2.10 docker save app:1.4.2 > /tmp/app.tar    # hasta el 19 de febrero la imagen es local, en app01
trivy image --input /tmp/app.tar --severity HIGH,CRITICAL --ignore-unfixed
```

## Accesos y fail2ban

```bash
journalctl -u ssh --since yesterday -p err
# -u ssh, no _COMM=sshd: en Debian 13 los accesos los registra sshd-session
journalctl -u ssh | grep "Failed password" | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head
grep "Failed password" /var/log/auth.log | wc -l
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip 203.0.113.7
sudo fail2ban-regex /var/log/nginx/error.log /etc/fail2ban/filter.d/nginx-http-auth.conf
```

## Fallos y rendimiento

```bash
dmesg -T | grep -i -E "oom|killed process"
coredumpctl list && coredumpctl info
cat /proc/sys/kernel/core_pattern
py-spy dump --pid $(pgrep -f gunicorn | head -1)
top -o %MEM; mpstat 1 5; vmstat 1 5; iostat -xz 1 5
ss -s; iftop -i ens18; pidstat 1 5
ps aux --sort=-%mem | head
```

## Copias con restic

```bash
# En el puesto de administración: el repositorio único del laboratorio
set -a; . /etc/restic/env; set +a                           # las cuatro variables de la A7.4, que son estas:
# RESTIC_REPOSITORY=s3:http://10.10.0.30:29000/backups   RESTIC_PASSWORD_FILE=/etc/restic/pass
# AWS_ACCESS_KEY_ID=restic   AWS_SECRET_ACCESS_KEY=...    # sin estas dos, restic ni siquiera abre el repositorio
restic init                                                 # una sola vez en todo el curso: paso 3 de la A7.4
ssh ops@10.20.3.10 'sudo -u postgres pg_dump -Fc servicio' | restic backup --stdin --stdin-filename servicio-pre.dump --tag pre
restic snapshots --tag pre
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
restic check --read-data-subset=5%
restic dump --tag pre latest /servicio-pre.dump > /tmp/servicio-pre.dump   # sacar un fichero sin restaurar todo
restic restore latest --target /tmp/restore
restic mount /mnt/restic
pg_restore -l /tmp/servicio-pre.dump | head                # el índice del volcado: si sale, el fichero se lee
systemctl list-timers | grep backup                         # en la empresa (UT6), la copia la lanza un timer
```

## Versiones y vulnerabilidades

```bash
docker images --digests
docker manifest inspect postgres:17.6 | jq '.config.digest'
docker compose -f /opt/servicio/compose.yml config | grep image:
# La imagen del servicio: app:<versión> en app01 hasta el 19 de febrero; registry.lab:5000/app:<versión> desde entonces
ssh ops@10.10.2.10 docker save app:1.4.2 > /tmp/app.tar
syft docker-archive:/tmp/app.tar -o cyclonedx-json > sbom.json
grype sbom:sbom.json --only-fixed
trivy image --input /tmp/app.tar --format json -o trivy.json
trivy image --severity CRITICAL --exit-code 1 registry.lab:5000/app:1.4.2   # puerta en el pipeline, desde el 19 de febrero
trivy fs . && trivy config .
pip-audit -r api/requirements.txt
# Integridad de datos antes y después, en db01 (o db01-pre)
sudo -u postgres psql -d servicio -c "SELECT relname, n_live_tup FROM pg_stat_user_tables ORDER BY 1;"
sudo -u postgres psql -d servicio -c "SELECT md5(string_agg(t::text, '' ORDER BY id)) FROM items t;"
```

## Terminación segura

```bash
# pre, siempre con IP y desde el puesto de administración; las evidencias, a una ruta del puesto
ssh ops@10.20.2.10 sudo docker compose -f /opt/servicio/compose.yml down -v --rmi all   # A8.2: liberar app01-pre
ssh ops@10.20.2.10 'sudo docker volume ls; sudo docker network ls; sudo docker images'   # comprobar que no queda nada
curl --cacert ~/ca/ca.crt -u <usuario> -X DELETE https://registry.lab:5000/v2/app/manifests/<digest>   # solo lo desplegado en pre
# En gitea01, con el registry parado: el recolector de basura libera las capas
docker compose -f /opt/registry/compose.yml stop registry
docker compose -f /opt/registry/compose.yml run --rm registry garbage-collect --delete-untagged /etc/docker/registry/config.yml
docker compose -f /opt/registry/compose.yml start registry
cd ~/iac-lab/envs/pre && tofu plan -destroy -out=destroy.plan && tofu apply destroy.plan   # A8.3, desde la rama baja-pre sin prevent_destroy
qm stop 200 && qm destroy 200 --purge     # router-pre, en el nodo: no es de OpenTofu, viene de la A2.5
restic forget --prune $(restic snapshots --tag pre --json | jq -r '.[].short_id')   # sin --keep-*, forget solo borra por identificador
restic snapshots --tag pre                # vacío
mc version info lab/backups                                      # un-versioned: lo que prune borra no se queda
mc rm --recursive --versions --non-current --force lab/backups   # solo si el bucket tiene versionado
shred -n 3 -z -v /dev/sdb          # solo discos magnéticos dedicados
blkdiscard /dev/nvme1n1            # SSD completo
# En una base que se conserva (en la A8.5, bajapre en db01). Dos -c y no uno: VACUUM no puede ir dentro de la transacción que abre un -c con varias órdenes
sudo -u postgres psql -d bajapre -c "UPDATE <tabla> SET <columna> = md5(<columna> || '<sal>');" -c "VACUUM FULL <tabla>;"
sudo -u postgres psql -d bajapre -c "DROP TABLE <tabla>;"
grep pre ~/ca/index.txt                   # vacío: en pre no se emitió ningún certificado
curl -sG http://localhost:29090/api/v1/query --data-urlencode 'query=count({env="pre"})'   # en mon01; [] si no queda nada
sudo photorec /d /tmp/recuperado /cmd /dev/sdb1 search
```
