# Настройка кластера Patroni + Consul + VIP-manager

## 1. Оборудование и ПО

__настроим 6 виртуальных машин__

``` text
- три хоста - кластер DSC   [ test-dcs1, test-dcs2, test-dcs3 ]
- три хоста - кластер СУБД  [ test-db1,  test-db2,  test-db3 ]
```

__- программное обеспечение__

``` text
- Astra Linux 1.8  (based on Debian 12)
- Consul 1.22
- Patroni 4.0.4 
- PostgresPro 1C 18  (free edition PostgresPro)
- Vip-manager 4.0
```

__- план__

``` text
- кластер  Consul + Patroni + Vip-manager
- как настроить синхронную реплику
- Patroni callback как замена Vip-manager
```

## 2. Consul - кластер

#### 2.1 Установка Consul

на каждом хосте DCS загружаем выполняемый файл Consul и создаем рабочие каталоги<br>
(1.22.3 - последняя верси на январь 2026)

``` bash
wget https://hashicorp-releases.mcs.mail.ru/consul/1.22.3/consul_1.22.3_linux_amd64.zip -O /tmp/consul.zip
sudo unzip /tmp/consul.zip -d /usr/bin
sudo chmod +x /usr/bin/consul

sudo groupadd consul
sudo useradd -m -d /var/lib/consul -g consul -r -c 'Consul DCS service' consul
sudo mkdir -p /etc/consul.d /var/lib/consul/data /var/log/consul
sudo chown -R consul: /etc/consul.d /var/lib/consul/data /var/log/consul
sudo chmod 775 /etc/consul.d /var/lib/consul /var/lib/consul/data /var/log/consul
```

/etc/consul.d         - каталог конфигураций<br> 
/var/lib/consul/data  - каталог для данных ноды кластера DCS<br>
/var/log/consul       - каталог для логов<br>

#### 2.2 Настройка и запуск Consul кластера

__получаем ключ шифрования на любом хосте кластера (один раз)__

``` bash
consul keygen
> L5o6P57/auweOjNSgJ8sOhoMf4BbiaTyPnDw097p/kk=
```

полученное значение надо использовать в параметре "encrypt" конфигурации

__- настройка для первого запуска__<br>
__создаём файл /etc/consul.d/config.json__  - файл на каждом хосте кластера для первого запуска

``` json
{
     "bind_addr": "0.0.0.0",
     "advertise_addr": "{{ GetInterfaceIP `eth0` }}",
     "bootstrap_expect": 3,
     "client_addr": "0.0.0.0",
     "datacenter": "test-dcs-cluster",
     "node_name": "test-dcs1",
     "data_dir": "/var/lib/consul/data",
     "domain": "consul",
     "disable_update_check": true,
     "enable_local_script_checks": true,
     "dns_config": {
         "enable_truncate": true,
         "only_passing": true
     },
     "enable_syslog": true,
     "encrypt": "L5o6P57/auweOjNSgJ8sOhoMf4BbiaTyPnDw097p/kk=",
     "leave_on_terminate": true,
     "log_level": "INFO",
     "log_file": "/var/log/consul/",
     "log_rotate_duration": "24h",
     "log_rotate_max_files": 30,
     "rejoin_after_leave": true,
     "retry_join": [ "test-dcs1", "test-dcs2", "test-dcs3" ],
     "server": true,
     "ui_config": { "enabled": true },
     "primary_datacenter": "test-dcs-cluster",
     "acl": {
         "enabled": true,
         "default_policy": "deny",
         "enable_token_persistence": true
     }
}
```

__важные параметры__

``` text
"retry_join": [ test-dcs1, test-dcs2, test-dcs3 ]  - сразу перечислены хосты кластера DCS
"datacenter": "test-dcs-cluster"  - имя кластера DCS
"node_name": "test-dcs1"   -  имя текущей ноды кластера DCS ( у всех разное !)
"primary_datacenter": "test-dcs-cluster" - основной кластер DCS - текущий
"encrypt": "L5o6P57/auweOjNSgJ8sOhoMf4BbiaTyPnDw097p/kk=" - ключ шифрования кластера
```

__проверка правильности конфигурации__

``` bash
consul validate /etc/consul.d/config.json
> bootstrap_expect > 0: expecting 3 servers
> Configuration is valid!
```

__создаём файл для службы Consul__ - /usr/lib/systemd/system/consyl.service

``` text
[Unit]
Description=Consul Service Discovery Agent
Documentation=https://www.consul.io/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=consul
Group=consul
PIDFile=/run/consul/consul.pid
RuntimeDirectory=consul
PermissionsStartOnly=true
ExecStart=/usr/bin/consul agent -config-dir=/etc/consul.d -pid-file=/run/consul/consul.pid
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGINT
TimeoutStopSec=5
Restart=on-failure
SyslogIdentifier=consul

[Install]
WantedBy=multi-user.target
```

__настройка и запуск__
( на каждом узле кластера )

``` bash
sudo systemctl daemon-reload
sudo systemctl start consul
--
sudo systemctl status consul
```

#### 2.3 Настройка Consul ACL

__получение мастер токена__<br>
( на одном из хостов кластера)

``` bash
consul acl bootstrap
> AccessorID:       01db8610-18c0-52b3-ec45-34f15ca01f55
> SecretID:         b5eb2128-eb32-7cbe-77fe-45959c2a444a
> Description:      Bootstrap Token (Global Management)
> Local:            false
> Create Time:      2026-01-28 11:31:55.748746127 +0300 MSK
> Policies:
>   00000000-0000-0000-0000-000000000001 - global-management
```

нам нужно значение SecterID - это мастер токен<br>

если присвоить это значение переменной окружения CONSUL_HTTP_TOKEN<br>
то можно через команды консоли проверить состояние кластера

``` bash
export CONSUL_HTTP_TOKEN=b5eb2128-eb32-7cbe-77fe-45959c2a444a
consul members
> Node       Address             Status  Type    Build   Protocol  DC                Partition  Segment
> test-dcs1  192.168.1.181:8301  alive   server  1.22.3  2         test-dcs-cluster  default    <all>
> test-dcs2  192.168.1.182:8301  alive   server  1.22.3  2         test-dcs-cluster  default    <all>
> test-dcs3  192.168.1.183:8301  alive   server  1.22.3  2         test-dcs-cluster  default    <all>
```

для текущего пользователя можно сохранить в профайле данный токен<br>
(можно на каждом хосте кластера такую настройку сделать)
``` bash
test -f ~/.profile  &&  sed -i "/CONSUL_HTTP/d"  ~/.profile
echo "export CONSUL_HTTP_TOKEN=b5eb2128-eb32-7cbe-77fe-45959c2a444a" >> ~/.profile
```

только надо пересоздать сессию подключения

__создание политик ACL__ для сервисов кластера DCS<br>

** создадим файл  policy-agent.hcl  для описания простого агентского доступа

``` hcl
node_prefix "" {
  policy = "write"
}
node "" {
  policy = "write"
}
service_prefix "" {
  policy = "write"
}
service "" {
  policy = "write"
}
```

** и файл policy-patroni.hcl  для описания доступа сервиса Patroni

``` hcl
service_prefix "" {
   policy = "write"
}

session_prefix "" {
  policy = "write"
}

key_prefix "" {
  policy = "write"
}

node_prefix "" {
  policy = "read"
}

agent_prefix "" {
  policy = "read"
}
```

Загрузим эти описания на кластер DCS и получим токены доступа

``` bash
export CONSUL_HTTP_TOKEN=b5eb2128-eb32-7cbe-77fe-45959c2a444a

consul acl policy create -name "agent" -rules @./policy-agent.hcl
> ID:           8abe907f-00eb-eb15-44e7-c2f553ac623a
> Name:         agent
> ...

consul acl token create -description "Token for Agent" -policy-name agent
> AccessorID:       1f2d33c8-29db-d786-eaef-d9da0f256d46
> SecretID:         61655c0a-07ff-7bf7-e22a-c855e2ec0f39
> Description:      Token for Agent
> Local:            false
> Create Time:      2026-01-28 12:56:29.528666389 +0300 MSK
> Policies:
>    8abe907f-00eb-eb15-44e7-c2f553ac623a - agent

consul acl policy create -name "patroni" -rules @./policy-patroni.hcl
> ID:           d4b0e4d2-5a79-436e-295a-b9c6ceaf0254
> Name:         patroni
> ...

consul acl token create -description "Token for Patroni" -policy-name patroni
> AccessorID:       a71ec65a-baec-bb88-ed18-14717dbb0c2b
> SecretID:         ea5a1ccc-e063-20c6-46b8-d751e7750111
> Description:      Token for Patroni
> Local:            false
> Create Time:      2026-01-28 12:59:53.186638812 +0300 MSK
> Policies:
>    d4b0e4d2-5a79-436e-295a-b9c6ceaf0254 - patroni
```

__получили токены__ (значения SecretID) <br> 
для агента  - 61655c0a-07ff-7bf7-e22a-c855e2ec0f39<br>
для Patroni - ea5a1ccc-e063-20c6-46b8-d751e7750111<br>
будем их использовать при дальнейшей конфигурации

#### 2.4 Настройка ACL хостов кластера DCS

на каждом хосте кластера DCS настроим ACL доступ (как хост)

``` bash
export CONSUL_HTTP_TOKEN=b5eb2128-eb32-7cbe-77fe-45959c2a444a
consul acl set-agent-token agent 61655c0a-07ff-7bf7-e22a-c855e2ec0f39
> ACL token "agent" set successfully
```

#### 2.5 Consul - как агент на серверах Patroni

__на хостах СУБД__  
** выполняем п.2.1 - установка Consul
** выполняем п.2.2 - только
  ключ шифрования уже есть общий и для агентов<br>
  файл конфигурации /etc/consul.d/config.json

``` json
{
     "bind_addr": "{{ GetInterfaceIP `eth0` }}",
     "client_addr": "127.0.0.1",
     "datacenter": "test-dcs-cluster",
     "node_name": "test-db1",
     "data_dir": "/var/lib/consul/data",
     "disable_update_check": true,
     "enable_local_script_checks": true,
     "enable_syslog": true,
     "encrypt": "L5o6P57/auweOjNSgJ8sOhoMf4BbiaTyPnDw097p/kk=",
     "leave_on_terminate": true,
     "log_level": "WARN",
     "log_file": "/var/log/consul/",
     "log_rotate_duration": "24h",
     "log_rotate_max_files": 30,
     "retry_join": [ "test-dcs1", "test-dcs2", "test-dcs3" ],
     "server": false,
     "acl": {
         "enabled": true,
         "tokens": {
             "default": "master-token"
         }
     }
}
```
  файл службы  /usr/lib/systemd/system/consul.service - такой-же

  также проверяем валидность конфигурационного файла

  запускаем службу
``` bash
sudo systemctl daemon-reload
sudo systemctl start consul
--
sudo systemctl status consul
```  

## PostgreSQL

__- на 3-х хостах кластера СУБД - разворачиваем PostgresPro 1C 18__<br>
_-- останавливаем службу и проверяем установленные Локальные языки_<br>
_-- во время инициализации кластера PostgreSQL будет определять текущую "локаль" и использует её_<br>

``` bash
wget https://repo.postgrespro.ru/1c/1c-18/keys/pgpro-repo-add.sh
chmod +x ./pgpro-repo-add.sh
sudo ./pgpro-repo-add.sh 
sudo apt -y install postgrespro-1c-18 postgrespro-1c-18-dev
...
  -- останавливаем и выключаем автозагрузку 
sudo systemctl stop postgrespro-1c-17
sudo systemctl disable postgrespro-1c-17

  -- Patroni сам будет запускать сервер PostgreSQL

  -- проверяем текущую локаль
locale
> LANG=ru_RU.UTF-8
> LANGUAGE=
> LC_CTYPE="ru_RU.UTF-8"
> LC_NUMERIC="ru_RU.UTF-8"
> ...

  -- проверяем установленные локали
locale -a    
> C
> C.utf8
> en_US.utf8
> POSIX
> ru_RU.utf8

  -- локаль текущая ru_RU.UTF-8
  -- локаль en_US.utf8 установлена - будет использоваться для логов
```

## Patroni

#### Python - виртуальное окружение

#### Установка Patroni 

##  Vip-manager
