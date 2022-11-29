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

We had discussed a lot about how to allocate and manage memory before. [RFC: Yet another simple idea for memory management](https://singularity-data.quip.com/CldAAcFmzZSO/Yet-another-simple-idea-for-memory-management) proposed a simple way to control the memory usage of streaming executors by controlling the *watermark epoch* of the LRU caches on a compute node, which was finally adopted in our system, surpassing [RFC: Simpler Dynamic Memory Management Among LRU-Caches](https://singularity-data.quip.com/A1kHAOBUo3Im/RFC-Simpler-Dynamic-Memory-Management-Among-LRU-Caches) and [RFC: Dynamic Memory Budget of Streaming Operators](https://singularity-data.quip.com/J9KYAQc2xIbr/RFC-Dynamic-Memory-Budget-of-Streaming-Operators).

Recently, [RFC: Coarse grained resource management for batch query engine](https://github.com/risingwavelabs/rfcs/pull/11) tried to control the memory of batch executors as well, but the 1st and 2nd version of the design was controversial because some designs were too complicated or infeasible. Luckily, the RFC has been refined a lot during the discussion, which finally comes to the solution of this RFC. 

## Goals

- The total memory usage is controlled to prevent OOM (Out-of-Memory).
- The streaming and batch executors should be isolated as far as possible.

## Design

Memory of compute node is a scarce resource. Due to the nature of streaming processing, the memory will be sooner or later consumed out. Therefore, we need to limit the memory used by each component to prevent OOM.

![](images/overall-memory-allocation.drawio.svg)

**This RFC will focus on the memory controlling on both streaming and batch executors**, assuming the storage, network buffer, system metadata, etc. always take a fixed size of memory.

Our approach uses a background coroutine to continuously run the loop below:

1. **Measure**: Get the current memory usage
2. **Policy**: Decide whether and which component needs to release memory
3. **Action**: Take actions to release memory

### Measure: How to measure memory usage?

Here we propose to use the simple and stupid way to calculate the memory usage - **estimating** the memory consumed by all the in-memory data structures and aggregating in levels. This is opposite to the solutions that rely on memory allocators to get an exact memory usage.

How to estimate memory usage depends on the data structures:

- For indexing structures like `HashMap`, `BTreeMap`, etc., we could estimate the memory size according to the `capacity()` or something else. 
   - @liurenjie1024: It also seems possible to inject an `Allocator` to count the exact allocated space, as long as it exposed such generic type parameter.
- For pure data structures like `DataChunk`, `Vec<Datum>`, etc., we could compute the exact size it used by summing up the sizes of internal arrays & variables as well as padding and other overhead.

A CN-level memory manager will collect the memory usage in a fixed interval like 100ms. It collects the memory recursively regarding the hierarchy: stream/batch -> fragments -> actors -> executors. To simplify the ownership, executors' memory usage can be stored in `AtomicUsize` and be updated by executors on every `next()`.

### Action: How to release the memory?

- **Streaming**: We have implemented a CN-level LRU Manager to control the watermark epoch of all stateful operators. Once memory usage reaches the limit, the watermark epoch will be increased to evict some least-recent entries. See [RFC: Yet another simple idea for memory management](https://singularity-data.quip.com/CldAAcFmzZSO/Yet-another-simple-idea-for-memory-management) for details.
- **Batch**: We release the memory from batch queries by simply killing the queries with the largest memory consumption.

Notice that the background coroutine works in an async style, that is, there is no strict guarantee that a task must be killed **immediately** once the memory usage exceeds the limit. 

### Policy: Which component should give out memory?

#### Without Overselling

2 straight-forward options are exposed to users:

- `batch_memory_limit_mb`: The **upper bound** of memory consumed by batch executors
- `streaming_memory_limit_mb`: The **upper bound** of memory consumed by streaming executors

![](images/policy-without-overselling.drawio.svg)

Once the memory usage exceeds the limit, we run the actions to release some memory from streaming or batch.

#### With Overselling (Optional)

In total, there are `batch_memory_limit_mb + streaming_memory_limit_mb` for both streaming and batch executors. @liurenjie1024 proposed to utilize the memory space by allowing streaming and batch executors to use memory from each other, but must return it back immediately once the "owner" requires.

2 additional parameters need to be introduced:

- `batch_memory_reserved_mb`: The memory that must be reserved for batch no matter whether it's free. Must be `<= batch_memory_limit_mb`
- `streaming_memory_reserved_mb`: The memory that must be reserved for streaming no matter whether it's free. Must be `<= streaming_memory_limit_mb`

![](images/policy-with-overselling.drawio.svg)

Then `free - reserved` is the memory size that could be borrowed by each other, denoted as `overflow`. For example, assuming `limit` of streaming is 500MB and `reserved` is 100MB:

- If `allocated` = 0MB and `free` = 500MB, then `overflow` = 500MB - 100MB = 400MB
- If `allocated` = 300MB and `free`= 200MB, then `overflow` = 200MB - 100MB = 100MB
- If `allocated` = 450MB and `free`= 50MB, then `overflow` = 50MB - 100M = -50MB, so overflow is prohibited now

Why `reserved` is necessary? This is because we are using an async way to control memory i.e. with a background coroutine. As a result, **any actions will be slightly later than the actual exhaustion of memory**. The `reserved` memory is designed to mitigate the problem by reserving some space in case that memory cannot be reclaimed immediately.

@fuyufjh prefer the solution without overselling to keep it simple and stupid.

## Implementation

The "measure" part takes the most effort to implement. However, albeit not perfectly, we can do this in 2 stages:

1. First, we implement the memory estimation for the batch engine, and the usage of the streaming engine can be estimated by `total - batch`, where the `total` is the process-level usage from `jemalloc`.
2. Then, we implement the memory estimation for the streaming engine. This will make the numbers more accurate and also allows us to inspect the memory taken by each fragment/actor/executor. 


## Unresolved questions

None


## Alternatives

None

## Future possibilities

Some potential extensions or optimizations can be done in the future based on the RFC.
