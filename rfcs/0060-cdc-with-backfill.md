# RFC: CDC Source with Backfill



## Motivation

Ingesting a CDC source consists of two phases: full snapshot phase and incremental binlog phase. Currently we leverage the Debzium connector to ingest data from upstream databases, however there are several issues with the Debzium connector:

- During the full snapshot phase, a global lock needs to be acquired on the upstream database, which is invasive to the upstream application.
- Taking a full snapshot of a large table can have performance limitations when using the Debzium connector. This is because the connector is single-threaded, which can cause the consuming of binlog events to stall, resulting in sub-optimal freshness of the entire pipeline.
- If a crash occurs during the full snapshot phase, the full snapshot needs to be restart from scratch after recovery.

Previously, we discussed a solution that Flink CDC uses to support parallel and incremental snapshot in CDC. However it is somehow an ad-hoc solution (e.g. source executor needs to report the binlog high watermark for each chunk, needs a config change to switch from snapshot phase to streaming phase)**.** This RFC proposes an alternative that aims to leverage the backfill algorithm.

## Recap the backfill algorithm in RisingWave

![Untitled](images/table.png)

Suppose the epoch of initial barrier is e1, backfill executor will scan the table with snapshot e1 and maintain the scan progress with an offset. Meanwhile, it will buffer the changelog events in memory until a new barrier comes.

When a new barrier comes, suppose the epoch is e2. The backfill executor will immediately stop fetching rows in snapshot e1 and flush changelog events with PK ≤ offset to downstream. Then it will continue scanning the table from the offset with snapshot e2.

The backfill executor will not emit duplicated or conflicts events because it can always read a snapshot of the table that is exactly the same as the given epoch. However, when selecting a MySQL/Postgres table, **we don’t know what the exact binlog position being reading is**. That is the reason why the watermark-based algorithm mentioned in [RFC: CDC Source Parallel Snapshot](https://www.notion.so/RFC-CDC-Source-Parallel-Snapshot-a6ee43baf7834f328c01e624391568a7?pvs=21) proposes to bound the select snapshot in a binlog window.

![                                 Watermark-based table selection](images/snapshot-window.png)

                                 Watermark-based table selection

For example, after we got the low watermark binlog position, we select a snapshot of the table. Between the time point of snapshot and the low watermark, there may be some events have been committed into the binlog because we don't acquire locks to prevent updates.

Thus to employ a CDC solution without database locks the application needs to handle those potential conflict events.

**Option 1**: Handle conflicts in the compute layer

This is what Flink does. Since the table created in Flink wouldn’t be materialized and it has to resolve conflict events in the source executor before emit to downstream. Flink will consume binlog events from `lw` to `hw`, and upsert the table snapshot with those events to resolve conflicts.

**Option 2**: Leverage the storage to handle conflicts (this RFC)

We have our own storage layer and `Table` will always be materialized before emit to downstream, so we can leverage the storage (e.g. mview operator) to handle conflict events.

## Design

### Single parallelism

![Single Parallelism](images/single-parallel.png)

- When receive a barrier, the source executor will first read the current binlog position as `binlog_begin`
- After reading the binlog position, start a select query order by primary key for the whole external table
- Iterate each row in the snapshot and emit to downstream, meanwhile we need to **buffer** upstream binlog events
- When a new barrier comes:
  - immediately stop the scanning of snapshot (suppose we stop at `current_offset`)
  - read the current binlog position as `binlog_end`
  - process the buffered upstream binlog which should contain events from `binlog_begin` to `binlog_end`:
    - Emit event with pk ≤ `current_offset`
    - Ignore event with pk > `current_offset`
  - Set `binlog_begin = binlog_end`
  - Start a **new** select query from the `current_offset` to the external table and continue iterating rows

### Multi parallelism

We can further parallel the above algorithm. And to prevent creating multiple changelog stream in the upstream database (each changelog stream requires an IO thread), we can add a new singleton **Changelog** operator to broadcast the changelog to each parallelism of the source executor.

![Multi Parallelism](images/multi-parallel.png)

## Failover

To support incremental snapshot, we need to maintain the upstream binlog offset and the offset of upstream table which denotes the progress of backfill.

Upon recovery, the Changelog executor will replay binlog events from the offset in the latest checkpoint and source executor will continue the backfill process from the stopped offset.

## Summary

With the approach in this RFC, we can achieve the following goals

- Lock-free snapshot loading
- Incremental snapshot loading
- Don't need an additional configuration change to switch from full snapshot phase and streaming phase
