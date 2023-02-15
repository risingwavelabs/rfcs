---
feature: create_sink_into_table
authors:
  - "st1page"
start_date: "2023/02/15"
---

# Create Sink into Table

## Motivation

### Functional Loss Compared with FlinkSQL
FlinkSQL use the `INSERT ... SELECT ...` statement to create a streaming job. It means that in FlinkSQL, the table's content does not bind to the one streaming query's result, even the table is with Flink table store(https://nightlies.apache.org/flink/flink-table-store-docs-master/). **Users can sink more than one stream into one table and let table handle the pk conflict by themselves**

However, RisingWave use `create materialized view as query` to declare a streaming query and bind a table with the query's result set. So compared with FlinkSQL, RisingWave can not express multiple streaming query into one internal table naturally.

We have meet some users use FlinkSQL to handle the situation. They use a hbase as a result table and create multiple streaming queries which sink into the table. Every query partially change some column in the hbase of the same key. 

###  RisingWave's Own Expressive Enhancement
It can also improve RisingWave's own expressiveness even we actually do not want to sink multiple stream into one table. with the ON CONFLICT clause ([RFC: on conflict clause](https://github.com/risingwavelabs/rfcs/pull/48)) and `Create sink into table` statement, user can define some user defined aggregation logic when we have not implemented [User-Defined Aggregates](https://www.postgresql.org/docs/current/xaggr.html).
The User-Defined Aggregates feature is also needed by some POC users. If we implement `on conflict clause` and `Create sink into table`, we can temporarily put the UDAF on hold. I think UDAF is a little hard in streaming because users should define their own state storage in the Aggregates function so that RisingWave can persist them.

## Design
### Syntax and semantics
```SQL
  CREATE SINK INTO [table_name] AS select_query;
```
All the behavior is the same with a sink into an external table with connector. 
The sink's schema should be the same with the table.
If the query generate an upsert sink, the sink's primary key should be the same with the table's.
A circle streaming plan is forbidden. with `dependent_relations` we can do it easily.

### Implementation
**TODO**
A simple idea is just emit the rows into DMLExecutor but the path is unstable and the data could be lost if failover happens. A "At most once" sink is not good enough.
We should do a configure change to connect the sink node with the table fragment, which can benefit from our checkpoint mechanism. I am not sure if it is easy to add an additional stream on a exist merge, **to be discussed**. 