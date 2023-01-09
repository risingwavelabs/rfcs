---
feature: use_backfill_to_let_mv_on_mv_stream_again
authors:
  - "DylanChen"
start_date: "2022/11/04"
---

# Use Backfill To Let Mv On Mv Stream Again

## Summary

Currently, if we want to create a mv A on another mv B, we need to first scan mv B with a snapshot which will cause us to buffer all stream update messages from mv B during the snapshot reading. This unbounded buffer will consume unbounded memory and also make the streaming processing stuck in some way. This RFC tries to use backfill to save mv-on-mv from buffering all update steam messages.

## Motivation

There is a paper *Online, Asynchronous Schema Change in F1* introduced by google used to solve the online schema change problem of distributed systems. They use online schema change to solve the creating index problem which somehow is similar to our creating index case or even our creating materialized view case. Online schema change uses a backfill operation to backfill all missing key-value pairs corresponding to the new element and this backfill operation can run concurrently with any upstream update. This idea can be used in our system. In this way, we can eliminate the unbounded buffer of upstream updates.

## Design

<img width="970" alt="image" src="https://user-images.githubusercontent.com/9352536/200274858-7880302a-952e-4f39-9c85-0f59fe9d4233.png">

We use creating index as an example to illustrate the idea, however, we can apply it to create mv too. Suppose that we have a table T which contains 10,000 rows. We want to create an index on it. This index is actually also a materialized view. The backfill operations work as follows.

### Backfill

Create a backfill operator which is similar to the Chain operator.

- Use a variable `current_pos` to keep track of the position (pk) we have backfilled which is initialized as 0.
- We select rows with pk between (`current_pos`, `current_pos` + `batch_size`) from table T into buffer. e.g. `batch_size` can be 1000.
- Handle all the streaming messages from upstream until the barrier comes. (Details are in the next Section)
- Flush buffer to the index when barrier comes.
- Set `current_pos = current_pos + batch_size` and repeatedly backfill batch data into the index.
- When `current_pos` becomes the end of Table T, the backfill finishes. we can forward upstream messages directly to the downstream and make the index visible to users.

### Deal With Streaming Message

Each iteration of backfill needs to use the latest commited epoch to scan the table T so that it can contain all the stream messages of the table in the previous epoch.
During the current epoch, we need to:
- Buffer all the batch data (`current_pos`, `current_pos` + `batch_size`) selected from table T. Apply any stream messages from table T if their pks are between (`current_pos`, `current_pos` + `batch_size`) to the buffer data.
- Forward stream message if its pk is smaller than `current_pos` to the downstream.
- Ignore stream message if its pk is bigger than `current_pos` + `batch_size`.
- When the barrier comes, flush all the buffered data to the downstream and increase the epoch and set `current_pos = current_pos + batch_size`.


### Failover & Recovery

We can persist `current_pos` to the state store and use it to help us recover after failure. It is useful if we need to create a mv on an existing mv with lots of data. It may take a long time to create and if any error happens during creating, we can recover from the last position we record to save expensive work that has been done before.

### How To Determine BatchSize

- The simplest way is to make `batch_size` constant or configurable. It could never catch up upstream if it is configured too small. Or it could increase the barrier latency if it is configured too large.
- Another way is that we don't predefine `batch_size` and stop the snapshot scan until the upstream barrier comes.
- The final way is measuring the actual barrier interval and comparing it with the system barrier interval. If the actual barrier interval is bigger than the system barrier interval, reduce `batch_size` otherwise increase it.

### Backfill Without Buffer

Actually the backfill operator doesn't need to buffer any data itself, it can select rows with pk between (`current_pos`, `current_pos` + `batch_size`) from upstream and insert them to the downstream directly. After sending the snapshot data, backfill operator consumes the upstream messages as follows:

- Forward stream message if its pk is smaller than `current_pos` + `batch_size` to the downstream.
- Ignore stream message if its pk is bigger than `current_pos` + `batch_size`.
- When the barrier comes, increase the epoch and set `current_pos = current_pos + batch_size`.

It is almost identical to the previous chain operator except for reading a small range rather than the whole table. It is also a perfect way to handle append only upstream.


## Future possibilities

Each iteration of backfill needs to immediately select next `epoch` data from the upstream, however, we need to wait epoch until the barrier goes through the whole DAG now. Optimization can be used to speed up the wait epoch.

