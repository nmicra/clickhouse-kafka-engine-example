# clickhouse-kafka-engine-example
Demonstrates how to use kafka engine of Clickhouse. You push data to kafka topic and the kafka-engine of clickhouse extracts the data to the table

### Run docker-compose
to start kafka & clickhouse `cd ch-kafka && docker-compose up`

### In clickhouse client (eg dbeaver) connect to localhost:8123
now run the next queries

1. create main table where the data should be located
`CREATE TABLE mylogger (
  id String, 
  area Nullable(String),	
  event_time DateTime64(6), 
  details_json String) 
ENGINE  = MergeTree() PARTITION BY toYYYYMM(event_time) ORDER BY (id, event_time) SETTINGS index_granularity=8192;
`
2. crate kafka-engine
`CREATE TABLE mylogger_kafka
(
    payload String
) ENGINE = Kafka('kafka1:9092', 'KAFKA2CH', 'KAFKA2CH_click', 'JSONAsString');`
3. create materialized view
`CREATE MATERIALIZED VIEW mylogger_kafka_consumer TO mylogger
AS
SELECT JSONExtractString(payload, 'payload', 'after', 'id')                               as id,
	   JSONExtractString(payload, 'payload', 'after', 'area')                             as area,
       toDateTime64(JSONExtractString(payload, 'payload', 'after', 'event_time'), 3, 'Asia/Jerusalem') as event_time,
       JSONExtractString(payload, 'payload', 'after', 'details_json')                     as details_json
FROM mylogger_kafka;`


### From command-line
4. Create the topic KAFKA2CH
`docker run -it --rm --network ch-kafka_app-tier -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper-server:2181 bitnami/kafka:latest kafka-topics.sh --create --topic KAFKA2CH --replication-factor 1 --partitions 1 --bootstrap-server kafka1:9092`
5. verify topic created
`docker run -it --rm --network ch-kafka_app-tier -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper-server:2181 bitnami/kafka:latest kafka-topics.sh --list --bootstrap-server kafka1:9092`
6. open kafka-console-producer and push the data
`docker run -it --rm --network ch-kafka_app-tier -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper-server:2181 bitnami/kafka:latest kafka-console-producer.sh --broker-list kafka1:9092 --topic KAFKA2CH`
data:
`{"payload":{"before":"null","after":{"id":"cf59290c-4627-4374-b2e4-93fff26c448b","area":"CA","event_time":"2022-01-01 00:00:00","type":"LOGIN_ERROR","user_id":"123","details_json":"{\"auth_method\":\"openid-connect\",\"grant_type\":\"password\",\"client_auth_method\":\"client-secret\",\"username\":\"kuku\"}"}}}`
