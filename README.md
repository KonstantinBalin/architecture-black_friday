# pymongo-api

## Как запустить

Запускаем mongodb и приложение

```shell
docker compose up -d
```

Заполняем mongodb данными

```shell
./scripts/mongo-init.sh
```

## Как проверить

### Если вы запускаете проект на локальной машине

Откройте в браузере http://localhost:8080

### Если вы запускаете проект на предоставленной виртуальной машине

Узнать белый ip виртуальной машины

```shell
curl --silent http://ifconfig.me
```

Откройте в браузере http://<ip виртуальной машины>:8080

## Доступные эндпоинты

Список доступных эндпоинтов, swagger http://<ip виртуальной машины>:8080/docs

# Задание 1 

### Cхемы

- /arch/sharding.drawio
- /arch/replication.drawio
- /arch/caching.drawio

# Задание 2
### Поднимите все сервисы

[//]: # (```shell)

[//]: # (docker compose -f mongo-sharding.yaml down --remove-orphans)

[//]: # (```)


### Запустить контейнеры
```shell
docker compose -f mongo-sharding.yaml up -d
```

### Подключиться к серверу конфигурации и сделайте инициализацию:
```bash
# Настройка сервера конфигурации
docker compose exec -T configSrv mongosh --port 27017 --quiet <<EOF
rs.initiate(
  {
    _id : "config_server",
    configsvr: true,
    members: [
      { _id : 0, host : "configSrv:27017" }
    ]
  }
)
EOF 
```
```bash
docker compose exec -T configSrv mongosh --port 27017
```
```
# Настройка шардов
docker compose exec -T shard1 mongosh --port 27018 --quiet << EOF

rs.initiate(
    {
        _id : "shard1",
        members: [
            { _id : 0, host : "shard1:27018" }
        ]
    }
);
```

```
docker exec -it shard2 mongosh --port 27019

rs.initiate(
{
    _id : "shard2",
    members: [
        { _id : 1, host : "shard2:27019" }
    ]
}
);

exit();
```

### Инцициализируйте роутер и наполните его тестовыми данными:
```
docker exec -it mongos_router mongosh --port 27020
sh.addShard( "shard1/shard1:27018");
sh.addShard( "shard2/shard2:27019");
sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
use somedb
for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})
db.helloDoc.countDocuments()
exit();
```

### Сделайте проверку на шардах:
```
docker exec -it shard1 mongosh --port 27018
use somedb;
db.helloDoc.countDocuments();
exit();
```
```
docker exec -it shard2 mongosh --port 27019
use somedb;
db.helloDoc.countDocuments();
exit();
```

### Сделайте проверку приложения
http://localhost:8080/helloDoc/users

### Остановите и удалите контейнеры
```shell
docker-compose -f mongo-sharding.yaml down
```

## Задание 3
### Запустите контейнеры
```bash
docker-compose -f mongo-sharding-repl.yaml up -d
```

### Инициализируйте Config Server Replica Set
```
docker exec -it config-srv1 mongosh --port 27017

rs.initiate(
   {
        _id: "config_server",
        configsvr: true,
        members: [
            { _id: 0, host: "config-srv1:27017"},
            { _id: 1, host: "config-srv2:27017"},
            { _id: 2, host: "config-srv3:27017"}
        ]
    }
);

exit();
```

### Шаг 2: Инициализируйте Shard 1 Replica Set
```
docker exec -it shard1-node1 mongosh --port 27018

rs.initiate(
    {
      _id: "shard1",
      members: [
        { _id: 0, host: "shard1-node1:27018"},
        { _id: 1, host: "shard1-node2:27018"},
        { _id: 2, host: "shard1-node3:27018"}
      ]
    }
);

exit();
```

### Инициализируйте Shard 2 Replica Set
```
docker exec -it shard2-node1 mongosh --port 27019

rs.initiate(
    {
      _id: "shard2",
      members: [
        { _id: 0, host: "shard2-node1:27019" },
        { _id: 1, host: "shard2-node2:27019" },
        { _id: 2, host: "shard2-node3:27019" }
      ]
    }
)

exit();
```

### Добавьте шарды в кластер через Mongos
```
docker exec -it mongos-router mongosh --port 27020
sh.addShard("shard1/shard1-node1:27018,shard1-node2:27018,shard1-node3:27018")
sh.addShard("shard2/shard2-node1:27019,shard2-node2:27019,shard2-node3:27019")
exit();
```
### Включите шардирование для базы данных
```
docker exec -it mongos-router mongosh --port 27020

sh.enableSharding("somedb")
use somedb
db.helloDoc.createIndex({ name: "hashed" });
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})
db.helloDoc.countDocuments()

exit();
```

### Сделайте проверку на шардах:
```
docker exec -it shard1-node1 mongosh --port 27018
use somedb;
db.helloDoc.countDocuments();
exit();
```

```
docker exec -it shard1-node2 mongosh --port 27018
use somedb;
db.helloDoc.countDocuments();
exit();
```

```
docker exec -it shard1-node3 mongosh --port 27018
use somedb;
db.helloDoc.countDocuments();
exit();
```

```
docker exec -it shard2-node1 mongosh --port 27019
use somedb;
db.helloDoc.countDocuments();
exit();
```

```
docker exec -it shard2-node2 mongosh --port 27019
use somedb;
db.helloDoc.countDocuments();
exit();
```

```
docker exec -it shard2-node3 mongosh --port 27019
use somedb;
db.helloDoc.countDocuments();
exit();
```

### Сделайте проверку приложения
http://localhost:8080/helloDoc/users

### Остановите и удалите контейнеры
```shell
docker-compose -f mongo-sharding-repl.yaml down
```

## Задание 4
### Запусти контейнер
```shell
docker-compose -f mongo-repl-cache.yaml up -d
```

### Инициализируй Config Server Replica Set
```
> docker exec -it config-srv1 mongosh --port 27017
rs.initiate(
    {
        _id: "config_server",
        configsvr: true,
        members: [
            { _id: 0, host: "config-srv1:27017"},
            { _id: 1, host: "config-srv2:27017"},
            { _id: 2, host: "config-srv3:27017"}
        ]
    }
);
exit();
```

### Инициализируй Shard 1 Replica Set
```
docker exec -it shard1-node1 mongosh --port 27018
rs.initiate(
{
    _id: "shard1",
        members: [
            { _id: 0, host: "shard1-node1:27018"},
            { _id: 1, host: "shard1-node2:27018"},
            { _id: 2, host: "shard1-node3:27018"}
        ]
    }
)
exit();
```

### Инициализируй Shard 2 Replica Set
```
docker exec -it shard2-node1 mongosh --port 27019
rs.initiate(
    {
    _id: "shard2",
        members: [
            { _id: 0, host: "shard2-node1:27019" },
            { _id: 1, host: "shard2-node2:27019" },
            { _id: 2, host: "shard2-node3:27019" }
        ]
    }
)
exit();
```

### Шаг 4: Добавление шардов в кластер через Mongos
```
docker exec -it mongos-router mongosh --port 27020
sh.addShard("shard1/shard1-node1:27018,shard1-node2:27018,shard1-node3:27018")
sh.addShard("shard2/shard2-node1:27019,shard2-node2:27019,shard2-node3:27019")
exit();
```

### Шаг 5: Включение шардирования для базы данных
```
docker exec -it mongos-router mongosh --port 27020
sh.enableSharding("somedb")
use somedb
db.helloDoc.createIndex({ name: "hashed" });
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})
db.helloDoc.countDocuments()
exit();
```
### Сделайте проверку на шардах:
```
docker exec -it shard1-node1 mongosh --port 27018
use somedb;
db.helloDoc.countDocuments();
exit();
```
```
docker exec -it shard1-node2 mongosh --port 27018
use somedb;
db.helloDoc.countDocuments();
exit();
```
```
docker exec -it shard1-node3 mongosh --port 27018
use somedb;
db.helloDoc.countDocuments();
exit();
```
```
docker exec -it shard2-node1 mongosh --port 27019
use somedb;
db.helloDoc.countDocuments();
exit();
```
```
docker exec -it shard2-node2 mongosh --port 27019
use somedb;
db.helloDoc.countDocuments();
exit();
```
```
docker exec -it shard2-node3 mongosh --port 27019
use somedb;
db.helloDoc.countDocuments();
exit();
```

### Сделайте проверку приложения
http://localhost:8080/helloDoc/users

### Остановите и удалите контейнеры
```shell
docker-compose -f mongo-repl-cache.yaml down
```

## Задание 5
Cхема /arch/gateway-discovery.drawio

## Задание 6
Cхема /arch/cdn.drawio