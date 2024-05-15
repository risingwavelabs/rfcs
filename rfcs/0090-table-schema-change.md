---
feature: table_schema_change
authors:
  - "@BugenZhao"
start_date: "2022/12/9"
---

> Migrated from [Notion](https://www.notion.so/Table-Schema-Change-dd4a452cde554005ae8668c7995f6d2a).

# Table Schema Change

## Overview

In the streaming system, data is continuously ingested and processed in real time. This requires the table or the source to accommodate new fields and evolve its schema over time, according to the actual business. Therefore, the ability to modify the schema is a must-have feature for almost all production scenarios.

This doc will mainly talk about a possible design of schema change of the **tables** in RisingWave, including the following aspects:

1. How to handle the `ALTER TABLE` requests by frontends and the meta service.
2. How to “update” the streaming pipeline of ingesting and materializing the table source data.
3. How to make later reads and writes compatible with the historical materialized data, and update them in a gradual manner if necessary.

Based on this design, applying schema change to the connector sources can be implemented similarly.

- For step 1, we need another mechanism to retrieve the schema change event from the source message channel in the **compute nodes** and report it to the meta service to apply, compared to the direct `ALTER TABLE` requests from the **frontends**. This significantly varies between different connectors, which may require another detailed doc.
- Then for append-only sources, we take step 2, and it’s done. For materialized sources, as it’s basically the same as the table, we’ll take step 2 and 3.

Note the schema change is not confused with the materialized view upgrade (or “alter mv”). By changing the schema of a base table, the materialized views based on it remain untouched (even for `SELECT *`). Therefore, we’re here mainly taking the **adding columns** as an example, while dropping columns is similar except for additional reference counting check for the columns used by the downstream materialized views.

## State Compatibility

The freshness or latency is critical to a streaming system. To change the schema without blocking the table for a long time, we must not transform the historical data in one turn. Instead, we need a mechanism for interpreting the old data with an updated schema.

Currently, to persist table (materialized view) data, we simply encode the datums in a tuple one by one as the storage value, according to the order specified by the columns in the catalog. To interpret it, one should know exactly the same schema to build a deserializer. Therefore, a natural way to achieve compatibility after schema change is to let the table layer hold every possible **version** of catalogs, and choose the corresponding schema for deserialization based on the version of each key-value record. Under the such approach, we need to carefully handle the synchronization and garbage collection of the multiple versions among all components of RisingWave, which sounds a little tricky. Below are the deprecated design about this approach.

- (deprecated) Details of **Multi-Version Table Catalog**
    
    The freshness or latency is critical to a streaming system. To change the schema without blocking the table for a long time, we must tolerate the state with a different schema for a while as an **intermediate state**. To handle that, multiple versions of metadata must be maintained simultaneously so that we can correctly resolve the state (compatible states in [**Compatible states**
    If the new executor can be compatible with the original states, then we can implement the logic of compatibility in executors and then replace it without other changes. For example, Materialize executor with multiple versions of schema can process the storage records before or after a schema change, like the [example 2](https://www.notion.so/Towards-the-Online-Streaming-Plan-Change-f179066d228a461dbb032e2367cba425?pvs=21).](https://www.notion.so/Compatible-states-If-the-new-executor-can-be-compatible-with-the-original-states-then-we-can-implem-98b31cf4820444dd992e5e065e3bf190?pvs=21)). In this case, it’s multi-version table catalog.
    
    Table catalog is persisted by the meta service, and used by frontends for planning query, compute nodes for accessing the table states, and compactor to maintain stuffs like bloom filter. As the source of the truth, it’s obvious that the meta service must maintain multiple versions of table catalog during the intermediate state. Considering the frontends take a full replica of the catalogs from the meta service, and use them to instruct how batch queries should interpret the states of materialized views, they also need to maintain multiple versions.
    
    ### Format
    
    The format for storing multiple versions can be trivial. One approach is to add a (reversed) `version` field after the `table_id` as a composed key, to mimic an MVCC key-value storage.
    
    ```protobuf
    table 1001, version 2 -> "int, decimal, varchar"
    table 1001, version 1 -> "int, decimal"
    table 1002, version 1 -> "double, int"
    ```
    
    Another approach can be directly saving a series of catalogs per `table_id`, sorting them in reverse order by `version`. A single entry can be directly used in a scanning plan for providing multi-versions of the catalog.
    
    ```protobuf
    table 1001 -> [
    	version 2, "int, decimal, varchar";
    	version 1, "int, decimal";
    ]
    table 1002 -> [
      version 1, "double, int";
    ]
    ```
    
    Schema change itself is a kind of configuration change: the version of a catalog bumps only if there’s a new epoch, that is, each version corresponds to an `[Included, Excluded/Unbounded)` range of epochs. Therefore, it’s also possible to directly use the `epoch` value here for the version. One should note that the semantic of `epoch` here is for a series of following epochs, instead of the **exact** one.
    
    ### Garbage Collection
    
    As the schema of the table being changed, it’s possible that we have more and more versions of catalog in the meta store and we should find a way to clean them up. A version of catalog can be cleaned up only if there’s no records in the corresponding schema.
    
    This is what the compactor is responsible for: it helps us to transform the existing data to the latest schema by inserting cells with default value when doing compaction in the background. We’ll talk about this later detailedly in the “storage” section, while typically it reports the version watermark of each table to the meta service afterwards, and the meta service and frontends will do garbage collection according to that.
    

What if the encoding itself contains enough information for schema compatibility? Then we don’t need any extra metadata like the schema history! Given the ID of a column in the table is **immutable and unique**, if we persist the column ID along with the datum in the storage value, we can always locate the columns we need regardless of whether the schema has changed. This leads to **column-aware row encoding**, which is a simplified version of the [TiDB row format](https://github.com/pingcap/tidb/blob/master/docs/design/2018-07-19-row-format.md).

> Manipulating primary key is not supported now.
> 

### Column-aware Row Encoding

```
(Vxxx WWBB) (1 byte)         ... flags
N (1~4 bytes)                ... number of non-null datums
[column_id] (1~4 * N bytes)  ... sorted array of column ids of non-null datums
[offset] (1~4 * (N-1) bytes) ... byte offset of each datum in [datum]
[datum]                      ... datums in value encoding (w/o type flag)
```

- The first byte is for flags.
    - `V` is always set to 1 for encoding compatibility.
    - `WW` indicates whether the row is **wide**, that is, whether the maximal non-null column ID in this row can be represented with `u8`, `u16`, or `u32`. This decides the unit size of `[column_id]` and the size of `N`.
    - `BB` indicates whether the row is **big**, that is, whether all offsets can be represented with  `u8`, `u16`, or `u32`. This decides the unit size of `[offset]`.
- The column IDs are sorted for better searching. If only a few columns are needed, we can apply binary search to locate the offsets and directly deserialize them. If most columns are needed, we may deserialize all columns and only keep the needed ones.
- The datums are serialized with value encoding, without the type flag as before. To deserialize a datum, one must provide the type for this field.

With the unique column ID encoded, we’re able to correctly read the materialized data without considering the synchronization between the storage value and the schema, like below. Even if we try to read a newly dropped column with the old schema, we’ll get a `NULL` for it, which also sounds reasonable.

| Column | In storage value | Not in storage value |
| --- | --- | --- |
| In schema | Non-null datum | Null
(null originally, or add column) |
| Not in schema | Invisible, ignore
(drop column) | N/A
(GC-ed) |

The pros of this encoding are the reduction of the overall complexity of the schema change, as well as the efficient column pruning support by column binary search and datum random accessing. The cons are more storage usage on S3, even if we’re using the minimal representation with utilization of big/wide flag. I believe this can be optimized by some trivial block compression technique, as the non-null column IDs are likely to be similar between rows.

For internal states, it might be okay to keep the original value-encoding format since there’s no short-term plan for supporting stateful operator upgrade. But I still suggest prepending a version flag other than column-aware encoding’s, just in case. For in-memory compacted row in cache, we should definitely keep the original one as the memory is precious.

### Catalog Version

Even if we don’t need to maintain a series of versions of the catalog for schema change, it’s still useful to introduce a per-table **version** to its catalog, which’ll be explained later. Considering that every DDL that changes the streaming graph is a configuration change and requires a checkpoint, we can directly use the **epoch** number ****of this checkpoint as the version.

```protobuf
message TableCatalog {
  ...
  Epoch version = 114514;
}
```

Note that the version epoch here may have slightly different semantics compared to the concrete epoch in each storage key-value record. Basically, a version epoch corresponds to an `[Included, Excluded/Unbounded)` range of epochs of states.

## Schema Change Procedure

In this section, we’ll talk about the procedure of how a schema change request will be executed.

1. User sends an `ALTER` DDL to one of the frontend.
    
    ```sql
    ALTER TABLE site_code_scan
    ADD last_pcr_test datetime;
    ```
    
2. The frontend will resolve this table and mutate the catalog **locally**, and plan a materialization streaming graph with the new catalog.
    
    > A table in RisingWave is the same as a materialized view with `SELECT * FROM table_source`, which is planned by the frontend and **not much understood** by the meta service. So for the mutated table, we still let the frontend to do the same thing **as if** it’s a new table.
    > 
    
    Each column in the catalog has a unique `column_id`, which is irrelevant to the position in the schema and should be never reused even if the column is dropped. To add a column, we allocate a new `column_id` for it so it won’t be confused with other columns.
    
3. The frontend sends a request to the meta service, asking to replace the complete streaming graph of the table with the newly planned one. This corresponds to [one case](https://www.notion.so/Towards-the-Online-Streaming-Plan-Change-f179066d228a461dbb032e2367cba425?pvs=21) of plan change.
    
    ```protobuf
    message ReplaceTablePlanRequest {
      // The new table catalog, with the correct table id.
    	TableCatalog table = 1;
      // The new materialization plan, with all schema being updated (re-planed).
      StreamFragmentGraph fragment_graph = 2;
      // The current version resolved by the frontend.
      Epoch current_version = 3;
    }
    ```
    
    As the catalog is pushed asynchronously to the frontend, it’s possible that the frontend operates on a stale catalog and leads to an unexpected result. So the `current_version` is required and the meta service will test it before taking action.
    
4. The meta service resolves, schedules, and builds the new actors based on the new graph. Then it issues a barrier to let all downstream materialized views to connect to the new Materialize fragment, following the procedure [here](https://www.notion.so/Towards-the-Online-Streaming-Plan-Change-f179066d228a461dbb032e2367cba425?pvs=21). During this step, all other DDLs to this table are rejected.
    - **MV on MV**
    We’re always working on a fixed set of column IDs in the downstream materialized views, including the snapshot part. So adding columns will never affect the downstream materialized views, even if they’re still creating.
        
        > For dropping referenced columns, we may leave the options to the users for whether to reject this, or tolerate inconsistency and fill the `NULL` by default.
        > 
    - **Ongoing DMLs**
    By dropping the old executors, the receiver part of DML channel is also dropped, so the ongoing DMLs will be naturally aborted and the user will receive an error for this.
        
        > This behavior is simple enough but not atomic. To achieve better atomicity, one alternative might be to add an `RwLock` for each DML channel, where the read guard is acquired for doing DMLs and the write guard is for doing DDLs like schema change. So we can always ensure there’s no ongoing DML by holding all write guards before replacing the plan.
        > 
5. Finish the procedure and notify all subscribers about the new catalog with `Update`.

After the procedure is done, the frontends will be able to handle DMLs again. Due to asynchronous notification, it’s also possible that a frontend issues a DML with old schema. For this case, the compute node is able to reject it as it already get known of a newer schema from the table source manager (or DML manager) according to the version field. However, for DQL with old schema, this can be acceptable as long as the pinned snapshot is earlier than the schema change.

## State Maintenance

### Adding Columns

As a downstream database, we’re not going to support specifying a non-null default value for columns. Even if the users really want to specify a default value for the historical data, considering the new column is invisible to current materialized views, they can create a new materialized views with explicit `COLAESE` on the column to achieve this.

This is why we omit the null columns in the encoding compared to the TiDB’s design: we don’t differentiate between a null cell and a default cell. As a result, the original row without the newly added columns has the exactly same representation **as if** the new columns are filled `NULL`, so there’s no need for backfilling the old states!

> If we do support non-null default value, we need to record the NULL column IDs in the row to distinguish with the default columns. The (latest) default value itself is always persisted in the table catalog. Note that altering the default value of a column affects historical data as well, under this approach.
> 

### Dropping Columns

However, for dropping columns, it’s beneficial to cleanup the non-null cells from the old states to save space, which can be done in a **gradual and asynchronous manner** by the compactors. To achieve this, the compactor peeks the column IDs in the value to find whether there’s a non-null column not in the current schema anymore. If so, the compactor deserialize this row and serialize it back according to the latest schema, omitting the dropped columns. This will introduce additional overhead of ser/de to the compaction, though.

As an optimization, we can maintain a mapping from table to its (single) catalog version in the SST metadata. For SSTs generated by the compute nodes, this is just the epoch of the checkpoint. The hummock manager maintains a **target catalog version** for each table which bumps **only if** there’re columns being dropped. Therefore, the compactor will not rebuild the row unless it finds the target catalog version is higher than the version recorded in SST.

> We cannot directly use the epoch after the key as the per-key catalog version, or rebuilding a row to a new version requires modifying the key’s epoch as well, which’ll be tricky to be implemented correctly.
> 

## Discussions

- How can we handle altering column types with casting?
    
    One approach is to allocate a new ID for this while recording a mapping in the table catalog. The mapping will be used for doing the cast on compute nodes or compactors. This is actually a minimal multi-version catalog, and GC seems unnecessary.
    
    Another approach is to encode the type flag in the row and let the deserializer to do the casting, while keeping the original column ID.
    
- How to handle Indexes?
    - (Dylan) PG behavior: dropping columns of the index include columns will cause the index to be dropped.
    
    ```sql
    tpch=# create table t (v1 int, v2 int, v3 int, v4 int);
    CREATE TABLE
    tpch=# create index iddd on t(v2,v3) include(v4, v1);
    CREATE INDEX
    tpch=# \d t;
                     Table "public.t"
     Column |  Type   | Collation | Nullable | Default
    --------+---------+-----------+----------+---------
     v1     | integer |           |          |
     v2     | integer |           |          |
     v3     | integer |           |          |
     v4     | integer |           |          |
    Indexes:
        "iddd" btree (v2, v3) INCLUDE (v4, v1)
    
    tpch=# alter table t drop column v1;
    ALTER TABLE
    tpch=# \d t;
                     Table "public.t"
     Column |  Type   | Collation | Nullable | Default
    --------+---------+-----------+----------+---------
     v2     | integer |           |          |
     v3     | integer |           |          |
     v4     | integer |           |          |
    ```
    
    - (Bugen) As we make including all columns the default behavior, maybe it’s better to also update the index to include the new column? This is a special and simplified case of upgrading a materialized view.
