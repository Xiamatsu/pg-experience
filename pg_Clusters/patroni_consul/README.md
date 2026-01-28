# Настройка кластера Patroni + Consul + (VIP-manager или callback Patroni)

##  Оборудование и ПО

__- настроим 6 виртуальных машин__
```
- три хоста - кластер DSC   [ test-dcs1, test-dcs2, test-dcs3 ]
- три хоста - кластер СУБД  [ test-db1,  test-db2,  test-db3 ]
```

__- программное обеспечение__
```
- Astra Linux 1.8  (based on Debian 12)
- Consul 1.22
- Patroni 4.0.4 
- PostgresPro 1C 18  (free edition PostgresPro)
- Vip-manager 4.0
```

__- план__
```
- кластер  Consul + Patroni + Vip-manager
- как настроить синхронную реплику
- Patroni callback как замена Vip-manager
```

##  Consul - кластер

#### Установка Consul

на каждом хосте DCS загружаем выполняемый файл Consul и создаем рабочие каталоги<br>
(1.22.3 - последняя верси на январь 2026)

```
wget https://hashicorp-releases.mcs.mail.ru/consul/1.22.3/consul_1.22.3_linux_amd64.zip -O /tmp/consul.zip
sudo unzip /tmp/consul.zip -d /usr/bin
sudo chmod +x /usr/bin/consul

sudo groupadd consul
sudo useradd -m -d /var/lib/consul -g consul -r -c 'Consul DCS service' consul
sudo mkdir -p /etc/consul.d /var/lib/consul/data /var/log/consul
sudo chown -R consul: /etc/consul.d /var/lib/consul/data /var/log/consul
sudo chmod -R 775 /etc/consul.d /var/lib/consul/data /var/log/consul
```

/etc/consul.d         - каталог конфигураций<br> 
/var/lib/consul/data  - каталог для данных ноды кластера DCS<br>
/var/log/consul       - каталог для логов<br>


#### Первый запуск Consul кластера

__- настройка для первого запуска__<br>
__создаём файл /etc/consul.d/config.json__  - файл на каждом хосте кластера для первого запуска
```
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
     "encrypt": "encrypt-key",
     "leave_on_terminate": true,
     "log_level": "INFO",
     "log_file": "/var/log/consul/",
     "log_rotate_duration": "24h",
     "log_rotate_max_files": 30,
     "rejoin_after_leave": true,
     "retry_join": [ test-dcs1, test-dcs2, test-dcs3 ],
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
```
"retry_join": [ test-dcs1, test-dcs2, test-dcs3 ]  - сразу перечислены хосты кластера DCS
"datacenter": "test-dcs-cluster"  - имя кластера DCS
"node_name": "test-dcs1"   -  имя текущей ноды кластера DCS ( у всех разное )
"primary_datacenter": "test-dcs-cluster" - основной кластер DCS - текущий
```
__проверка правильности конфигурации__
```
sudo consul validate /etc/consul.d/config.json
```

__создаём файл для службы Consul__ - /usr/lib/systemd/system/consyl.service
```
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

__окончательная настройка и первый запуск__
```
sudo systemctl daemon-reload
sudo systemctl start consul
--
sudo systemctl status consul
```


#### Настройка Consul ACL 

#### Второй запуск Consul кластера

##  Consul - как агент на серверах Patroni

##  PostgreSQL


##  Patroni

#### Python - виртуальное окружение

#### Установка Patroni 

##  Vip-manager
