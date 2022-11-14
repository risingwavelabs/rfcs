---
feature: batch_resource_management
authors:
  - "Renjie Liu"
start_date: "2022/11/03"
---

# Batch Query Resource Management

## Summary

We will introduce coarse grained resource management for batch query engine:

1. Limit number of concurrent distributed queries in each frontend node.
2. Limit the total amount of memory used by batch query engine.

## Motivation

When user executes complex sql in distributed mode in our system with huge amount of data, our compute node will abort(not panic) when running out of memory. We want to reduce process failure and impact to other users as much as possible.

### Non Goal

There exists many different fine grained resource management strategies in industry, and the goal of this rfc is not to bring them into our system.

## Design

### Limit number of concurrent distributed queries

To reduce the chance of query failure, we need to limit the number of concurrent distributed queries running in cluster. To implement this mechanism, we have two assumptions:

1. Load balancer works well, so each frontend node receives almost same amount of queries.
2. Enforcing this limitation is not mandatory, just best effort is enough.

Based on above assumptions, we will introduce a cluster wide `max_concurrent_distributed_queries` config, which limits number of concurrent distributed queries in cluster. Meta node will divide this value according to number of frontend nodes, and sync to each frontend node. Frontend node will limit this concurrent queries according to this value.

### Limit number of memory used in batch query engine

To avoid aborting in allocation failure, we will introduce a config `batch_query_max_memory`, which limits the number of memory used by all batch query executors on each compute node. To enforce this limitation, we need to pass an `Allocator` implementation to each executor, which internally uses an atomic number to count memory usage, and returns error when  There are several cases to consider when using `Allocator` in executor:

1. Falliable collections. This is the eaiest case, and we should use `try_reserve`/`try_insert` api as mush as possible.
2. Infalliable collections. In this case we should estimate its memory usage before allocation. Notice that this is managing memory by ourself, and we should not forget to free it in executor destruction.
3. Data chunks. In fact, a large part of memory is used in buffering data chunks, so we need to add `Allocator` to array and array builders, and returns error when out of memory.

With this approach, we can add memory constraints to batch executor one by one.

Other memory usages are considered as system memory, and we still need to abort process when oom.

## Unresolved questions

1. With this design, we statically divide memory between different runtimes, e.g. batch/streaming runtime. This may lead to under utilizing memory usage.

## Alternatives

A panic based approach can be found here: <https://github.com/risingwavelabs/rfcs/pull/11/files#diff-044dd2e47982ff2f25e56eaa49e7b1b9b190ecc6308a74667a8a22fdb3178af8>

## Future possibilities

Fine grained resource management.
