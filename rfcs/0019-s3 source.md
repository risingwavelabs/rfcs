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

In the source abstraction, one source can only have one enumerator to discover the upstream changes. However, the executors in different fragments have different consuming process and different state table.
It is a trivial idea to discover the changes by every executor and the enumerator only confines the concurrency.

The source executor will start a thread when starting a S3 connector and list all objects in the given bucket.
After a regex match, the object names will be passed to reader by a channel. 

- [ ] The channel can break the abstraction of source, we may consider a new file-system source executor apart from the original one.

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

The scale in and scale out operations are intuitive, just adjust the `min_slot` and `max_slot` for each partition.

### Failover

As we only sustain `(object_name, latest_offset)` in the state store, it is unacceptable to download all objects and check whether the reader has reached the end.
The best practice here is that if the object name is in the state table, it is considered have been read, where we bring a strict rule that **every object must be read completely within one epoch**.

The failover scene can be really simple in such case, just filter out the objects not in the state table.

## Unresolved questions

* This RFC imposes a strict limit on commitment: we can only consider the current epoch finished when an object has been read.
  * We may experience a relative large chunk if encountered a large object.

## Future possibilities

To avoid the state table expands endlessly, we need a TTL for it.
