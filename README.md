# Prereq

1. Docker
2. 2 GB RAM Size
3. 6 GB Storage HDD/SSD

# Diagram
![alt text](https://github.com/nugrohosam/kafka-etl-pipeline/blob/master/images/diagram.png?raw=true)

# Try This

1. clone this repo
2. install `confluent-hub-client` where ever it's
3. MacBook : `brew tap confluentinc/homebrew-confluent-hub-client`
4. `$ docker-compose up`
5. get into postgres container and do something
    - `-$ docker exec -it postgres /bin/bash`
    - `-$ psql -U postgres-user customers`
    - `-$ CREATE TABLE customers (id TEXT PRIMARY KEY, name TEXT, age INT);`
    - 
        ```
        INSERT INTO customers (id, name, age) VALUES ('5', 'fred', 34);
        INSERT INTO customers (id, name, age) VALUES ('7', 'sue', 25);
        INSERT INTO customers (id, name, age) VALUES ('2', 'bill', 51);
        ```
    - `-$ exit`
6. get into mongo container and do something
    - `-$ docker exec -it mongo /bin/bash`
    - `-$ mongo -u $MONGO_INITDB_ROOT_USERNAME -p mongo-pw admin`
    - `-$ use config`
    - 
        ``` 
        db.createRole({
        role: "dbz-role",
        privileges: [
            {
                resource: { db: "config", collection: "system.sessions" },
                actions: [ "find", "update", "insert", "remove" ]
            }
                ],
                roles: [
                { role: "dbOwner", db: "config" },
                { role: "dbAdmin", db: "config" },
                { role: "readWrite", db: "config" }
                ]
            })
        ```
    - `-$ use admin`
    - 
        ```
        db.createUser({
        "user" : "dbz-user",
        "pwd": "dbz-pw",
        "roles" : [
            {
            "role" : "root",
            "db" : "admin"
            },
            {
            "role" : "readWrite",
            "db" : "logistics"
            },
            {
            "role" : "dbz-role",
            "db" : "config"
            }
        ]
        })
        ```
    - `-$ use logistics`
    - 
        ```
        db.createCollection("orders")
        ```
    - 
        ```
        db.createCollection("shipments")
        ```
    - 
        ```
        db.orders.insert({"customer_id": "2", "order_id": "13", "price": 50.50, "currency": "usd", "ts": "2020-04-03T11:20:00"})
        db.orders.insert({"customer_id": "7", "order_id": "29", "price": 15.00, "currency": "aud", "ts": "2020-04-02T12:36:00"})
        db.orders.insert({"customer_id": "5", "order_id": "17", "price": 25.25, "currency": "eur", "ts": "2020-04-02T17:22:00"})
        db.orders.insert({"customer_id": "5", "order_id": "15", "price": 13.75, "currency": "usd", "ts": "2020-04-03T02:55:00"})
        db.orders.insert({"customer_id": "7", "order_id": "22", "price": 29.71, "currency": "aud", "ts": "2020-04-04T00:12:00"})
        ```
    - 
        ```
        db.shipments.insert({"order_id": "17", "shipment_id": "75", "origin": "texas", "ts": "2020-04-04T19:20:00"})
        db.shipments.insert({"order_id": "22", "shipment_id": "71", "origin": "iowa", "ts": "2020-04-04T12:25:00"})
        db.shipments.insert({"order_id": "29", "shipment_id": "89", "origin": "california", "ts": "2020-04-05T13:21:00"})
        db.shipments.insert({"order_id": "13", "shipment_id": "92", "origin": "maine", "ts": "2020-04-04T06:13:00"})
        db.shipments.insert({"order_id": "15", "shipment_id": "95", "origin": "florida", "ts": "2020-04-04T01:13:00"})
        ```
    - `-$ exit`
7. go into ksqlDB container
    - `-$ docker exec -it ksqldb-cli ksql http://ksqldb-server:8088`
    - `-$ SET 'auto.offset.reset' = 'earliest';`
    - 
        ```
        CREATE SOURCE CONNECTOR customers_reader WITH (
            'connector.class' = 'io.debezium.connector.postgresql.PostgresConnector',
            'database.hostname' = 'postgres',
            'database.port' = '5432',
            'database.user' = 'postgres-user',
            'database.password' = 'postgres-pw',
            'database.dbname' = 'customers',
            'database.server.name' = 'customers',
            'table.whitelist' = 'public.customers',
            'transforms' = 'unwrap',
            'transforms.unwrap.type' = 'io.debezium.transforms.ExtractNewRecordState',
            'transforms.unwrap.drop.tombstones' = 'false',
            'transforms.unwrap.delete.handling.mode' = 'rewrite'
        );
        ```
    - 
        ```
        CREATE SOURCE CONNECTOR logistics_reader WITH (
            'connector.class' = 'io.debezium.connector.mongodb.MongoDbConnector',
            'mongodb.hosts' = 'mongo:27017',
            'mongodb.name' = 'my-replica-set',
            'mongodb.authsource' = 'admin',
            'mongodb.user' = 'dbz-user',
            'mongodb.password' = 'dbz-pw',
            'collection.whitelist' = 'logistics.*',
            'transforms' = 'unwrap',
            'transforms.unwrap.type' = 'io.debezium.connector.mongodb.transforms.ExtractNewDocumentState',
            'transforms.unwrap.drop.tombstones' = 'false',
            'transforms.unwrap.delete.handling.mode' = 'drop',
            'transforms.unwrap.operation.header' = 'true'
        );
        ```
    - 
        ```
        CREATE STREAM customers WITH (
            kafka_topic = 'customers.public.customers',
            value_format = 'avro'
        );
        ```
    - 
        ```
        CREATE STREAM orders WITH (
            kafka_topic = 'my-replica-set.logistics.orders',
            value_format = 'avro',
            timestamp = 'ts',
            timestamp_format = 'yyyy-MM-dd''T''HH:mm:ss'
        );
        ```
    -
        ```
        CREATE STREAM shipments WITH (
            kafka_topic = 'my-replica-set.logistics.shipments',
            value_format = 'avro',
            timestamp = 'ts',
            timestamp_format = 'yyyy-MM-dd''T''HH:mm:ss'
        );
        ```
    - 
        ```
        CREATE TABLE customers_by_key AS
        SELECT id,
            latest_by_offset(name) AS name,
            latest_by_offset(age) AS age
        FROM customers
        GROUP BY id
        EMIT CHANGES;
        ```
    -
        ```
        CREATE STREAM enriched_orders AS
        SELECT o.order_id,
            o.price,
            o.currency,
            c.id AS customer_id,
            c.name AS customer_name,
            c.age AS customer_age
        FROM orders AS o
        LEFT JOIN customers_by_key c
        ON o.customer_id = c.id
        EMIT CHANGES;
        ```
    -
        ```
        CREATE STREAM shipped_orders WITH (
            kafka_topic = 'shipped_orders'
        )   AS
            SELECT o.order_id,
                s.shipment_id,
                o.customer_id,
                o.customer_name,
                o.customer_age,
                s.origin,
                o.price,
                o.currency
        FROM enriched_orders AS o
        INNER JOIN shipments s
        WITHIN 7 DAYS
        ON s.order_id = o.order_id
        EMIT CHANGES;
        ```
    -
        ```
        CREATE SINK CONNECTOR enriched_writer WITH (
            'connector.class' = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
            'connection.url' = 'http://elastic:9200',
            'type.name' = 'kafka-connect',
            'topics' = 'shipped_orders'
        );
        ```

