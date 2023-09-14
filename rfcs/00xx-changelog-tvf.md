---
feature: changelog_tvf
authors:
  - "st1page"
start_date: "2023/08/24"
---

# 

Please feel free to add or remove sections.

## Summary

Introduce a new TVF 
```
changelog(table/mv name) -> Append only stream(
  original schema | 
  op: varchar, 
  event_time: TIMESTAMPTZ, 
  _row_id(hidden): Serial,
  old_value: STRUCT<original schema>)
```
And users can get a append-only stream and express more flexible behavior. 
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

  CREATE MATERIALIZED VIEW error_minutes as 
  SELECT 
    distinct on window_start,
    window_start,
  FROM changelog(windowed_result) 
  WHERE error_count > all_count * 0.9 AND op = "insert";

  CREATE MATERIALIZED VIEW detailed_msg_in_error_minutes as 
  SELECT events.*
  FROM error_minutes 
  JOIN events FOR SYSTEM_TIME AS OF PROCTIME()
  ON events.event_time between window_start AND window_start + INTERVAL '1 MINUTES';
```



## Motivation

Why are you want to introduce the feature?

## Design

Explain the feature in detail.

## Unresolved questions

* Are there some questions that haven't been resolved in the RFC?
* Can they be resolved in some future RFCs?
* Move some meaningful comments to here.

## Alternatives

What other designs have been considered and what is the rationale for not choosing them?

## Future possibilities

Some potential extensions or optimizations can be done in the future based on the RFC.
