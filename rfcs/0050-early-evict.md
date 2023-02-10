---
feature: early_evict
authors:
  - "soundOfDestiny"
start_date: "2023/02/10"
---

# Early Evict in Block Cache

## Summary

Evict blocks that will never be involved in reads in block cache.

## Motivation

To reduce cache miss.

## Design

Create a mpsc: `HummockStorage` is the consumer, which will receive messages on a block having entered/exited LRU list. Therefore `HummockStorage` knows whether each block is currently in LRU list or not.

`HummockStorage` can then collect blocks which are both in a SST not belonging to latest `HummockVersion` and in LRU list. `HummockStorage` can then tell block cache that these blocks can be evicted immediately.

## Future possibilities

More accurate arrangement in block cache by context info.
