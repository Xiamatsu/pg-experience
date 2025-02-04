### загрузка потока команды journalctl в базу InfluxDB

- дополнительные настройки на **Ubuntu 2404** - настраиваем виртуальное окружение для python
- активируем и загружаем библиотеку клиента **influxdb**
```
sudo apt install -y python3-pip
sudo apt install -y python3.12-venv
python3 -m venv venv
source ./venv/bin/activate
pip install influxdb-client
```

- программа на **python** - testx.py
```python
import sys
import json
import influxdb_client
from influxdb_client.client.write_api import SYNCHRONOUS


URL = "http://192.168.1.244:8086"
BUCKET = "test1"
ORG = "sirus"
TOKEN = "ICEmDRlB94HhXZmur02hlxEPTe1QCIOE7_j0V-ivdwAokYGs132qpIpDqKkGQtuTZ4vgwnEP3lJe-8EEfaS6lw=="

client = influxdb_client.InfluxDBClient( url=URL, token=TOKEN, org=ORG )
write_api = client.write_api( SYNCHRONOUS )

# файл для логов (блок with - для тестирования)
with open('test_log.txt', 'w') as f:
    # бесконечный цикл по стандартному потоку ввода 
    for line in sys.stdin:
        # читаем строку
        ss = line.strip()
        data = json.loads(ss)
        # выбираем нужные данные - время преобразуем в (нс)
        xtime = int(data['__REALTIME_TIMESTAMP'])*1000
        xhost = data['_HOSTNAME']
        xtran = data['_TRANSPORT']
        xmess = data['MESSAGE']
        xscop = data['_RUNTIME_SCOPE']
        xprio = data['PRIORITY']
        # готовим данные
        p = influxdb_client.Point(xhost).time(xtime).tag("transport",xtran).tag("scope",xscop).field("message",xmess).field("priority",xprio)
        # записываем данные
        write_api.write( BUCKET, ORG, p )
        # контрольная запись в файл
        f.write( p.to_line_protocol() )
        f.write("\n")
        pass
```

- запускаем в консоли с выборочными данными с ожиданием
```
journalctl -f -ek -o json | python testx.py
```
// вывод в формате json - выдаётся построчно, что проще обрабатывать ( не надо ловить "{}" )<br>
// программа в связке с journalctl - ожидает<br>
// и служит мостом для отправки данных в базу


- пример программы выборки данных
```python
import influxdb_client 
from influxdb_client.client.write_api import SYNCHRONOUS

URL = "http://192.168.1.244:8086"
BUCKET = "bitcoin"
ORG = "sirus"
TOKEN = "ICEmDRlB94HhXZmur02hlxEPTe1QCIOE7_j0V-ivdwAokYGs132qpIpDqKkGQtuTZ4vgwnEP3lJe-8EEfaS6lw=="

client = influxdb_client.InfluxDBClient( url=URL, token=TOKEN, org=ORG )

query_api = client.query_api()

query = 'from(bucket: "test1")\
   |> range(start: 2025-01-19T08:00:00Z)\
   |> filter(fn: (r) => r._measurement == "ubutest")\
   |> filter(fn: (r) => r._field == "message")\
   |> filter(fn: (r) => r.transport == "kernel")'

result = query_api.query( query, ORG )

results = []
for table in result:
    for record in table.records:
        results.append( (record.get_time(), record.get_field(), record.get_value()) )

print(results)
```