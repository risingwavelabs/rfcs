---
feature: temporal_filter
authors:
  - "Eric Fu"
start_date: "2022/11/08"
---

# RFC: Temporal Filter

## Background

A temporal filter is a filtering condition on some particular time column according to the current time, usually used to **represent a sliding window** with a flexible start or end. For example,

* `time_col >= now() - INTERVAL '1 hour'` 
    — a sliding window of recent 1 hour
* `time_col BETWEEN now() - INTERVAL '2 hour' AND NOW() - INTERVAL '1 hour'` 
    — a sliding window of 1 hour before 1 hour
* `time_col >= date_trunc('day', now())` 
    — a sliding window of today (from 0:00)

On the contrary, below are ***NOT*** temporal filters

* `time_col >= '2022-10-01 00:00:00'` 
    — an ordinary static filtering condition
* `to_char(time_col, ’yyyymmdd') = to_char(now(), 'yyyymmdd')
    `— the condition is not applied on datetime/timestamp data type

Besides the semantics itself, a temporal filter is also useful to **bound the size of states for an arbitrary query**. For example,
as mentioned in [State Cleaning in Flink](https://singularity-data.quip.com/Z4KPAzFlFKz8), a query without any proper temporal conditions naturally involve all historical data, thus the size of states will grow infinitely. To solve this, one possible solution is to put a temporal filter there:

* `SELECT customer_id, amount FROM orders GROUP BY customer_id
    `— the size of states is unbounded
* `SELECT customer_id, amount FROM orders GROUP BY customer_id
    WHERE order_date > NOW() - INTERVAL '7 DAYS'
    `— the size of states is bounded


You may also be interested in [the implementation of temporal filters in Materialize](https://materialize.com/blog/temporal-filters/#appendex-implementation). It’s quite clever, but unluckily, not applicable for RisingWave. I would strongly recommend taking a glance at that blog article, and you will actually find some connectivity between these 2 approaches - at least equivalent time & space compelexity.


## Design

### Lower-bound temporal conditions

A lower-bound temporal condition let the incoming rows live for some bounded time, and will delete them at some time (mostly) in some future time point.

Temporal filters can be probably considered as a special case of [RFC: Dynamic Filter - A New Streaming Operator](https://singularity-data.quip.com/AE06Ao1kAIaZ), where the expression with `now()` as the dynamic part. For example, to deal with

```
WHERE time_col > now() - INTERVAL '1 hour'
```

What we will do is create a table ordered by `time_col` and find out the position of `now() - INTERVAL '1 hour'` in that table. Once a new epoch comes with a barrier, the value of that expression will be updated, and the position it points to also changes, and some delete events may be emitted accordingly.

![](https://user-images.githubusercontent.com/10192522/201015841-ad84dda9-807a-498d-a141-8d4d362f4e9c.png)

A nice property of time is that it will never be set back. Therefore, unlike an ordinary dynamic filter, here we can drop the rows once being filtered out to keep the size of states bounded.

### (Optional) Upper-bound temporal conditions

Sometimes users may define a window with both lower and upper bound, like

```
WHERE time_col BETWEEN now() - INTERVAL '2 hour' AND NOW() - INTERVAL '1 hour' 
```

> Well, I’m not sure whether it’s useful but [Materialize took this as an example](https://materialize.com/docs/sql/patterns/temporal-filters/#details). Anyway, it’s doable in our approach.


The idea is quite similar except that now we need 2 positions: one for past rows and one for future rows. Both positions need to be updated according to epoch changes, and some insert or delete events will be triggered accordingly.

![](https://user-images.githubusercontent.com/10192522/201015808-a051748a-3ee0-45d4-83e0-df6ae8a2feed.png)


By the way, a temporal condition without lower-bound but only upper-bound is somehow weird to me, and it cannot limit the state size. I prefer to forbid such cases, because it may just be a mistake by users. 

### (Optional) Expression transformation

Sometimes people may want to write the temporal filter in other forms, like

```
WHERE now() <= time_col + INTERVAL '1 hour'
```

A simple transformation can be applied implicitly to normalize it into our favored preferred form i.e. `time_col >= now() - INTERVAL '1 hour'` . I’m afraid that users may feel very confused without this.

## Discussions

**Does this proposal have to be implemented based on the dynamic filter operator?**

Not necessary actually. The temporal filter differs from ordinary dynamic filters in 2 aspects

* Only one input stream is needed for a temporal filter operator.
    * To implement it on dynamic filter, we can introduce a new operator (or TVF like generate_series) which always outputs 1-row 1-column table with a single value of current timestamp i.e. `now()` 
* The condition `now()` is monotonically increasing, so the past states can be dropped
    * To implement it on dynamic filter, we can let the new operator mentioned above emit watermark messages (see https://github.com/risingwavelabs/rfcs/pull/2/files) on barriers, and make the dynamic filter operator to clean state on watermark messages.
    * Note that in this schemem, state cleaning and emitting `delete` events are independent. State cleanning is driven by `Watermark` message, that is, the events below the watermark will be removed from state. Emitting `delete` (or `insert`) events is driven by `Barrier` message, because the epoch in barriers will advance the current time (`now()`) and then trigger new `delete` or `insert` events to downsteam.

Either introducing a new `TempralFilter` operator or reusing `DynamicFilter` looks good to me. Let's leave the choice to the implementer.

## References

* [Temporal Filters: Enabling Windowed Queries in Materialize](https://materialize.com/blog/temporal-filters/#appendex-implementation)

## Q&A


* ***Tianyi Zhuang***: A key point is that how users decide to use Windowing TVF or temporal filter?
    * A rough idea: Materialize recommend users to use temporal filter to implement hop window too, but not us, the temporal filter can be used in the following two cases:
        * real sliding window: When users want a near-future sliding window, they should use temporal filter.
        * temporal filter over TVF: When users want the result of a recent hop window, they can use temporal filter over a TVF.
    * ***Eric Fu replies***: Agree with you. It actually depends on the user’s query: if the `where` condition applies on time column of table, then it’s a “real sliding window”; otherwise, if the condition applies on the window_start of TVF, that will be “temporal filter over TVF”. We could guide users in the docs/tutorials.
* ***Zilin Chen***: What’s the semantic of `now()` in select-clause aka. Project executor?
    * ***Eric Fu replies***: I think `now()` should be be the current time for all rows rather than per row. However, this will be very expensive - similar to a nested-loop join with a constantly changing inner side. I prefer to ban this situation now.

