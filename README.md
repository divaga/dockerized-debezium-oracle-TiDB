# Kafka Connect with Oracle Database Using Debezium

---
## 1. Prepare Docker Environments

1. **Create compose.yaml:**

```yml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    container_name: zookeeper
    depends_on:
      - oracle
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:latest
    container_name: kafka
    depends_on:
      - oracle
      - zookeeper
    ports:
      - 9092:9092
    environment:
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper:2181"
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_BROKER_ID: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 100

  schema-registry:
    image: confluentinc/cp-schema-registry:5.5.3
    container_name: schema-registry
    ports:
      - 8081:8081
    depends_on:
      - oracle
      - zookeeper
      - kafka
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL: zookeeper:2181
  debezium:
    image: debezium/connect:1.9
    container_name: connect
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: "1"
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: "1"
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: "1"
      KEY_CONVERTER: io.confluent.connect.avro.AvroConverter
      VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      LD_LIBRARY_PATH: '/kafka/external_libs/instantclient_19_6/'
    depends_on: [ oracle,kafka,zookeeper,schema-registry ]
    ports:
      - 8083:8083
  oracle:
    image: oracleinanutshell/oracle-xe-11g:latest
    container_name: oracle
    ports:
      - 1521:1521
```

2. **Start:**

   Run the following command
   ```bash
   sudo docker-compose up
   ```

---

## 2. Prepare Database for CDC

**Step 1: Create recovery area folder**

go to oracle container 
```
sudo docker exec -it oracle bash
```

```
cd /u01/app/oracle/oradata
mkdir -p recovery_area
```

**Step 2: Setup Logminer**

Run following from Oracle docker container

```
sqlplus sys/oracle@//localhost:1521 as sysdba
```

```
   alter system set db_recovery_file_dest_size = 100G;
   alter system set db_recovery_file_dest = '/u01/app/oracle/oradata/recovery_area' scope=spfile;
   shutdown immediate;
   connect sys/oracle as sysdba;
   startup mount;
   alter database archivelog;
   alter database open;
   archive log list;
   ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
   ALTER PROFILE DEFAULT LIMIT FAILED_LOGIN_ATTEMPTS UNLIMITED;

   CREATE TABLESPACE LOGMINER_TBS DATAFILE '/u01/app/oracle/oradata/XE/logminer_tbs.dbf' SIZE 25M REUSE AUTOEXTEND ON MAXSIZE UNLIMITED;

   CREATE USER c##dbzuser IDENTIFIED BY dbz DEFAULT TABLESPACE LOGMINER_TBS QUOTA UNLIMITED ON LOGMINER_TBS;
   GRANT CREATE SESSION TO c##dbzuser;
   GRANT CREATE SEQUENCE TO c##dbzuser;
   GRANT SELECT ON V_$DATABASE TO c##dbzuser;
   GRANT FLASHBACK ANY TABLE TO c##dbzuser;
   GRANT SELECT ANY TABLE TO c##dbzuser;
   GRANT SELECT_CATALOG_ROLE TO c##dbzuser;
   GRANT EXECUTE_CATALOG_ROLE TO c##dbzuser;
   GRANT SELECT ANY TRANSACTION TO c##dbzuser;
   GRANT SELECT ANY DICTIONARY TO c##dbzuser;
   GRANT LOGMINING TO c##dbzuser;
   GRANT CREATE TABLE TO c##dbzuser;
   GRANT LOCK ANY TABLE TO c##dbzuser;
   GRANT CREATE SEQUENCE TO c##dbzuser;
   GRANT EXECUTE ON DBMS_LOGMNR TO c##dbzuser;
   GRANT EXECUTE ON DBMS_LOGMNR_D TO c##dbzuser;
   GRANT SELECT ON V_$LOGMNR_LOGS TO c##dbzuser;
   GRANT SELECT ON V_$LOGMNR_CONTENTS TO c##dbzuser;
   GRANT SELECT ON V_$LOGFILE TO c##dbzuser;
   GRANT SELECT ON V_$ARCHIVED_LOG TO c##dbzuser;
   GRANT SELECT ON V_$ARCHIVE_DEST_STATUS TO c##dbzuser;


   CREATE USER debezium IDENTIFIED BY dbz DEFAULT TABLESPACE USERS;
   GRANT CONNECT TO debezium;
   GRANT CREATE SESSION TO debezium;
   GRANT CREATE TABLE TO debezium;
   GRANT CREATE SEQUENCE to debezium;
   ALTER USER debezium QUOTA 100M on users;
```

**Step 3: Create Sample Table**


```bash
sqlplus debezium/dbz@//localhost:1521
```

```
CREATE TABLE customers
(
    id         NUMBER(4) NOT NULL PRIMARY KEY,
    first_name VARCHAR2(255) NOT NULL,
    last_name  VARCHAR2(255) NOT NULL,
    email      VARCHAR2(255) NOT NULL UNIQUE
);
GRANT SELECT ON customers to c##dbzuser;
ALTER TABLE debezium.customers
    ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;

INSERT INTO customers
VALUES (1, 'Sally', 'Thomas', 'sally.thomas@acme.com');
INSERT INTO customers
VALUES (2, 'George', 'Bailey', 'gbailey@foobar.com');
INSERT INTO customers
VALUES (3, 'Edward', 'Walker', 'ed@walker.com');
INSERT INTO customers
VALUES (4, 'Anne', 'Kretchmar', 'annek@noanswer.org');
```

## 2. Prepare Debizium Connect for CDC

**Step 1: Install the Required Drivers**

Go to Debezium bash terminal

```
sudo docker exec -it connect bash
cd libs
```

```bash
curl https://maven.xwiki.org/externals/com/oracle/jdbc/ojdbc8/12.2.0.1/ojdbc8-12.2.0.1.jar -o ojdbc8-12.2.0.1.jar
curl https://repo1.maven.org/maven2/com/thoughtworks/xstream/xstream/1.3.1/xstream-1.3.1.jar -o xstream-1.3.1.jar
curl https://repo1.maven.org/maven2/com/oracle/database/xml/xdb/21.6.0.0/xdb-21.6.0.0.jar -o xdb-21.6.0.0.jar
```

**Step 2: Setup the Instant Client Tool**

Instant Client is used to Connect to Talk with Oracle db and XStream Api

> Change the Directory to : cd /kafka/external_libs`

```bash
curl "https://download.oracle.com/otn_software/linux/instantclient/19600/instantclient-basiclite-linux.x64-19.6.0.0.0dbru.zip" -O /tmp/ic.zip
unzip instantclient-basiclite-linux.x64-19.6.0.0.0dbru.zip
```

**Step 3: Setup the Debizium JDBC Sink Plugins for MySQL**

> Change the Directory to : cd connect

```shell
curl https://repo1.maven.org/maven2/io/debezium/debezium-connector-jdbc/2.5.0.Final/debezium-connector-jdbc-2.5.0.Final-plugin.tar.gz -O /tmp/ic.zip
tar xvfz  debezium-connector-jdbc-2.5.0.Final-plugin.tar.gz
```

**Step 4: Restart the Debizium Connector**
> _Note: Use Docker to Restart the connector Service_


---

## 3. Create Connector for Testing

> Create a Oracle Source Connector

**Step 1: Create oracle11.json**


```json
{
  "name": "oracle-11",
  "config": {
    "connector.class": "io.debezium.connector.oracle.OracleConnector",
    "database.hostname": "oracle",
    "database.port": "1521",
    "database.user": "c##dbzuser",
    "database.password": "dbz",
    "database.server.name": "oracle",
    "database.history.kafka.topic": "history",
    "database.dbname": "XE",
    "database.connection.adapter": "LogMiner",
    "database.history.store.only.captured.tables.ddl":"true",
    "schema.history.internal.store.only.captured.databases.ddl":"true",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "table.include.list": "DEBEZIUM.CUSTOMERS",
    "database.schema": "DEBEZIUM",
    "database.oracle.version": "11",
    "snapshot.mode": "schema_only",
    "include.schema.changes": "true",
    "key.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url": "http://schema-registry:8081",
    "value.converter.schema.registry.url": "http://schema-registry:8081"
  }
}
```
> Create Connector

```shell
curl -vX POST http://localhost:8083/connectors -d @oracle11.json --header "Content-Type: application/json"
```

> Now check the Connector is Working Fine:

```shell
curl -s "http://localhost:8083/connectors?expand=info&expand=status" | \
jq '. | to_entries[] | [ .value.info.type, .key, .value.status.connector.state,.value.status.tasks[].state,.value.info.config."connector.class"]|join(":|:")' | \
column -s : -t| sed 's/\"//g'| sort
```

> The Output Should be like this

```text
source | oracle-11 | RUNNING | RUNNING | io.debezium.connector.oracle.OracleConnector
```

**Step 2: Listen the Topic and Read Message messages**

> Create docker network and attach it to kafka container

```shell
sudo docker network create my-network
sudo docker network connect my-network kafka
```

> Access Terminal and paste this command

```shell
docker run --tty --network resources_default confluentinc/cp-kafkacat kafkacat -b kafka:9092 -C -s key=s -s value=avro -r http:/schema-registry:8081 -t test.DEBEZIUM.CUSTOMERS
```


## 4.Verify the Changes (CDC)

**Step1: Access the BASH of Oracle DB**

```shell
docker exec -it oracle bash -c 'sleep 1; sqlplus debezium/dbz@localhost:1521'
```

**Step2: Enable Auto Commit by using this query**

```oracle
SET AUTOCOMMIT ON;
```

**Step3: Perform Some Changes**

`INSERT: `

```oracle
INSERT INTO CUSTOMERS
VALUES (5, 'Peter', 'Parker', 'peter.parker@marvel.com');
```

`UPDATE: `

```oracle
UPDATE CUSTOMERS
SET email = 'new_email@gmail.com'
WHERE id = 1041;
```

`DELETE: `

```oracle
DELETE
FROM CUSTOMERS
WHERE id = 1024;
```

_Now check the changes in the terminal where we are listening ou TOPIC_

---

## 5. Create a TiDB Connector as a Sink


**Step 1: Create a Sink request using this payload**

> Create tidb.json

```json
{
  "name": "tidb-sink",
  "config": {
    "connector.class": "io.debezium.connector.jdbc.JdbcSinkConnector",
    "key.converter": "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url": "http://schema-registry:8081",
    "tasks.max": "1",
    "connection.url": "jdbc:mysql://110.239.x.x:4000/test",
    "connection.username": "dbz",
    "connection.password": "dbz",
    "hibernate.dialect": "org.hibernate.dialect.MySQL8Dialect",
    "insert.mode": "upsert",
    "delete.enabled": "true",
    "auto.create": "true",
    "primary.key.mode": "record_key",
    "schema.evolution": "none",
    "database.time_zone": "UTC",
    "topics": "test.DEBEZIUM.CUSTOMERS",
    "table.name.format": "customers"
  }
}
```

```
curl -vX POST http://localhost:8083/connectors -d @tidb.json --header "Content-Type: application/json"
```

> To Delete Connector:
```
curl -i -X DELETE localhost:8083/connectors/tidb-sink/
```

> To See the Status:

```
curl  https://localhost:8083/connectors/tidb-sink/status -k   | jq
```

**Step 2: Check the connector is Working Fine**

```shell
curl -s "http://localhost:8083/connectors?expand=info&expand=status" | \
jq '. | to_entries[] | [ .value.info.type, .key, .value.status.connector.state,.value.status.tasks[].state,.value.info.config."connector.class"]|join(":|:")' | \
column -s : -t| sed 's/\"//g'| sort
```

`The Output Should be like this `

```text
source | oracle-11        | RUNNING | RUNNING | io.debezium.connector.oracle.OracleConnector
sink   | tidb-sink        | RUNNING | RUNNING | io.debezium.connector.oracle.OracleConnector
```


> 2. Check the Tables in TiDB

```sql
SELECT *
FROM customers;
```

> There will be a same Data which was in Oracle DB


