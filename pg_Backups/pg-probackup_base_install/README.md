### Базовая установка pg_probackup (free 2.5.15) 

```
- Astra Linux 1.8.1.12     (not free - for test)
- PostgresPro-1C-16        (free) (database taxi - chicago taxi 2023 - 3287 MB) 
- pg_probackup-16  2.5.15  (free)
```

###  Установка для  версии  Astra Linux  1.8

$\textsf{\color{blue}- Регистрируем репозиторий}$
```
sudo -- sh -c "echo 'deb https://repo.postgrespro.ru/pg_probackup/deb bookworm main-bookworm' > /etc/apt/sources.list.d/pg_probackup.list"

  -- Загружаем ключ GPG, преобразовываем и записываем в реестр   (  wget, gpg   -  должны быть установлены )
wget --progress=bar:force https://repo.postgrespro.ru/pg_probackup/keys/GPG-KEY-PG-PROBACKUP  -P /tmp/
sudo -- sh -c "gpg --dearmor < /tmp/GPG-KEY-PG-PROBACKUP > /etc/apt/trusted.gpg.d/pg_probackup.gpg"
sudo apt update
```

###  Установка для  версии  Ubuntu 2404

$\textsf{\color{blue}- Регистрируем репозиторий}$
```
sudo -- sh -c "echo 'deb https://repo.postgrespro.ru/pg_probackup/deb noble main-noble' > /etc/apt/sources.list.d/pg_probackup.list"

  -- Загружаем ключ GPG, преобразовываем и записываем в реестр   (  wget, gpg   -  должны быть установлены )
wget --progress=bar:force https://repo.postgrespro.ru/pg_probackup/keys/GPG-KEY-PG-PROBACKUP  -P /tmp/
sudo -- sh -c "gpg --dearmor < /tmp/GPG-KEY-PG-PROBACKUP > /etc/apt/trusted.gpg.d/pg_probackup.gpg"
sudo apt update
```



$\textsf{\color{blue}- Установка нужной версии программы ( для каждой версии PostgreSQL - своя программ pg-probackup )}$
```
sudo apt install pg-probackup-16
  ( при установке пакета первый "дефис" -  но при запуске программы  pg_probackup-16   !!!! )
  ( программа сразу установилась в каталог /usr/bin )
```

### Настройка - Создание пользователя админа бэкапов

$\textsf{\color{blue}- создадим пользователя  probackup  и выдадим права  в базе postgres ( для PG15+ )}$
```
sudo su - postgres
psql

BEGIN;
CREATE ROLE backup_admin WITH LOGIN REPLICATION password 'AbraBraa';
GRANT USAGE ON SCHEMA pg_catalog TO backup_admin;
GRANT EXECUTE ON FUNCTION pg_catalog.current_setting(text) TO backup_admin;
GRANT EXECUTE ON FUNCTION pg_catalog.set_config(text, text, boolean) TO backup_admin;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_is_in_recovery() TO backup_admin;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_start(text, boolean) TO backup_admin;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_stop(boolean) TO backup_admin;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_create_restore_point(text) TO backup_admin;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_switch_wal() TO backup_admin;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_last_wal_replay_lsn() TO backup_admin;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current() TO backup_admin;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current_snapshot() TO backup_admin;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_snapshot_xmax(txid_snapshot) TO backup_admin;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_control_checkpoint() TO backup_admin;
COMMIT;
```

- добавляем в файл ~postgres/.pgpass (права  обязательно  0600)
```
echo "*:*:*:backup_admin:AbraBraa" >> ~/.pgpass
chmod 600 ~/.pgpass
```

- добавляем в файл pg_hba.conf - следующие строки - желательно в начало правил
```
local   all             backup_admin                      md5
local   replication     backup_admin                      md5
```

- если предусматривается удаленный запуск pg_probackup - ещё добавляем ( для текущей сети )
```
host    all             backup_admin     samenet          md5
host    replication     backup_admin     samenet          md5
```

$\textsf{\color{blue}- перезагрузить конфиг незабываем}$
```
psql
select pg_reload_conf();
```

### Включение контрольных сумм

$\textsf{\color{blue}- если контрольные суммы не включены}$
```
sudo systemctl stop postgrespro-1c-16
sudo su - postgres
pg_checksums -e -D /var/lib/pgpro/1c-16/data
exit
sudo systemctl start postgrespro-1c-16
```

### Настройка программы pg_probackup

$\textsf{\color{blue}- создаём каталог для бэкапов ( или монтируем сетевой ресурс ) и назначаем переменную}$
```
mkdir /backups/test
echo "export BACKUP_PATH=/backups/test" >> ~/.profile
source ~/.profile
```

$\textsf{\color{blue}-  инициализируем каталог , добавляем инстанс, устанавливаем настройки по умолчанию}$
```
pg_probackup-16 init
pg_probackup-16 add-instance -D /var/lib/pgpro/1c-16/data --instance taxi
pg_probackup-16 set-config --instance taxi --pgdatabase=postgres -h /tmp -U backup_admin --compress-algorithm=zlib --compress-level=6
pg_probackup-16 set-config --instance taxi --retention-window=10 --retention-redundancy=2 
pg_probackup-16 set-config --instance taxi --log-level-file=INFO --log-filename=pimsdb-%d-%m-%y.log
pg_probackup-16 set-config --instance taxi --log-rotation-age=10d
```  
$\textsf{\color{orange}// в бесплатной версии нет компрессии zstd и lz4}$<br> 
$\textsf{\color{orange}//  - подключение к локальному серверу к БД postgres пользователем probackup}$<br>
$\textsf{\color{orange}//  - компрессия zlib с уровнем 6}$<br>
$\textsf{\color{orange}//  - архивы на 10 дней и 2 полных архива -  // retention-policy}$<br>
$\textsf{\color{orange}//  - сохранять логи с уровнем INFO - держать 10 дней  // папка для логов  /backups/test/log}$

$\textsf{\color{blue}-  посмотреть настройки для инстанса}$
```
pg_probackup-16 show-config --instance taxi
```

$\textsf{\color{blue}-  примеры команд бэкапа с дополнительными параметрами настройки}$ 
```
pg_probackup-16 backup --instance taxi -j 2 -b FULL --stream --temp-slot --delete-expired --merge-expired --backup-pg-log 
pg_probackup-16 backup --instance taxi -j 2 -b DELTA --stream --temp-slot --delete-expired --merge-expired --backup-pg-log 
```
$\textsf{\color{orange}//  - многопоточная - 2 потока}$<br>
$\textsf{\color{orange}//  - вид архива  (FULL, DELTA, PAGE, PTRACK)}$<br>
$\textsf{\color{orange}//  - режим  stream  -  после бэкапа загружаются wal}$<br>
$\textsf{\color{orange}//  - создается временный слот контрольной точки для корректного отрабатывания по wal файлам}$<br>
$\textsf{\color{orange}//  - удалять старые архивы согласно retention-policy}$<br>
$\textsf{\color{orange}//  - объединять старые архивы согласно retention-policy}$<br>
$\textsf{\color{orange}//  - сохранять также логи сервера postgresql в бэкап}$

$\textsf{\color{blue}-  посмотреть список бэкапов инстанса, или всех инстансов}$
```
pg_probackup-16 show --instance taxi
pg_probackup-16 show 
```

### Удаленный запуск pg_probackup


$\textsf{\color{blue}- для удалённого запуска pg-probackup c host1 к host2 сделаем доступ по ssh (user - postgres)}$
```
ssh-keygen -t ed25519
ssh-copy-id postgres@host2
```
// на сервере host2 - также настраивается пользователь backup_admin (см. выше)

$\textsf{\color{blue}- создаём отдельный инстанс на базу с host2}$
```
pg_probackup-16 add-instance -D /pgdata/main --instance=host2 --remote-host=host2 --remote-user=postgres -U backup_admin
pg_probackup-16 set-config --instance=host2 --remote-host=host2 --remote-user=postgres 
pg_probackup-16 set-config --instance=host2 --compress-algorithm=zlib --compress-level=6
```

$\textsf{\color{blue}- создание бэкапа}$
```
pg_probackup-16 backup --instance=host2 -b FULL --stream -j 2 
```




