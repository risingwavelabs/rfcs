---
feature: Reusable Source Executor
authors:
  - "Eric Fu"
start_date: "2023/08/17"
---

# Reusable Source Executor


## Motivation

Conceptionally, a source is far different from a table or sink in terms of whether it is **instantiated**. A source defined by a `create source` statement is not an instance but only a set of metadata stored in the catalog, until it's been referenced by a meterialized view, upon which time a `SourceExecutor` is instantiated.

As a result, when creating multiple materialized views on top of one source, the `SourceExecutor` is not reused between them. It leads to multiple times of resource ultilization and multiple consuming offsets, metrics, etc. Furthermore, it will become a bug if the source definistion uses some exclusive property, such as the consumer group of Kafka source.

## Design

Recall that most source has more or less histrical data. For message brokers including Kafka, Pulsar, Kinesis, etc, this can be configured with option `scan.startup.mode`. When `scan.startup.mode` is not `latest`, the consumer must read all the historical data before consuming any new events. This process is implicitly done by the client library, and RisingWave is not aware of it.

Let's assume that the Kafka client library is stupid and RisingWave needs to read the historical data by itself. Then, it turns out this procedure is quite similar to the backfilling procedure of tables. Recall that when a new materialized view is created, we introduce a `BackfillExecutor` to read the historical data. Here we can solve the problem with exactly the same idea by dividing it into 2 phases: 1) backfill and 2) consume new events.

![](images/0061-reusable-source/reusable-source.png)

Here,

- `KafkaSourceExecutor` is created on executing `CREATE SOURCE` statement, although a trivial optimization is doing nothing when there is no downstream operator attached.
- `KafkaBackfillExecutor` is created along with the `CREATE MATERIALIZED VIEW` statement. Each MV has its own `KafkaBackfillExecutor` respectively.

Obviously, for a source without histrical data or message broker source `scan.startup.mode = latest`, the `BackfillExecutor` can be omitted.

### Implementation for Message Brokers

Events from Kafka should be processed according to their sequence in partition, no matter for historical events or new incomming events.

So how to combine the backfill data with incoming data without duplicated or missing events?

Since everything happen inside the `BackfillExecutor`, the implementation is not difficult. Here is the pesedo-code.

```rust
let source_stream = the stream from upstreaming SourceExecutor
let backfill_stream = read_from_mq(scan_startup_mode)

// In fact, these should be per kafka partition
let backfill_offset = 0;
let backfill_completed = false;

next_event = select! {
    biased;
    e = source_stream.next() => {
        if backfill_completed {
          yield e;
        } else if e.offset > backfill_offset {
          // do nothing. ignore e
        } else {
          // the progress catched up. backfilling is completed
          backfill_stream.close();
          backfill_completed = true;
          yield e;
        }
    }
    e = backfill_stream.next() => {
      assert!(!backfill_completed);
      backfill_offset = e.offset;
      yield e;
    }
}
```

### Implementation for File Source

Naturally, the file source doesn't have any guarentee on ordering. This actually makes it more difficult to do backfilling because it become more challenging to record which part of data has been consumed.

A intuitive approach is to take a snapshot of current progress of the `SourceExecutor`, which is a list of `file_name:offset`. Then, the `BackfillExecutor` scans files according to this list. Meanwhile, all of the new incoming events will be streamed, because the event order doesn't matter.

Note that the snapshot i.e. the list of `file_name:offset` for backfilling is large it might need to be persisted as a state in order to achieve fault-tolerance. This is why I say it is more difficult.

### Batch Scan (Kafka-only)

Batch scan is exactly the same as the backfilling process. Hopefully we should reuse the implementation as much as possible. 
