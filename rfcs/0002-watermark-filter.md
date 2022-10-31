---
feature: watermark_filter
authors:
  - "TennyZhuang"
  - "st1page"
  - "BugenZhao"
start_date: "2022/10/20"
---

# The WatermarkFilter and StreamSort operator

## Summary

We will introduce the watermark strategy in the doc. The main changes are two new oprators, `WatermarkFilter` and `StreamSort`.

## Motivation

We have the following purposes:

* Support some operators which can only accept ordered input.
  * SessionWindow
  * OverAgg
  * MatchRecognize
* Support sink a complete result set to some append-only warehouse.
* State cleaning

## Design

### The watermark message

```rust
pub enum Message {
    Chunk(StreamChunk),
    Barrier(Barrier),
    Watermark(col_idx, Timestamp),
}
```

**Term 1.1: Every record has multiple watermark columns with the timestamp type.**

**Term 1.2: If the order of a data record *d* in the stream is after another watermark record *w*, then *d[watermark_col] > w.val*.**

Based on Term 1.2, we can get a noteworthy property:

**Prop 1.1: We can postpone or remove any watermark record in a valid stream.**

The next question: Where does the watermark message come from?

### The WatermarkFilter operator

For simplicity, we chose to support only the append-only stream as the input, we can discuss the retractable stream later.

We will introduce a new operator, `WatermarkFilter`, which will filter the outdated records and output some `Watermark` messages **over one column**.

The design is highly different from [[Deprecated] RFC: Watermark in RisingWave (WaterMark part I)](https://www.notion.so/Deprecated-RFC-Watermark-in-RisingWave-WaterMark-part-I-b97ed46b310549ae921abe61e1f30226), we will not buffer the out-of-order records, and no orders are guaranteed on the output stream.

```python
watermark = recovered.or(0)
watermark_col: int = user_defined()
def process(msg):
  match msg:
    # We use a row-level representation instead of the Chunk in the pseudo-code.
    case Record(rec):
      rec_event_time = rec[watermark_col]
      if rec_event_time < watermark:
        # Filter the outdated records.
        return;
      if rec_event_time - timeout > watermark:
        watermark = rec_event_time - timeout
        # Send a watermark message, optionally
        emit(Watermark(watermark))
      emit(Chunk(chunk))
    case Barrier(barrier):
      if barrier.checkpoint:
        # Watermark should be checkpointed.
        checkpoint(watermark)
      emit(Barrier(barrier))
    case Watermark(upper_col_idx, w):
      # Watermark from the upstream
      emit(Watermark(upper_col_idx, w))
```

The WatermarkFilter mainly does the following two things:

1. Insert some watermarks to a message stream, which satisfies Term 1.2.
2. Filtered out some outdated records.

The following graph shows a simple example; 3 and 7 are filtered due to outdated.

And multiple watermarks are produced in the output stream (Note, we can omit some of them for performance).

![https://viewer.diagrams.net/?border=0&tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1&title=watermark-1.drawio&lightbox=1&chrome=0#R5VdNc5swEP01nkkP6fBhsHN0bCc9tDOd8XTSnjoKyKBGICJEwP31XSHJIAP5bi%2FNRehptSvt27dyZv46a645KtIvLMZ05jlxM%2FM3M89z514Ig0QOClnM5wpIOIm1UQfsyG%2BsQUejFYlxaRkKxqgghQ1GLM9xJCwMcc5q22zPqB21QAkeALsI0SF6Q2KRKnTpLTr8EyZJaiK74YVayZAx1jcpUxSzugf525m%2F5owJ9ZU1a0xl8kxe1L6ridXjwTjOxXM2OM0dC%2FNdk9xef7svdouidn%2BeB8rLA6KVvrA%2BrDiYDOAYEqKnjIuUJSxHdNuhl5xVeYxlGAdmnc1nxgoAXQB%2FYSEOml1UCQZQKjKqV%2FcsF2tGGW8j%2BldXDvwBPryivnXJKh7hR%2B6l7eTZext1Yq4xy7DgBzDgmCJBHmyyka6Z5Gh33PqVETgKxFQWQajJ1dU9dxzbhUA8wULv6siBj94xOqil7AX0uSP0hVTolFo8hvcVMwvnZcvECgzcoGjaTJt1%2BErkeIME5hnid9sGR8AYPxMkw6ySXluHjv%2FBBIOzq3hqq4FvuUFq40ztXAyLjFJQtCymOiUC7wrU0ltDU7FLZbIkHjAXuHmUbL3qLW3SfENi3QkcmFRY2hP30pmuD4vZl9LoPUOFebyS7QxmEUVlSSI7L7YGITn88L0%2F%2BSEnHwMz3TT9xc1Bzyazq%2Br4HQTXy3EwkmKDvVGXR%2FqmdKkayECXA0fzE0f%2Bxb8VuD%2BoDMgFuHI8NYRqOFF9dOylnar9%2Fb7tqgOh%2B5Mqlp7nagjUsFSDfJzfGnYxGfak8kHVwq51REmSSyFAlWIIeCm1T%2BDNXumFjMSxepkwdDp027qS9V1InlrmgstZsJG%2BoLWpbti6LgVnd9i8RTnLpZc9ofQEeoc25DonpbUMBm3IHWtD%2Ft9qQ%2BFTr8kowY4ThmME35w50w9EW0WvreTpiI88SZPF%2FLaIwRN3fIFSpoMspoP8J2I5%2FaE1JpbRB%2BUVYoFp96tcNfLufxt%2F%2Bwc%3D](https://user-images.githubusercontent.com/9161438/197964182-b9b78e8c-1940-43c4-b819-ff999bb16796.png)

### The Sort operator in Stream

We now have a bounded out-of-order stream, but some operators can’t handle an unordered stream so we may need the order property on stream:

* OverAgg (SQL Window Function)
* MatchRecognize
* SortAgg

To support these operators, we can introduce the Sort operator.

The Sort operator will buffer all records before the watermark that it received recently and output the ordered stream. **In a database context, the Sort operator is also similar to a Filter, which will filter some incomplete records out. (We can see the records in the future, but at this point, they are absent).**

The following graph shows a simple example of the Sort operator.

Since the output of the Sort operator always outputs an ordered stream, every record can pair with a watermark, but we may still omit some watermarks for performance.

![https://viewer.diagrams.net/?border=0&tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1&title=Untitled.drawio&lightbox=1&chrome=0#R7VZLj5swEP41SMlhK4KB0GMeu%2B1hV6oarWiPTnDAXYOpMXn013eMDcQh2Ye27akne76xx3jmm884aJEfPglcZg88Iczx3OTgoKXjeRPfC2FQyFEjU9%2FXQCpoYhb1wIr%2BIgZ0DVrThFTWQsk5k7S0wQ0vCrKRFoaF4Ht72ZYz%2B9QSp2QArDaYDdGYJjLTaORNe%2FwzoWnWnjwJP2pPjtvF5iZVhhO%2BP4HQrYMWgnOpZ%2FlhQZhKXpsXve%2Fuirf7MEEK%2BZoN60f09dtTIOOZeLjbPdbzHb2%2FMdXZYVabC5uPlcc2AySBhBiTC5nxlBeY3fboXPC6SIg6xgWrX3PPeQngBMAfRMqjqS6uJQcokzkz3i0vpHFOArD1N6iDr97VQBWvxYY8c0GvyzRQlPCcSHGEfYIwLOnOjo8NV9JuXbf1C6dwsucaXoeuKaphtd%2FabQiJRUqk2dUXBSYnn9FDTaneUDbvQtlCJk0qrfqFP2vFsLk1S834%2FBbluKmausxgwSQoDxcDraDkbTC4jo5nn7EWLbKut1sCltsc5TrBPHK8hQoP86We7rEkIsfiqVs2HQa6euI5fxkDsVA83WdUklWJG8LsQa9sFpqcEiHJ4XneDflkNqDQ5kVn73vt8FquZCe6EbnXKWiR561MQa9o8CKZKaUEa8NwVdGNnRe7vc97FTIjjt%2BMszG%2BK%2BND51seTp3Lo7Fe3%2BO6l97e4yc5Dy6kvMXeKQXIe0EKtEQNpGAQyPfPuBP8W03xB0yJR%2B7Y9KYePD2EeohHyLh9PQR6iFp3MO462yDT8YB90GzS5htmNC0UGYEIoBRorlqSwpM8M46cJol%2BeAioE143oRSnSpWbJlvBXIkJxIK3pjJsBbOSgj%2BRBWcc4i4LXqgoW8rYOXTxQXqXNJzRBEXBQBnQJWVAf0sZopfrHY8mY6vy8cizSx6P%2FLFV%2FL7qHUvC8f%2FyD%2F4YLtY%2F%2BjP1B7P%2FrdTy0P%2Bco9vf](https://user-images.githubusercontent.com/9161438/197971481-56e25377-914f-41fb-9c3f-7818a9eabec3.png)

### Watermark derivation

We need to derive the watermark during some operators, e.g., for the tumbling window:

For a Tumble(time_col) with a watermark on time_col, we will have the following derivation:

```plain!
W(time_col, t) → W(time_col, t), W(window_start, tumble_start(t)), W(window_end, tumble_end(t))
```

We can get three watermarks over three columns after the projection.

The following operators may have some special watermark derivation rules, which we will discuss in another doc:

* Tumble Window
* Hop Window
* Session Window
* Project with expressions that keep the order (e.g., EXTRACT)

A tumble window example:

Watermark(10:40, order_time) → Tumble(30 minutes)

1. Watermark(10:40, order_time)
2. Watermark(10:30, window_start)
3. Watermark(11:00, window_end)

We will also need to derive the watermark for some operators which have multiple inputs; the typical examples are **Join and Union**. The solution is very easy enough; **the output watermark should be the smaller value of watermarks from upstreams.**

### State cleaning

We can use the watermark to clean the states in many operators, which we will also discuss in another doc, but we can give a simple example here:

```sql
SELECT customer_id, window_end as order_hour, SUM(price) as sum_price
FROM TUMBLE(
  WATERMARK(orders, order_time, INTERVAL '1' MINUTE), 
  order_time, 
  INTERVAL '1' HOUR)
GROUP BY window_end, customer_id;
```

In the Aggregation operator, we will get two watermarks, `order_time` and `window_end`, and `window_end` matches the prefix of the grouping key, so when we receive `Watermark(window_end, w)`, we can confirm that we will not receive records whose `window_start` is smaller than `w`, then we can clean the states before that.

Note: we may need Range-Delete here.

### Exchange over watermarks

We need to handle the `Watermark` message correctly, and the strategy is very hard, and @Bugen Zhao will introduce that in another doc.

The core idea of Exchange is **Prop 1.1.** We need also to align the watermarks, but we can achieve that by delaying the watermarks.

## Unresolved questions

* Are there some questions that haven't been resolved in the RFC?
* Can they be resolved in some future RFCs?
* Move some meaningful comments to here.

## Syntax

We can discuss the syntax later, but now we can implement that as a table function and expose the watermarkFilter’s behavior directly.

```plain
WATERMARK(orders, time_column, watermark_strategy_expression)
```

And the concept `watermark_strategy_expression` refers [FlinkSQL](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/table/sql/create/#watermark). The expression is evaluated for every record and update the watermark if the result greater than the current watermark.
The new syntax is more friendly for flinkSQL user and offers user methods defining their strategy.

```sql
CREATE SOURCE `orders` (
  `id` BIGINT,
  `order_time` TIMESTAMP,
  `price` DECIMAL,
  `customer_id` BIGINT,
) WITH (
  'connector' = ...,
);

-- normal streaming plan, just for state cleaning
SELECT customer_id, window_end as order_hour, SUM(price) as sum_price
FROM TUMBLE(
  WATERMARK(orders, order_time, order_time - INTERVAL '1' MINUTE), 
  order_time, 
  INTERVAL '1' HOUR)
GROUP BY window_end, customer_id;

-- Using the option `EMIT ON WINDOW CLOSE` on agg to make sink append-only
CREAT SINK AS
SELECT customer_id, window_end as order_hour, SUM(price) as sum_price
FROM TUMBLE(
  WATERMARK(orders, order_time, order_time - INTERVAL '1' MINUTE), 
  order_time, 
  INTERVAL '1' HOUR)
GROUP BY window_end, customer_id EMIT ON WINDOW CLOSE;

-- session window
-- `EMIT ON WINDOW CLOSE` is necessary for `SESSION` due to implementation.
SELECT customer_id, window_end as order_hour, SUM(price) as sum_price
FROM SESSION(
  WATERMARK(orders, order_time, order_time - INTERVAL '1' MINUTE),
  order_time,
  customer_id,
  INTERVAL '1' HOUR)
GROUP BY window_end, customer_id EMIT ON WINDOW CLOSE;

-- SQL WINDOW FUNCTION, which only use the Watermark table function but not the time window.
-- We allow window functions to be defined with the time attribute column of a stream. 
-- In brief, the event time column with a watermark or the omitted processing time.
-- Fraud Detection, more details: 
--   https://singularity-data.quip.com/8BJoAulRYblq/Flink-Demo-Fraud-Detection-in-Online-Shopping-Assignee- 
create materialized view ALERTS as
SELECT
  customer_id,
  price, 
  order_time,
  lead(price, 1) over w as next_price,
  lead(order_time, 1) over w as next_order_time
FROM 
  WATERMARK(orders, order_time, order_time - INTERVAL '1' MINUTE), 
WHERE
  abs(next_price - price) > 10000
  AND next_order_time - order_time < INTERVAL '30' SECOND
WINDOW w AS (
  PARTITION BY customer_id 
  ORDER BY order_time
)
```

And `watermark_strategy_expression` offers potential to make user define their own strategy. Some following case is fancy and not very well-defined. You can treat them as a pseudocode which we might implement in furture

```sql
CREATE SOURCE `orders` (
  `id` BIGINT,
  `order_time` TIMESTAMP,
  `price` DECIMAL,
  `customer_id` BIGINT,
) WITH (
  'connector' = ...,
);

-- normal timeout watermark
with watermarked_orders as (
  WATERMARK(orders, order_time, order_time - INTERVAL '1' MINUTE), 
) 

-- normal timeout watermark with simple check
with watermarked_orders as (
  WATERMARK(
  orders, 
  order_time, 
  CASE
  -- we have not determined the design about `PROC_TIME()`
    WHEN order_time > PROC_TIME() THEN Null
    ELSE order_time - INTERVAL '1' MINUTE
  END), 

  -- normal timeout watermark with removing outliers
with watermarked_orders as (
  WATERMARK(
  (
    -- we have not determined the window funtion on unordered stream
    select *, 
      max(order_time) as win_max_time 
        OVER(BETWEEN 10 PRECEDING AND CURRENT ROW)
    from orders
  ), 
  order_time, 
  CASE
    WHEN order_time == win_max_time THEN Null
    ELSE order_time - INTERVAL '1' MINUTE
  END), 
) 
```

## Future possibilities

### Should we support extra columns in Sort?

Yes, it definitely works, we can also sort columns prefixed by a watermarked column, and it may be useful for some optimization such as SortAgg and SortMergeJoin.

### GroupWatermark over Source

Flink has a watermark strategy that is aware of the partitioning. The typical use case is that the partitions upstream are skewed heavily but records in every partition are almost ordered. We can introduce a `GroupWatermark` operator for the scenario. We can discuss it later.

### The type of watermark column

In the doc, we require the type of watermark column to be `Timestamp`, but it’s not necessary. The only limitation is `Ord + Sub`. At least, we have to support `Timestamp`, `TimestampZ`, and `int64` (for unix epoch). We can even support `Decimal` easily, but I don’t think it’s meaningful.
