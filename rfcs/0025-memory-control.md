---
feature: Memory Control
authors:
  - "Eric Fu"
start_date: "2022/10/24"
---

# Memory Control



## Summary

This RFC proposes a detailed and practical way to control the memory usage for both streaming and batch executors on compute nodes.

## Motivation

We had discussed a lot about how to allocate and manage memory before. [RFC: Yet another simple idea for memory management](https://singularity-data.quip.com/CldAAcFmzZSO/Yet-another-simple-idea-for-memory-management) proposed a simple way to control the memory usage of streaming executors by controlling the *watermark epoch* of the LRU caches on a compute node, which was finaally adopted in our system, surpassing [RFC: Simpler Dynamic Memory Management Among LRU-Caches](https://singularity-data.quip.com/A1kHAOBUo3Im/RFC-Simpler-Dynamic-Memory-Management-Among-LRU-Caches) and [RFC: Dynamic Memory Budget of Streaming Operators](https://singularity-data.quip.com/J9KYAQc2xIbr/RFC-Dynamic-Memory-Budget-of-Streaming-Operators).

Recently, [RFC: Coarse grained resource management for batch query engine](https://github.com/risingwavelabs/rfcs/pull/11) tried to control the memory of batch executors as well, but the 1st and 2nd version of design was controversary because some designs were too complicated or infeasible. Luckily, the RFC has been refined a lot during the discussion, which finally comes to the solution of this RFC. 

## Goals

The streaming and batch executors should be isolated as far as possible. Specifically, given that the streaming jobs are running, the memory consumpted by batch queries should be strictly limited. In another word, streaming engine has higher priority and a portion of memory is guaranteed to be usable by it whenever needed.


## Design

Memory of compute node is a scarce resource. Due to the nature of streaming processing, the memory will be sooner or later consumed out. Therefore, we need to limit the memory used by each component to prevent OOM (Out-of-Memory).

![](images/overall-memory-allocation.png)

**This doc will focus on the memory control on both streaming and batch executors**, assuming the storage, network buffer, system metadata, etc. always take a fixed size of memory.

Our approach uses a background coroutine to continously run the loop below:

1. Get the current memory usage (**Measure**)
2. Decide whether and which component needs to release memory (**Policy**)
3. Take actions to release memory (**Action**)

### Measure: How to measure the memory usage?

Here we propose to use the simple and stupid way to calcute the memory usage - **esitimating** the memory consumed by all the in-memroy data structures and aggregate in levels. This is opposite to the approaches that relies on memory allocators to get an accurate memory usage.

How to estimate the memory usage depends on the data structures:

- For indexing structures like `HashMap`, `BTreeMap`, etc., we could estimate the memory size according to the `capacity()` or something else. 
   - @liurenjie1024: It also seems possible to inject an `Allocator` to count the exact allocated space, as long as it exposed such generic type.
- For pure data structures like `DataChunk`, `Vec<Datum>`, etc., we could compute the exact size it used by summing up the sizes of internal arrays & variables as well as padding and other overhead.

A CN-level memory manager will collect the memory usage in fixed interval like 100ms. It collects the memory recusively regarding to the hierarchy: stream/batch -> fragments -> actors -> executors. To simplify the ownership, executors's memory usage can be stored in `AtomicUsize` and be updated by executor on every `next()`.

### Action: How to release the memory?

- **Streaming**: We have implemented a CN-level LRU Manager to control the watermark epoch of all stateful operators. Once memory usage reaches the limit, the watermark epoch will be increased to evict some least-recent entries. See [RFC: Yet another simple idea for memory management](https://singularity-data.quip.com/CldAAcFmzZSO/Yet-another-simple-idea-for-memory-management) for details.
- **Batch**: We release the memory from batch queries by simply killing the queries with largest memory consumption.

Notice that the background coroutine works in an asyncronized style, that is, there is no strict guarantee that a task must be killed immediately once the memory usage exceeds limit. 

### Policy: Which component should give out memory?

#### Without overselling

2 straight-forward options are exposed to users:

- `batch_memory_limit_mb`: The **upper bound** of memory consumed by batch executors
- `streaming_memory_limit_mb`: The **upper bound** of memory consumed by streaming executors

![](images/policy-without-overselling.png)

Once the memory usage exceeds the limit, we run the actions to release some memory from streaming or batch.

#### With overselling (optional)

In total, there are `batch_memory_limit_mb + streaming_memory_limit_mb` for both streaming and batch executors. @liurenjie1024 proposes to ultilize the memory space by allowing streaming and batch executors to use memory from each other, but must return it back immediately once the "owner" requires.

2 additional parameters need to be introduces:

- `batch_memory_reserved_mb`: The memory that must be reserved for batch no matter whether it's free. Must be `<= batch_memory_limit_mb`
- `streaming_memory_reserved_mb`: The memory that must be reserved for streaming no matter whether it's free. Must be `<= streaming_memory_limit_mb`

![](images/policy-with-overselling.png)

The `free - reserved` size could be borrowed by each other. For example, if `limit` of streaming is 500MB, within which 300MB has been `allocated`, and `reserved` is 100MB, then 200MB - 100MB = 100MB is allows to be `overflowed` by batch.

Why `reserved` is necessary? This is because we are using an async style way to manage memory i.e. with a background coroutine. As a result, **any actions will be later than actual exhaustion of memory**. The `reserved` area is designed to mitigate the problem by reserving some space in case that memory cannot be reclaimed immediately.

## Implementation

Obviously, the "measure" part takes the most effort to implement. However, albeit not perfect, we can do this in 2 stages:

1. First, we implement the memory estimation for batch engine, and the usage of streaming engine can be estimated by `total - batch`, where the `total` is the process-level usage from `jemalloc`.
2. Then, we implement the memory estimation for streaming engine. This  will make the numbers more accurate and also allows us to inspect the memory taken by each fragment/actor/executor. 


## Unresolved questions

None


## Alternatives

TODO

## Future possibilities

Some potential extensions or optimizations can be done in the future based on the RFC.
