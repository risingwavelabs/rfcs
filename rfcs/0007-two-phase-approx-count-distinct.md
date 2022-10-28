---
feature: two_phase_approx_count_distinct
authors:
  - "soundOfDestiny"
start_date: "2022/10/28"
---

# 2-phase Approx Count Distinct Agg

## Summary

2-phase Approx Count Distinct Agg which uses the bucket_id as distribution key to calculate vnode for 2-phase agg, as jon-chuang proposed in https://github.com/risingwavelabs/risingwave/issues/3414.

## Motivation

Currently the `partial_to_total_agg` of `Approx Count Distinct Agg` is `Sum0`, but it is a bad estimation because $Cardinality(S1 \cup S2) \neq Cardinality(S1) + Cardinality(S2)$.

## Design

* Alter group-by key of input into `bucket_id || group_by_keys`. (For example, in GlobalSimpleAgg whose `group_by_keys` is `None`, the new group-by key will be `bucket_id`.) We need to leverage `StreamExchange` to alter distribution key accordingly if necessary.
* The partial agg is hash agg.
* The partial agg should maintain bucket state in HyperLogLog algorithm. The implementation of the partial agg is detailedly described in https://github.com/risingwavelabs/risingwave/issues/3414.
* The partial-to-all agg is global simple agg.
* The partial-to-all agg should compute the harmonic mean of partial results, which is easy.

## Unresolved questions

We use as many as `bucket_num` groups to do 2-phase aggregation for Approx Count Distinct Agg. However, is it faster than Count Agg?

## Alternatives

We can keep original distribution (aka distribution key of input) in 2-phase agg, but with very huge state as described in https://github.com/risingwavelabs/risingwave/issues/3414.

## Future possibilities

We can replace HyperLogLog algorithm with an estimation algorithm which is more suitable with streaming process.
