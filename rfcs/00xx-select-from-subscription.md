---
feature: select_from_subscription
authors:
  - "hzxa21"
start_date: "2024/05/08"
---

# 

## Motivation

This RFC aims to provide a solution to enable user to run query on a changelog stream from a MV/Table, like https://github.com/risingwavelabs/rfcs/pull/83,
but with a different syntax. It uses the `subscription` concept instead of a TVF to represent the changelog stream.

Let's first recap on the current usage on `subscription`:
```SQL
CREATE TABLE t (pk int PRIMARY KEY, v int);

CREATE SUBSCRIPTION sub FROM t WITH (
    retention = '1D'
);

DECLARE cur SUBSCRIPTION CURSOR FOR sub;

FETCH NEXT FROM cur;
  pk   |   v   | op | rw_timestamp 
-------+-------+----+--------------
 25184 | 25184 |  1 |             
(1 row)

FETCH NEXT FROM cur;
 t.pk  |  t.v  | t.op | rw_timestamp  
-------+-------+------+---------------
 18704 | 18704 |    2 | 1715021447040
(1 row)

...

```

Basically we support `FETCH NEXT FROM <cursor>` on a `subscription` to get the changelog stream with schema `(original schema | op | rw_timestamp)`.


This RFC will extend the usage of `subscription` to support `SELECT` query on it as well. The output schema of the SELECT query is `(original schema | op | rw_timestamp)`.

## Design
### Batch SELECT on subscription
```SQL
CREATE TABLE t (pk int PRIMARY KEY, v int);

INSERT INTO t values (1, 1);

CREATE SUBSCRIPTION sub FROM t WITH (
    retention = '1D'
);


INSERT INTO t VALUES (2, 2);

UPDATE t SET v = 11 WHERE pk = 1;

# op = 1 insert
# op = 2 delete
# op = 3 update_insert
# op = 4 update_delete
SELECT * FROM sub;
 t.pk  |  t.v  | t.op | rw_timestamp  
-------+-------+------+---------------
 2     | 2     |    1 | 1715021447030    
 1     | 1     |    4 | 1715021447040
 1     | 11    |    3 | 1715021447040
(3 rows)

SELECT t.pk as pk, t.v as v, CASE WHEN t.op = 2 or t.op = 4 THEN TRUE ELSE FALSE END as is_delete FROM sub;
 pk    |   v   |   is_delete   
 ------+-------+------------
 2     | 2     |    f     
 1     | 1     |    t 
 1     | 11    |    f 
(3 rows)
```

- Similar to the requirement for CURSOR query, `retention` must be specified when creating a subscription, that is eligble for batch select.
- Unlike CUSOR query, batch select won't see historical snapshot data when the subscription is created. It only sees the data after the subscription is created. Technically the following two queries are equivalent:
```SQL
SELECT * FROM sub;

# equivalent to FETCH NEXT from cursor created with since begin() till rw_timestamp == now()

DECLARE cur SUBSCRIPTION CURSOR FOR sub SINCE BEGIN();
FETCH NEXT FROM cur;
...
```

Use case: one-shot inspection on the MV/Table changelog for debugging


### Streaming SELECT on subscription
```SQL
CREATE TABLE t (pk int PRIMARY KEY, v int);

CREATE SUBSCRIPTION sub FROM t;

CREATE MATERIALIZED VIEW t_changelog AS
SELECT * FROM sub;

CREATE SINK t_changelog_sink
AS SELECT t.pk as pk, t.v as v, CASE WHEN t.op = 2 or t.op = 4 THEN TRUE ELSE FALSE END as is_delete FROM sub;
```

- User can create sink/mv on the subscription to get the changelog stream of the MV/Table the subscription is created upon.
- `retention` is optional for subscription eligble for streaming query.
  - When `retention` is set, the corresponding log store will be created and the subscription is eligble for cursor query, batch select query, and streaming query.
  - When `retention` is not set, no log store will be created for the subscription and the subscription is only eligble for streaming query.
- A streaming projection executor will be created to convert schema of the upstream stream chunk to `(original schema | op | rw_timestamp)` for streaming query on subscription. Implementation-wise, this is essentially the table value function mentioned in https://github.com/risingwavelabs/rfcs/pull/83.

#### Use-case 1: Alerting MV/Sink based on the first event passing a threshold
(this example is borrowed from https://github.com/risingwavelabs/rfcs/pull/83)
```SQL
  CREATE SOURCE events_source(
    id: int, 
    msg: varchar, 
    status: varchar, 
    event_time timestamptz, 
    proc_time timestampz AS proctime()
  );
  
  CREATE MATERIALIZED VIEW events as
  SELECT * FROM events_source 
  WHERE proc_time > NOW() - INTERVAL '7 days' 
  ORDER BY event_time;
  
  CREATE MATERIALIZED VIEW windowed_result as 
  SELECT 
    window_start,
    count(*) as all_count,
    count(*) filter(where status = 'error') as error_count
  FROM TUMBLE (events, event_time, INTERVAL '1 MINUTES')
  GROUP BY window_start;

  CREATE SUBSCRIPTION windowed_result_sub from windowed_result;

  # op = 1: insert
  CREATE MATERIALIZED VIEW error_minutes as 
  SELECT 
    distinct on window_start,
    window_start,
  FROM windowed_result_sub 
  WHERE error_count > all_count * 0.9 AND op = 1;

  CREATE MATERIALIZED VIEW detailed_msg_in_error_minutes as 
  SELECT events.*
  FROM error_minutes 
  JOIN events FOR SYSTEM_TIME AS OF PROCTIME()
  ON events.event_time between window_start AND window_start + INTERVAL '1 MINUTES';
```

#### Use-case 2: create sink from non-append-only MV/Table to append-only external sink
```SQL
CREATE TABLE t (pk int PRIMARY KEY, v int);

# op = 1 insert
# op = 2 delete
# op = 3 update_insert
# op = 4 update_delete
CREATE SUBSCRIPTION sub FROM t;


# create sink to a starrocks unique key table (support upsert but not delete) with schema (t, v, is_delete) by filtering out update delete
CREATE SINK t_sr_unique_key_table
AS SELECT t.pk as pk, t.v as v, CASE WHEN t.op = 2 THEN TRUE ELSE FALSE END as is_delete FROM sub where op != 4
WITH (
  connector = 'starrocks',
  ...
)

# create sink to append-only snowflake table (snowflake doesn't support streaming CDC) with schema (t, v, __delete)
CREATE SINK t_snowflake_append_only
AS SELECT t.pk as pk, t.v as v, CASE WHEN t.op = 2 or t.op = 4 THEN TRUE ELSE FALSE END as __delete FROM sub
WITH (
  connector = 'snowflake',
  ...
)
```


## Alternatives
https://github.com/risingwavelabs/rfcs/pull/83