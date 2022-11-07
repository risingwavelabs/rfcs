---
feature: use_backfill_to_let_mv_on_mv_stream_again
authors:
  - "DylanChen"
start_date: "2022/11/04"
---

# Use Backfill To Let Mv On Mv Stream Again

## Summary

Currently, if we want to create a mv A on another mv B, we need to scan mv B with a snapshot at first which will cause we need to buffer all stream upadte message from mv B during the snapshot reading. This unbounded buffer will consume unbounded memory and also make the streaming processing stuck in some way. This RFC try to use backfill to save mv-on-mv from buffering all update steam message.

## Motivation

There is a paper *Online, Asynchronous Schema Change in F1* introduced by google used to solve the online schema change problem of distributed system. They use online schema change to solve the creating index problem which somehow is similar to our creating index case or even our creating materialized view case. Online schema change use a backfill operation to backfill all missing key-value pairs corresponding to the new element and this backfill operation can run concurrently with any upstream update. This idea can be used to our system. In this way, we can eliminate the unbounded buffer of upstream update.

## Design

We use creating index as a example to illustrate the idea, however, we can apply it to create mv too. Suppose that we have a table T which contains 10,000 rows. We want to create an index on it. This index is actually also a materialized view. The backfill operations work like as following.

<img width="970" alt="image" src="https://user-images.githubusercontent.com/9352536/200274858-7880302a-952e-4f39-9c85-0f59fe9d4233.png">


### Backfill

Create a backfill operator which is similar to the Chain operator.

- Use a variable `current_pos` to keep track of the position (pk) we have backfilled which is initialized as 0.
- We select rows pk within (`current_pos`, `current_pos` + `batch_size`) from table T into buffer and flush buffer to the index after barrier. e.g. `batch_size` can be 1000.
- Repeatedly backfill batch data into the index and set `current_pos = current_pos + batch_size`.
- When current_pos become the end of Table T, backfill finishes. we can forward upstream message directly to the downstream and make the index visible to users.

### More Detail

Each step of backfill needs to use the latest commited epoch to scan the table T so that it can contain all the stream messages of the table in previous epoch.
During current epoch, we need to:
- Buffer all the batch data (`current_pos`, `current_pos` + `batch_size`) selected from table T. Apply any stream message from table T if their pks are within (`current_pos`, `current_pos` + `batch_size`) to the buffer data.
- Forward stream message if its pk is smaller than `current_pos` to the down stream.
- Ignore stream message if its pk is bigger than `current_pos` + `batch_size`.
- When the barrier comes, flush all the buffered data to downstream and increase the epoch and `current_pos = current_pos + batch_size`.


### Failover & Recovery

We can persist `current_pos` to the state store and use it to help us recover after failure. It is useful if we need to create a mv on a existing mv with lots of data. It may takes a long time to create and if any error happen during creating, we can recover from the last position we record to save expensive work have been done before.

### How to determine BatchSize

- The simplest way is make `batch_size` constant or configurable. It could never catch up upstream if it is configured too small. Or it could increase the barrier latency if it is configured too large.
- Another way is that we don't predefine `batch_size` and stop the snapshot scan util upstream barrier come.

## Future possibilities

Each iteration of backfill need to immediately select next `epoch` data from the upstream, however, we need to wait epoch until the barrier go through the whole DAG now. Optimization can be used to speed up the wait epoch.

