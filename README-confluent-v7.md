# Fun with Sensors' Data

## Demo : Sensor Data
### Setup

1. Setup MQTT Broker : Mosquitto

    Follow instruction from [Mosquitto documentation](https://mosquitto.org/download/)

    **Mac**

    ```bash
    brew install mosquitto
    brew services start mosquitto
    ```

1. Setup Confluent Platform

    1. Download confluent platform and start local service

        ```bash
        wget "https://packages.confluent.io/archive/7.0/confluent-7.0.1.tar.gz"
        tar -xvf confluent-7.0.1.tar.gz
        export CONFLUENT_HOME=$PWD/confluent-7.0.1
        export PATH=$PATH:$CONFLUENT_HOME/bin
        confluent local services start
        ```

    1. Install Yugabyte Sink Connector

        ```bash
        confluent-hub install yugabyteinc/yb-kafka-connector:1.0.0 --no-prompt
        ```

        **Output**

        ```log

        Implicit confirmation of the question: You are about to install 'yb-kafka-connector' from Yugabyte, Inc., as published on Confluent Hub.
        Downloading component Kafka Connect Yugabyte 1.0.0, provided by Yugabyte, Inc. from Confluent Hub and installing into <current-directory>/confluent-7.0.1/share/confluent-hub-components
        Adding installation directory to plugin path in the following files:
          <current-directory>/confluent-7.0.1/etc/kafka/connect-distributed.properties
          <current-directory>/confluent-7.0.1/etc/kafka/connect-standalone.properties
          <current-directory>/confluent-7.0.1/etc/schema-registry/connect-avro-distributed.properties
          <current-directory>/confluent-7.0.1/etc/schema-registry/connect-avro-standalone.properties

        Completed
        ```

    1. Install MQTT Connector Connector

        ```bash
        confluent-hub install confluentinc/kafka-connect-mqtt:latest --no-prompt
        ```

        **Output**

        ```bash
        Running in a "--no-prompt" mode
        Implicit acceptance of the license below:
        Confluent Software Evaluation License
        https://www.confluent.io/software-evaluation-license
        Downloading component Kafka Connect MQTT 1.5.1, provided by Confluent, Inc. from Confluent Hub and installing into <current-directory>/confluent-7.0.1/share/confluent-hub-components
        Adding installation directory to plugin path in the following files:
          <current-directory>/confluent-7.0.1/etc/kafka/connect-distributed.properties
          <current-directory>/confluent-7.0.1/etc/kafka/connect-standalone.properties
          <current-directory>/confluent-7.0.1/etc/schema-registry/connect-avro-distributed.properties
          <current-directory>/confluent-7.0.1/etc/schema-registry/connect-avro-standalone.properties

        Completed
        ```

1. Setup Kafka MQTT Connect

    1. Create connector for `mqtt-source`

        ```bash
        curl -s -X POST -H 'Content-Type: application/json' http://localhost:8083/connectors \
        -d '{
          "name" : "mqtt-source",
          "config" : {
            "connector.class" : "io.confluent.connect.mqtt.MqttSourceConnector",
            "tasks.max" : "1",
            "mqtt.server.uri" : "tcp://127.0.0.1:1883",
            "mqtt.topics" : "temperature",
            "key.converter": "",
            "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
            "kafka.topic" : "mqtt.temperature",
            "confluent.topic.bootstrap.servers": "localhost:9092",
            "confluent.topic.replication.factor": "1",
            "confluent.license":""
          }
        }'
        ```

        **Output**

        ```json
        {"name":"mqtt-source","config":{"connector.class":"io.confluent.connect.mqtt.MqttSourceConnector","tasks.max":"1","mqtt.server.uri":"tcp://127.0.0.1:1883","mqtt.topics":"temperature","kafka.topic":"mqtt.temperature","confluent.topic.bootstrap.servers":"localhost:9092","confluent.topic.replication.factor":"1","confluent.license":"","name":"mqtt-source"},"tasks":[],"type":"source"}
        ```

    1. (Optional) List connector status

        ```bash
        curl http://localhost:8083/connectors
        ```

        **Output**

        ```json
        ["mqtt-source"]
        ```

        OR using `confluent`

        ```bash
        confluent local services connect connector status
        ```

        **Output**

        ```log
        The local commands are intended for a single-node development environment only,
        NOT for production usage. https://docs.confluent.io/current/cli/index.html

        [
          "mqtt-source"
        ]
        ```

    1. (Optional) Check `mqtt-source` status

        ```bash
        curl -s "http://localhost:8083/connectors/mqtt-source/status"
        ```

        **Output**

        ```json
        {"name":"mqtt-source","connector":{"state":"RUNNING","worker_id":"10.20.30.229:8083"},"tasks":[{"id":0,"state":"RUNNING","worker_id":"10.20.30.229:8083"}],"type":"source"}
        ```

        OR using `confluent`

        ```bash
        confluent local services connect connector status mqtt-source
        ```

        **Output**

        ```log
        The local commands are intended for a single-node development environment only,
        NOT for production usage. https://docs.confluent.io/current/cli/index.html

        {
          "name": "mqtt-source",
          "connector": {
            "state": "RUNNING",
            "worker_id": "<your-machine-ip>:8083"
          },
          "tasks": [
            {
              "id": 0,
              "state": "RUNNING",
              "worker_id": "<your-machine-ip>:8083"
            }
          ],
          "type": "source"
        }
        ```

    1. FYI Only - Delete connector

        ```bash
        curl -s -X DELETE localhost:8083/connectors/mqtt-source
        ```

        **Output**

        _No Output_

1. Create topic for mqtt temperatures

    ```bash
    kafka-topics --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic mqtt.temperature
    ```

    **Output**

    ```log
    WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.
    Created topic mqtt.temperature.
    ```

1. (Optional) Test MQTT and Kafka Setup

    1. Terminal 2: Run `mosquitto_sub` to listen on temperature

        ```bash
        mosquitto_sub -h 127.0.0.1 -t temperature
        ```

        **Output**

        _No Output_

    1. Terminal 3: Run `kafka-console-consumer` to listen to `mqtt.temperature` topic

        ```bash
        kafka-console-consumer --bootstrap-server localhost:9092 --topic mqtt.temperature --property print.key=true --from-beginning
        ```

        **Output**

        _No Output_

1. Setup Yugabyte

    1. [Install YugabyteDB](https://docs.yugabyte.com/quick-start/install/).  Any latest version can be chosen, `yugabyte-2.13.0.1` is just a sample

        **Mac**

        ```bash
        wget https://downloads.yugabyte.com/releases/2.13.0.1/yugabyte-2.13.0.1-b2-darwin-x86_64.tar.gz
        tar xvfz yugabyte-2.13.0.1-b2-darwin-x86_64.tar.gz
        ```

        OR

        **Linux x86_64**

        ```bash
        wget https://downloads.yugabyte.com/releases/2.13.0.1/yugabyte-2.13.0.1-b2-linux-x86_64.tar.gz
        tar xvfz yugabyte-2.13.0.1-b2-linux-x86_64.tar.gz
        ```

        OR

        **Linux ARM64**

        ```bash
        wget https://downloads.yugabyte.com/releases/2.13.0.1/yugabyte-2.13.0.1-b2-el8-aarch64.tar.gz
        tar xvfz yugabyte-2.13.0.1-b2-el8-aarch64.tar.gz
        ```

    1. [Create a local cluster](https://docs.yugabyte.com/latest/quick-start/create-local-cluster)

        ```bash
        yugabyte-2.13.0.1/bin/yugabyted start --listen 127.0.0.1
        ```

        **Output**

        ```log
        Starting yugabyted...
        ✅ System checks

        +--------------------------------------------------------------------------------------------------+
        |                                            yugabyted                                             |
        +--------------------------------------------------------------------------------------------------+
        | Status              : Running. Leader Master is present                                          |
        | Web console         : http://127.0.0.1:7000                                                      |
        | JDBC                : jdbc:postgresql://127.0.0.1:5433/yugabyte?user=yugabyte&password=yugabyte  |
        | YSQL                : bin/ysqlsh   -U yugabyte -d yugabyte                                       |
        | YCQL                : bin/ycqlsh   -u cassandra                                                  |
        | Data Dir            : <home-dir>/var/data                                                        |
        | Log Dir             : <home-dir>/var/logs                                                        |
        | Universe UUID       : <random-uuid>                                                              |
        +--------------------------------------------------------------------------------------------------+
        🚀 yugabyted started successfully! To load a sample dataset, try 'yugabyted demo'.
        🎉 Join us on Slack at https://www.yugabyte.com/slack
        👕 Claim your free t-shirt at https://www.yugabyte.com/community-rewards/
        ```

    1. (Optional) Check status

        ```bash
        yugabyte-2.13.0.1/bin/yugabyted status
        ```

        **Output**

        ```log

        +--------------------------------------------------------------------------------------------------+
        |                                            yugabyted                                             |
        +--------------------------------------------------------------------------------------------------+
        | Status              : Running. Leader Master is present                                          |
        | Web console         : http://127.0.0.1:7000                                                      |
        | JDBC                : jdbc:postgresql://127.0.0.1:5433/yugabyte?user=yugabyte&password=yugabyte  |
        | YSQL                : bin/ysqlsh   -U yugabyte -d yugabyte                                       |
        | YCQL                : bin/ycqlsh   -u cassandra                                                  |
        | Data Dir            : <home-dir>/var/data                                                        |
        | Log Dir             : <home-dir>/var/logs                                                        |
        | Universe UUID       : <random-uuid>                                                              |
        +--------------------------------------------------------------------------------------------------+

        ```

    1. Launch ycqlsh. You can find `ycqlsh` in the `bin` subdirectory located inside the YugabyteDB installation folder.

        ```bash
        yugabyte-2.13.0.1/bin/ycqlsh
        ```

        **Output**

        ```log
        [ycqlsh 5.0.1 | Cassandra 3.9-SNAPSHOT | CQL spec 3.4.2 | Native protocol v4]
        Use HELP for help.
        ycqlsh>
        ```

    1. Create a keyspace and table by running the following command.

        Run following in the `cqlsh`

        ```sql
        CREATE KEYSPACE IF NOT EXISTS demo;
        CREATE TABLE demo.test_table (key text, value bigint, ts timestamp, PRIMARY KEY (key));
        ```

        **Output**

        _No Output_

1. Setup Kafka Yugabyte Connector

    1. Create kafka connect

        ```bash
        curl -s -X POST -H 'Content-Type: application/json' http://localhost:8083/connectors \
        -d '{
          "name" : "yugbayte-sink",
          "config" : {
            "connector.class" : "com.yb.connect.sink.YBSinkConnector",
            "topics": "mqtt.temperature",
            "tasks.max" : "1",
            "key.converter": "org.apache.kafka.connect.storage.StringConverter",
            "value.converter": "org.apache.kafka.connect.storage.StringConverter"
            "key.converter.schemas.enable": "false",
            "value.converter.schemas.enable":"false",
            "yugabyte.cql.keyspace": "demo",
            "yugabyte.cql.tablename": "test_table",
            "yugabyte.cql.contact.points": "127.0.0.1:9042,127.0.0.2:9042,127.0.0.3:9042",
            "confluent.license":""
          }
        }'
        ```

        **Output**

        ```json
        {"name":"yugbayte-sink","config":{"connector.class":"com.yb.connect.sink.YBSinkConnector","topics":"mqtt.temperature","tasks.max":"1","yugabyte.cql.keyspace":"demo","yugabyte.cql.tablename":"test_table","yugabyte.cql.contact.points":"127.0.0.1:9042,127.0.0.2:9042,127.0.0.3:9042","confluent.topic.bootstrap.servers":"localhost:9092","confluent.topic.replication.factor":"1","confluent.license":"","name":"yugbayte-sink"},"tasks":[],"type":"sink"}
        ```





## IoT Fleet Management


### Setup
1. Create YCQL Tables

    1. Start `ycqlsh`


        ```bash
        ycqlsh
        ```

        **Output**

        ```log
        Connected to local cluster at 127.0.0.1:9042.
        [ycqlsh 5.0.1 | Cassandra 3.9-SNAPSHOT | CQL spec 3.4.2 | Native protocol v4]
        Use HELP for help.
        ycqlsh>
        ```

    1. Create keyspace

        ```sql
        CREATE KEYSPACE TrafficKeySpace
        ```

        **Output**

        _No Output_

    1. Create table for sensor data `Origin_Table`

        ```sql
        CREATE TABLE TrafficKeySpace.Origin_Table (
          vehicleId text,
          routeId text,
          vehicleType text,
          longitude text,
          latitude text,
          timeStamp timestamp,
          speed double,
          fuelLevel double,
          PRIMARY KEY ((vehicleId), timeStamp)
        ) WITH default_time_to_live = 3600;
        ```

    1. Create tables for UI - `Total_Traffic`, `Window_Traffic` and `Poi_Traffic`

        ```sql
        CREATE TABLE TrafficKeySpace.Total_Traffic (
          routeId text,
          vehicleType text,
          totalCount bigint,
          timeStamp timestamp,
          recordDate text,
          PRIMARY KEY (routeId, recordDate, vehicleType)
        );

        CREATE TABLE TrafficKeySpace.Window_Traffic (
          routeId text,
          vehicleType text,
          totalCount bigint,
          timeStamp timestamp,
          recordDate text,
          PRIMARY KEY (routeId, recordDate, vehicleType)
        );

        CREATE TABLE TrafficKeySpace.Poi_Traffic(
          vehicleid text,
          vehicletype text,
          distance bigint,
          timeStamp timestamp,
          PRIMARY KEY (vehicleid)
        );
        ```

    1. Quit `ysqlsh` by Pressing `Ctrl+D` or `quit` command

1. FYI - Sample Payload

    ```json
    {
      "vehicleId":"0bf45cac-d1b8-4364-a906-980e1c2bdbcb",
      "vehicleType":"Taxi",
      "routeId":"Route-37",
      "longitude":"-95.255615",
      "latitude":"33.49808",
      "timestamp":"2017-10-16 12:31:03",
      "speed":49.0,
      "fuelLevel":38.0
    }
    ```

1. Create topic on kafka

    ```bash
    kafka-topics --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic iot-data-event
    ```

    **Output**

    ```log
    Created topic iot-data-event.
    ```

1. Create stream on ksql

    1. Launch ksql

        ```bash
        ksql
        ```

        **Output**

        ```log

                          ===========================================
                          =       _              _ ____  ____       =
                          =      | | _____  __ _| |  _ \| __ )      =
                          =      | |/ / __|/ _` | | | | |  _ \      =
                          =      |   <\__ \ (_| | | |_| | |_) |     =
                          =      |_|\_\___/\__, |_|____/|____/      =
                          =                   |_|                   =
                          =        The Database purpose-built       =
                          =        for stream processing apps       =
                          ===========================================

        Copyright 2017-2021 Confluent Inc.

        CLI v7.0.1, Server v7.0.1 located at http://localhost:8088
        Server Status: RUNNING

        Having trouble? Type 'help' (case-insensitive) for a rundown of how things work!

        ksql>
        ```

    1. Create `traffic_stream`

        ```sql
        CREATE STREAM traffic_stream (
                  vehicleId varchar,
                  vehicleType varchar,
                  routeId varchar,
                  timeStamp varchar,
                  latitude varchar,
                  longitude varchar)
            WITH (
                  KAFKA_TOPIC='iot-data-event',
                  VALUE_FORMAT='json',
                  TIMESTAMP='timeStamp',
                  TIMESTAMP_FORMAT='yyyy-MM-dd HH:mm:ss');
        ```

        **Output**

        ```log
         Message
        ----------------
        Stream created
        ----------------
        ```

    1. Create POI steam

        ```sql
         CREATE STREAM poi_traffic
              WITH ( PARTITIONS=1,
                    KAFKA_TOPIC='poi_traffic',
                    TIMESTAMP='timeStamp',
                    TIMESTAMP_FORMAT='yyyy-MM-dd HH:mm:ss') AS
              SELECT vehicleId,
                    vehicleType,
                    cast(GEO_DISTANCE(cast(latitude AS double),cast(longitude AS double),33.877495,-95.50238,'KM') AS bigint) AS distance,
                    timeStamp
              FROM traffic_stream
              WHERE GEO_DISTANCE(cast(latitude AS double),cast(longitude AS double),33.877495,-95.50238,'KM') < 30;
        ```

        **Output**

        ```log
         Message
        ------------------------------------------
        Created query with ID CSAS_POI_TRAFFIC_7
        ------------------------------------------
        ```


    1. Create aggregations

        ```sql
        CREATE TABLE total_traffic
            WITH ( PARTITIONS=1,
                    KAFKA_TOPIC='total_traffic',
                    TIMESTAMP='timeStamp',
                    FORMAT='JSON'
                    ) AS
            SELECT routeId,
                    vehicleType,
                    count(vehicleId) AS totalCount,
                    max(rowtime) AS timeStamp,
                    TIMESTAMPTOSTRING(max(rowtime), 'yyyy-MM-dd') AS recordDate
            FROM traffic_stream
            GROUP BY routeId, vehicleType;
        ```

        **Output**

        ```log
         Message
        --------------------------------------------
        Created query with ID CTAS_TOTAL_TRAFFIC_3
        --------------------------------------------
        ```

    1. Create aggregation - `window_traffic`

        ```sql
        CREATE TABLE window_traffic
            WITH ( TIMESTAMP='timeStamp',
                    KAFKA_TOPIC='window_traffic',
                    PARTITIONS=1,
                    FORMAT='JSON') AS
            SELECT routeId,
                    vehicleType,
                    count(vehicleId) AS totalCount,
                    max(rowtime) AS timeStamp,
                  TIMESTAMPTOSTRING(max(rowtime), 'yyyy-MM-dd') AS recordDate
            FROM traffic_stream
            WINDOW HOPPING (SIZE 30 SECONDS, ADVANCE BY 10 SECONDS)
            GROUP BY routeId, vehicleType;
        ```

        **Output**

        ```log
         Message
        ---------------------------------------------
        Created query with ID CTAS_WINDOW_TRAFFIC_5
        ---------------------------------------------
        ```

    1. Quit `ksql`

        ```sql
        quit
        ```

1. Create Kafka connect connectors

    1. Create connector for raw data

        ```bash
        curl -s -X POST -H 'Content-Type: application/json' http://localhost:8083/connectors \
        -d '{
          "name" : "yugabyte-sink",
          "config" : {
            "confluent.license": "",
            "key.converter": "org.apache.kafka.connect.storage.StringConverter",
            "value.converter": "org.apache.kafka.connect.json.JsonConverter",
            "key.converter.schemas.enable": false,
            "value.converter.schemas.enable": false,
            "offset.flush.interval.ms": "10000",
            "connector.class": "com.yb.connect.sink.YBSinkConnector",
            "yugabyte.cql.contact.points": "127.0.0.1:9042,127.0.0.2:9042,127.0.0.3:9042",
            "yugabyte.cql.keyspace": "TrafficKeySpace",
            "yugabyte.cql.tablename": "Origin_Table",
            "topics": "iot-data-event"
          }
        }'
        ```

        **Output**

        ```json
        {"name":"yugabyte-sink","config":{"confluent.license":"","key.converter":"org.apache.kafka.connect.storage.StringConverter","value.converter":"org.apache.kafka.connect.json.JsonConverter","key.converter.schemas.enable":"false","value.converter.schemas.enable":"false","offset.flush.interval.ms":"10000","connector.class":"com.yb.connect.sink.YBSinkConnector","yugabyte.cql.contact.points":"127.0.0.1:9042,127.0.0.2:9042,127.0.0.3:9042","yugabyte.cql.keyspace":"TrafficKeySpace","yugabyte.cql.tablename":"Origin_Table","topics":"iot-data-event","name":"yugabyte-sink"},"tasks":[],"type":"sink"}
        ```

    1. Create connector for poi data

        ```bash
        curl -s -X POST -H 'Content-Type: application/json' http://localhost:8083/connectors \
        -d '{
          "name" : "yugabyte-sink-poi",
          "config" : {
            "confluent.license": "",
            "key.converter": "org.apache.kafka.connect.storage.StringConverter",
            "value.converter": "org.apache.kafka.connect.json.JsonConverter",
            "key.converter.schemas.enable": false,
            "value.converter.schemas.enable": false,
            "offset.flush.interval.ms": "10000",
            "connector.class": "com.yb.connect.sink.YBSinkConnector",
            "yugabyte.cql.contact.points": "127.0.0.1:9042,127.0.0.2:9042,127.0.0.3:9042",
            "yugabyte.cql.keyspace": "TrafficKeySpace",
            "yugabyte.cql.tablename": "Poi_Traffic",
            "topics": "poi_traffic"
          }
        }'
        ```

        **Output**

        ```json
        {"name":"yugabyte-sink-poi","config":{"confluent.license":"","key.converter":"org.apache.kafka.connect.storage.StringConverter","value.converter":"org.apache.kafka.connect.json.JsonConverter","key.converter.schemas.enable":"false","value.converter.schemas.enable":"false","offset.flush.interval.ms":"10000","connector.class":"com.yb.connect.sink.YBSinkConnector","yugabyte.cql.contact.points":"127.0.0.1:9042,127.0.0.2:9042,127.0.0.3:9042","yugabyte.cql.keyspace":"TrafficKeySpace","yugabyte.cql.tablename":"Poi_Traffic","topics":"poi_traffic","name":"yugabyte-sink-poi"},"tasks":[],"type":"sink"}
        ```

    1. Create connector for totals data

        ```bash
        curl -s -X POST -H 'Content-Type: application/json' http://localhost:8083/connectors \
        -d '{
          "name" : "yugabyte-sink-total",
          "config" : {
            "confluent.license": "",
            "key.converter": "org.apache.kafka.connect.json.JsonConverter",
            "value.converter": "org.apache.kafka.connect.json.JsonConverter",
            "key.converter.schemas.enable": false,
            "value.converter.schemas.enable": false,
            "offset.flush.interval.ms": "10000",
            "connector.class": "com.yb.connect.sink.YBSinkConnector",
            "yugabyte.cql.contact.points": "127.0.0.1:9042,127.0.0.2:9042,127.0.0.3:9042",
            "yugabyte.cql.keyspace": "TrafficKeySpace",
            "yugabyte.cql.tablename": "Total_Traffic",
            "topics": "total_traffic"
          }
        }'
        ```

        **Output**

        ```json
        {"name":"yugabyte-sink-total","config":{"confluent.license":"","key.converter":"org.apache.kafka.connect.json.JsonConverter","value.converter":"org.apache.kafka.connect.json.JsonConverter","key.converter.schemas.enable":"false","value.converter.schemas.enable":"false","offset.flush.interval.ms":"10000","connector.class":"com.yb.connect.sink.YBSinkConnector","yugabyte.cql.contact.points":"127.0.0.1:9042,127.0.0.2:9042,127.0.0.3:9042","yugabyte.cql.keyspace":"TrafficKeySpace","yugabyte.cql.tablename":"Total_Traffic","topics":"total_traffic","name":"yugabyte-sink-total"},"tasks":[],"type":"sink"}
        ```

    1. Create connector for window data

        ```bash
        curl -s -X POST -H 'Content-Type: application/json' http://localhost:8083/connectors \
        -d '{
          "name" : "yugabyte-sink-window",
          "config" : {
            "confluent.license": "",
            "key.converter": "org.apache.kafka.connect.storage.StringConverter",
            "value.converter": "org.apache.kafka.connect.json.JsonConverter",
            "key.converter.schemas.enable": false,
            "value.converter.schemas.enable": false,
            "offset.flush.interval.ms": "10000",
            "connector.class": "com.yb.connect.sink.YBSinkConnector",
            "yugabyte.cql.contact.points": "127.0.0.1:9042,127.0.0.2:9042,127.0.0.3:9042",
            "yugabyte.cql.keyspace": "TrafficKeySpace",
            "yugabyte.cql.tablename": "Window_Traffic",
            "topics": "window_traffic"
          }
        }'
        ```

        **Output**

        ```json
        {"name":"yugabyte-sink-window","config":{"confluent.license":"","key.converter":"org.apache.kafka.connect.storage.StringConverter","value.converter":"org.apache.kafka.connect.json.JsonConverter","key.converter.schemas.enable":"false","value.converter.schemas.enable":"false","offset.flush.interval.ms":"10000","connector.class":"com.yb.connect.sink.YBSinkConnector","yugabyte.cql.contact.points":"127.0.0.1:9042,127.0.0.2:9042,127.0.0.3:9042","yugabyte.cql.keyspace":"TrafficKeySpace","yugabyte.cql.tablename":"Window_Traffic","topics":"window_traffic","name":"yugabyte-sink-window"},"tasks":[],"type":"sink"}
        ```

### Run Sample


1. Run Dashboard

    ```bash

    ```

1. Run IoT Data Producer




