---
feature: create_sink_into_table
authors:
  - "st1page"
start_date: "2023/02/15"
---

# Create Sink into Table

## Motivation

### Loss of functionality compared to FlinkSQL
FlinkSQL uses the `INSERT ... SELECT ...` statement to create a streaming job. This means that in FlinkSQL, the table content is not bound to the result of a streaming query, even if the table is stored in Flink table store (https://nightlies.apache.org/flink/flink-table-store-docs-master/). **Users can sink more than one stream into a table and let the table handle the pk conflict by itself**.

However, RisingWave uses `create materialised view as query' to declare a streaming query and bind a table with the query's result set. So compared to FlinkSQL, RisingWave cannot express multiple streaming query into one internal table naturally.

We have met some users who use FlinkSQL to handle this situation. They use a hbase as a result table and create multiple streaming queries that sink into the table. Each query partially changes some column in the hbase with the same key. 

### RisingWave's own expressive enhancement
It can also improve RisingWave's own expressiveness even if we actually do not want to sink multiple stream into one table. using the ON CONFLICT clause ([RFC: on conflict clause](https://github.com/risingwavelabs/rfcs/pull/48)) and `Create sink into table` statement, user can define some user-defined aggregation logic if we have not implemented [User-Defined Aggregates](https://www.postgresql.org/docs/current/xaggr.html).
The user-defined aggregates feature is also needed by some POC users. If we implement `on conflict clause' and `create sink into table', we can temporarily put UDAF on hold. I think UDAF is a bit hard in streaming because users should define their own state storage in the Aggregates function so that RisingWave can persist them.

## Design
### Syntax and semantics
```SQL
  CREATE SINK INTO [table_name] AS select_query
```
All the behavior is the same with a sink into an external table with a connector. 
The schema of the sink should be the same as the table.
If the query generates an upsert sink, the primary key of the sink should be the same as that of the table.
A circular streaming plan is forbidden, with `dependent_relations` we can easily do it.

### Implementation
A simple idea is to just sink the rows into the DMLExecutor, but the path is unstable and the data could be lost if failover happens. A "At most once" sink is not good enough.
We should do a configure change to connect the sink node to the table fragment, which can benefit from our checkpoint mechanism.

### Schema Change
to be discussed if the behavior is ok
- adding a column one a table which has been sink, the new column will be null.
- dropping column on a table which has been sink is forbidden.
Or just simply ban all the DDL on a table where exists an sink.
