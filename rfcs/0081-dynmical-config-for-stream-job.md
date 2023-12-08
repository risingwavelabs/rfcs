---
feature: Dynamic Config For Stream Job
authors:
  - "st1page"
start_date: "2023/12/07"
---

# Dynamic Config For Stream Job

Thanks all the people took part in the discussion of https://github.com/risingwavelabs/risingwave/issues/11929 and implantation of `stream_rate_limit`. The RFC is hight inspired by them.

## Background
Currently, we have lots of config, switch and parameter for the streaming execution. They often used for performance optimization for different workload. But they distributed different places currently.
- hardcode as constant value, can not alter, such as 
  - `STATE_CLEANING_PERIOD_EPOCH` https://github.com/risingwavelabs/risingwave/blob/a8aa905ef26ef8c00f55a7c5b8ce76e6a0f7b72d/src/stream/src/common/table/state_table.rs#L65
  - `TOPN_CACHE_HIGH_CAPACITY_FACTOR` https://github.com/risingwavelabs/risingwave/blob/a8aa905ef26ef8c00f55a7c5b8ce76e6a0f7b72d/src/stream/src/executor/top_n/top_n_cache.rs#L30
- in specified stream plan node's proto, such as 
  - `OverWindowCachePolicy` https://github.com/risingwavelabs/risingwave/blob/a8aa905ef26ef8c00f55a7c5b8ce76e6a0f7b72d/proto/stream_plan.proto#L676-L690, 
  - `rate_limit` https://github.com/risingwavelabs/risingwave/blob/a8aa905ef26ef8c00f55a7c5b8ce76e6a0f7b72d/proto/stream_plan.proto#L164
    - And it can be altered with a special barrier command https://github.com/risingwavelabs/risingwave/blob/a8aa905ef26ef8c00f55a7c5b8ce76e6a0f7b72d/proto/stream_plan.proto#L85
- stored in the `StreamGraph` as a `StreamEnvironment`, such as 
  - `timezone`. https://github.com/risingwavelabs/risingwave/blob/a8aa905ef26ef8c00f55a7c5b8ce76e6a0f7b72d/proto/stream_plan.proto#L829-L832, https://github.com/risingwavelabs/risingwave/blob/a8aa905ef26ef8c00f55a7c5b8ce76e6a0f7b72d/proto/meta.proto#L100
    - It is expected not to alter, which will be explained below
- stores as the global config such as 
  - `max_dirty_groups_heap_size` https://github.com/risingwavelabs/risingwave/blob/a8aa905ef26ef8c00f55a7c5b8ce76e6a0f7b72d/src/common/src/config.rs#L801

And in future we might introduce more parameters for streaming, such as `log_store_capacity`.

In other database, those parameter is declared by session variable or session config. It is because the compute task is bounded in those batch system and it is not necessary to alter the config for a running job. But for a long-running streaming job, alter the config for a created and processing job is needed.

This RFC will introduce a dynamic config framework to make user
- can declare those configuration when create streaming job
- can alter those configuration for a long-running streaming job. 
### Non-Goals

The RFC will not concern those config can change the streaming graph and its topology, such as
- The configs that can change the plan such as `RW_TWO_PHASE_AGG`
- The configs that can change the actor's number such as `STREAMING_PARALLELISM` (maybe we can use the same syntax to alter it later, but it will be only a semantic sugar of scale)

Also, the RFC do not associate with those node-level config such as `exchange_initial_permits` 

These are just some example configs that **must not** including in the framework, there still are some vague config need to be discussed case by case, which mainly depends on if it is needed to be different between different streaming jobs, such as `chunk_size` and `connector_message_buffer_size`.

## Unified streaming job configs

We will introduce "dynamic stream config" concept to the user. The config entries in that can be set in the with clause when creating and we have a lots of discussion in the issue https://github.com/risingwavelabs/risingwave/issues/11929.
```SQL
create materialized view mv with ( streaming_rate_limit = 200 );
```

detailed implementation, a `DynamicConfig` struct will be stored in the `TableFragments` just like current `StreamEnvironment`. And when creating the actors and executors inside, the executor will get their needed parameter for the config.
Btw, I‘d like to still maintain the `StreamEnvironment` as the immutable parameters.

### relation with session config

Please notice that the config is independent with session config. In Conceptual space, we can define a dynamic stream config without its corresponding session variable.
The session config of the corresponding dynamic stream config just define its default value, in other words, when the config entry is not defined in the with clause, what's the value should be.
Furthermore, do we need add a copy of a session configs in the system parameters as the default value of the session config, that is another topic and will not talked in this RFC.

## Alter dynamic configs

First to clarify, there is some parameter can not be altered such as `timezone`. Because it can change the result of the streaming query in SQL semantic. It is strange to alter such a config for a "running query", it is a little like alter mv which change the SQL statement.

User will be able to alter the dynamic stream config for specified MV or SINK with the grammar.
```SQL
ALTER [MATERIALIZED VIEW | SINK] SET parameter TO [value | 'value'];
```
The statement will send a RPC to meta. After reducing the config in the common place, it is easier to alter the config of some relations. And we need some implementation complexity trade-off here. There are three implementation sort from easy to hard.

1. alter with recovery: change the configs in the meta store and trigger a recovery. Then the newly created actor will adopt the new configs.
2. alter by the local stream manager: the meta send a barrier with a alter config command. When the barrier receiver receive a barrier with alter config command want to change its own actor, it will stop to poll the input channel and attach the input channel on the barrier. then pass the barrier. When local stream manager collect this barrier, it can build the new actor with the channel rx on the barrier. And because the barrier has been committed in the local state store and no data after the barrier input, the correctness can be ensured. The disadvantages is that all cache are lost with the old actors and the performance is influenced.
3. alter in each executor: every stream executor must can receive the alter config barrier and adopt the new config.

Maybe the third is easier than the second one. We can discuss it later.