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
