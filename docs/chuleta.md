# Chuleta de comandos

Lo que se teclea una y otra vez en esta asignatura. Los comandos de Proxmox, OpenTofu, Ansible, Jenkins y certificados están en la [chuleta de la asignatura de despliegue](https://victor-educ.github.io/apuntes-5166/chuleta/).

## Docker: observar un contenedor

```bash
docker stats --no-stream
docker logs -f --since 10m app
docker logs app 2>&1 | grep -E "ERROR|FATAL" | tail -50
docker events --filter type=container --format '{{json .}}'
docker inspect app --format '{{.RestartCount}} {{.State.ExitCode}} {{.State.OOMKilled}}'
docker inspect app --format '{{.State.Health.Status}}'
docker compose ps && docker compose top
docker system df -v
# Límites del driver de logs (/etc/docker/daemon.json)
# { "log-driver": "json-file", "log-opts": { "max-size": "50m", "max-file": "5" } }
sudo systemctl restart docker
# cgroups v2 del contenedor
cat /sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' app).scope/memory.current
```

## Prometheus y PromQL

```bash
promtool check config prometheus.yml
promtool check rules rules.yml alerts.yml
promtool test rules tests.yml
curl -s -X POST http://mon01:9090/-/reload
curl -s 'http://mon01:9090/api/v1/query?query=up' | jq .
curl -s http://app01:9102/metrics | grep -E '^app_'
```

```text
sum(rate(app_requests_total{status=~"5.."}[5m])) / sum(rate(app_requests_total[5m]))
histogram_quantile(0.95, sum by(le)(rate(app_request_seconds_bucket[5m])))
container_memory_working_set_bytes{name="app"} / container_spec_memory_limit_bytes{name="app"}
increase(container_start_time_seconds{name="app"}[1h])
pg_stat_activity_count / pg_settings_max_connections
predict_linear(node_filesystem_avail_bytes{mountpoint="/data"}[7d], 30*86400) < 0   # /data: el disco de datos de db01
absent(up{job="app"})
count({job="app"})
```

## Loki y LogQL

```bash
logcli --addr=http://mon01:3100 labels
logcli --addr=http://mon01:3100 query '{container="app"}' --since=1h --limit=50
logcli --addr=http://mon01:3100 query 'sum(count_over_time({container="app"} |= "ERROR" [5m]))'
curl -s http://app01:9080/metrics | grep promtail_dropped
# Borrar streams (compactor con retention_enabled y delete_request)
curl -X POST 'http://mon01:3100/loki/api/v1/delete?query={container="app"}&start=0'
```

```text
{container="app"} |= "ERROR"
{container="app"} | json | level="error" | msg=~"timeout.*"
sum(count_over_time({container="app"} | json | level="error" [5m]))
count_over_time({job="docker-events"} | json | Action="oom" [10m]) > 0
rate({job="nginx"} | logfmt | status >= 500 [1m])
```

## Alertmanager

```bash
amtool check-config alertmanager.yml
amtool --alertmanager.url=http://mon01:9093 alert
amtool --alertmanager.url=http://mon01:9093 alert add alertname=Prueba severity=warning service=app
amtool --alertmanager.url=http://mon01:9093 silence add alertname=AppRestarting --duration=2h --comment "mantenimiento"
amtool --alertmanager.url=http://mon01:9093 silence query
amtool --alertmanager.url=http://mon01:9093 silence expire <id>
curl -s http://mon01:9093/api/v2/alerts | jq '.[] | {alertname: .labels.alertname, state: .status.state}'
```

## Red y firewall del host

```bash
ss -tlnup
docker ps --format '{{.Names}} {{.Ports}}'
nmap -sS -p- -T4 app01
nmap -sV -p 9100,8081,9187,9102 app01
tcpdump -i eth0 -n port 3100
nft list ruleset
nft add rule inet fw input ip saddr 10.10.0.20 tcp dport { 9100, 8081, 9187, 9102 } accept
nft add rule inet fw input tcp dport { 9100, 8081, 9187, 9102 } drop
# TLS
openssl s_client -connect app01:9100 -servername app01 </dev/null 2>/dev/null | openssl x509 -noout -subject -dates
curl -sk --cacert ca.crt -u prometheus:secreto https://app01:9100/metrics | head
htpasswd -nBC 10 prometheus          # hash bcrypt para web.config.file
```

## Pruebas

```bash
# Funcionales
pytest tests/ --junitxml=report.xml
newman run coleccion.json -e pre.json --reporters cli,junit --reporter-junit-export report.xml
# Carga y estrés
k6 run test.js
k6 run --vus 50 --duration 5m test.js
k6 run --out json=result.json test.js
k6 run --out experimental-prometheus-rw test.js     # con K6_PROMETHEUS_RW_SERVER_URL
# Seguridad
docker run --rm -t -v $PWD:/zap/wrk zaproxy/zap-stable zap-baseline.py -t https://api.dev.lab -r zap.html
trivy image --severity HIGH,CRITICAL --ignore-unfixed registry.lab:5000/app:1.4.2
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
ss -s; iftop -i eth0; pidstat 1 5
ps aux --sort=-%mem | head
```

## Copias con restic

```bash
export RESTIC_REPOSITORY=s3:http://10.10.0.30:9000/backups RESTIC_PASSWORD_FILE=/etc/restic/pass
export AWS_ACCESS_KEY_ID=restic AWS_SECRET_ACCESS_KEY=...   # sin estas dos, restic ni siquiera abre el repositorio
restic init                                                 # una sola vez en todo el curso
pg_dump -h db01 -U app -Fc app > /var/backups/app.dump   # PostgreSQL vive en db01, no en el compose
restic backup /var/backups/app.dump /opt/app/compose.yml /opt/app/data --tag app
restic snapshots
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
restic check --read-data-subset=5%
restic restore latest --target /restore
restic mount /mnt/restic
pg_restore -h db01 -U app -d app_test /restore/var/backups/app.dump
systemctl list-timers | grep backup
systemctl status backup-app.timer
```

## Versiones y vulnerabilidades

```bash
docker images --digests
docker manifest inspect postgres:17.6 | jq '.config.digest'
docker compose config | grep image:
syft registry.lab:5000/app:1.4.2 -o cyclonedx-json > sbom.json
grype sbom:sbom.json --only-fixed
trivy image --format json -o trivy.json registry.lab:5000/app:1.4.2
trivy image --severity CRITICAL --exit-code 1 registry.lab:5000/app:1.4.2   # puerta en el pipeline
trivy fs . && trivy config .
pip-audit -r requirements.txt
npm audit --audit-level=high
# Integridad de datos antes/después
psql -h db01 -U app -c "SELECT relname, n_live_tup FROM pg_stat_user_tables ORDER BY 1;"
psql -h db01 -U app -c "SELECT md5(string_agg(t::text, '' ORDER BY id)) FROM items t;"
```

## Terminación segura

```bash
docker compose down -v --rmi all
docker volume ls; docker network ls; docker image prune -a
curl -X DELETE https://registry.lab:5000/v2/app/manifests/<digest>
docker exec registry registry garbage-collect /etc/docker/registry/config.yml
tofu destroy -var-file=pre.tfvars
qm destroy 220 --purge          # app01 de pre
restic key list && restic key remove <id>          # borrado criptográfico: sin clave no hay copia
aws --endpoint-url http://10.10.0.30:9000 s3api list-object-versions --bucket backups
aws --endpoint-url http://10.10.0.30:9000 s3api delete-objects --bucket backups --delete file://versiones.json
shred -n 3 -z -v /dev/sdb          # solo discos magnéticos dedicados
blkdiscard /dev/nvme1n1            # SSD completo
psql -U app -c "UPDATE users SET email = md5(email), name = 'anon'; VACUUM FULL users;"
psql -U app -c "DROP TABLE pedidos; VACUUM FULL;"
curl -s 'http://mon01:9090/api/v1/query?query=count({env="pre"})'
sudo photorec /d /tmp/recuperado /cmd /dev/sdb1 search
```
