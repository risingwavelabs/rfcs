---
feature: use_backfill_to_let_mv_on_mv_stream_again
authors:
  - "DylanChen"
start_date: "2022/11/04"
---

# Use Backfill To Let Mv On Mv stream again

## Summary

Currently, if we want to create a mv A on another mv B, we need to scan mv B with a snapshot at first which will cause we need to buffer all stream upadte message from mv B during the snapshot reading. This unbounded buffer will consume unbounded memory and also make the streaming processing stuck in some way. This RFC try to use backfill to save mv-on-mv from buffering all update steam message.

## Motivation

There is a paper *Online, Asynchronous Schema Change in F1* introduced by google used to solve the online schema change problem of distributed system. They use online schema change to solve the creating index problem which somehow is similar to our creating index case or even our creating materialized view case. Online schema change use a backfill operation to backfill all missing key-value pairs corresponding to the new element and this backfill operation can run concurrently with any upstream update. This idea can be used to our system. In this way, we can eliminate the unbounded buffer of upstream update.

## Design

We use creating index as a example to illustrate the idea, however, we can apply it to create mv too.Suppose that we have a table T which contains 10,000 rows. We want to create an index on it. This index is actually also a materialized view. The backfill operations work like as following.

### Backfill

- Use a variable `current_pos` to keep track the position (pk) we have backfilled which initialized as 0.
- We select (`current_pos`, `current_pos` + `batch_size`) from table T and insert them to the index. e.g. `batch_size` can be 1000.
- Repeatedly backfill next batch after current batch to the index.
- We can forward any update stream from table T to the index if its pk is lower than the `current_pos` concurrently, otherwise, igonre the update.

### More Detail

Each step of backfill needs to use the latest commited epoch to select the table T so that it can contain all the update of the table in previous epoch.
During current epoch, we need to:
- buffer all the batch data (`current_pos`, `current_pos` + `batch_size`) selected from table T.Apply any updates from table T if their pks are within (`current_pos`, `current_pos` + `batch_size`) to the buffer data.
- Forward update if its pk is less than `current_pos` to the index.
- Ignore update if its pk is bigger than `current_pos` + `batch_size`.
- When the barrier comes, flush all the buffered data to downstream and increase the epoch and `current_pos = current_pos + batch_size`.
- Once we finish the backfill, we can keep forward update from upstream to the downstream.


### Failover & Recovery

We can save `current_pos` to the state and use it to help us recover after failover. It is useful if we need to create a mv on a large existing mv. It may takes a long time to create and if any error happen during creating, we can recover from the last position we record to save expensive work have been done before.


## Unresolved questions


## Future possibilities

Each iteration of backfill need to immediately select next `epoch` data from the upstream, however, we need to wait epoch until the barrier go through the whole DAG now. Optimization can be used to speed up the wait epoch.

