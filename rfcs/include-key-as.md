---
feature: New Syntax Include Key As
authors:
  - "TabVersion"
start_date: "2023/11/22"
---

# New Syntax: Include \<column\> as \<new name\>

## Background

> This RFC is a follow-up to [RFC-0064](./0064-new-source-ddl.md).

In previous implementation, we only load message key into a [~~hidden~~](https://github.com/risingwavelabs/risingwave/pull/13521) 
column called `_rw_key` as `bytea` on default.

Here are some cases when RisingWave derives the `_rw_key` column.

* For all connectors with `format upsert`, RisingWave derives the column as primary key to perform upsert semantic.
  * An exception: for `format upsert encode json`, RisingWave allows the PK to be part of the message payload instead of
  the message key. This behavior is more like `format plain encode json` with PK constraint.
  * **To Be Discussed**: whether we can afford to perform a breaking change for the above exception.
* For all connectors with `format plain`, RisingWave derives the `_rw_key` column as a normal one
  ([ref](https://github.com/risingwavelabs/risingwave/pull/13278)) at a POC request. This enforces an additional column
  for all tables, which is not ideal.

Aside from message keys, users may want to ingest the timestamp as well as the topic headers into the table for further
analysis.

This RFC proposes a new syntax to allow users to specify the column name for message components other than payload.

## Proposed Syntax

```sql
create source/table (
    ..,
    primary key ( <key-column> )
)
    [ include key [ as <key-column> ] ]
    [ include timestamp [ as <timestamp-column>] ]
    [ include header [as <header-column>] ]
    [ include ... [ as ... ] ]
with ( ... )
format ... encode ... ( ... )
[ key encode [ ( ... ) ] ]
```

* If `as` name is not specified, a connector-component naming template will be applied
  * For connector kafka and component key, the derived message key column name is `_rw_kafka_key`.
* The default type for message key column is `bytea`. The priority of the type definition is: 
  `key encode` > infer from `format ... encode ...` > default type  
* **Important**: `include key` is required for `format upsert` and RisingWave will use the key column as one and 
  only primary key to perform upsert semantic. It does not allow to specify multiple columns as primary key
  even if they are part of the key.

## Specifications

### Kafka

| Allowed Components | Default Type                               | Note                                              |
|--------------------|--------------------------------------------|---------------------------------------------------|
| key                | `bytea`                                    | Allow overwritten by `encode` and `key encode`    |
| timestamp          | `timestamp with time zone` (i64 in millis) | Refer to `CreateTime` rather than `LogAppendTime` |
| partition          | `i64`                                      | The message is from which partition               |
| offset             | `i64`                                      | The offset in the partition                       |
| header             | `struct<varchar, bytea>[]`                 | KV pairs along with message                       |

### Pulsar

| Allowed Components | Default Type | Note                                                                                      |
|--------------------|--------------|-------------------------------------------------------------------------------------------|
| key                | `bytea`      | Allow overwritten by `encode` and `key encode`. Refer to `MessageMetadata::partition_key` |

More components are available at [here](https://docs.rs/pulsar/latest/pulsar/message/proto/struct.MessageMetadata.html).

### Kinesis

| Allowed Components | Default Type                             | Note                                                                             |
|--------------------|------------------------------------------|----------------------------------------------------------------------------------|
| key                | `bytea`                                  | Allow overwritten by `encode` and `key encode`. Refer to `Record::partition_key` |
| timestamp          | `timestamp with time zone` (from chrono) | refer to `Record::approximate_arrival_timestamp`                                 |

More components are available at [here](https://docs.rs/aws-sdk-kinesis/latest/aws_sdk_kinesis/types/struct.Record.html).

### S3/GCS

| Allowed Components | Default Type | Note                              |
|--------------------|--------------|-----------------------------------|
| file               | varchar      | The record comes from which file. |
