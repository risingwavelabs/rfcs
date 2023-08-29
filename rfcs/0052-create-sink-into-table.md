---
feature: create_sink_into_table
authors:
  - "st1page"
start_date: "2023/02/15"
---

# Create Sink into Table

## Background && Summary

Currently, RisingWave use two ways to create a streaming job. They both construct a streaming job to maintain the changes of the streaming query's result continually. But it 
- `CREATE MATERIALIZED VIEW`.
  - Materialize the result as a RW's table and apply the changes on it.
  - The table is coupled with the streaming job. They trust some property of each other.
    - The streaming job is the only changes source of the table. No other changes such as DML is allowed on this table.
    - The table's PK and other constraint is inferred by the streaming query's own property. They trust each other and do not need additional check.
    - The result table is in the same global checkpoint with the upstream table. So the changes are applied exactly-once naturally. 
- `CREATE SINK`
  - Apply the changes into an external system's table(such as MySQL).
  - The table in the external storage and the streaming job in RW do not trust any assumption each other.
    - The user can do DML on the external in the storage. The can create multiple sink into the same table too.
    - The table's PK and other constraint is maintained by the external storage. 
    - The RW's streaming execution need additional 2pc mechanism with external storage to achieve exactly-once.

This RFC give a new SQL grammar `CREATE SINK sink_name TO table_name AS SELECT ...` to create streaming job and apply its changes into a RW's table, which decouples the streaming job and the result table.
- It's changes events behavior is same with other external sinks.
- When table's constraint is violated by the changes, the behavior should be determined by the RW's table. Specifically, it can be defined by ON CONFLICT clause ([RFC: on conflict clause](https://github.com/risingwavelabs/rfcs/pull/48)). 
- align the barrier and epoch with upstream, satisfying exactly-once and snapshot isolation with upstream.

Also, the `ALTER MATERIALIZED VIEW [mv_table] TO TABLE` is used for transforming an existent MV to a table and a Sink to the table. 
## Motivation
There are some use cases.

### Multiple data streams into a single table
Required by some of our Poc users.

In wide table model, the table's field of one row could be from multiple sources. Traditional data warehouse or ETL uses multi tables join according to the primary. But the streaming join brings issues such as low efficiency and large memory consumption. So some people do it by the result table's partial update feature. https://docs.starrocks.io/en-us/latest/loading/Load_to_Primary_Key_tables#partial-updates

```SQL 
CREATE SOURCE dwd_click_events (
    user VARCHAR,
    product VARCHAR,
    event_time TIMESTAMP
) WITH (
    'connector.type' = 'kafka',
		...
);

CREATE TABLE dwd_orders (
    user VARCHAR,
    product VARCHAR,
    price BIGINT,
    event_time TIMESTAMP,
) WITH (
    'connector.type' = 'kafka',
		...
);

CREATE TABLE dws_product_per_hour (
  hour TIMESTAMP,
  product VARCHAR,
  click_cnt BIGINT,
  sell_cnt BIGINT,
  PRIMARY KEY (hour, product)
) WITH (
  'ttl' = '7d'
) ON CONFLICT PARTIAL UPDATE;

CREATE SINK click_production_analyze_per_hour_job INTO dws_product_per_hour AS 
SELECT 
  window_start,
  product,
  COUNT(*),
  NULL as sell_cnt
FROM TUMBLE(dwd_click_events, event_time, INTERVAL '1' HOUR)
GROUP BY window_start, product;

CREATE SINK order_production_analyze_per_hour_job INTO dws_product_per_hour AS 
SELECT 
  window_start,
  product,
  NULL as click_cnt,
  COUNT(*)
FROM TUMBLE(dwd_orders, order_time, INTERVAL '1' HOUR)
GROUP BY window_start, product;

alter dws_product_per_hour ADD sell_cnt BIGINT;

CREATE SINK order_production_sum_analyze_per_hour_job INTO dws_product_per_hour AS 
SELECT 
  window_start,
  product,
  NULL as click_cnt,
  NULL as sell_cnt,
  sum(price)
FROM TUMBLE(dwd_orders, order_time, INTERVAL '1' HOUR)
GROUP BY window_start, product;
```

### provide user more control over the result table
- Do DML on it and do modifications
- define Primary key and on conflict behavior
- define a TTL on the table

![](./images/0052-sink-to-table/1.png)
```SQL 
-- append-only event source with watermark
CREATE SOURCE dwd_user_behavior_with_watermark (
    user VARCHAR,
    product VARCHAR,
    price BIGINT,
    event_time TIMESTAMP,
    behavior VARCHAR,
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
    'connector.type' = 'kafka',
		...
);

-- the detailed events table which considered as the source of the truth
CREATE TABLE dwd_user_behavior (
    user VARCHAR,
    product VARCHAR,
    price BIGINT,
    event_time TIMESTAMP,
    behavior VARCHAR)
WITH (
    'connector.type' = 'kafka',
    'ttl' = '30d'
		...
);

CREATE TABLE dws_per_hour (
  hour TIMESTAMP PRIMARY KEY,
  buy_cnt BIGINT,
  sell_cnt BIGINT)
WITH (
  'ttl' = '7d'
);

-- the real-time aggregation per hour. 
-- the recent result might not be accurate because of the watermark and will be corrected by dml later
CREATE SINK events_analyze_per_hour_job INTO dws_per_hour AS 
SELECT 
  window_start,
  COUNT(*) filter(where behavior = 'buy'), 
  COUNT(*) filter(where behavior = 'sell'),   
FROM TUMBLE(dwd_user_behavior_with_watermark, event_time, INTERVAL '1' HOUR)
GROUP BY window_start;

-- a cron job to correct the table dwd_user_buy_events's result
INSERT INTO dws_per_hour
SELECT 
  window_start,
  COUNT(*) filter(where behavior = 'buy'), 
  COUNT(*) filter(where behavior = 'sell'),   
FROM TUMBLE(dwd_user_behavior_with_watermark, event_time, INTERVAL '1' HOUR)
WHERE DAY(event_time) = '${1-days-ago}'
GROUP BY window_start;
```

### SQL evolution
This case is very similar to some AP system's conditional-updates behavior. https://docs.starrocks.io/en-us/latest/loading/Load_to_Primary_Key_tables#conditional-updates

```SQL
  CREATE TABLE dwd_orders (
    user VARCHAR,
    product VARCHAR,
    price BIGINT,
    event_time TIMESTAMP,
) WITH (
    'connector.type' = 'kafka',
		...
);

CREATE TABLE dws_product_per_hour (
  hour TIMESTAMP,
  product VARCHAR,
  order_cnt BIGINT,
  PRIMARY KEY (hour, product)
) WITH (
  'ttl' = '7d'
  -- only update when the new incoming row's order_cnt is larger than the old order_cnt
) ON CONFLICT DO UPDATE SET (hour, product, order_cnt) = (
      CASE WHEN EXCLUDED.order_cnt IS NOT NULL AND EXCLUDED.order_cnt > order_cnt 
        THEN EXCLUDED.hour ELSE hour 
      END,
      CASE WHEN EXCLUDED.order_cnt IS NOT NULL AND EXCLUDED.order_cnt > order_cnt 
        THEN EXCLUDED.product ELSE product 
      END,
      CASE WHEN EXCLUDED.order_cnt IS NOT NULL AND EXCLUDED.order_cnt > order_cnt 
        THEN EXCLUDED.order_cnt, order_cnt ELSE order_cnt 
      END,)

CREATE SINK order_production_analyze_per_hour_job INTO dws_product_per_hour AS 
SELECT 
  window_start,
  product,
  COUNT(*) as order_cnt,
FROM TUMBLE(dwd_orders, order_time, INTERVAL '1' HOUR)
GROUP BY window_start, product
WITH (
  -- because the table's on- conflict behavior can only applied on the “INSERT” conflict，so we need use force append-only sink here.
  force_append_only = 'true'
);

-- when the user want to add an new aggregator
alter dws_product_per_hour ADD sell_sum BIGINT ON CONFLICT DO UPDATE SET 
  sell_cnt = CASE WHEN EXCLUDED.order_cnt IS NOT NULL AND EXCLUDED.order_cnt > order_cnt 
                  THEN EXCLUDED.sell_sum ELSE sell_sum 
             END;
CREATE SINK new_order_production_analyze_per_hour_job INTO dws_product_per_hour AS 
SELECT 
  window_start,
  product,
  COUNT(*) as order_cnt,
  sum(price) as sell_sum,
FROM TUMBLE(dwd_orders, order_time, INTERVAL '1' HOUR)
-- omit unnecessary historical data
WHERE order_time > '${1-days-ago}'
GROUP BY window_start, product
WITH (
  force_append_only = 'true'
);

-- when the new job catch up
DROP SINK order_production_analyze_per_hour_job;
ALTER SINK new_order_production_analyze_per_hour_job RENAME TO order_production_analyze_per_hour_job;
```

## Design
### Syntax and semantics
```SQL
  CREATE SINK [sink_name] INTO [table_name] AS select_query
```
All the behavior is the same with a sink into an external table with a connector. 
The schema of the sink should be the same as the table.
If the query generates an upsert sink, the primary key of the sink should be the same as that of the table.
A circular streaming plan is forbidden, with `dependent_relations` we can easily do it.

Additionally, we may introduce a `CREATE TABLE AS` syntax, which can be considered as a sugar of `CREATE TABLE (...)` and `CREATE SINK INTO` combined together. This syntax is also adopted by [Flink SQL](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=199541185) and [ksqlDB](https://docs.ksqldb.io/en/latest/developer-guide/ksqldb-reference/create-table-as-select/).

```sql
CREATE TABLE [table_name] AS select_query
```

To make it more usable for users, I would like to introduce another function to alter an exiting materialzied view to a table.

```sql
ALTER MATERIALIZED VIEW [mv_table] TO TABLE
```

This is because now we still recommend `create materialized view` at the first place instead of `create table as`. However, application developers are facing changes all the time, so it's very likely that they may realize they need to 'alter' an existing materialized view. Meanwhile, the strong consistency gurantee of MV prevents us to do any altering. With the help of `ALTER MATERIALIZED VIEW TO TABLE`, users can choose to break this consistency gurantee and alter their streaming pipelines.

The two SQL statement both require **Allow a sink and a table have the same name**. It is ok in our system because the sink will never appear in the FROM clause. An alternative is that we can add a `_sink` suffix on the sink's name.

### Implementation

#### CREATE SINK into TABLE
In our current design, a table's plan must be like this. The Hash's exchange must exist because the DML executor depend on it.
```
  DML -> (optional) RowIdGen -> Exchange(Hash) -> Materialize
```
So when a new sink created into this table, we can just connect the blackhole Sink plan to a exchange.

#### Alter MV TO TABLE

Some Limitation:
1. the mv's primary key can not be hidden column
2. we can ban the mv with singlton distribution temporally

```
  upstream -> Materialize
```
We have 2 choice here:
just add a DML fragment on the top of the materialized view plan. This is easy to support but might introduce some complexity when drop the sink.
```
  upstream -> Sink -> exchange(NoShuffle) -> DML -> exchange(hash) -> Materialize(check pk)
```
And we can also rebuild the whole plan which is the same with the `CREATE TABLE` and `CREATE SINK TO TABLE` to unify the patterns.
```
  SourceExec -> DML ->
  upstream -> Sink -> exchange(hash) -> Materialize(check pk)
```

The catalog changes 
1. the sink and the table's streaming plan should be independent and in different `table fragments`
2. the `definition` of the table should be the SQL that "create table"

### Schema Change
we can just support the add column first. Other behavior will related to the on table's conflict behavior. 