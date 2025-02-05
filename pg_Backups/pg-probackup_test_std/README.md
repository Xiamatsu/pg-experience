### Тестирование pg_probackup-17 2.8.5 из комплекта PostgresPro Std 17

- виртуальная машина создана  в гипервизоре VMWare ESXi ( задания HW01,HW02,HW05 )
  параметры изменены на vCPU=4/vRAM=8G/vHDD=40Gb + vHDD2=25Gb + vHDD3=50Gb
- установлена  ubuntu-2404 (host ubutest; user boss; ip 192.168.1.244)
- установлен PostgreSQL версии PostgresPro-1c-16 


$\textsf{\color{blue}- Но - установим PostgresPro-Std-17 (not free) на порт 5433}$<br>
$\textsf{\color{blue}- репозиторий std версии содержит pg-probackup}$
```
wget https://repo.postgrespro.ru/std/std-17/keys/pgpro-repo-add.sh 
chmod +x ./pgpro-repo-add.sh
sudo ./pgpro-repo-add.sh 
sudo apt install postgrespro-std-17
```

$\textsf{\color{blue}- Загрузим демо базу данных ( предварительно создав пустую базу air )}$ 
```
postgres@ubutest:~$ pg_restore -p 5433 --format=custom --no-owner -d air /tmp/postgres_air_2024.backup
postgres@ubutest:~$ psql -p 5433
psql (17.2)
Введите "help", чтобы получить справку.

postgres=# \l+
                                                                                             Список баз данных
    Имя    | Владелец | Кодировка | Провайдер локали | LC_COLLATE  |  LC_CTYPE   | Локаль | Правила ICU |     Права доступа     | Размер  | Табл. пространство |                  Описание
-----------+----------+-----------+------------------+-------------+-------------+--------+-------------+-----------------------+---------+----------------------------------------------------------------
 air       | postgres | UTF8      | icu              | ru_RU.UTF-8 | ru_RU.UTF-8 | ru-RU  |             |                       | 7992 MB | pg_default         |
 postgres  | postgres | UTF8      | icu              | ru_RU.UTF-8 | ru_RU.UTF-8 | ru-RU  |             |                       | 7477 kB | pg_default         | default administrative connection database
 template0 | postgres | UTF8      | icu              | ru_RU.UTF-8 | ru_RU.UTF-8 | ru-RU  |             | =c/postgres          +| 7321 kB | pg_default         | unmodifiable empty database
           |          |           |                  |             |             |        |             | postgres=CTc/postgres |         |                    |
 template1 | postgres | UTF8      | icu              | ru_RU.UTF-8 | ru_RU.UTF-8 | ru-RU  |             | =c/postgres          +| 7549 kB | pg_default         | default template for new databases
           |          |           |                  |             |             |        |             | postgres=CTc/postgres |         |                    |
(4 строки)

```

$\textsf{\color{blue}- Данная версия PostgreSQL содержит в репозитории pg-probackup 2.8.5 (not free) - установим}$
```
sudo apt install pg-probackup-std-17
-- добавим ссылку на программу
sudo ln -s /opt/pgpro/std-17/bin/pg_probackup /usr/bin/pg_probackup
```
**// при  установке имя пакета с "-" (дефис) после "pg"**<br>
**// а сама программа с "_" (подчеркивание) после "pg"**<br>
**// для бесплатных версий также пакет pg-probackup-16 - программа pg_probackup-16**


### Настройка pg_probackup

$\textsf{\color{blue}-- создаём каталог для бэкапов}$
```
boss@ubutest:~$ sudo mkdir /backups/probackup
boss@ubutest:~$ sudo chown postgres: /backups/probackup
```
$\textsf{\color{blue}-- инициализируем каталог}$
```
sudo -i -u postgres
postgres@ubutest:~$ pg_probackup init -B /backups/probackup
INFO: Backup catalog '/backups/probackup' successfully initialized
```
$\textsf{\color{blue}-- установим переменную на данный каталог и можно использовать без ключа -B}$
```
postgres@ubutest:~$ echo "export BACKUP_PATH=/backups/probackup" >> ~/.profile
postgres@ubutest:~$ source ~/.profile
```
$\textsf{\color{blue}-- добавляем инстанс ( учитываем порт 5433 )}$
```
postgres@ubutest:~$ pg_probackup add-instance -D /pgdata/air -p 5433 --instance=air
INFO: Instance 'air' successfully initialized
```
$\textsf{\color{blue}-- посмотрим какие параметры по умолчанию назначены}$
```
postgres@ubutest:~$ pg_probackup show-config --instance=air
# Backup instance information
pgdata = /pgdata/air
system-identifier = 7460572854766238660
xlog-seg-size = 16777216
# Connection parameters
pgdatabase = postgres
pgport = 5433
# Archive parameters
archive-timeout = 5min
# Logging parameters
log-level-console = INFO
log-level-file = OFF
log-format-console = PLAIN
log-format-file = PLAIN
log-filename = pg_probackup.log
log-rotation-size = 0TB
log-rotation-age = 0d
# Retention parameters
retention-redundancy = 0
retention-window = 0
wal-depth = 0
# Compression parameters
compress-algorithm = none
compress-level = 1
# Remote access parameters
remote-proto = ssh
# Backup instance information
write-rate-limit = 0GBps
```

$\textsf{\color{blue}- сдедаем полный архив с разной упаковкой и потоками}$<br>
$\textsf{\color{blue}  // здесь на тестах работаем от пользователя postgres - что нежелательно - WARNING предупреждение}$<br>
$\textsf{\color{blue}  // (как настраивается отдельная роль - см. ссылку в конце статьи)}$

```
postgres@ubutest:~$ pg_probackup backup --instance=air -b FULL --stream
INFO: Backup start, pg_probackup version: 2.8.5, instance: air, backup ID: SQ73B8, backup mode: FULL, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
WARNING: Current PostgreSQL role is superuser. It is not recommended to run pg_probackup under superuser.
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: PGDATA size: 8014MB
INFO: Current Start LSN: 2/90000028, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 1m:25s
INFO: wait for pg_stop_backup()
INFO: pg_stop_backup() successfully executed
INFO: stop_stream_lsn 2/91000000 currentpos 2/91000000
INFO: backup->stop_lsn 2/900001C8
INFO: Getting the Recovery Time from WAL
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 2s
INFO: Validating backup SQ73B8
INFO: Backup SQ73B8 data files are valid
INFO: Backup SQ73B8 resident size: 8038MB
INFO: Backup SQ73B8 completed

postgres@ubutest:~$ pg_probackup backup --instance=air -b FULL --stream --compress-algorithm=lz4 --compress-level=9
...
postgres@ubutest:~$ pg_probackup backup --instance=air -b FULL --stream --compress-algorithm=lz4 --compress-level=9 -j 3
...
postgres@ubutest:~$ pg_probackup backup --instance=air -b FULL --stream --compress-algorithm=zstd --compress-level=3 -j 3
...
postgres@ubutest:~$ pg_probackup backup --instance=air -b FULL --stream --compress-algorithm=zlib --compress-level=3 -j 3
...
postgres@ubutest:~$ pg_probackup backup --instance=air -b FULL --stream --compress-algorithm=zlib --compress-level=6 -j 3
...
```

$\textsf{\color{blue}- статистика по алгоритмам и сжатию ( быстрее и лучше сжимает - zstd , но вместе с lz4 его нет в бесплатной версии )}$
```
postgres@ubutest:~$ pg_probackup show --instance=air
===================================================================================================================================================
 Instance  Version  ID      Recovery Time                  Mode  WAL Mode  TLI    Time    Data   WAL  Zalg  Zratio  Start LSN   Stop LSN    Status
===================================================================================================================================================
 air       17       SQ73YG  2025-01-16 19:12:29.943367+00  FULL  STREAM    1/0  1m:28s  2311MB  16MB  zlib    3.47  2/9A000028  2/9A0001C8  OK      
 air       17       SQ73W0  2025-01-16 19:10:29.710545+00  FULL  STREAM    1/0     56s  2370MB  16MB  zlib    3.38  2/98000028  2/980001C8  OK
 air       17       SQ73UJ  2025-01-16 19:09:16.479732+00  FULL  STREAM    1/0     36s  2286MB  16MB  zstd    3.51  2/96000028  2/960001C8  OK
 air       17       SQ73RO  2025-01-16 19:07:56.480770+00  FULL  STREAM    1/0     59s  3476MB  16MB   lz4    2.31  2/94000028  2/940001C8  OK
 air       17       SQ73N7  2025-01-16 19:06:38.375030+00  FULL  STREAM    1/0  2m:20s  3476MB  16MB   lz4    2.31  2/92000028  2/920001C8  OK
 air       17       SQ73B8  2025-01-16 18:58:33.749503+00  FULL  STREAM    1/0  1m:27s  8022MB  16MB  none    1.00  2/90000028  2/900001C8  OK
```

$\textsf{\color{blue}- удалим большинство бэкапов и оставим последний}$
```
postgres@ubutest:~$ pg_probackup delete --instance=air -i SQ73B8
INFO: Resident data size to free by delete of backup SQ73B8 : 8038MB
INFO: Delete: SQ73B8 2025-01-16 18:58:33+00
postgres@ubutest:~$ pg_probackup delete --instance=air -i SQ73N7
INFO: Resident data size to free by delete of backup SQ73N7 : 3492MB
INFO: Delete: SQ73N7 2025-01-16 19:06:38+00
postgres@ubutest:~$ pg_probackup delete --instance=air -i SQ73RO
INFO: Resident data size to free by delete of backup SQ73RO : 3492MB
INFO: Delete: SQ73RO 2025-01-16 19:07:56+00
postgres@ubutest:~$ pg_probackup delete --instance=air -i SQ73UJ
INFO: Resident data size to free by delete of backup SQ73UJ : 2302MB
INFO: Delete: SQ73UJ 2025-01-16 19:09:16+00
postgres@ubutest:~$ pg_probackup delete --instance=air -i SQ73W0
INFO: Resident data size to free by delete of backup SQ73W0 : 2386MB
INFO: Delete: SQ73W0 2025-01-16 19:10:29+00
```

### Восстановление "кластера" на другой ВМ

$\textsf{\color{blue}- настроим доступ к папке /backups с других ВМ по сети как NFS}$
```
boss@ubutest:~$ sudo apt update
...
boss@ubutest:~$ sudo apt install nfs-kernel-server
...
boss@ubutest:~$ sudo -- sh -c "echo '/backups  192.168.1.0/24(rw,no_root_squash,no_subtree_check)' >> /etc/exports"
boss@ubutest:~$ sudo exportfs -a
  -- выдадим полный доступ для всех (other) 
boss@ubutest:~$ sudo chmod -R o+rwX /backups/
```

$\textsf{\color{blue}- есть вторая ВМ с установленным Astra Linux 1.8  и также PostgresPro-std-17}$<br>
$\textsf{\color{blue}  настроим доступ к диску /backups первой ВМ по сети}$
```
boss@astra8:~$ sudo apt update
...
boss@astra8:~$ sudo apt install nfs-common
...
  -- временно подключим NFS ресурс
boss@astra8:~$ sudo mkdir /backups
boss@astra8:~$ sudo mount -t nfs 192.168.1.244:/backups /backups 
```

$\textsf{\color{blue}- настраивать pg-probackup и базу на второй ВМ - не надо ( работаем от пользователя postgres )}$<br>
$\textsf{\color{blue}  проверим что видим бэкапы по сети}$
```
postgres@astra8:~$ pg_probackup show -B /backups/probackup/

BACKUP INSTANCE 'air'
===================================================================================================================================================
 Instance  Version  ID      Recovery Time                  Mode  WAL Mode  TLI    Time    Data   WAL  Zalg  Zratio  Start LSN   Stop LSN    Status
===================================================================================================================================================
 air       17       SQ73YG  2025-01-17 00:12:29.943367+05  FULL  STREAM    1/0  1m:28s  2311MB  16MB  zlib    3.47  2/9A000028  2/9A0001C8  OK
```

$\textsf{\color{blue}- восстановим "кластер" из бэкапа на второй ВМ ( /var/lib/pgpro/std-17/data )}$
```
boss@astra8:~$ sudo systemctl stop postgrespro-std-17.service
boss@astra8:~$ sudo rm -r /var/lib/pgpro/std-17/data
boss@astra8:~$ sudo su - postgres
postgres@astra8:~$ pg_probackup restore -B /backups/probackup/ --instance=air -D /var/lib/pgpro/std-17/data
INFO: Validating backup SQ73YG
INFO: Backup SQ73YG data files are valid
INFO: Backup SQ73YG WAL segments are valid
INFO: Backup SQ73YG is valid.
INFO: Restoring the database from backup SQ73YG
INFO: Start restoring backup files. PGDATA size: 8030MB
INFO: Backup files are restored. Transferred bytes: 8030MB, time elapsed: 1m:16s
INFO: Restore incremental ratio (less is better): 100% (8030MB/8030MB)
INFO: Syncing restored files to disk
INFO: Restored backup files are synced, time elapsed: 1s
INFO: Restore of backup SQ73YG completed.
```

$\textsf{\color{blue}- запускаем СУБД}$
```
boss@astra8:~$ sudo systemctl start postgrespro-std-17.service
boss@astra8:~$ sudo systemctl status postgrespro-std-17.service
● postgrespro-std-17.service - Postgres Pro std 17 database server
     Loaded: loaded (/lib/systemd/system/postgrespro-std-17.service; enabled; preset: enabled)
     Active: active (running) since Sat 2025-01-18 11:16:34 +05; 7s ago
    Process: 82934 ExecStartPre=/opt/pgpro/std-17/bin/check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
   Main PID: 82936 (postgres)
      Tasks: 7 (limit: 2307)
     Memory: 95.7M
        CPU: 118ms
     CGroup: /system.slice/postgrespro-std-17.service
             ├─82936 /opt/pgpro/std-17/bin/postgres -D /var/lib/pgpro/std-17/data
             ├─82943 "postgres: logger "
             ├─82944 "postgres: checkpointer "
             ├─82945 "postgres: background writer "
             ├─82953 "postgres: walwriter "
             ├─82954 "postgres: autovacuum launcher "
             └─82955 "postgres: logical replication launcher "

янв 18 11:16:31 astra8 systemd[1]: Starting postgrespro-std-17.service - Postgres Pro std 17 database server...
янв 18 11:16:32 astra8 postgres[82936]: 2025-01-18 06:16:32.174 UTC [82936] LOG:  redirecting log output to logging collector process
янв 18 11:16:32 astra8 postgres[82936]: 2025-01-18 06:16:32.174 UTC [82936] HINT:  Future log output will appear in directory "log".
янв 18 11:16:34 astra8 systemd[1]: Started postgrespro-std-17.service - Postgres Pro std 17 database server.
```

$\textsf{\color{blue}- прорверим наличие базы}$
```
boss@astra8:~$ sudo su - postgres
postgres@astra8:~$ psql -p 5433
WARNING:  database "postgres" has a collation version mismatch
ПОДРОБНОСТИ:  The database was created using collation version 153.121.44.8, but the operating system provides version 153.120.42.
ПОДСКАЗКА:  Rebuild all objects in this database that use the default collation and run ALTER DATABASE postgres REFRESH COLLATION VERSION, or build PostgreSQL with the right library version.
psql (17.2)
Введите "help", чтобы получить справку.

postgres=# \l+
                                                                                             Список баз данных
    Имя    | Владелец | Кодировка | Провайдер локали | LC_COLLATE  |  LC_CTYPE   | Локаль | Правила ICU |     Права доступа     | Размер  | Табл. пространство |       Описание
-----------+----------+-----------+------------------+-------------+-------------+--------+-------------+-----------------------+---------+--------------------+--------------------------------------------
 air       | postgres | UTF8      | icu              | ru_RU.UTF-8 | ru_RU.UTF-8 | ru-RU  |             |                       | 7991 MB | pg_default         |
 postgres  | postgres | UTF8      | icu              | ru_RU.UTF-8 | ru_RU.UTF-8 | ru-RU  |             |                       | 7477 kB | pg_default         | default administrative connection database
 template0 | postgres | UTF8      | icu              | ru_RU.UTF-8 | ru_RU.UTF-8 | ru-RU  |             | =c/postgres          +| 7321 kB | pg_default         | unmodifiable empty database
           |          |           |                  |             |             |        |             | postgres=CTc/postgres |         |                    |
 template1 | postgres | UTF8      | icu              | ru_RU.UTF-8 | ru_RU.UTF-8 | ru-RU  |             | =c/postgres          +| 7393 kB | pg_default         | default template for new databases
           |          |           |                  |             |             |        |             | postgres=CTc/postgres |         |                    |
(4 строки)
```
$\textsf{\color{orange}// база в наличии - востановление успешно, если не считать проблемы с сортировкой на разных версиях Linux}$<br>
$\textsf{\color{orange}// подобная проблема была HW05 - там провайдер был libc здесь icu - разные версии на Astra Linux и Ubuntu}$<br>
$\textsf{\color{orange}// Надо следить за этим и при реализации реплики и кластеров Patroni ( Linux должен быть СТРОГО одной версии )}$ 


$\textsf{\color{blue}- определить версию провайдера ICU можно так}$
```
boss@astra8:~$ uconv -V
uconv v2.1  ICU 72.1

boss@ubutest:~$ uconv -V
uconv v2.1  ICU 74.2
```

$\textsf{\color{blue}- привести в норму сортироку можно командами (для каждой базы включая template1)}$
```
psql -p 5433
ALTER DATABASE postgres REFRESH COLLATION VERSION;
REINDEX DATABASE postgres;
\c air
ALTER DATABASE air REFRESH COLLATION VERSION;
REINDEX DATABASE air;
\c template1
ALTER DATABASE template1 REFRESH COLLATION VERSION;
REINDEX DATABASE template1;
```


### Под нагрузкой снимаем бэкап


$\textsf{\color{blue}- запускаем виртуальную нагрузку со второй ВМ}$
```
  -- установим sysbench
boss@astra8:~$ sudo apt -y install sysbench
  -- создадим пользователя и базу данных на первой ВМ
boss@astra8:~$ sudo su - postgres
postgres@astra8:~$ psql -p 5433 -h 192.168.1.244
psql (17.2)
Введите "help", чтобы получить справку.

postgres=# create role sbtest with login password 'sbtest';
CREATE ROLE
postgres=# create database sbtest owner sbtest;
CREATE DATABASE

  -- инициализируем sysbench ( 24 таблицы, 100000 записей )
postgres@astra8:~$ sysbench \
--db-driver=pgsql \
--oltp-table-size=100000 \
--oltp-tables-count=24 \
--threads=1 \
--pgsql-host=192.168.1.244 \
--pgsql-port=5433 \
--pgsql-user=sbtest \
--pgsql-password=sbtest \
--pgsql-db=sbtest \
/usr/share/sysbench/tests/include/oltp_legacy/parallel_prepare.lua \
run
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 1
Initializing random number generator from current time


Initializing worker threads...

Threads started!

thread prepare0
Creating table 'sbtest1'...
Inserting 100000 records into 'sbtest1'
Creating secondary indexes on 'sbtest1'...
Creating table 'sbtest2'...
Inserting 100000 records into 'sbtest2'
Creating secondary indexes on 'sbtest2'...
...
Creating table 'sbtest24'...
Inserting 100000 records into 'sbtest24'
Creating secondary indexes on 'sbtest24'...
SQL statistics:
    queries performed:
        read:                            0
        write:                           912
        other:                           48
        total:                           960
    transactions:                        1      (0.03 per sec.)
    queries:                             960    (27.22 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          35.2653s
    total number of events:              1

Latency (ms):
         min:                                35264.72
         avg:                                35264.72
         max:                                35264.72
         95th percentile:                    35191.04
         sum:                                35264.72

Threads fairness:
    events (avg/stddev):           1.0000/0.00
    execution time (avg/stddev):   35.2647/0.00
```

$\textsf{\color{blue}- запускаем со второй ВМ нагрузку чтения-записи ПРОБНУЮ}$<br> 
$\textsf{\color{blue}  ( 8 потоков, 24 таблицы, 100000 записей, 120 сек, отчет каждые 4 сек )}$<br>
$\textsf{\color{blue}  нагружает первыую ВМ на 2/3 }$
```
postgres@astra8:~$ sysbench \
--db-driver=pgsql \
--report-interval=4 \
--oltp-table-size=100000 \
--oltp-tables-count=24 \
--threads=8 \
--time=120 \
--pgsql-host=192.168.1.244 \
--pgsql-port=5433 \
--pgsql-user=sbtest \
--pgsql-password=sbtest \
--pgsql-db=sbtest \
/usr/share/sysbench/tests/include/oltp_legacy/oltp.lua \
run

sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 8
Report intermediate results every 4 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 4s ] thds: 8 tps: 604.05 qps: 12104.75 (r/w/o: 8476.70/2417.95/1210.10) lat (ms,95%): 21.89 err/s: 0.00 reconn/s: 0.00
[ 8s ] thds: 8 tps: 883.25 qps: 17670.56 (r/w/o: 12370.04/3533.26/1767.26) lat (ms,95%): 11.45 err/s: 0.25 reconn/s: 0.00
[ 12s ] thds: 8 tps: 882.73 qps: 17651.20 (r/w/o: 12354.79/3531.19/1765.22) lat (ms,95%): 12.08 err/s: 0.00 reconn/s: 0.00
[ 16s ] thds: 8 tps: 898.00 qps: 17959.97 (r/w/o: 12571.98/3591.99/1796.00) lat (ms,95%): 11.65 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 8 tps: 910.99 qps: 18215.51 (r/w/o: 12751.33/3641.70/1822.48) lat (ms,95%): 11.24 err/s: 0.00 reconn/s: 0.00
[ 24s ] thds: 8 tps: 911.52 qps: 18231.44 (r/w/o: 12762.31/3646.09/1823.04) lat (ms,95%): 11.04 err/s: 0.00 reconn/s: 0.00
[ 28s ] thds: 8 tps: 837.26 qps: 16753.21 (r/w/o: 11727.40/3351.29/1674.52) lat (ms,95%): 13.70 err/s: 0.00 reconn/s: 0.00
[ 32s ] thds: 8 tps: 741.75 qps: 14825.71 (r/w/o: 10376.22/2966.24/1483.25) lat (ms,95%): 22.69 err/s: 0.00 reconn/s: 0.00
[ 36s ] thds: 8 tps: 864.49 qps: 17291.96 (r/w/o: 12105.79/3457.19/1728.97) lat (ms,95%): 13.46 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 8 tps: 944.75 qps: 18902.56 (r/w/o: 13231.80/3780.76/1890.01) lat (ms,95%): 10.65 err/s: 0.00 reconn/s: 0.00
[ 44s ] thds: 8 tps: 844.51 qps: 16889.19 (r/w/o: 11821.88/3378.29/1689.02) lat (ms,95%): 13.22 err/s: 0.00 reconn/s: 0.00
[ 48s ] thds: 8 tps: 908.00 qps: 18158.75 (r/w/o: 12710.50/3632.25/1816.00) lat (ms,95%): 11.65 err/s: 0.00 reconn/s: 0.00
[ 52s ] thds: 8 tps: 949.68 qps: 18989.80 (r/w/o: 13294.49/3795.96/1899.36) lat (ms,95%): 10.65 err/s: 0.00 reconn/s: 0.00
[ 56s ] thds: 8 tps: 941.07 qps: 18814.69 (r/w/o: 13168.76/3763.79/1882.14) lat (ms,95%): 10.84 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 8 tps: 915.50 qps: 18316.00 (r/w/o: 12821.25/3663.75/1831.00) lat (ms,95%): 11.04 err/s: 0.00 reconn/s: 0.00
[ 64s ] thds: 8 tps: 949.25 qps: 18980.76 (r/w/o: 13287.26/3795.25/1898.25) lat (ms,95%): 10.65 err/s: 0.00 reconn/s: 0.00
[ 68s ] thds: 8 tps: 936.75 qps: 18746.41 (r/w/o: 13124.19/3748.23/1873.99) lat (ms,95%): 10.84 err/s: 0.00 reconn/s: 0.00
[ 72s ] thds: 8 tps: 937.75 qps: 18754.78 (r/w/o: 13128.52/3750.76/1875.50) lat (ms,95%): 11.04 err/s: 0.00 reconn/s: 0.00
[ 76s ] thds: 8 tps: 929.74 qps: 18586.50 (r/w/o: 13007.82/3719.20/1859.47) lat (ms,95%): 10.65 err/s: 0.00 reconn/s: 0.00
[ 80s ] thds: 8 tps: 957.26 qps: 19153.77 (r/w/o: 13410.44/3828.80/1914.53) lat (ms,95%): 10.65 err/s: 0.00 reconn/s: 0.00
[ 84s ] thds: 8 tps: 707.74 qps: 14146.75 (r/w/o: 9899.82/2831.20/1415.72) lat (ms,95%): 29.72 err/s: 0.00 reconn/s: 0.00
[ 88s ] thds: 8 tps: 790.76 qps: 15814.21 (r/w/o: 11069.65/3163.29/1581.27) lat (ms,95%): 14.21 err/s: 0.00 reconn/s: 0.00
[ 92s ] thds: 8 tps: 839.76 qps: 16793.86 (r/w/o: 11756.83/3357.27/1679.76) lat (ms,95%): 12.75 err/s: 0.00 reconn/s: 0.00
[ 96s ] thds: 8 tps: 935.49 qps: 18715.12 (r/w/o: 13100.91/3742.72/1871.49) lat (ms,95%): 10.84 err/s: 0.25 reconn/s: 0.00
[ 100s ] thds: 8 tps: 930.01 qps: 18596.23 (r/w/o: 13016.16/3720.30/1859.77) lat (ms,95%): 10.84 err/s: 0.00 reconn/s: 0.00
[ 104s ] thds: 8 tps: 924.75 qps: 18495.74 (r/w/o: 12946.99/3699.00/1849.75) lat (ms,95%): 11.24 err/s: 0.00 reconn/s: 0.00
[ 108s ] thds: 8 tps: 926.50 qps: 18538.51 (r/w/o: 12977.01/3708.00/1853.50) lat (ms,95%): 10.84 err/s: 0.50 reconn/s: 0.00
[ 112s ] thds: 8 tps: 945.49 qps: 18914.54 (r/w/o: 13243.10/3780.21/1891.23) lat (ms,95%): 11.04 err/s: 0.00 reconn/s: 0.00
[ 116s ] thds: 8 tps: 927.95 qps: 18559.82 (r/w/o: 12990.85/3712.81/1856.16) lat (ms,95%): 10.84 err/s: 0.00 reconn/s: 0.00
[ 120s ] thds: 8 tps: 936.79 qps: 18731.78 (r/w/o: 13112.30/3745.66/1873.83) lat (ms,95%): 10.65 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            1490538
        write:                           425853
        other:                           212941
        total:                           2129332
    transactions:                        106463 (886.74 per sec.)
    queries:                             2129332 (17735.43 per sec.)
    ignored errors:                      4      (0.03 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          120.0598s
    total number of events:              106463

Latency (ms):
         min:                                    4.03
         avg:                                    9.02
         max:                                  163.70
         95th percentile:                       11.87
         sum:                               960000.46

Threads fairness:
    events (avg/stddev):           13307.8750/617.07
    execution time (avg/stddev):   120.0001/0.02
```

$\textsf{\color{blue}- запускаем на второй ВМ снова нагрузку}$<br>
$\textsf{\color{blue}- а на первой ВМ сразу запускаем создание бэкапа}$ 
```
postgres@ubutest:~$ pg_probackup backup --instance=air -b FULL --stream --compress-algorithm=zstd --compress-level=3 -j 3
INFO: Backup start, pg_probackup version: 2.8.5, instance: air, backup ID: SQBPX7, backup mode: FULL, wal mode: STREAM, remote: false, compress-algorithm: zstd, compress-level: 3
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
WARNING: Current PostgreSQL role is superuser. It is not recommended to run pg_probackup under superuser.
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: PGDATA size: 8679MB
INFO: Current Start LSN: 3/9016968, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 40s
INFO: wait for pg_stop_backup()
INFO: pg_stop_backup() successfully executed
INFO: stop_stream_lsn 3/1403B188 currentpos 3/1403B188
INFO: backup->stop_lsn 3/13125660
INFO: Getting the Recovery Time from WAL
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 3s
INFO: Validating backup SQBPX7
INFO: Backup SQBPX7 data files are valid
INFO: Backup SQBPX7 resident size: 2762MB
INFO: Backup SQBPX7 completed
 
postgres@ubutest:~$ pg_probackup show --instance=air
====================================================================================================================================================
 Instance  Version  ID      Recovery Time                  Mode  WAL Mode  TLI    Time    Data    WAL  Zalg  Zratio  Start LSN   Stop LSN    Status
====================================================================================================================================================
 air       17       SQBPX7  2025-01-19 06:56:36.608318+00  FULL  STREAM    1/0     44s  2570MB  192MB  zstd    3.38  3/9016968   3/13125660  OK
 air       17       SQBPTA  2025-01-19 06:54:11.369512+00  FULL  STREAM    1/0     40s  2569MB   16MB  zstd    3.38  3/2000028   3/20001C8   OK
 air       17       SQ73YG  2025-01-16 19:12:29.943367+00  FULL  STREAM    1/0  1m:28s  2311MB   16MB  zlib    3.47  2/9A000028  2/9A0001C8  OK
```

$\textsf{\color{blue}- статистика нагрузочного запуска на второй ВМ}$
```
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 8
Report intermediate results every 4 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 4s ] thds: 8 tps: 789.39 qps: 15805.78 (r/w/o: 11066.94/3158.06/1580.78) lat (ms,95%): 13.22 err/s: 0.00 reconn/s: 0.00
[ 8s ] thds: 8 tps: 750.22 qps: 15016.94 (r/w/o: 10512.11/3004.39/1500.44) lat (ms,95%): 23.52 err/s: 0.00 reconn/s: 0.00
[ 12s ] thds: 8 tps: 506.77 qps: 10134.88 (r/w/o: 7093.52/2027.58/1013.79) lat (ms,95%): 22.69 err/s: 0.00 reconn/s: 0.00
[ 16s ] thds: 8 tps: 457.50 qps: 9140.49 (r/w/o: 6397.50/1827.75/915.25) lat (ms,95%): 23.95 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 8 tps: 499.49 qps: 9992.13 (r/w/o: 6996.91/1996.23/998.99) lat (ms,95%): 21.89 err/s: 0.00 reconn/s: 0.00
[ 24s ] thds: 8 tps: 524.76 qps: 10491.19 (r/w/o: 7341.38/2100.54/1049.27) lat (ms,95%): 19.65 err/s: 0.00 reconn/s: 0.00
[ 28s ] thds: 8 tps: 357.25 qps: 7155.50 (r/w/o: 5010.25/1430.25/715.00) lat (ms,95%): 26.68 err/s: 0.00 reconn/s: 0.00
[ 32s ] thds: 8 tps: 508.74 qps: 10163.88 (r/w/o: 7115.67/2030.98/1017.24) lat (ms,95%): 22.69 err/s: 0.00 reconn/s: 0.00
[ 36s ] thds: 8 tps: 485.25 qps: 9713.77 (r/w/o: 6800.27/1942.75/970.75) lat (ms,95%): 23.52 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 8 tps: 381.50 qps: 7631.24 (r/w/o: 5342.24/1526.00/763.00) lat (ms,95%): 24.38 err/s: 0.00 reconn/s: 0.00
[ 44s ] thds: 8 tps: 532.01 qps: 10637.37 (r/w/o: 7444.08/2129.52/1063.76) lat (ms,95%): 21.89 err/s: 0.00 reconn/s: 0.00
[ 48s ] thds: 8 tps: 587.49 qps: 11746.86 (r/w/o: 8222.90/2348.72/1175.24) lat (ms,95%): 20.00 err/s: 0.00 reconn/s: 0.00
[ 52s ] thds: 8 tps: 543.49 qps: 10873.40 (r/w/o: 7612.43/2173.98/1086.99) lat (ms,95%): 31.94 err/s: 0.00 reconn/s: 0.00
[ 56s ] thds: 8 tps: 514.01 qps: 10277.11 (r/w/o: 7193.83/2055.27/1028.01) lat (ms,95%): 23.52 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 8 tps: 514.24 qps: 10292.11 (r/w/o: 7203.90/2059.72/1028.49) lat (ms,95%): 23.10 err/s: 0.00 reconn/s: 0.00
[ 64s ] thds: 8 tps: 844.50 qps: 16878.57 (r/w/o: 11814.80/3374.51/1689.26) lat (ms,95%): 17.95 err/s: 0.00 reconn/s: 0.00
[ 68s ] thds: 8 tps: 913.75 qps: 18282.44 (r/w/o: 12798.46/3656.49/1827.49) lat (ms,95%): 11.24 err/s: 0.00 reconn/s: 0.00
[ 72s ] thds: 8 tps: 934.51 qps: 18689.73 (r/w/o: 13082.66/3738.05/1869.02) lat (ms,95%): 10.84 err/s: 0.00 reconn/s: 0.00
[ 76s ] thds: 8 tps: 927.25 qps: 18542.09 (r/w/o: 12980.31/3707.27/1854.51) lat (ms,95%): 11.24 err/s: 0.00 reconn/s: 0.00
[ 80s ] thds: 8 tps: 927.75 qps: 18560.48 (r/w/o: 12991.24/3713.00/1856.25) lat (ms,95%): 11.04 err/s: 0.25 reconn/s: 0.00
[ 84s ] thds: 8 tps: 826.50 qps: 16524.32 (r/w/o: 11567.30/3304.01/1653.01) lat (ms,95%): 13.70 err/s: 0.00 reconn/s: 0.00
[ 88s ] thds: 8 tps: 869.99 qps: 17402.09 (r/w/o: 12182.14/3479.97/1739.98) lat (ms,95%): 13.46 err/s: 0.00 reconn/s: 0.00
[ 92s ] thds: 8 tps: 897.51 qps: 17943.43 (r/w/o: 12558.63/3589.79/1795.02) lat (ms,95%): 11.65 err/s: 0.00 reconn/s: 0.00
[ 96s ] thds: 8 tps: 728.69 qps: 14581.78 (r/w/o: 10208.65/2915.76/1457.38) lat (ms,95%): 26.68 err/s: 0.00 reconn/s: 0.00
[ 100s ] thds: 8 tps: 915.04 qps: 18295.55 (r/w/o: 12805.56/3660.41/1829.58) lat (ms,95%): 11.87 err/s: 0.00 reconn/s: 0.00
[ 104s ] thds: 8 tps: 661.52 qps: 13252.23 (r/w/o: 9275.84/2652.85/1323.55) lat (ms,95%): 33.12 err/s: 0.00 reconn/s: 0.00
[ 108s ] thds: 8 tps: 411.50 qps: 8230.01 (r/w/o: 5761.01/1645.50/823.50) lat (ms,95%): 73.13 err/s: 0.00 reconn/s: 0.00
[ 112s ] thds: 8 tps: 584.50 qps: 11670.77 (r/w/o: 8171.27/2330.25/1169.25) lat (ms,95%): 71.83 err/s: 0.00 reconn/s: 0.00
[ 116s ] thds: 8 tps: 895.75 qps: 17915.43 (r/w/o: 12541.20/3582.74/1791.49) lat (ms,95%): 11.65 err/s: 0.00 reconn/s: 0.00
[ 120s ] thds: 8 tps: 910.00 qps: 18199.59 (r/w/o: 12739.81/3640.02/1819.76) lat (ms,95%): 11.45 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            1131382
        write:                           323242
        other:                           161634
        total:                           1616258
    transactions:                        80812  (673.19 per sec.)
    queries:                             1616258 (13463.98 per sec.)
    ignored errors:                      1      (0.01 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          120.0420s
    total number of events:              80812

Latency (ms):
         min:                                    3.67
         avg:                                   11.88
         max:                                  451.78
         95th percentile:                       20.37
         sum:                               959979.10

Threads fairness:
    events (avg/stddev):           10101.5000/860.55
    execution time (avg/stddev):   119.9974/0.01
```

$\textsf{\color{orange}- по результатам тестирования можно сделать вывод, что бэкап создавался с большим приоритетом}$<br>
$\textsf{\color{orange}  бэкап без нагрузки - 40с , а с нагрузкой - 44с}$<br>
$\textsf{\color{orange}  в нагрузочной статистике есть провал по скорости работы,}$<br>
$\textsf{\color{orange}  а также большая часть данных ( база air ) была незадействована в нагрузке}$


### Работа с репликой

$\textsf{\color{blue}- для реплики создадим клон первой ВМ - тогда имеем}$
```
ВМ1 - Ubuntu-2404 - ubutest  - 192.168.1.244  ( PG17  port=5433  PGDATA=/pgdata/air/ ) 
ВМ2 - Astra 1.8   - astra8   - 192.168.1.28
ВМ3 - Ubuntu-2404 - ubutest3 - 192.168.1.203  ( PG17  port=5433  PGDATA=/pgdata/air/ ) 
  -- на всех ВМ прописали хосты в файле /etc/hosts
```

$\textsf{\color{blue}- План}$
```
-- ВМ1 - основной, ВМ3 - реплика
-- делаем бэкап реплики базы с ВМ3 удаленно с ВМ2 и сохраняем сразу на NFS диск ВМ1
```

// на ВМ1 разрешим использование репликации в текущей сети - в файл pg_hba.conf - добавим<br>
//  ( не забываем перечитать конфигурацию - pg_reload_conf() )
```
host    replication     all             samenet               md5
```

$\textsf{\color{blue}- Создаем реплику на ВМ3}$
```
boss@ubutest3:~$ sudo systemctl stop postgrespro-std-17.service
boss@ubutest3:~$ sudo rm -r /pgdata/air/
boss@ubutest3:~$ sudo su - postgres
postgres@ubutest3:~$ pg_basebackup -h ubutest -p 5433 -R -D /pgdata/air 
...
boss@ubutest3:~$ sudo systemctl start postgrespro-std-17.service
boss@ubutest3:~$ sudo systemctl status postgrespro-std-17.service
● postgrespro-std-17.service - Postgres Pro std 17 database server
     Loaded: loaded (/usr/lib/systemd/system/postgrespro-std-17.service; enabled; preset: enabled)
     Active: active (running) since Sun 2025-01-19 08:20:17 UTC; 5s ago
    Process: 6098 ExecStartPre=/opt/pgpro/std-17/bin/check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
   Main PID: 6101 (postgres)
      Tasks: 6 (limit: 9387)
     Memory: 85.2M (peak: 85.6M)
        CPU: 138ms
     CGroup: /system.slice/postgrespro-std-17.service
             ├─6101 /opt/pgpro/std-17/bin/postgres -D /pgdata/air
             ├─6102 "postgres: logger "
             ├─6103 "postgres: checkpointer "
             ├─6104 "postgres: background writer "
             ├─6105 "postgres: startup recovering 000000010000000300000021"
             └─6106 "postgres: walreceiver streaming 3/21000060"

янв 19 08:20:17 ubutest3 systemd[1]: Starting postgrespro-std-17.service - Postgres Pro std 17 database server...
янв 19 08:20:17 ubutest3 postgres[6101]: 2025-01-19 08:20:17.387 UTC [6101] LOG:  redirecting log output to logging collector process
янв 19 08:20:17 ubutest3 postgres[6101]: 2025-01-19 08:20:17.387 UTC [6101] HINT:  Future log output will appear in directory "log".
янв 19 08:20:17 ubutest3 systemd[1]: Started postgrespro-std-17.service - Postgres Pro std 17 database server.
```

$\textsf{\color{blue}- для удалённого запуска pg-probackup c ВМ2 на ВМ3 сделаем доступ по ssh}$
```
postgres@astra8:~$ ssh-keygen -t ed25519
Generating public/private ed25519 key pair.
Enter file in which to save the key (/var/lib/postgresql/.ssh/id_ed25519):
Created directory '/var/lib/postgresql/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /var/lib/postgresql/.ssh/id_ed25519
Your public key has been saved in /var/lib/postgresql/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:pKsY8U3q0LKJc2ah606Jwq7/MT0ZUat7kHid7J4AS3M postgres@astra8
The key's randomart image is:
+--[ED25519 256]--+
|        .        |
|       . .       |
|      . o        |
|     . O .       |
|  . = E S        |
|o o= @ B         |
|o=+.B O o        |
|*.+O + = .       |
|B@=.+   o        |
+----[SHA256]-----+

postgres@astra8:~$ ssh-copy-id postgres@ubutest3
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/var/lib/postgresql/.ssh/id_ed25519.pub"
The authenticity of host 'ubutest3 (192.168.1.203)' can't be established.
ED25519 key fingerprint is SHA256:lKU663GTvn81PEFHYUw+IrA/dei9nPdWZE6d7QHBCyE.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
postgres@ubutest3's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'postgres@ubutest3'"
and check to make sure that only the key(s) you wanted were added.
```

$\textsf{\color{blue}- создание бэкапа}$
```
postgres@astra8:~$ pg_probackup backup --instance=air -b FULL --remote-host=ubutest3 --stream --compress-algorithm=zstd --compress-level=3 -j 3
INFO: Backup start, pg_probackup version: 2.8.5, instance: air, backup ID: SQBV6K, backup mode: FULL, wal mode: STREAM, remote: true, compress-algorithm: zstd, compress-level: 3
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
WARNING: Current PostgreSQL role is superuser. It is not recommended to run pg_probackup under superuser.
INFO: Backup SQBV6K is going to be taken from standby
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: PGDATA size: 8679MB
INFO: Current Start LSN: 3/21000060, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 1m:26s
INFO: wait for pg_stop_backup()
INFO: pg_stop_backup() successfully executed
INFO: stop_stream_lsn 3/21000168 currentpos 3/21000168
INFO: backup->stop_lsn 3/21000168
INFO: Getting the Recovery Time from WAL
INFO: Failed to find Recovery Time in WAL, forced to trust current_timestamp
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 0
INFO: Validating backup SQBV6K
INFO: Backup SQBV6K data files are valid
INFO: Backup SQBV6K resident size: 2590MB
INFO: Backup SQBV6K completed

postgres@astra8:~$ pg_probackup show --instance=air
====================================================================================================================================================
 Instance  Version  ID      Recovery Time                  Mode  WAL Mode  TLI    Time    Data    WAL  Zalg  Zratio  Start LSN   Stop LSN    Status
====================================================================================================================================================
 air       17       SQBV6K  2025-01-19 13:51:04.663937+05  FULL  STREAM    1/0  1m:35s  2574MB   16MB  zstd    3.37  3/21000060  3/21000168  OK
 air       17       SQBSSL  2025-01-19 12:58:45.288153+05  FULL  STREAM    1/0     48s  2574MB   16MB  zstd    3.37  3/1E000028  3/1E0001C8  OK
 air       17       SQBPX7  2025-01-19 11:56:36.608318+05  FULL  STREAM    1/0     44s  2570MB  192MB  zstd    3.38  3/9016968   3/13125660  OK
 air       17       SQBPTA  2025-01-19 11:54:11.369512+05  FULL  STREAM    1/0     40s  2569MB   16MB  zstd    3.38  3/2000028   3/20001C8   OK
 air       17       SQ73YG  2025-01-17 00:12:29.943367+05  FULL  STREAM    1/0  1m:28s  2311MB   16MB  zlib    3.47  2/9A000028  2/9A0001C8  OK
```
$\textsf{\color{orange}// pg-probackup определил что бэкап делается со standby сервера}$<br>
$\textsf{\color{orange}// ---}$<br>
$\textsf{\color{orange}// в free версии замечено, что при работе с standby идет ожидание блоков wal (archive-timeout = 5min) если совсем неактивная БД}$<br>
$\textsf{\color{orange}// уменьшаем до 10s}$

### Итоги

```
- Выполнен бэкап тестовой БД на ВМ1 
- Каталог бэкапов выведен в сетевой ресурс NFS на ВМ1
- Восстановление на другой сервер ( ВМ2 ) с другой Linux !!! ( через сетевой ресурс NFS ВМ1 )
- Выполнение бэкапа под нагрузкой с другого сервера ( ВМ2 )
- Создание реплики БД на ВМ3  с  ВМ1
- Выполнен бэкап с реплики ВМ3 удаленно с ВМ2 на сетевой ресурс NFS ВМ1
```

### См. также

[ - Базовая настройка pg_probackup-16 (free) с отдельной ролью для бэкапирования](https://github.com/Zikon328/pg-times/tree/main/PG_PROBACKUP)<br>
[ - Базовая настройка и использование WAL-G](https://github.com/Zikon328/pg-times/tree/main/WAL-G)<br>

