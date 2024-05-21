---
feature: iceberg_source
authors:
  - "Eric Fu"
start_date: "2024/1/5"
---

# RFC: Combine Historical and Incremental Data

## Motivation

Some users need to create a table that combines data from historical data (such as file source or Iceberg Source) and incremental data (such as Kafka or Pulsar)

Secondly, We’d like to allow users to analyze data from Iceberg or other data stores directly. As a product strategy, we would offer more ability to run analysis on historical using RisingWave.

## Design

The design comprises the following parts.

1. **Iceberg Source**, which is queryable but unable to be base table of streaming jobs. 
2. **Alter the connector of a table**

With these new features, users will be able to 

1. Create an iceberg source for historical data
2. Create a RisingWave table and insert data selected from the iceberg source
3. Alter the table to add connector from the upstream Kafka

With this design, we give users more flexibility. For examples,

- Case 1: if the historical data and Kafka data have different schema (e.g. one in relational table and another in JSON), they would be able to restructure the data with SQL.
- Case 2: If the historical data is from another Kafka topic, they can simply create a table with the old Kafka connector, and then alter it to another new Kafka topic.
- Case 3: If the users want to load a snapshot from Postgres but not sync data in the future, they can create a PG-connected table and alter the connector to none after the data is loaded.

### 1. **Iceberg Source**

Iceberg Source introduces a new category of source - queryable but unable to be base table of streaming jobs. The reason behind this is straightforward: Apache Iceberg as well as other data lake projects does not provide a good interface for CDC, and it doesn’t sound like a good practice because data lakes are usually treated as the final stop of data. 

> **Source or External Table?**
> 
> Shall we introduce this “queryable but not streamable” thing as external table? 
>
> **Pros:**
> - It clearly shows that it’s “queryable but not streamable”
>
> **Cons:**
> - It’s a new concept
> - Kafka source is already queryable (and it turns out to be very useful), so the concept is somehow overlapped with source. So does the file source.
> - It will be even worse if one day we decide to support CDC for iceberg source. Actually, [limited support](https://iceberg.apache.org/docs/latest/spark-structured-streaming/) is feasible.

Detailed implementation of the Iceberg source won’t be discussed today. But to give you a brief feeling of the workload, the following features seem to be required:

- Scan from tables including MOR (Merge-on-Read) tables
- Filter pushdown (optional)
- Partition / Row group pruning (optional)

We may refer to [ClickHouse](https://clickhouse.com/docs/en/engines/table-engines/integrations/iceberg) for the feature set that we need to support and their priorities.

Besides, it’s also the time to support scanning for the file source. ☺️

### 2. Alter the Connector of a Table

The new syntax allows users to change the connector of a table.

```sql
ALTER TABLE table_name WITH (
	connector = 'kafka',
  ....
) format ... encode ... ( ... )

ALTER TABLE table_name WITH (
	connector = 'none'
)
```

Under the hood, the implementation relies on 

- The new extensible table write path
    - Designed in https://github.com/risingwavelabs/rfcs/pull/52
    - Implemented in https://github.com/risingwavelabs/risingwave/pull/12240
- Replacing an actor via Mutation barrier
    - Implemented in https://github.com/risingwavelabs/risingwave/pull/8063

As far as I am concerned, this feature can be done in acceptable complexity.

### 3. Enhance `INSERT INTO SELECT` to be distributed

The current implementation of `INSERT INTO SELECT` is simply combining `INSERT` and a batch query, which is single-threaded. To make our workflow usable, we need a distributed implementation.

> 💭 **How about `CREATE TABLE AS`?**
> 
> Perhaps we shall also enhance the `CREATE TABLE AS` command to be distributed as well, because here we expect users to create a new table and insert data immediately into it. People familiar with SQL might use `CREATE TABLE AS` instead of `INSERT INTO SELECT`.
>
> However, this will be conflicted with the proposal of `CREATE TABLE > AS` syntax in [RFC: Create Sink into Table](https://github.com/risingwavelabs/rfcs/blob/e227e4379c46302bb72ef4f1fa6736469055424b/rfcs/0052-create-sink-into-table.md#create-table-as).

## Alternatives

### [Hybrid Source](https://nightlies.apache.org/flink/flink-docs-master/docs/connectors/datastream/hybridsource/) from Flink

The design requires the first source to be *bounded*, because it will begin to output events for source 2 after source 1 has been exhausted. Unfortunately, we don’t have such concept of “bounded/unbounded stream”.

Secondly, the hybrid source is not as flexible as this design. Ported to RisingWave, it will be like: a hybrid source can combine multiple connectors in one table/source. This makes everything in “one shot”, but users won’t know whether it’s correct until the first source is exhausted.

## Discussions

**Shall we name it a Batch Source to differentiate from existing source?**

I feel that it’s like an attribute of source e.g.

```
Source      can_batch_read?   can_stream?
Kafka       yes               yes
Iceberg     yes               no
SQS         no                yes
```

**What about external lookup join? 🤔 Asking because of the question Source vs Ext Table. It sounds like it can have such ability.**

No, because Iceberg doesn’t have pk and can’t select row efficiently. Maybe we can support external lookup table in the future, but out of scope of this.

**Abort `INSERT INTO SELECT`, simply distributed is not enough. We need to think about failover. Also we need a lot of optimizations to make it practical, especially when MOR is required.**

- (Eric) Regarding of failover, I believe it’s nice-to-have, but I don’t have any good ideas.
   - This is not a good user experience, I think we need to rethink about it  
      - Any ideas to improve?


- To ingest a large amount of historical data by inserting dml would take a long time. It is a large transaction, but currently, our atomicity is only provided for small transactions.
  - We don't have a recovery mechanism to restart DML from the last failure point. It seems users need to delete partial data from the table and then insert data from the beginning instead.