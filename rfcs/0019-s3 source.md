---
feature: S3 Source Design
authors:
- "Bohan Zhang"
  start_date: "2022/11/09"
---

# S3 Source Design

## Motivation

Support load data from Amazon S3 and other Simple Storage Services.

- The previous design is available [here](https://www.notion.so/risingwave-labs/Deprecated-RFC-S3-Source-Design-632aa78264b347f9a4a37fd89a91d785).

## Design

More and more scenarios require us to support data synchronization from S3, which challenges our existing abstractions.
The main problem in the previous design is that a source could be read many times by multiple materialized views but the SQS message can only be consumed once, which leads to data inconsistency.

First of all, we believe that s3 source is append only. This is consistent with the semantics of s3, which does not allow a file to be modified, but only to delete the old one and add the new one. So s3 source will re-read the full amount of updated files.

In the source abstraction, one source can only have one enumerator to discover the upstream changes. Although we cannot implement multiple consumptions on an SQS, we can have the enumerator cache the queue results.

At the first start, the enumerator first `ls` the object in the bucket and stores it in a hashset for persistence, at which point we consider the source creation complete. Then it starts to consume SQS and maintains a list of all objects that need to be consumed by SQS.

When the source executor is started, the elements in the hashset are assigned to the specific executor by a consistent hash, and the new added objects are distributed by config change. One of the optimization points here is to transfer only the changed paths, which reduces memory consumption.

### Scale

In this design the enumerator always provides M hashslots and assigns these slots to N nodes. 
For example, the enumerator provides 256 (0~255) slots and there are three executors, each partition will look like:

```plain
S3Partition{ "min_slot": 0, "max_slot": 85 }
S3Partition{ "min_slot": 86, "max_slot": 171 }
S3Partition{ "min_slot": 171, "max_slot": 255 }
```

Each executor get the object names and use a hash function to filter out the names meet the restriction. 
We can use consistent hash here for load balancing.

We can easily scale in or out by changing the slot assignment in the source manager, and it will push down new assignments by barrier.

### Failover

Just like other connectors, meta is responsible for push full objects assignments to the executor, regardless whether we have read the file.

As we sustain `(object_name, latest_offset, is_reach_end)` in the state store. When we reach the end of one file, the source executor will set `is_reach_end` to `true`. For failover or migration scenes, we can filter the unfinished objects. Even if the compute node fails when reading a large file, we know it will continue reading later.

## Unresolved questions

* If CN fails when reading a large file A, and SQS sends a message to delete the file, whether we still distribute it?

## Future possibilities

To avoid the state table expands endlessly, we need a TTL for it.
