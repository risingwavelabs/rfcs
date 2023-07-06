---
feature: New Source DDL
authors:
  - "TabVersion"
start_date: "2023/05/26"
---

# RFC: New Source DDL

## Background and Motivation

Traditionally, we have overlooked the key part of the message queue, which serves as the main data source for RisingWave, 
assuming it merely assists in load balancing for the message queue's partitions. However, we have encountered an 
increasing number of cases demonstrating the message key's critical role in production scenarios, such as implementing 
upsert semantics. We have also seen a growing complexity in the structure of the message key, and in some cases, 
users even want a portion of the columns to come from the message key. 
This prompted us to reconsider the importance of the message key, ~~aiming to achieve the same level of support 
as we have for the message value portion in the aftermath of this restructuring~~.

Simultaneously, we are supporting an increased number of connectors and message formats, inevitably leading to higher 
maintenance and cognitive costs. Taking Debezium as an example, even minor API differences due to varying underlying 
encoding formats have compelled us to support both Debezium JSON and Debezium Avro, despite their identical core logic. 
A notable area for optimization is segregating these issues into two dimensions, checking respective constraints, 
and implementing the core logic.

Furthermore, placing all properties within the 'with clause' is not an elegant solution, despite this being Flink's approach. 
In a stream system, there are numerous configuration items, and it is necessary to fill in the configurations based on 
the configured entities to reduce the cognitive burden on the users.

In conclusion, I propose a new `create source` and `create table with connector` syntax.

## Overview

```sql
CREATE SOURCE [IF NOT EXIST] <source-name>
    [ schema_definition ]
FORMAT <format-name>
ENCODE <encode-name> [ encode_specification ]
[ KEY ENCODE <encode-name> [ encode_specification ] [ DATATYPE (sql datatype) ] ]
WITH (<property-list>)
```

## Examples

### Plain Format

Here is an example of a plain format source, without loading message key. 

If you have the [message payload](https://www.risingwave.dev/docs/current/fast-twitter-events-processing/#step-2-connect-risingwave-to-data-streams) show as below:

```json
{
  "tweet": {
    "created_at": "2020-02-12T17:09:56.000Z",
    "id": "1227640996038684673",
    "text": "Doctors: Googling stuff online does not make you a doctor\n\nDevelopers: https://t.co/mrju5ypPkb",
    "lang": "English"
  },
  "author": {
    "created_at": "2013-12-14T04:35:55.000Z",
    "id": "2244994945",
    "name": "Singularity Data",
    "username": "singularitty"
  }
}
```

The corresponding new DDL is shown below, creating an append-only source.

```sql
CREATE SOURCE [IF NOT EXIST] source_1 (
    "tweet" STRUCT<reated_at TIMESTAMP,
        id VARCHAR,
        text VARCHAR,
        lang VARCHAR >,
    "author" STRUCT < created_at TIMESTAMP,
        id VARCHAR,
        name VARCHAR,
        username VARCHAR,
        followers INT > )
    FORMAT PLAIN ENCODE [ JSON | AVRO | PROTOBUF ]
    WITH (
        'connector' = '...',
        ...
    );
```

### UPSERT FORMAT

Upsert semantic means that RisingWave will see the message key as the superset of primary key, 
and update the old record if the key exists, otherwise insert a new record. Note that if a message only contains the key part, 
RisingWave will treat it as a delete operation.

Here I will separate the situation into four cases, separated by dimension `key clause` and `pk`.

> * No `Key Clause` means RisingWave will automatically deduct the primary key data type by value's encode.
>   * encode to `_key` datatype mapping
>     * `JSON` -> `jsonb`
>     * `AVRO` -> parse from schema registry
>     * `PROTOBUF` -> parse from schema registry
>     * `BYTES` -> `bytea`
> * No `PK` means RisingWave will add a hidden column `_key` and see it as primary key.

The examples will share the same schema preset.

> key schema
> ```json
> {
>   "tweet": {
>     "created_at": "2020-02-12T17:09:56.000Z",
>     "id": "1227640996038684673",
>     "text": "Doctors: Googling stuff online does not make you a doctor\n\nDevelopers: https://t.co/mrju5ypPkb",
>     "lang": "English"
>   }
> }
> ```

> value schema
> ```json
> {
>   "tweet": {
>     "created_at": "2020-02-12T17:09:56.000Z",
>     "id": "1227640996038684673",
>     "text": "Doctors: Googling stuff online does not make you a doctor\n\nDevelopers: https://t.co/mrju5ypPkb",
>     "lang": "English"
>   },
>   "author": {
>     "created_at": "2013-12-14T04:35:55.000Z",
>     "id": "2244994945",
>     "name": "Singularity Data",
>     "username": "singularitty"
>   }
> }
> ```

#### No Key Clause and No PK

The below DDL will create a table with schema `(_key jsonb, tweet struct<...>, author struct<...>)` if value encodes 
with json and RisingWave will use `_key` as primary key.

```sql
create table [if not exist] source_1 (
    "tweet" STRUCT<reated_at TIMESTAMP,
        id VARCHAR,
        text VARCHAR,
        lang VARCHAR >,
    "author" STRUCT < created_at TIMESTAMP,
        id VARCHAR,
        name VARCHAR,
        username VARCHAR,
        followers INT > )
    format upsert encode [ JSON | AVRO | PROTOBUF ]
    with (
        'connector' = '...',
        ...
    );
```
If users uses encode `avro` or `protobuf` with schema registry, RisingWave will load the schema from schema registry. 
So there is no need to specify the schema and primary key in DDL.

#### No Key Clause and Has PK

The DDL will create a table with schema `(tweet struct<...>, author struct<...>)`. RisingWave will not add a hidden column and 
the key part is essential only when the message payload is `NULL`, for delete operation.

```sql
create table [if not exist] source_1 (
    "tweet" STRUCT<reated_at TIMESTAMP,
        id VARCHAR,
        text VARCHAR,
        lang VARCHAR >,
    "author" STRUCT < created_at TIMESTAMP,
        id VARCHAR,
        name VARCHAR,
        username VARCHAR,
        followers INT >,
    primary key ( tweet ) )
    format upsert encode [ JSON | AVRO | PROTOBUF ]
    with (
        'connector' = '...',
        ...
    );
```

#### Has Key Clause and No PK

Specifying the key's datatype is doing the deduction work for RisingWave, which identifies `_key` type.
The datatype here represents how the message key is composed, and it has nothing to do with other columns.
In this case, we don't enforce the primary key must come from the message value, at the price of possibly storing the same data twice.
The following DDL will create a table with schema `(_key struct<...>, tweet struct<...>, author struct<...>)`.

```sql
create table [if not exist] source_1 (
    "tweet" STRUCT<reated_at TIMESTAMP,
        id VARCHAR,
        text VARCHAR,
        lang VARCHAR >,
    "author" STRUCT < created_at TIMESTAMP,
        id VARCHAR,
        name VARCHAR,
        username VARCHAR,
        followers INT > )
    format upsert encode [ JSON | AVRO | PROTOBUF ]
    KEY ENCODE [ JSON | AVRO | PROTOBUF ] DATATYPE ( struct<...> )
    with (
        'connector' = '...',
        ...
    );
```

#### Has Key Clause and Has PK

Things can get tricky in this case, because the message key part and primary key part may be overlapped.
Therefore, we set a rule that the message key part must be a superset of the primary key part and all the primary key must come from message key.

For example the key schema is `(a int, b int, c int)` and the value is `(a int, b int, c int, d int)`. Then you write the DDL like this:

```sql
create table [if not exist] source_1 (
    "a" int,
    "b" int,
    "c" int,
    "d" int,
    primary key ( a, b, c ) )
    format upsert encode [ JSON | AVRO | PROTOBUF ]
    KEY ENCODE [ JSON | AVRO | PROTOBUF ] DATATYPE ( struct<a int, b int, c int> )
    with (
        'connector' = '...',
        ...
    );
```

Everything is fine, because the message key part and the primary key part are the same. RisingWave will do the upsert semantic.
But when you write the DDL like this:

```sql
create table [if not exist] source_1 (
    "a" int,
    "b" int,
    "c" int,
    "d" int,
    primary key ( a, b ) ) -- c is missing
    format upsert encode [ JSON | AVRO | PROTOBUF ]
    KEY ENCODE [ JSON | AVRO | PROTOBUF ] DATATYPE ( struct<a int, b int, c int> )
    with (
        'connector' = '...',
        ...
    );
```

It still goes well on most cases, there is no difference when handling insert. When getting a message like
`(a1, b1, c1)^(...)` (), RisingWave will treat it as an update/delete event, and the primary key is `(a1, b1)`.
In another word, even if the message key part is `(a1, b1, c1)`, RisingWave will only use `(a1, b1)` as primary key.

### Debezium / Maxwell / Canal Format

These three formats are similar, they all have a `before` and `after` part (or other names) in the message value and specifications 
on message key and value. So they will not as flexible as `upsert` format.

```sql
create table [if not exist] source_1 (
    "id" int,
    "name" varchar,
    primary key ( id ) )
    format [ debezium | maxwell | canal ] encode [ JSON | AVRO | PROTOBUF ]
    with (
        'connector' = '...',
        ...
    );
```

Here we always assume key and value share the same encode unless you specify them separately.
RisingWave will not add the hidden column `_key` as in `upsert` format because we expect the message is generated by the protocol.
The specified message datatype helps to interpret the key part and RisingWave will only get the needed parts from it. 

```sql
create table [if not exist] source_1 (
    "id" int,
    "name" varchar,
    primary key ( id ) )
    format [ debezium | maxwell | canal ] encode [ JSON | AVRO | PROTOBUF ]
    KEY ENCODE [ JSON | AVRO | PROTOBUF ] DATATYPE ( struct<id int> )
    with (
        'connector' = '...',
        ...
    );
```

#### Debezium_mongo Format

`debeizum_mongo` is an exception as it enforce a fixed table schema and primary key, which is `(_id jsonb primary key, payload jsonb)`.

```sql
create table [if not exist] source_1 (
    "_id" jsonb,
    "payload" varchar,
    primary key ( _id ) )
    format debezium_mongo encode [ JSON | AVRO | PROTOBUF ]
    with (
        'connector' = '...',
        ...
    );
```
