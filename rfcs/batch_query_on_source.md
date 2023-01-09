---
feature: batch_query_on_source
authors:
  - "Renjie Liu"
start_date: "2022/11/10"
---

# Batch Query on Source

## Summary

Support executing the batch queries on sources like Kafka/S3.

## Motivation

We have already supported ingesting data from sources like kafka/s3, but user may want to query directly from them.

### Non Goal

* We assume that data stored in source are json or protobuf, which has no index/statistics associated with it, so we will not introduce any optimization on it.

## Design

Essentially, after user creates a source, we want to allow user to execute following sql to query data directly:

```sql
CREATE SOURCE IF NOT EXISTS source_abc (
   column1 varchar,
   column2 integer,
)
WITH (
   connector='kafka',
   topic='demo_topic',
   properties.bootstrap.server='172.10.1.1:9090,172.10.1.2:9090',
   scan.startup.mode='latest',
   scan.startup.timestamp_millis='140000000',
   properties.group.id='demo_consumer_name'
)
ROW FORMAT JSON;

select * from source_abc limit 10;
```

In general, we need to add source plan node for batch plan and source scan executor for batch query engine. The parallelism of source scan executor is determined by the number of source splits, which can be fetched from sources.

### Extra columns

By default, we allow user to scan data using earliest or latest offset. This default option works well in streaming, but maybe not suitable for batch execution. For example, user may just want to scan kafka data in latest 10 minutes, or lastest 10000 messages. Or when user has already partitioned data, they may want to query only some partition. To fufill these requirements, we can expose message meta as extra columns in schema. Let's use kafka as an example, we can attach following extra columns to each source from kafka:

| Column name                | Type      | Comment                                                                                   |
|----------------------------|-----------|-------------------------------------------------------------------------------------------|
| _rw_kafka_partition        | int       | Kafka partition this message from.                                                        |
| _rw_kafka_partition_offset | bigint    | Kafka offset in partition. 0 means starts from earliest, and -1 means starts from latest. |
| _rw_kafka_timestamp        | timestamp | Message timestamp                                                                         |

With these extra columns, we can implement several optimizations:

1. To query messages from some time, we can write `_rw_kafka_timestamp > '2022-11-10 10:00:00'`. We can use kafka's seek time offset api to determine smallest offset later than `'2022-11-10 10:00:00'`.
2. To limit scan by range of offsets, we can use `_rw_kafka_partition_offset > 1000`.  If the offset range is postive number, it's counted from earliest. We can use `_rw_kafka_partition_offset > -1000` to express latest 1000 message in each partition.
3. To limit partition to scan, we can use `_rw_kafka_partition = 0` to scan data from partition 0 only.

Above queries use kafka as an example, we can have similar stategies for other sources(pulsar, kinesis, etc).

## Unresolved questions

## Alternatives

To count offset from latest, we use negative numbers. This maybe counterintuitive for some users. Another way is to use `_rw_kafka_max_partition_offset()` as a special placeholder.

## Future possibilities
