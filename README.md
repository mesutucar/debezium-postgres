# Debezium sample application to synchronize 2 postgresql databases
The demo synchronizes **all the tables** in **postgres1.inventory** db with **postgres2.inventory** db.

There are many official Debezium examples but I could not find any such above.

Starting point of this demo was the [official tutorial](https://debezium.io/documentation/reference/stable/tutorial.html).

### Requirements:
* Docker
* Database access & management tool of choice (DBeaver, pgadmin, psql etc.)
* curl command

## Running the Demo

### Run Zookeeper:
```console
docker run -it --rm --name zookeeper -p 2181:2181 -p 2888:2888 -p 3888:3888 quay.io/debezium/zookeeper:2.2
```

### Run Kafka:
```console
docker run -it --rm --name kafka -p 9092:9092 --link zookeeper:zookeeper quay.io/debezium/kafka:2.2
```

### Run Postgresql 1:
```console
docker run -it --rm --name postgres1 -p 5432:5432 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=inventory quay.io/debezium/example-postgres:2.2.0.Final
```
* At this point, connect to the DB and **drop table**: inventory.geom

### Run Postgresql 2:
```console
docker run -it --rm --name postgres2 -p 5433:5432 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=inventory quay.io/debezium/example-postgres:2.2.0.Final
```

### Run Debezium Connect:
```console
docker run -it --rm --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link zookeeper:zookeeper --link kafka:kafka --link postgres1:postgres1 --link postgres2:postgres2 quay.io/debezium/connect:2.2
```

### Register source connector:
```console
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @source-postgres.json
```

### Register sink connector:
```console
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @sink-postgres.json
```

### Result:
After the above applied successfully, all **row level operations** (insert, update, delete) on postgres1 are synched to the db on postgres2.

### Utilities:
* List Kafka Topics:
```console
docker exec -ti kafka bash bin/kafka-topics.sh --list --bootstrap-server 0.0.0.0:9092
```
* Kafka - Watch Topic:
```console
docker run -it --rm --name watcher --link kafka:kafka quay.io/debezium/kafka:2.2 watch-topic -a -k inventory.customers
```
* List connectors:
```console
curl http://localhost:8083/connectors
```
* Delete a connector:
```console
curl -X DELETE http://localhost:8083/connectors/sink-connector
```
* Clean Up:
```console
docker stop postgres1 postgres2 watcher connect kafka zookeeper
```