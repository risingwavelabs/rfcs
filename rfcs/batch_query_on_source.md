---
feature: batch_query_on_source
authors:
  - "Renjie Liu"
start_date: "2022/11/10"
---

# Batch Query on Source

## Summary

Introduce execute batch query on sources like kafka/s3.

## Motivation

We have already supported ingesting data from sources like kafka/s3, but user may want to query directly from them.

### Non Goal

* We assume that data stored in source are json or protobuf, which has no index/statistics associated with it, so we will not introduce any optimization on it.

## Design

Essentially, after user creates a source, we want to allow user to execute following sql to query data directly:

```sql

```

## Unresolved questions

* Are there some questions that haven't been resolved in the RFC?
* Can they be resolved in some future RFCs?
* Move some meaningful comments to here.

## Alternatives

What other designs have been considered and what is the rationale for not choosing them?

## Future possibilities

Some potential extensions or optimizations can be done in the future based on the RFC.
