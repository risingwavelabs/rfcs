---
feature: A new schedule way for distributed lookup join
authors:
  - "Dylan Chen"
start_date: "2022/10/28"
---

# A New Schedule Way For Distributed Lookup Join


## Summary

Currently, we implement our distributed lookup join by scheduling them along with outer side table scan. In this RFC I propose to schedule distributed lookup join along with inner side table scan which can make it possible for each LookupJoin executor locally lookup their inner side data.

## Motivation

Currently our system has 2 query execution mode: local and distributed execution mode. For local executoin, we expect user will execute OLTP like queries which touch a litte data, while for distributed execution, we expect user will execute OLAP like queries which need to process a large amount of data. Currently, our lookup join perform lookup operations by calling a RPC to the inner side each time which is fine for local execution while it is expensive for distributed execution.
For this reason we need to find a better way to implement our distributed lookup join.

## Design

<img width="863" alt="image" src="https://user-images.githubusercontent.com/9352536/198513912-dfd0fea5-baef-4ed1-98fb-996d158caa12.png">

I propose a new schedule way for distributed lookup join by scheduling all the lookup join operators along with the inner side table scan. We use a new kind of Exchange operator to shuffle outer side data to match inner side data distribution. After shuffling outer side data to inner side node, we can execute LookupJoin locally without calling a RPC to other CN to lookup data.

### New kind of exchange/distribution

```
/// TableId is used to represent the data distribution(`vnode_mapping`) of this
/// UpstreamHashShard None means we don't have exact knowledge of its data distribution.
/// If UpstreamHashShard is used to encore distribution of some plan node, you must specify the
/// TableId and then the schedule can fetch its corresponding `vnode_mapping` to do shuffle.
UpstreamHashShard(Vec<usize>, Option<TableId>)
```

We can extend `UpstreamHashShard` to accept a `Option<TableId>` to represent its exact data distribution aka. `vnode_mapping`.
In this way we can use `UpstreamHashShard` to enforce plan node to satisfy a specific table distribution we want.

```
message ExchangeInfo {
  ...
  message ConsistentHashInfo {
    // `vmap` maps virtual node to down stream task id
    repeated uint32 vmap = 1;
    repeated uint32 key = 2;
  }

  message HashInfo {
    uint32 output_count = 1;
    repeated uint32 key = 3;
  }
}
```

Scheduler will replace the `TableId` of `UpstreamHashShard` by an actual `vnode_mapping` to construct a `ConsistentHashInfo` to shuffle data.

Notice: `vnode_mapping` maps virtual node to parallel unit, while `vmap` of `ConsistentHashInfo` maps virtual node to down stream task id.

Let's compare the difference between `ConsistentHashInfo` and `HashInfo`. `HashInfo` will calculate the hash code of a row and then modulo output_count to get the down stream task id, while `ConsistentHashInfo` will calculate the vnode of the row and then get the down stream task id by `vmap`.

On the scheduler side we need to schedule LookupJoin executor to its corresponding worker. In our case, we use `vnode_mapping` to determine where the LookupJoin executes.

For example:
```
// For simplicity, we considering vnode_mapping only contains 8 vnodes and their corresponding parallel units are as follow.
vnode_mapping = [2, 2, 2, 3, 3, 4, 4, 5]

// its corresponding `vmap`:
vmap = [0, 0, 0, 1, 1, 2, 2, 3]

// we create 4 LookupJoin with task id 0, 1, 2 and 3 , because we have 4 distinct parallel unit

// LookupJoin should be scheduled by their task id to corresponding worker
LookupJoin 0 -> parallel unit 2 -> worker A
LookupJoin 1 -> parallel unit 3 -> worker B
LookupJoin 2 -> parallel unit 4 -> worker C
LookupJoin 3 -> parallel unit 5 -> worker D
```

### Distributed LookupJoin
For each inner side table parallel unit we will schedule a LookupJoin executor.
Each LookupJoin executor will fetch the outer side data already shuffled by ConsistentHashInfo and then do the lookup join locally.
We can maintain 2 kind of lookup implementation by checking LookupJoin executioin env. If LookupJoin executed in frontend node it must be in local execution mode. If LookupJoin executed in commpute node it must be in distributed execution mode.


### Efficient multi point get
Distributed LookupJoin can produce lots of multi-point-get, so we need to implement a more efficient multi-point-get method avoiding small chunk and use sequential point-get instead of parallel point-get to reduce the contention.


## Future possibilities

The new kind of Exchange can be used to another scenario like HashJoin to shuffle one side of data against another side data distribution.
