---
feature: Auto Execution Mode Selection
authors:
  - "DylanChen"
start_date: "2022/11/03"
---

# Auto Execution Mode Selection

## Summary

This RFC introduces an auto execution mode selection feature for user's batch workloads so that OLTP queries can run in local mode and OLAP queries can run in distributed mode. As a result, users will experience a lower latency automatically.

## Motivation

Currently, our system exists 2 kind of execution mode: local and distributed. We want to run OLTP queries in local execution mode, while OLAP queries in distributed execution mode. However we leave the choice to users by `set query_mode = [local | distributed]` which in some way makes our distributed execution useless because most users don't know or understand what those options exactly mean. As a database, we have the ability and should make a good choice for users to reduce their tuning works. 

## Design

Since we don't have any statistics about table/mv, we are unable to calculate a meaningful cost for queries, so we focus on the oltp workload read pattern to decide what queries should run in local mode.

### Query should run in local mode:

- Single table primary key lookup. we can check our physical plan to ensure whether there is only one table scan in the plan tree and this table scan contains a point lookup scan range.
- Single table two side bounded range scan. we can check our physical plan to ensure whether there is only one table scan in the plan tree and this table scan contains a two side bounded scan range.
- Single table Index lookup. we can check our logical plan and physical plan to ensure there is only one table scan in the logical plan tree and there is a lookup join(non covering index) or there is a table scan in the physical plan tree(covering index).



### Query should run in distributed mode

- Any other plans not exists in above section.
- Example:
- Single Table Scan have no scan range which means it is a full table scan.
- Multi join complex queries like TPCH querys.

### How to verify

We expect all queries of Sysbench can run in local mode automatically and all queries of TPCH can run in distributed mode automatically.

Example of Sysbench query
```
local stmt_defs = {
   point_selects = {
      "SELECT c FROM sbtest%u WHERE id=?",
      t.INT},
   simple_ranges = {
      "SELECT c FROM sbtest%u WHERE id BETWEEN ? AND ?",
      t.INT, t.INT},
   sum_ranges = {
      "SELECT SUM(k) FROM sbtest%u WHERE id BETWEEN ? AND ?",
       t.INT, t.INT},
   order_ranges = {
      "SELECT c FROM sbtest%u WHERE id BETWEEN ? AND ? ORDER BY c",
       t.INT, t.INT},
   distinct_ranges = {
      "SELECT DISTINCT c FROM sbtest%u WHERE id BETWEEN ? AND ? ORDER BY c",
      t.INT, t.INT},
   index_updates = {
      "UPDATE sbtest%u SET k=k+1 WHERE id=?",
      t.INT},
   non_index_updates = {
      "UPDATE sbtest%u SET c=? WHERE id=?",
      {t.CHAR, 120}, t.INT},
   deletes = {
      "DELETE FROM sbtest%u WHERE id=?",
      t.INT},
   inserts = {
      "INSERT INTO sbtest%u (id, k, c, pad) VALUES (?, ?, ?, ?)",
      t.INT, t.INT, {t.CHAR, 120}, {t.CHAR, 60}},
   random_points = {
      "SELECT id, k, c, pad
          FROM sbtest1
          WHERE k IN (%s)"
   }
}
```

### New Query Mode

Add a new type of query mode `auto` and use it by default. User can still choose `local` or `distributed` if they want.


## Unresolved questions

There is no guarantee of how much IO or rows being fetched in local mode. 

Think about two side bounded range scan. We don't know how much rows will return. 

Think about index point lookup if this is a poor index with low cardinality.

## Future possibilities

If we have a cost model in the future, we can decide what execution mode should used for the query by its cost.
