---
feature: unify_materialized_source_and_table
authors:
  - "TennyZhuang"
start_date: "2022/9/13"
tag: 
---

# RFC: Unify the materialized source and table

## Summary

In RisingWave, we have three types of sources now:

* `CREATE SOURCE`: Subscribe to some connectors, and don't materialize the data.
* `CREATE MATERIALIZED SOURCE`: A syntax sugar that composes a `SOURCE` with a `MATERIALIZE` operator.
* `CREATE TABLE`: A table that supported `DML` on it.

TLDR, In the RFC, we want to unify the materialized source and table.

## Design

* `CREATE SOURCE`
  * ****Optionally**** subscribe to some connectors
  * ****Don't**** materialize the data
  * ****Don't**** allow primary key constraints
  * ****Only**** allow append-only streams as input (debezium or other CDC sources are banned).
  * ****Only**** support `INSERT`
  * ****Usually**** used as a fact table
* `CREATE TABLE`
  * ****Optionally**** subscribe to some connectors
  * ****Always**** Materialize the data
  * ****Allow**** the primary key constraints (We will ignore the duplicated insert operations)
  * ****Allow**** retractable streams as input (We will ignore the delete operation on unexist records)
  * ****Allow**** `INSERT`, `DELETE,` and `UPDATE` (but no good transactional guarantees)
  * ****Usually**** used as a dimension table

## Benefits

There are multiple motivations to do that:

* For table or materialize source
  * The materialized source and table are very similar, and the key point it we can treat RisingWave as ****the source of truth**** for them.
  * If we are the source of truth, why not allow DMLs on a materialized source?
  * In our previous design, users can choose the materialized source for subscribing data or can choose a table for revising data, but they can't choose both, so the table is almost useless.
  * Users can subscribe to some connectors on materialized sources but use DMLs to revise their data by DMLs.
  * We can check the consistency and constraints on materialized sources, so retractable streams and constraints are allowed on that.
* For source:
  * We can't check the consistency on non-materialized sources, so only append-only streams can be used as input.
  * We can't check the constraints on non-materialized sources, so we will generate a row-id for every record and PK is disallowed.
  * `INSERT` on source is very useful for testing, so we may implement it.

## Unresolved questions

* (Eric) What’s the to-do list? From my understanding it seems the major change is on concepts, so is there anything need to be done in implementations? Need further design of for **TableExecutor** (the executor to write changes into *table*). Particularly, the table may has PK so the change stream must be shuffled by PK before being processed by TableExecutor.
