---
feature: dedicated-serving-cluster
authors:
  - "Eric Fu"
start_date: "2022/11/07"
---

# Dedicated Batch Serving Cluster

## Summary

Run a dedicated cluster to serve batch queries.

## Motivation

- **Different traffic pattern**. Batch queries, especially the high-concurrent point queries (aka. “online serving”), requires millisecond-level low latency and high availability, while streaming jobs focus more on high throughput and cost efficiency.
- **Different access pattern on storage** of batch & streaming. Batch queries are issued by services on user side, while streaming operators always handle the changes from upstreaming system. It's possible that they have quite different distribution of access aka. hotspot. 
- In some extreme use cases, the QPS on serving side could be 100K or even higher, which may result in severe **resource contention** (CPU, IOPS, etc.) against streaming jobs.

To address these problems, here we propose to run a dedicated cluster to serve batch queries (hereafter referred to as “serving cluster”). Meanwhile, data source, sink and maintenance of MViews are still done in the primary streaming cluster.

Additionally, this should be an **optional** feature. Some simple use cases, like dashboard, may not need to support such high concurrency. Besides, free tier RisingWave Cloud users are definitely not its target users.  

## Non-Goals

The objective is to **isolate the resources** of streaming and batch instead of providing any new functional or performance features. Users pay more money and get a more stable streaming and batching performance, and that's all.

1. **Columar store**. This design primarily focuses on serving instead of OLAP, so we will not discuss about anything about columnar format. The serving cluster can be used to run some short-time analytical queries as long as the resources are sufficient.
2. **Row store in another distribution**. This design does nothing about range scan or any other requirements. Hash or range distribution is a forever debate and we should not continue it here.

## Design

Decoupling barrier and checkpoint (reference: [Support barrier-checkpoint decoupling](https://singularity-data.quip.com/QLOaAegyeApR/Support-barrier-checkpoint-decoupling)) allows us to decouple the streaming barrier from the expensive checkpointing process, but also brings more complexity to us.  

### Read Checkpointed Snapshots

In the first stage, we can simply boot up several compute nodes without scheduling any streaming tasks on them. The meta service pushes the new storage versions to the compute nodes as usual, therefore, whenever a new storage version generated, the compute node will be informed.

In the catalog, beside the existing `vnode -> parallel unit` mapping, we have to maintain another mapping of serving cluster for each table & materialized view. Below is an example, where “PU” refers to Parallel Unit. Apparently, this mapping should be updated once nodes joined or leaved the serving cluster. 

```jsx
Main Cluster: 
- PU#1: vnodes [0, 1]
- PU#2: vnodes [2, 3]
- PU#3: vnodes [4, 5]

Serving Cluster: 
- PU#1: vnodes [0, 1, 2]
- PU#2: vnodes [3, 4, 5]
```

High availability can be achieved by allocating multiple Parallel Units for each vnode, where one of them is primary and the other(s) are stand-by. Once the primary replica is unavaialble, the stand-by replica will immediately be picked by frontend as new route. This provides us both high-availability and cache-locality. For example, given 3 nodes and `replicas = 2`,

```jsx
Serving Cluster: 
- PU#1: vnodes [0, 1, 2, 4]
- PU#2: vnodes [3, 4, 5, 1]
- PU#3: vnodes [0, 2, 3, 5]
```

### Read Non-Checkpointed Snapshots

In the current design, non-checkpointing barriers are much more frequent than checkpointing barriers, so it provided both freshness and lower IO cost. However, these non-checkpointed updates are only kept in memory, more specifically, in the `StateStore` of the `MaterializedExecutor` with corresponding vnode.

#### Approach 1. Read non-checkpoint writes from primary cluster (Preferred)

> Suggested by @BowenXiao1999 and @hzxa21

Inform the serving cluster with the vnode mapping of primary streaming cluster. Once the servering cluster find it lack of the data of epoch being requested (the read snapshot epoch is piggy-backed in frontend's RPC), it can issue a RPC to fetch the changes (i.e. immutable MemTables) from the corresponding streaming CNs.

#### Approach 2. Read non-checkpoint writes from S3

Here we propose to **store** these barrier-level changes of non-checkpoint barriers to shared storage **temporarily**. These immutable MemTable SSTs will only be temporarily placed in L0, and these SST files can be dropped as long as next checkpoint completes.

To make it simpler, we can reuse the concept “compaction group”. A compaction group can be tagged as checkpointing on every barrier. Table_ids in such compaction groups will upload its immutable MemTable on non-checkpointing barriers, and a “serving-only” version will be created.

Note that serving-only versions will not affect the recovery process. On recovery, all nodes should roll back the latest completed checkpoint like before.

#### Approach 3. Contantly streaming changes from primary cluster

> Suggested by @hzxa21

Streaming (push) changes to serving nodes. 

But I think the streaming is not that "streaming". Since we hope to allow both cluster to have different SLA, and furthermore, can be scaled independently, it may be complicated than imagination. 

Perhaps we can introduce some additional machanism to push changes. This seems to be harder than approach 1 so I stopped here, but proposals are welcomed.

## Unresolved questions

None


## Future possibilities

None