# Kafka Connect with Oracle Database to TiDB Using Debezium

---
## 1. Prepare Docker Environments

1. **Install Docker and Docker Compose:**

```
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
dnf install docker-ce -y

systemctl start docker
systemctl enable docker

dnf install -y curl
curl -L https://github.com/docker/compose/releases/download/v2.21.0/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

2. **Create compose.yaml:**

```yml
version: '2'
services:
  zookeeper:
    image: quay.io/debezium/zookeeper:2.1
    ports:
     - 2181:2181
     - 2888:2888
     - 3888:3888
  kafka:
    image: quay.io/debezium/kafka:2.1
    ports:
     - 9092:9092
    links:
     - zookeeper
    environment:
     - ZOOKEEPER_CONNECT=zookeeper:2181
  connect:
    image: debezium/connect-with-oracle-jdbc:2.1
    build:
      context: debezium-with-oracle-jdbc
      args:
        DEBEZIUM_VERSION: 2.1
    ports:
     - 8083:8083
     - 5005:5005
    links:
     - kafka
    environment:
     - BOOTSTRAP_SERVERS=kafka:9092
     - GROUP_ID=1
     - CONFIG_STORAGE_TOPIC=my_connect_configs
     - OFFSET_STORAGE_TOPIC=my_connect_offsets
     - STATUS_STORAGE_TOPIC=my_connect_statuses
     - LD_LIBRARY_PATH=/instant_client
     - KAFKA_DEBUG=true
     - DEBUG_SUSPEND_FLAG=n
     - JAVA_DEBUG_PORT=0.0.0.0:5005
```

3. **Start:**

   Create context folder (debezium-with-oracle-jdbc) in same location with this compose.yaml with its Dockerfile inside and run the following command
   ```bash
   sudo docker-compose up
   ```

---

## 2. Prepare Oracle Database for CDC

**Step 1: Create recovery area folder**

```
cd /opt/oracle/oradata
mkdir -p recovery_area
```

**Step 2: Setup Log Miner**

Run following scripts (please adjust SYS password and CDB/PDB name)

```bash
./setup-logminer.sh
```

**Step 3: Create Sample Table**


```bash
sqlplus debezium/dbz@//localhost:1521/ORCLPDB1
```

```
-- Create some customers ...
CREATE TABLE customers (
  id NUMBER(4) GENERATED BY DEFAULT ON NULL AS IDENTITY (START WITH 1001) NOT NULL PRIMARY KEY,
  first_name VARCHAR2(255) NOT NULL,
  last_name VARCHAR2(255) NOT NULL,
  email VARCHAR2(255) NOT NULL UNIQUE
);
GRANT SELECT ON customers to c##dbzuser;
ALTER TABLE customers ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;

INSERT INTO customers
  VALUES (NULL,'Sally','Thomas','sally.thomas@acme.com');
INSERT INTO customers
  VALUES (NULL,'George','Bailey','gbailey@foobar.com');
INSERT INTO customers
  VALUES (NULL,'Edward','Walker','ed@walker.com');
INSERT INTO customers
  VALUES (NULL,'Anne','Kretchmar','annek@noanswer.org');
```

## 3. Prepare Debizium Connect for CDC

Go to Debezium bash terminal (change "connect" with your container name)

```bash
sudo docker exec -it connect bash
```

**Step 1: Setup the Debezium JDBC Sink Plugins for MySQL**

> Change the Directory to connect:
`cd connect`

```shell
curl https://repo1.maven.org/maven2/io/debezium/debezium-connector-jdbc/2.5.0.Final/debezium-connector-jdbc-2.5.0.Final-plugin.tar.gz -O /tmp/ic.zip
tar xvfz  debezium-connector-jdbc-2.5.0.Final-plugin.tar.gz
```

**Step 2: Restart the Debezium Connector**
> Note: Use Docker to Restart the connector Service

`sudo docker restart <container-name>`

---

## 4. Create Connector for Testing

> Create a Oracle Source Connector

**Step 1: Create oracle-source.json**


```json
{
  "name": "oracle-source",
  "config": {
    "connector.class" : "io.debezium.connector.oracle.OracleConnector",
    "tasks.max" : "1",
    "topic.prefix" : "server1",
    "database.hostname" : "<oracle-ip-address>",
    "database.port" : "1521",
    "database.user" : "c##dbzuser",
    "database.password" : "dbz",
    "database.dbname" : "ORCLCDB",
    "database.pdb.name" : "ORCLPDB1",
    "schema.history.internal.kafka.bootstrap.servers" : "kafka:9092",
    "schema.history.internal.kafka.topic": "schema-changes.inventory"
  }
}
```
> Create Connector

```shell
curl -vX POST http://localhost:8083/connectors -d @oracle-source.json --header "Content-Type: application/json"
```

> Now check the Connector is working:

```shell
curl -s "http://localhost:8083/connectors?expand=info&expand=status" | \
jq '. | to_entries[] | [ .value.info.type, .key, .value.status.connector.state,.value.status.tasks[].state,.value.info.config."connector.class"]|join(":|:")' | \
column -s : -t| sed 's/\"//g'| sort
```

> The Output Should be like this

```text
source | oracle | RUNNING | RUNNING | io.debezium.connector.oracle.OracleConnector
```

**Step 2: Listen the Topic and Read Message messages**

> Execute this on other terminal

```shell
sudo /usr/local/bin/docker-compose -f compose.yaml exec kafka /kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server kafka:9092 \
    --from-beginning \
    --property print.key=true \
    --topic server1.DEBEZIUM.CUSTOMERS
```


## 5.Verify the Changes (CDC)

**Step 1: Perform Some Changes on Oracle DB**

> INSERT:

```oracle
INSERT INTO CUSTOMERS
VALUES (NULL, 'Peter', 'Parker', 'peter.parker@marvel.com');
```

Now check the changes in the terminal where we are listening on Kafka topic

---

## 6. Create a TiDB Connector as a Sink


**Step 1: Create a Sink request using this payload**

> Create tidb-sink.json, change TiDB host address, user and password

```json
{
  "name": "tidb-sink",
  "config": {
    "connector.class": "io.debezium.connector.jdbc.JdbcSinkConnector",
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
    "topics": "server1.DEBEZIUM.CUSTOMERS",
    "table.name.format": "customers"
  }
}
```

> Register Sink

```
curl -vX POST http://localhost:8083/connectors -d @tidb-sink.json --header "Content-Type: application/json"
```

> To See the Status:

```
curl  https://localhost:8083/connectors/tidb-sink/status -k   | jq

```



> To Delete Connector (If needed):
```
curl -i -X DELETE localhost:8083/connectors/tidb-sink/
```

**Step 2: Check all the connectors are working**

```shell
curl -s "http://localhost:8083/connectors?expand=info&expand=status" | \
jq '. | to_entries[] | [ .value.info.type, .key, .value.status.connector.state,.value.status.tasks[].state,.value.info.config."connector.class"]|join(":|:")' | \
column -s : -t| sed 's/\"//g'| sort
```

`The Output Should be like this `

```text
source | oracle-source    | RUNNING | RUNNING | io.debezium.connector.oracle.OracleConnector
sink   | tidb-sink        | RUNNING | RUNNING | io.debezium.connector.jdbc.JdbcSinkConnector
```


> 2. Check the Tables in TiDB

```sql
SELECT *
FROM customers;
```

> There will be similar data with Oracle DB


