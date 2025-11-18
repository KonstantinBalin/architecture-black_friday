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

### Остановка контейнера
```shell
docker compose -f mongo-sharding-repl.yaml down
```

---

# Задание 1 

### Cхемы
- /arch/sharding.drawio
- /arch/replication.drawio
- /arch/caching.drawio

---

# Задание 2
### Запустить контейнеры
```shell
docker compose -f mongo-sharding.yaml up -d
```

### Настройка сервера конфигурации
```shell
docker compose -f mongo-sharding.yaml exec -T configSrv mongosh --port 27017 --quiet <<EOF
rs.initiate({_id : "config_server",configsvr: true,members: [{ _id : 0, host : "configSrv:27017" }]})
EOF
```

### Настройка шардов
```shell
docker compose -f mongo-sharding.yaml exec -T shard1 mongosh --port 27018 --quiet <<EOF
rs.initiate({_id : "shard1",members: [{ _id : 0, host : "shard1:27018" }]})
EOF
```

```shell
docker compose -f mongo-sharding.yaml exec -T shard2 mongosh --port 27019 --quiet <<EOF
rs.initiate({_id : "shard2",members: [{ _id : 1, host : "shard2:27019" }]})
EOF
```

### Настройка шардов на роутере
```shell
docker compose -f mongo-sharding.yaml exec -T router mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1:27018")
sh.addShard("shard2/shard2:27019")
EOF
```

### Настройка шардирования БД
```shell
docker compose -f mongo-sharding.yaml exec -T router mongosh --port 27020 --quiet <<EOF
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
EOF
```
### Наполнение БД тестовыми данными
```shell
docker compose -f mongo-sharding.yaml exec -T router mongosh --port 27020 --quiet <<EOF
use somedb
for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})
EOF
```

### Проверка наполнения БД тестовыми данными
```shell
docker compose -f mongo-sharding.yaml exec -T router mongosh --port 27020 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

### Проверка состояния БД
```shell
docker compose -f mongo-sharding.yaml exec -T router mongosh --port 27020 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
db.helloDoc.getShardDistribution()
EOF
```

###  Проверка приложения
http://localhost:8080/helloDoc/users

### Остановка контейнера
```shell
docker compose -f mongo-sharding.yaml down
```
---

## Задание 3
### Запуск контейнеров
```bash
docker compose -f mongo-sharding-repl.yaml up -d
```

### Настройка Config Server Replica Set
```shell
docker compose -f mongo-sharding-repl.yaml exec -T config-srv1 mongosh --port 27017 --quiet <<EOF
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
EOF
```

### Настройка Shard 1 Replica Set
```shell
docker compose -f mongo-sharding-repl.yaml exec -T shard1-node1 mongosh --port 27018 --quiet <<EOF
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
EOF
```

### Настройка Shard 2 Replica Set
```shell
docker compose -f mongo-sharding-repl.yaml exec -T shard2-node1 mongosh --port 27019 --quiet <<EOF
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
EOF
```

### Добавление шардов в кластер через Mongos
```shell
docker compose -f mongo-sharding-repl.yaml exec -T router mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1-node1:27018,shard1-node2:27018,shard1-node3:27018")
sh.addShard("shard2/shard2-node1:27019,shard2-node2:27019,shard2-node3:27019")
EOF
```

### Настройка шардирования для базы данных и наполнение тестовыми данными
```shell
docker compose -f mongo-sharding-repl.yaml exec -T router mongosh --port 27020 --quiet <<EOF
sh.enableSharding("somedb")
use somedb
db.helloDoc.createIndex({ name: "hashed" });
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})
db.helloDoc.countDocuments()
EOF
```

### Проверка наполнения БД тестовыми данными
```shell
docker compose -f mongo-sharding-repl.yaml exec -T router mongosh --port 27020 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

### Проверка состояния БД
```shell
docker compose -f mongo-sharding-repl.yaml exec -T router mongosh --port 27020 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
db.helloDoc.getShardDistribution()
EOF
```

### Проверка приложения
http://localhost:8080/helloDoc/users

### Остановка контейнера
```shell
docker compose -f mongo-sharding-repl.yaml down
```

---

## Задание 4
### Запуск контейнеров
```shell
docker compose -f mongo-repl-cache.yaml up -d
```

### Настройка redis кластера
```shell
docker compose -f mongo-repl-cache.yaml exec -T redis-node-1 sh <<EOF
echo yes | redis-cli --cluster create redis-node-1:7001 redis-node-2:7002 redis-node-3:7003 redis-node-4:7004 redis-node-5:7005 redis-node-6:7006 --cluster-replicas 1
EOF
```
### Проверка redis кластера
```shell
docker compose -f mongo-repl-cache.yaml exec -T redis-node-1 redis-cli -c -p 7001 cluster info
```

### Настройка Config Server Replica Set
```shell
docker compose -f mongo-repl-cache.yaml exec -T config-srv1 mongosh --port 27017 --quiet <<EOF
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
EOF
```

### Настройка Shard 1 Replica Set
```shell
docker compose -f mongo-repl-cache.yaml exec -T shard1-node1 mongosh --port 27018 --quiet <<EOF
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
EOF
```

### Настройка Shard 2 Replica Set
```shell
docker compose -f mongo-repl-cache.yaml exec -T shard2-node1 mongosh --port 27019 --quiet <<EOF
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
EOF
```

### Настройка шардов в кластер через Mongos
```shell
docker compose -f mongo-repl-cache.yaml exec -T router mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1-node1:27018,shard1-node2:27018,shard1-node3:27018")
sh.addShard("shard2/shard2-node1:27019,shard2-node2:27019,shard2-node3:27019")
EOF
```


### Настройка шардирования для базы данных
```shell
docker compose -f mongo-repl-cache.yaml exec -T router mongosh --port 27020 --quiet <<EOF
sh.enableSharding("somedb")
use somedb
db.helloDoc.createIndex({ name: "hashed" });
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})
db.helloDoc.countDocuments()
EOF
```
### Проверка состояния БД
```shell
docker compose -f mongo-repl-cache.yaml exec -T router mongosh --port 27020 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
db.helloDoc.getShardDistribution()
EOF
```

### Проверку приложения
http://localhost:8080/helloDoc/users

### Остановка контейнеров
```shell
docker compose -f mongo-repl-cache.yaml down
```

---

## Задание 5
Cхема /arch/gateway-discovery.drawio

---

## Задание 6
Cхема /arch/cdn.drawio