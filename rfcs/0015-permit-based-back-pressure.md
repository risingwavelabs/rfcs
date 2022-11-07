---
feature: permit-based-back-pressure
authors:
  - "Bugen Zhao"
start_date: "2022/10/24"
---

# Permit-based Back-pressure with Bidirectional gRPC Streaming

## Background

The back-pressure is necessary in a streaming system to ensure the resources are not exhausted. Luckily, it’s natural and easy to implement back-pressure in RisingWave, as what we’re doing now:

- **Local Exchange**
We use a channel bounded with the chunk number to exchange. When the downstream actor processing slowly, the channel will be filled gradually, and the upstream will be blocked.
- **Remote Exchange**
We rely on the HTTP/2 flow control under gRPC. When the downstream actor processing slowly, the window of the HTTP/2 stream will be filled gradually, and the upstream will be blocked.

## Problems

According to the recent benchmarks, we find a serious problem: since we bound the local exchange with the **chunk number** while the remote exchange with the **window size (bytes)**, which has different effects under different workloads, there can be significant imbalance between local and remote exchange.

> [#6099 streaming: imbalance between local and remote inputs](https://github.com/risingwavelabs/risingwave/issues/6099)

Take TPCH Q12 as an example:

```jsx
StreamMaterialize { columns: [l_shipmode, high_line_count, low_line_count], pk_columns: [l_shipmode] }
└─StreamProject { exprs: [lineitem.l_shipmode, sum(Case(((orders.o_orderpriority = '1-URGENT':Varchar) OR (orders.o_orderpriority = '2-HIGH':Varchar)), 1:Int32, 0:Int32)), sum(Case(((orders.o_orderpriority <> '1-URGENT':Varchar) AND (orders.o_orderpriority <> '2-HIGH':Varchar)), 1:Int32, 0:Int32))] }
  └─StreamHashAgg { group_key: [lineitem.l_shipmode], aggs: [count, sum(Case(((orders.o_orderpriority = '1-URGENT':Varchar) OR (orders.o_orderpriority = '2-HIGH':Varchar)), 1:Int32, 0:Int32)), sum(Case(((orders.o_orderpriority <> '1-URGENT':Varchar) AND (orders.o_orderpriority <> '2-HIGH':Varchar)), 1:Int32, 0:Int32))] }
    └─StreamExchange { dist: HashShard(lineitem.l_shipmode) }
      └─StreamProject { exprs: [lineitem.l_shipmode, Case(((orders.o_orderpriority = '1-URGENT':Varchar) OR (orders.o_orderpriority = '2-HIGH':Varchar)), 1:Int32, 0:Int32), Case(((orders.o_orderpriority <> '1-URGENT':Varchar) AND (orders.o_orderpriority <> '2-HIGH':Varchar)), 1:Int32, 0:Int32), orders.o_orderkey, lineitem.l_orderkey, lineitem.l_linenumber] }
        └─StreamHashJoin { type: Inner, predicate: orders.o_orderkey = lineitem.l_orderkey, output: [orders.o_orderpriority, lineitem.l_shipmode, orders.o_orderkey, lineitem.l_orderkey, lineitem.l_linenumber] }
          ├─StreamExchange { dist: HashShard(orders.o_orderkey) }
          | └─StreamTableScan { table: orders, columns: [orders.o_orderkey, orders.o_orderpriority], pk: [orders.o_orderkey], dist: UpstreamHashShard(orders.o_orderkey) }
          └─StreamExchange { dist: HashShard(lineitem.l_orderkey) }
            └─**StreamProject { exprs: [lineitem.l_orderkey, lineitem.l_shipmode, lineitem.l_linenumber] }
              └─StreamFilter { predicate: In(lineitem.l_shipmode, 'FOB':Varchar, 'SHIP':Varchar) AND (lineitem.l_commitdate < lineitem.l_receiptdate) AND (lineitem.l_shipdate < lineitem.l_commitdate) AND (lineitem.l_receiptdate >= '1994-01-01':Varchar::Date) AND (lineitem.l_receiptdate < ('1994-01-01':Varchar::Date + '1 year 00:00:00':Interval)) }
                └─StreamTableScan { table: lineitem, columns: [lineitem.l_orderkey, lineitem.l_shipmode, lineitem.l_linenumber, lineitem.l_shipdate, lineitem.l_commitdate, lineitem.l_receiptdate], pk: [lineitem.l_orderkey, lineitem.l_linenumber], dist: UpstreamHashShard(lineitem.l_orderkey, lineitem.l_linenumber) }**
```

As one side of the join, the `lineitem` is firstly filtered with a strict predicate, then projected to a few columns. Note that the Filter operator only manipulate the visibility of the chunk and don’t buffer the records or reconstruct the chunks, the chunks sent to the Exchange may **only contain few visible rows**.

Based on the current configuration, we have…

- For local exchange, we buffer 16 chunks. Assuming the selectivity is 5% and the chunks from sources have 1024 rows, we buffer ~819 rows.
- For remote exchange, we buffer 16 chunks in the upstream *(should actually be shrink)* and additional 65536 bytes in HTTP/2. Assuming the memory size of one row is 20 bytes, we buffer ~4096 rows.

Different size of buffer will lead to imbalance back-pressure. To verify this, we scale the Hash Join to a single parallel unit, then it’s shown that the throughput of the local upstream (in the same compute node, `:1222`) will be significantly lower, and the back-pressure rate increases from 5.0% to 96.3%.

![Throughput 20,000 vs 100,000](Permit-based%20Back-pressure%20with%20Bidirectional%20gRPC%20a11d74903c9b4784b14010a2886f5093/Untitled.png)

Throughput 20,000 vs 100,000

![Back-pressure rate 96.3% vs 5.0%](Permit-based%20Back-pressure%20with%20Bidirectional%20gRPC%20a11d74903c9b4784b14010a2886f5093/Untitled%201.png)

Back-pressure rate 96.3% vs 5.0%

Can we resolve this by increasing the capacity of the local channel? Partially. Consider a counterexample that the chunk is really wide and includes some large blobs, for example, average 500 bytes per row. Then we can only buffer ~131 rows in the network but 16384 rows in the channel. 😄 Increasing the capacity of the local channel only makes it worse.

The main reason is that, “chunk number” is not comparable with the “network buffer size”, considering the visibility, the cardinality, and the schema of the chunks. On the contrary, if the network bandwidth is sufficient, the “width” of a chunk does not count much: most columns will not be involved to the computing and won’t affect the performance. Therefore, the **row number** (record count) should be the most reasonable bounding unit.

## Possible Solutions

I’d like to introduce a `permit(credit, token)`-based back-pressure strategy for both local and remote exchanges, where one permit indicates that one row is allowed to send to the downstream.

Initially, the upstream and the downstream has a consensus of `EXCHANGE_ROW_COUNT = 32768` permits. Every time the upstream want to send a chunk, it should first acquire permits of the **cardinality** of the chunk. The permits are only added back when the downstream reports that has processed the chunk. If the chunk is too large (which rarely happens), we treat its cardinality as the max value of `EXCHANGE_ROW_COUNT`. Barrier and watermark messages are not counted.

For detailed implementations…

### Local Exchange

The idea is the same as semaphore and is trivial to implement locally.

- Initialize an unbounded channel, and a semaphore of `EXCHANGE_ROW_COUNT` permits.
- Upstream: acquire the semaphore for “many” permits before sending the chunk by `await`.
- Downstream: receive the chunk passively in the actor thread and add permits back.

### Remote Exchange

To avoid hacking the network stack and make everything simple, we can adopt the gRPC bidirectional streaming and implement it on the application layer. Note that every flow control algorithm needs a whole RTT to receive a “window update” message, it doesn't matter much which level we implement it on, considering this little overhead compared to the network latency.

We make the exchange service a bidirectional stream. Since we’re handling the flow control by ourselves, we can set the HTTP/2 window size to a relaxed value in MB level.

```protobuf
message GetStreamRequest {
  message Get {
    uint32 up_actor_id = 1;
    uint32 down_actor_id = 2;
    uint32 up_fragment_id = 3;
    uint32 down_fragment_id = 4;
  }
  message AddPermits {
    uint32 permits = 1;
  }
  oneof value {
    Get get = 1;
    AddPermits add_permits = 2;
  }
}
message GetStreamResponse {
  stream_plan.StreamMessage message = 1;
}

service ExchangeService {
  rpc GetStream(**stream GetStreamRequest**) returns (stream GetStreamResponse);
}
```

- The downstream will first send a `Get` message to establish the channel. The upstream then initialize a semaphore of `EXCHANGE_ROW_COUNT` permits, and spawn a background task to serve the future `AddPermits` messages.
- Upstream: acquire the semaphore for “many” permits before sending the chunk by `await`.
- Upstream gRPC server: receive the chunk and forward it to the downstream **without** adding the permits back.
- Downstream: receive the chunk passively in the actor thread, and send a `AddPermits` message according to the cardinality.
- Upstream background: receive `AddPermits` messages and add the permits to the semaphore.

It’s possible that there’re a lot of small chunks, so we can batch the permits at the downstream to reduce the backward messages as an optimization, which is exactly what HTTP/2 does. In this case, we should set the maximum permits of a chunk to `EXCHANGE_ROW_COUNT - PERMIT_BATCH` to avoid deadlocks.
