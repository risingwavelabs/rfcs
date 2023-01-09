---
feature: watermark-operators-explained
authors:
  - "Bugen Zhao"
start_date: "2022/10/25"
---

# Watermark Operators Explained

In [RFC: The WatermarkFilter and StreamSort Operator](https://github.com/risingwavelabs/rfcs/pull/2), we showed how watermark will be used in RisingWave. This doc will introduce how the key operators should be implemented in our system.

![Overview of the design. Image explained in detail below](https://user-images.githubusercontent.com/25862682/200244705-67cc7f80-d89f-408f-a47a-26414db55a5f.png)

> Overview of the design. Image explained in detail below

Basically, the general idea is similar to the design of Flink like above, while one should note that there’re several major differences in our system:

- Watermark can be generated everywhere, not associated with or to be a property of the sources.
- We’re able to scale the stream graph online. Therefore, the state persistence and recovery should be considered.

We’ll introduce 3 operators one by one in the following.

## Watermark Filter (Stateful)

The Watermark Filter maintains the timestamp-like column as the watermark while taking input records, filtering out the outdated records based on the watermark and a fixed **timeout**. Thus, the Watermark Filter is the source of truth of the `Watermark` messages, and it does not need to reorder or buffer the input chunks.

![https://viewer.diagrams.net/?border=0&tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1&title=watermark-1.drawio&open=R5VdNc5swEP01nkkP6fBhsHN0bCc9tDOd8XTSnjoKyKBGICJEwP31XSHJIAP5bi%2FNRehptSvt27dyZv46a645KtIvLMZ05jlxM%2FM3M89z514Ig0QOClnM5wpIOIm1UQfsyG%2BsQUejFYlxaRkKxqgghQ1GLM9xJCwMcc5q22zPqB21QAkeALsI0SF6Q2KRKnTpLTr8EyZJaiK74YVayZAx1jcpUxSzugf525m%2F5owJ9ZU1a0xl8kxe1L6ridXjwTjOxXM2OM0dC%2FNdk9xef7svdouidn%2BeB8rLA6KVvrA%2BrDiYDOAYEqKnjIuUJSxHdNuhl5xVeYxlGAdmnc1nxgoAXQB%2FYSEOml1UCQZQKjKqV%2FcsF2tGGW8j%2BldXDvwBPryivnXJKh7hR%2B6l7eTZext1Yq4xy7DgBzDgmCJBHmyyka6Z5Gh33PqVETgKxFQWQajJ1dU9dxzbhUA8wULv6siBj94xOqil7AX0uSP0hVTolFo8hvcVMwvnZcvECgzcoGjaTJt1%2BErkeIME5hnid9sGR8AYPxMkw6ySXluHjv%2FBBIOzq3hqq4FvuUFq40ztXAyLjFJQtCymOiUC7wrU0ltDU7FLZbIkHjAXuHmUbL3qLW3SfENi3QkcmFRY2hP30pmuD4vZl9LoPUOFebyS7QxmEUVlSSI7L7YGITn88L0%2F%2BSEnHwMz3TT9xc1Bzyazq%2Br4HQTXy3EwkmKDvVGXR%2FqmdKkayECXA0fzE0f%2Bxb8VuD%2BoDMgFuHI8NYRqOFF9dOylnar9%2Fb7tqgOh%2B5Mqlp7nagjUsFSDfJzfGnYxGfak8kHVwq51REmSSyFAlWIIeCm1T%2BDNXumFjMSxepkwdDp027qS9V1InlrmgstZsJG%2BoLWpbti6LgVnd9i8RTnLpZc9ofQEeoc25DonpbUMBm3IHWtD%2Ft9qQ%2BFTr8kowY4ThmME35w50w9EW0WvreTpiI88SZPF%2FLaIwRN3fIFSpoMspoP8J2I5%2FaE1JpbRB%2BUVYoFp96tcNfLufxt%2F%2Bwc%3D](https://user-images.githubusercontent.com/9161438/197964182-b9b78e8c-1940-43c4-b819-ff999bb16796.png)

There’re two notable facts:

- The watermark filters run independently on each partition (parallel unit), so they’ll emit `Watermark` messages **based on the local input** like the specific splits assigned to the upstream source executor, which can be slightly different from others.
- The semantics of watermark requires that, the timestamp of all records after the `Watermark(t)` message **must** not be less than `t`.

Based on the above, consider a case of scale-in: the partition of one parallel unit (to be removed) with watermark `5` is split and merged into a parallel unit with watermark `3`. Then we must treat the watermark of this new partition as `max(3, 5) = 5` and filters out all records with timestamp less than `5`, including those from the previous partition. We can simplify the design and avoid buffering records based on this.

This leads to the design:

- **Schema:** only buffer the watermark messages
  `Vnode(..) [key], Watermark`

- **On checkpoint**
  For all vnodes handled by this parallel unit, persist the same local watermark value.

- **On fail-over**
  Get the state with any of the vnodes handled by this parallel unit, take the value as the local watermark.

- **On scaling**
  Scan **all states**, take the **maximum** value as the watermark.

- **On any initialization (the first barrier after creating / scaling)**
  (May) initialize, and always emit the `Watermark` message immediately. Explained in section [Exchange (Stateless)](#exchange-stateless).

By always using the global-maximum value as the watermark after scaling, we ensure that no existing or new partition will emit a `Watermark` message with a smaller value after scaling. This is necessary for implementing the stateless Exchange.

When implementing, if we don’t want to recognize fail-over against scaling, we may always take the global maximum.

For source operators, the `Vnode` here makes less sense and what we really want is the Split ID. However, we have no “external distribution” support currently: we may only treat the vnode as the **division** assigned by the meta service and should not care about the value itself.

## Exchange (Stateless)

After the Watermark Filter per parallel unit, we need to handle the `Watermark` message across the Exchange. The basic idea is very similar to Flink. We show that, there will be no significant changes on the Exchange and it will still be **stateless** based on the design of the Watermark Filter above, with support for fail-over and scaling naturally.

![Global Watermark](https://user-images.githubusercontent.com/25862682/200244696-12cd9629-d769-4064-b867-dfb0f209fd89.png)

### Dispatcher

- **On `Watermark`**
  Broadcast the message to all outputs, like `Barrier`.

### Merger

There will be multiple inputs (upstreams) for the Merger. As the watermark indicates that there’s no records with a smaller timestamp from now, the Merger should always **take the minimum watermark** from all inputs.

- **On any initialization**
  As the Watermark Filter always emit a `Watermark` message this time, we’ll receive watermarks from all inputs now.

- **On `Watermark`**
  Buffer the message to a in-memory per-input queue.
  **Only if** all queues are non-empty, we’re sure that which `Watermark` the minimum of all inputs is now, and we can emit it to the downstream. After that, clean-up all the `Watermark` messages with the same value as the emitted one.

- **On `Chunk`**
  Forward the chunk immediately, regardless of the current watermark. Note that deferring a `Watermark` message after a chunk doesn’t break the correctness.

- **On `Barrier`**
  Align the barrier like before.

- **On upstream scaling**
  Initialize an empty queue for newly registered upstream.
  As we’ve ensured the watermark emitted right after scaling must be the global-maximum, it must be **not smaller than** the watermark emitted by the Merger before the scaling. So the monotonicity of `Watermark` messages is still held.

- **On current fragment scaling**
  The Merger on the new partition will receive watermarks from all inputs immediately, and reach the global watermark.

Right before aligning all barriers, the mergers in all (downstream) partitions will finish receiving **the same set** of `Watermark` messages from all inputs during this epoch. It’s easy to see that the algorithm above is deterministic, so a same watermark value is aligned in all partitions, that is, the watermark after the Exchange will become a **global watermark** (marked as blue). This is a must-have property for the Sort operator.

~~We also have Barrier Aligner for those operators with multiple upstreams. The behavior should be similar and also simpler, compared to the Merger.~~

## Sort (Stateful)

It’s impossible to sort an infinite stream basically, so we’re not able to benefit from the order property in streaming, like the Sort Aggregation and Window Functions. Luckily, with watermark, we cut the stream into finite parts and sort them individually when watermark bumps, so the stream can be transformed to be ordered and append-only. We split this responsibility to a separate operator named Sort, which must be stateful as it must buffer the records until watermark bumps.

![https://viewer.diagrams.net/?border=0&tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1&title=Untitled.drawio&open=R7VZLj5swEP41SMlhK4KB0GMeu%2B1hV6oarWiPTnDAXYOpMXn013eMDcQh2Ye27akne76xx3jmm884aJEfPglcZg88Iczx3OTgoKXjeRPfC2FQyFEjU9%2FXQCpoYhb1wIr%2BIgZ0DVrThFTWQsk5k7S0wQ0vCrKRFoaF4Ht72ZYz%2B9QSp2QArDaYDdGYJjLTaORNe%2FwzoWnWnjwJP2pPjtvF5iZVhhO%2BP4HQrYMWgnOpZ%2FlhQZhKXpsXve%2Fuirf7MEEK%2BZoN60f09dtTIOOZeLjbPdbzHb2%2FMdXZYVabC5uPlcc2AySBhBiTC5nxlBeY3fboXPC6SIg6xgWrX3PPeQngBMAfRMqjqS6uJQcokzkz3i0vpHFOArD1N6iDr97VQBWvxYY8c0GvyzRQlPCcSHGEfYIwLOnOjo8NV9JuXbf1C6dwsucaXoeuKaphtd%2FabQiJRUqk2dUXBSYnn9FDTaneUDbvQtlCJk0qrfqFP2vFsLk1S834%2FBbluKmausxgwSQoDxcDraDkbTC4jo5nn7EWLbKut1sCltsc5TrBPHK8hQoP86We7rEkIsfiqVs2HQa6euI5fxkDsVA83WdUklWJG8LsQa9sFpqcEiHJ4XneDflkNqDQ5kVn73vt8FquZCe6EbnXKWiR561MQa9o8CKZKaUEa8NwVdGNnRe7vc97FTIjjt%2BMszG%2BK%2BND51seTp3Lo7Fe3%2BO6l97e4yc5Dy6kvMXeKQXIe0EKtEQNpGAQyPfPuBP8W03xB0yJR%2B7Y9KYePD2EeohHyLh9PQR6iFp3MO462yDT8YB90GzS5htmNC0UGYEIoBRorlqSwpM8M46cJol%2BeAioE143oRSnSpWbJlvBXIkJxIK3pjJsBbOSgj%2BRBWcc4i4LXqgoW8rYOXTxQXqXNJzRBEXBQBnQJWVAf0sZopfrHY8mY6vy8cizSx6P%2FLFV%2FL7qHUvC8f%2FyD%2F4YLtY%2F%2BjP1B7P%2FrdTy0P%2Bco9vf](https://user-images.githubusercontent.com/9161438/197971481-56e25377-914f-41fb-9c3f-7818a9eabec3.png)

Every record in the output can pair with a watermark as it’s ordered, but we may still omit some watermarks for performance.

To make it correct and simple, a Sort operator **requires a global watermark** emitted from the Exchange. Thus, it cannot be directly put after the Watermark Filter in the same fragment. We may need to manually insert an Exchange for this.

The behavior of Sort operator:

- **Schema:** only buffer the records
  `(Timestamp, Stream Key) [order key], Record` distributed by the consistent hash

- **On `Chunk`**
  Buffer the records to a in-memory B-Tree. No need to operate on the storage memory table.

- **On checkpoint**
  Persist all in-memory records to the state store to make them consistent.

- **On any initialization**
  Fetch all records from the state store to the in-memory buffer.

- **On `Watermark`**
  Drain the range of `..watermark` for both in-memory buffer and the state store (with `RangeDelete` operation), yield them in order. Then propagate the `Watermark`.
  No need to maintain the watermark, both in memory and in state store.

- **On scaling**
  Reset the state table with the new partition info.
  The scaling only occur after the checkpoint, so we **must have aligned** the global watermark. Even if the Watermark Filter is also scaled, the watermark emitted won’t decrease. Therefore, merging records from the buffer of other partitions won’t break the order of the output.

Furthermore, the logic of sorting can also be embedded to existing stateful operators to reduce unnecessary updates. For example, if one want to “freeze” the aggregation result of a window when it’s still not closed, there’s no need to maintain this aggregation group any more.
