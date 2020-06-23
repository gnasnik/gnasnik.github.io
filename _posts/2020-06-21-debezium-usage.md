---
layout:     post
title:      "debezium 使用笔记"
subtitle:   ""
date:       2020-06-22 16:23:50
author:     "frank"
header-style: text
# header-img: "img/post-bg-halcssting.jpg"
tags:
    - debezium
---

# Debezium 使用笔记

Debezium 是一个 CDC 数据同步`Change data capture for a variety of databases.`的解决方案，因为最近项目有数据库数据同步的需求，目前是在代码开启协程处理数据同步问题，后续数据多了不能保证效率和稳定性，于是考虑把数据同步交给 CDC 工具来做。


下面使用 Mongodb 为例进行搭建测试，更多数据库支持请查看官方文档 [Debezium Tutorial](https://github.com/debezium/debezium-examples/tree/master/tutorial#debezium-tutorial)

## Getting Start

新建 `docker-compose.yml` 文件  [github地址](https://github.com/debezium/debezium-examples/blob/master/tutorial/docker-compose-mongodb.yaml)
```
version: '2'
services:
  zookeeper:
    image: debezium/zookeeper:${DEBEZIUM_VERSION}
    ports:
     - 2181:2181
     - 2888:2888
     - 3888:3888
  kafka:
    image: debezium/kafka:${DEBEZIUM_VERSION}
    ports:
     - 9092:9092
    links:
     - zookeeper
    environment:
     - ZOOKEEPER_CONNECT=zookeeper:2181
  mongodb:
    image: debezium/example-mongodb:${DEBEZIUM_VERSION}
    hostname: mongodb
    ports:
     - 27017:27017
    environment:
     - MONGODB_USER=debezium
     - MONGODB_PASSWORD=dbz
  connect:
    image: debezium/connect:${DEBEZIUM_VERSION}
    ports:
     - 8083:8083
    links:
     - kafka
     - mongodb
    environment:
     - BOOTSTRAP_SERVERS=kafka:9092
     - GROUP_ID=1
     - CONFIG_STORAGE_TOPIC=my_connect_configs
     - OFFSET_STORAGE_TOPIC=my_connect_offsets
     - STATUS_STORAGE_TOPIC=my_connect_statuses
```

启动 docker-compose

```
# Start the topology as defined in https://debezium.io/docs/tutorial/
export DEBEZIUM_VERSION=1.1
docker-compose -f docker-compose-mongodb.yaml up

# Initialize MongoDB replica set and insert some test data
docker-compose -f docker-compose-mongodb.yaml exec mongodb bash -c '/usr/local/bin/init-inventory.sh'

# Start MongoDB connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-mongodb.json

# Consume messages from a Debezium topic
docker-compose -f docker-compose-mongodb.yaml exec kafka /kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server kafka:9092 \
    --from-beginning \
    --property print.key=true \
    --topic dbserver1.inventory.customers

# Modify records in the database via MongoDB client
docker-compose -f docker-compose-mongodb.yaml exec mongodb bash -c 'mongo -u $MONGODB_USER -p $MONGODB_PASSWORD --authenticationDatabase admin inventory'

db.customers.insert([
    { _id : NumberLong("1005"), first_name : 'Bob', last_name : 'Hopper', email : 'thebob@example.com', unique_id : UUID() }
]);

# Shut down the cluster
docker-compose -f docker-compose-mongodb.yaml down

# show connectors
curl http://localhost:8083/connectors

# remove connector
curl -i -X DELETE http://localhost:8083/connectors/connectorName
```


新建文件 `register-mongodb.json`    
```
{
    "name": "inventory-connector",
    "config": {
        "connector.class" : "io.debezium.connector.mongodb.MongoDbConnector",
        "tasks.max" : "1",
        "mongodb.hosts" : "rs0/mongodb:27017",
        "mongodb.name" : "dbserver1",
        "mongodb.user" : "debezium",
        "mongodb.password" : "dbz",
        "database.whitelist" : "inventory", # 监控的 collections
        "database.history.kafka.bootstrap.servers" : "kafka:9092"
    }
}
```


init-inventory.sh

```
HOSTNAME=`hostname`

mongo localhost:27017/inventory <<-EOF
    rs.initiate({
        _id: "rs0",
        members: [ { _id: 0, host: "${HOSTNAME}:27017" } ]
    });
EOF
echo "Initiated replica set"

sleep 3
mongo localhost:27017/admin <<-EOF
    db.createUser({ user: 'admin', pwd: 'admin', roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] });
EOF

mongo -u admin -p admin localhost:27017/admin <<-EOF
    db.runCommand({
        createRole: "listDatabases",
        privileges: [
            { resource: { cluster : true }, actions: ["listDatabases"]}
        ],
        roles: []
    });

    db.createUser({
        user: 'debezium',
        pwd: 'dbz',
        roles: [
            { role: "readWrite", db: "inventory" },
            { role: "read", db: "local" },
            { role: "listDatabases", db: "admin" },
            { role: "read", db: "config" },
            { role: "read", db: "admin" }
        ]
    });
EOF

echo "Created users"

mongo -u debezium -p dbz --authenticationDatabase admin localhost:27017/inventory <<-EOF
    use inventory;

    db.products.insert([
        { _id : NumberLong("101"), name : 'scooter', description: 'Small 2-wheel scooter', weight : 3.14, quantity : NumberInt("3") },
        { _id : NumberLong("102"), name : 'car battery', description: '12V car battery', weight : 8.1, quantity : NumberInt("8") },
        { _id : NumberLong("103"), name : '12-pack drill bits', description: '12-pack of drill bits with sizes ranging from #40 to #3', weight : 0.8, quantity : NumberInt("18") },
        { _id : NumberLong("104"), name : 'hammer', description: "12oz carpenter's hammer", weight : 0.75, quantity : NumberInt("4") },
        { _id : NumberLong("105"), name : 'hammer', description: "14oz carpenter's hammer", weight : 0.875, quantity : NumberInt("5") },
        { _id : NumberLong("106"), name : 'hammer', description: "16oz carpenter's hammer", weight : 1.0, quantity : NumberInt("0") },
        { _id : NumberLong("107"), name : 'rocks', description: 'box of assorted rocks', weight : 5.3, quantity : NumberInt("44") },
        { _id : NumberLong("108"), name : 'jacket', description: 'water resistent black wind breaker', weight : 0.1, quantity : NumberInt("2") },
        { _id : NumberLong("109"), name : 'spare tire', description: '24 inch spare tire', weight : 22.2, quantity : NumberInt("5") }
    ]);

    db.customers.insert([
        { _id : NumberLong("1001"), first_name : 'Sally', last_name : 'Thomas', email : 'sally.thomas@acme.com' },
        { _id : NumberLong("1002"), first_name : 'George', last_name : 'Bailey', email : 'gbailey@foobar.com' },
        { _id : NumberLong("1003"), first_name : 'Edward', last_name : 'Walker', email : 'ed@walker.com' },
        { _id : NumberLong("1004"), first_name : 'Anne', last_name : 'Kretchmar', email : 'annek@noanswer.org' }
    ]);

    db.orders.insert([
        { _id : NumberLong("10001"), order_date : new ISODate("2016-01-16T00:00:00Z"), purchaser_id : NumberLong("1001"), quantity : NumberInt("1"), product_id : NumberLong("102") },
        { _id : NumberLong("10002"), order_date : new ISODate("2016-01-17T00:00:00Z"), purchaser_id : NumberLong("1002"), quantity : NumberInt("2"), product_id : NumberLong("105") },
        { _id : NumberLong("10003"), order_date : new ISODate("2016-02-19T00:00:00Z"), purchaser_id : NumberLong("1002"), quantity : NumberInt("2"), product_id : NumberLong("106") },
        { _id : NumberLong("10004"), order_date : new ISODate("2016-02-21T00:00:00Z"), purchaser_id : NumberLong("1003"), quantity : NumberInt("1"), product_id : NumberLong("107") }
    ]);
EOF

echo "Inserted example data"
```