---
feature: managed_range_cache_range_lru
authors:
  - "jon-chuang"
start_date: "2022/12/01"
---

# The Design of `ManagedRangeCache`'s Last-access-time Oracle - `RangeLru`

## Summary
The design of a mechanism for evicting ranges last accessed before a certain epoch for the `DynamicFilter` operator

## Details

The dynamic filter requests a given range at every epoch. As this range is the difference between the previous and current pointer, the path of these ranges as a function of the epoch is always continuous.

The current `DynamicFilter` cache, the `RangeCache` differs from other caches in that it caches logical ranges rather than physical elements. That's because the basic unit of storage access for `DynamicFilter` are logical ranges rather than concrete keys.

At any point in time, the `RangeCache` owns a contiguous range.

In-line with our non-OOM and memory-efficiency requirements, we require a way to evict ranges that have not been requested by the operator for a long time. We choose to do so in a similar fashion to `ManagedLru` - based on the last-accessed time of a given subrange (the subranges being a partition of the total range owned by `RangeCache`). 

However, the problem is complicated by the fact that a given subrange may be accessed at a later time. Thus, we need to be able to update the last accessed time of old subranges, as well as easily determine the ranges whose last-accessed time falls below some watermark.


### Requirement
The managed LRU cache requires an oracle to determine the subranges last accessed at a given epoch. This way, manager can request deletion of those subranges.`

### Properties of the Last-Accessed Subranges
We know the following facts:
1. Due to the properties of `DynamicFilter` whose requested ranges form a continuous path, every continuous range of epochs (including a single epoch) maps onto a continuous range.
2. In particular, this means that evicting e.g. [x, y) from [x, now], resulting in the epoch ranges [x, y), [y, now] will produce two contiguous ranges: range to evict, and remaining range. In particular, the ranges will be contiguous at one of their endpoints.


### Design

Our oracle for determining subrange last-accessed time will be boiled down into a datastructure, `RangeLru`.

`RangeLru` has two operations.

```rust
/// This updates the last accessed time for a given range. 
/// `epoch` must be greater than any previously seen epoch.
fn access(&mut self, range: ScalarRange, epoch: u64)

/// This obtains the ranges subject to deletion in a given epoch
fn evict_before(&self, epoch: u64) -> Vec<ScalarRange>
```

1. The main data structure (main index) is a `BTreeMap { epoch -> Vec<ScalarRange> } ` to facilitate the `evict_before` operation.
2. The secondary index is a modified `BTreeMap`,  storing the left bounds of all the subranges, via the data `BTreeMap { (ScalarImpl, is_exclusive: bool) -> epoch) }`,  allowing one to look up the epochs intersecting with a given range, with an additional field `lower_unbounded_range_epoch: Option<u64>` as lower-unbounded range's existence and epoch cannot be represented. 

Notes:
1. `evict_before` needs to delete all ranges below the watermark from both the main index and the secondary index
2. `access` needs to overwrite any range that overlaps with the accessed range in both the main and secondary indexes. In the former, it will modify the endpoints of the ranges that it partially overlaps with, and delete ranges that it fully overlaps with. In the latter, it will delete the left bounds that it fully overlaps with.



