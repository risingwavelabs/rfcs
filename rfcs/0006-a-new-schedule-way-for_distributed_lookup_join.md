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

I propose a new way schedule way for distributed lookup join by scheduling all the lookup join operators along with the inner side table scan. We use a new kind of Exchange operator to shuffle outer side data to match inner side data distribution. After shuffling outer side data to inner side node, we can execute LookupJoin locally without calling a RPC to other CN to lookup data.

### New kind of exchange/Distribution

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
  message VHashInfo {
    repeated uint32 vnode_mapping = 1;
    repeated uint32 key = 2;
  }
}
```


### 

Each LookupJoin executor will fetch the outer side data already shuffled by VHashInfo and then do the lookup join locally.




## Unresolved questions

* Are there some questions that haven't been resolved in the RFC?
* Can they be resolved in some future RFCs?
* Move some meaningful comments to here.

## Alternatives

What other designs have been considered and what is the rationale for not choosing them?

## Future possibilities

Some potential extensions or optimizations can be done in the future based on the RFC.
