
- Ubuntu 2404 (ubutest 192.168.1.244) - виртуальная машина на ESXi гипервизоре


###  Запуск docker контейнера InfluxDB



- первый запуск неудачный
```
boss@ubutest:~$ docker run --name influxdb2 --publish 8086:8086 \
           --mount type=volume,source=influxdb2-data,target=/var/lib/influxdb2 \
           --mount type=volume,source=influxdb2-config,target=/etc/influxdb2 \
                   --env DOCKER_INFLUXDB_INIT_MODE=setup \
                   --env DOCKER_INFLUXDB_INIT_USERNAME=influx \
                   --env DOCKER_INFLUXDB_INIT_PASSWORD=influx \
                   --env DOCKER_INFLUXDB_INIT_ORG=sirus \
                   --env DOCKER_INFLUXDB_INIT_BUCKET=office \
                   influxdb:2
docker: invalid reference format.
See 'docker run --help'.
--env: command not found
--env: command not found
--env: command not found
influxdb:2: command not found
```

- выход - создал файл  env.list
```
DOCKER_INFLUXDB_INIT_MODE=setup
DOCKER_INFLUXDB_INIT_USERNAME=influx
DOCKER_INFLUXDB_INIT_PASSWORD=influx
DOCKER_INFLUXDB_INIT_ORG=sirus
DOCKER_INFLUXDB_INIT_BUCKET=office
```

- команда запуска стала короче - но неудачно - в файле env.list лишний пробел после слова setup
```
boss@ubutest:~$ docker run --name influxdb2 --publish 8086:8086 \
--mount source=influxdb2-data,target=/var/lib/influxdb2 \
--mount source=influxdb2-config,target=/etc/influxdb2 \
--env-file ./env.list \
influxdb:2
2025-01-21T06:24:16.355169598Z  error   found invalid DOCKER_INFLUXDB_INIT_MODE, valid values are 'setup' and 'upgrade' {"system": "docker", "DOCKER_INFLUXDB_INIT_MODE": "setup "}
```

- И опять неудачно - пароль должен быть длиннее
```
boss@ubutest:~$ docker run --name influxdb2 --publish 8086:8086 \
--mount source=influxdb2-data,target=/var/lib/influxdb2 \
--mount source=influxdb2-config,target=/etc/influxdb2 \
--env-file ./env.list \
influxdb:2
...
ts=2025-01-21T06:36:35.962503Z lvl=error msg="failed to onboard user influx" log_id=0uDu4aUW000 handler=onboard error="passwords must be between 8 and 72 characters long" took=1.142ms
ts=2025-01-21T06:36:35.962531Z lvl=error msg="api error encountered" log_id=0uDu4aUW000 error="passwords must be between 8 and 72 characters long"
Error: failed to setup instance: 400 Bad Request: passwords must be between 8 and 72 characters long
...
```

- поправил пароль и запустилось !!!
- и теперь данная сессия занята запущенным контейнером

###  Страничка в браузере

на Windows адресс вводим виртуальной машины Ubuntu

http://192.168.1.244:8086

вводим логин и пароль - и веб-клиент открылся



###  Откоываем вторую сессию терминала

- контейнер в работе
```
boss@ubutest:~$ docker ps
CONTAINER ID   IMAGE        COMMAND                  CREATED          STATUS         PORTS                                       NAMES
56c811f7c254   influxdb:2   "/entrypoint.sh infl…"   10 seconds ago   Up 9 seconds   0.0.0.0:8086->8086/tcp, :::8086->8086/tcp   influxdb2
```

```
boss@ubutest:~$ docker exec -it influxdb2 influx config ls
Active  Name    URL                     Org
*       default http://localhost:8086   sirus
```

- конфигурация сервера 

```
boss@ubutest:~$ docker exec -it influxdb2 influx server-config
{
        "assets-path": "",
        "bolt-path": "/var/lib/influxdb2/influxd.bolt",
        "e2e-testing": false,
        "engine-path": "/var/lib/influxdb2/engine",
        "feature-flags": null,
        "flux-log-enabled": false,
        "hardening-enabled": false,
        "http-bind-address": ":8086",
        "http-idle-timeout": 180000000000,
        "http-read-header-timeout": 10000000000,
        "http-read-timeout": 0,
        "http-write-timeout": 0,
        "influxql-max-select-buckets": 0,
        "influxql-max-select-point": 0,
        "influxql-max-select-series": 0,
        "instance-id": "",
        "log-level": "info",
        "metrics-disabled": false,
        "nats-max-payload-bytes": 0,
        "nats-port": 4222,
        "no-tasks": false,
        "overwrite-pid-file": false,
        "pid-file": "",
        "pprof-disabled": false,
        "query-concurrency": 1024,
        "query-initial-memory-bytes": 0,
        "query-max-memory-bytes": 0,
        "query-memory-bytes": 0,
        "query-queue-size": 1024,
        "reporting-disabled": false,
        "secret-store": "bolt",
        "session-length": 60,
        "session-renew-disabled": false,
        "sqlite-path": "/var/lib/influxdb2/influxd.sqlite",
        "storage-cache-max-memory-size": 1073741824,
        "storage-cache-snapshot-memory-size": 26214400,
        "storage-cache-snapshot-write-cold-duration": "10m0s",
        "storage-compact-full-write-cold-duration": "4h0m0s",
        "storage-compact-throughput-burst": 50331648,
        "storage-max-concurrent-compactions": 0,
        "storage-max-index-log-file-size": 1048576,
        "storage-no-validate-field-size": false,
        "storage-retention-check-interval": "30m0s",
        "storage-series-file-max-concurrent-snapshot-compactions": 0,
        "storage-series-id-set-cache-size": 0,
        "storage-shard-precreator-advance-period": "30m0s",
        "storage-shard-precreator-check-interval": "10m0s",
        "storage-tsm-use-madv-willneed": false,
        "storage-validate-keys": false,
        "storage-wal-flush-on-shutdown": false,
        "storage-wal-fsync-delay": "0s",
        "storage-wal-max-concurrent-writes": 0,
        "storage-wal-max-write-delay": 600000000000,
        "storage-write-timeout": 10000000000,
        "store": "disk",
        "strong-passwords": false,
        "template-file-urls-disabled": false,
        "testing-always-allow-setup": false,
        "tls-cert": "",
        "tls-key": "",
        "tls-min-version": "1.2",
        "tls-strict-ciphers": false,
        "tracing-type": "",
        "ui-disabled": false,
        "vault-addr": "",
        "vault-cacert": "",
        "vault-capath": "",
        "vault-client-cert": "",
        "vault-client-key": "",
        "vault-client-timeout": 0,
        "vault-max-retries": 0,
        "vault-skip-verify": false,
        "vault-tls-server-name": "",
        "vault-token": ""
}
```

```
boss@ubutest:~$ docker exec -it influxdb2 influxd inspect -d
Error: unknown shorthand flag: 'd' in -d
See 'influxd -h' for help
```
// нерабочая команда - там нет таких ключей



- устанавливаем клиента (тут же в Ubuntu)

```
boss@ubutest:~$ mkdir infl
boss@ubutest:~$ cd infl
boss@ubutest:~/infl$ wget https://dl.influxdata.com/influxdb/releases/influxdb2-client-2.7.5-linux-amd64.tar.gz
--2025-01-21 07:00:45--  https://dl.influxdata.com/influxdb/releases/influxdb2-client-2.7.5-linux-amd64.tar.gz
Resolving dl.influxdata.com (dl.influxdata.com)... 54.240.174.54, 54.240.174.102, 54.240.174.94, ...
Connecting to dl.influxdata.com (dl.influxdata.com)|54.240.174.54|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 11663685 (11M) [application/x-tar]
Saving to: ‘influxdb2-client-2.7.5-linux-amd64.tar.gz’

influxdb2-client-2.7.5-linux-amd64.tar.gz  100%[=======================================================================================>]  11,12M  23,2MB/s    in 0,5s

2025-01-21 07:00:45 (23,2 MB/s) - ‘influxdb2-client-2.7.5-linux-amd64.tar.gz’ saved [11663685/11663685]

boss@ubutest:~/infl$ tar xvzf ./influxdb2-client-2.7.5-linux-amd64.tar.gz
./
./LICENSE
./README.md
./influx
boss@ubutest:~/infl$ sudo cp ./influx /usr/local/bin/

boss@ubutest:~/infl$ influx
NAME:
   influx - Influx Client

USAGE:
   influx [command]

HINT: If you are looking for the InfluxQL shell from 1.x, run "influx v1 shell"

COMMANDS:
   version              Print the influx CLI version
   write                Write points to InfluxDB
   bucket               Bucket management commands
   completion           Generates completion scripts
   query                Execute a Flux query
   config               Config management commands
   org, organization    Organization management commands
   delete               Delete points from InfluxDB
   user                 User management commands
   task                 Task management commands
   telegrafs            List Telegraf configuration(s). Subcommands manage Telegraf configurations.
   dashboards           List Dashboard(s).
   export               Export existing resources as a template
   secret               Secret management commands
   v1                   InfluxDB v1 management commands
   auth, authorization  Authorization management commands
   apply                Apply a template to manage resources
   stacks               List stack(s) and associated templates. Subcommands manage stacks.
   template             Summarize the provided template
   bucket-schema        Bucket schema management commands
   scripts              Scripts management commands
   ping                 Check the InfluxDB /health endpoint
   setup                Setup instance with initial user, org, bucket
   backup               Backup database
   restore              Restores a backup directory to InfluxDB
   remote               Remote connection management commands
   replication          Replication stream management commands
   server-config        Display server config
   help, h              Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --help, -h  show help
```


- определяем токен для настройки клиента

```
boss@ubutest:~$ docker volume inspect influxdb2-config
[
    {
        "CreatedAt": "2025-01-21T06:34:45Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/influxdb2-config/_data",
        "Name": "influxdb2-config",
        "Options": null,
        "Scope": "local"
    }
]
boss@ubutest:~$ sudo ls/var/lib/docker/volumes/influxdb2-config/_data
sudo: ls/var/lib/docker/volumes/influxdb2-config/_data: command not found
boss@ubutest:~/infl$ sudo ls /var/lib/docker/volumes/influxdb2-config/_data
influx-configs
boss@ubutest:~$ sudo cat /var/lib/docker/volumes/influxdb2-config/_data/influx-configs
[default]
  url = "http://localhost:8086"
  token = "ICEmDRlB94HhXZmur02hlxEPTe1QCIOE7_j0V-ivdwAokYGs132qpIpDqKkGQtuTZ4vgwnEP3lJe-8EEfaS6lw=="
  org = "sirus"
  active = true
#
# [eu-central]
#   url = "https://eu-central-1-1.aws.cloud2.influxdata.com"
#   token = "XXX"
#   org = ""
#
# [us-central]
#   url = "https://us-central1-1.gcp.cloud2.influxdata.com"
#   token = "XXX"
#   org = ""
#
# [us-west]
#   url = "https://us-west-2-1.aws.cloud2.influxdata.com"
#   token = "XXX"
#   org = ""
```

- настраиваем клиента CLI для доступа к Influx 

```
boss@ubutest:~$ ./influx config create --config-name MY_CONFIG \
--host-url http://192.168.1.244:8086 \
--org sirus \
--token = ICEmDRlB94HhXZmur02hlxEPTe1QCIOE7_j0V-ivdwAokYGs132qpIpDqKkGQtuTZ4vgwnEP3lJe-8EEfaS6lw== \
--active
```
// параметры клиента сохраняются в файле  **~/.influxdbv2/configs**


- создаём bucket и заполняем данным из CSV

```
boss@ubutest:~/infl$ ./influx bucket create --name bitcoin
boss@ubutest:~/infl$ ./influx write -b bitcoin -f ./bitcoin-historical-annotated.csv
```

- выполняем запрос

```
boss@ubutest:~$ ./influx query '
from(bucket: "bitcoin")
   |> range(start: 2025-01-01T08:00:00Z, stop: 2025-01-10T20:00:01Z)
   |> filter(fn: (r) => r._measurement == "coindesk")
   |> filter(fn: (r) => r._field == "price")
   |> filter(fn: (r) => r.code == "USD")
'
Result: _result
Table: keys: [_start, _stop, _field, _measurement, code, crypto, description, symbol]
                   _start:time                      _stop:time           _field:string     _measurement:string             code:string           crypto:string      description:string           symbol:string                      _time:time                  _value:float
------------------------------  ------------------------------  ----------------------  ----------------------  ----------------------  ----------------------  ----------------------  ----------------------  ------------------------------  ----------------------------
2025-01-01T08:00:00.000000000Z  2025-01-10T20:00:01.000000000Z                   price                coindesk                     USD                 bitcoin    United States Dollar                   &#36;  2025-01-02T01:52:43.000000000Z                     95324.742
2025-01-01T08:00:00.000000000Z  2025-01-10T20:00:01.000000000Z                   price                coindesk                     USD                 bitcoin    United States Dollar                   &#36;  2025-01-03T01:53:40.000000000Z                    98180.7686
2025-01-01T08:00:00.000000000Z  2025-01-10T20:00:01.000000000Z                   price                coindesk                     USD                 bitcoin    United States Dollar                   &#36;  2025-01-04T01:52:03.000000000Z                    99019.4068
2025-01-01T08:00:00.000000000Z  2025-01-10T20:00:01.000000000Z                   price                coindesk                     USD                 bitcoin    United States Dollar                   &#36;  2025-01-05T01:57:28.000000000Z                    99080.4417
2025-01-01T08:00:00.000000000Z  2025-01-10T20:00:01.000000000Z                   price                coindesk                     USD                 bitcoin    United States Dollar                   &#36;  2025-01-06T01:56:22.000000000Z                    99779.8982
2025-01-01T08:00:00.000000000Z  2025-01-10T20:00:01.000000000Z                   price                coindesk                     USD                 bitcoin    United States Dollar                   &#36;  2025-01-07T01:54:23.000000000Z                   102422.5572
2025-01-01T08:00:00.000000000Z  2025-01-10T20:00:01.000000000Z                   price                coindesk                     USD                 bitcoin    United States Dollar                   &#36;  2025-01-08T01:54:06.000000000Z                    97142.1298
2025-01-01T08:00:00.000000000Z  2025-01-10T20:00:01.000000000Z                   price                coindesk                     USD                 bitcoin    United States Dollar                   &#36;  2025-01-09T02:09:08.000000000Z                    94951.6729
2025-01-01T08:00:00.000000000Z  2025-01-10T20:00:01.000000000Z                   price                coindesk                     USD                 bitcoin    United States Dollar                   &#36;  2025-01-10T01:55:07.000000000Z                     93890.043
```

- пример в браузере 
- диапазон времени 30 дней надо установить
- а то ничего не выдавал

![DataExplorer](./Images/lesson1-2.jpg)