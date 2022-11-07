---
feature: DML design 
authors:
  - "st1page"
start_date: "2022/10/31"
---

# DML design

## Motivation

the [Unify the materialized source and table](https://github.com/risingwavelabs/rfcs/pull/4) will lead to 2 things

- table's data are materialized in storage and we should check the consistency and constraints when materializing
- table can both subscribe to some connectors and support `INSERT`, `DELETE,` and `UPDATE`, which meas it should serve two sources of data changes.

This rfc is to give the details of that.

## Design

### Different objects's DML

#### Source

A source get append-only records from external source and `INSERT` statment and our system will generate a row_id for each record. User can not define PK and other constraints on that. Because all row id are generated internally, we do not need to check the pk's unique constraint.

#### Table

A table can get any operation from external source and `INSERT`, `UPDATE`, `DELETE` statement. The later operations can overwrite the previous ones. If user does not define a primary key, we will generate a row_id for each record. But because user can do modification on table, we must always do the check no matter if the pk is generated internally.

#### Append Only Table(undocumented)

To simplify the definitions in our system, that is a kind of undocumented feature and we do not encourage user do that. It is just for test now.
We offer a syntax like that: `create table t1 (v1 int, v2 int) with (appendonly = true)`. Like some other OLAP systems, the "append only" indicate two things.

1. user can not define primiary key or other unique constraints on that table
2. user can not delete/update the table

### Implementation details

#### Materialize check

The consistency and constraints check will be done in materialize executor. For every operation from upstream, the materialize executor will lookup its pk from storage and fix the changes if it is conflict with the storage's data. And because of the look up queries, a cache is needed.

#### DML executor

Currently we treat the DML as a kind of source. But now we need serve the data from both connector source and DML. So the `DML executor` is introduced to merge the DML changes stream into main stream. It will hold the `TableSource` as a RX to receive the operations from batch DML executors.

#### Generate row_id in BatchInsert

When user do an insert on a table without PK, we should help to generate the row_id for each records. Now we can only generate row_id in the SourceExecutor. But now, we might need to generate row id for both external data and DML insert's data. So the `BatchInsert` executor and `SourceExecutor` will share the `RowIdGenerator`.

#### DML manager

The `TableSourceManager` will rename as `DMLManager`. It will be indexed as `table_id` instead of `source_id` of the table source. The DML batch executors can get the `TableSource` for DML operation and the `Mutex<RowIdGenerator>` easily.
