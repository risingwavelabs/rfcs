---
feature: over_window_on_watermark
authors:
  - "TennyZhuang"
start_date: "2022/10/27"
---

# Over Window on watermark

## Summary

We will introduce the Over Window (OverWindow) to RisingWave in the RFC.

Another well-known name of OverWindow is [Window Function][window_function_wiki]. Unfortunately, the name is highly ambiguous in streaming.

Flink named the feature [Over Aggregation][over_aggregation_flink], but we think Over Window can describe the part more concisely.

[window_function_wiki]: https://en.wikipedia.org/wiki/Window_function_(SQL)#:~:text=In%20SQL%2C%20a%20window%20function,single%20value%20for%20multiple%20rows.
[over_aggregation_flink]: https://nightlies.apache.org/flink/flink-docs-master/docs/dev/table/sql/queries/over-agg

Note: We'll only implement a limited OverWindow based on watermark strategy.

## Definitions

Window function call:

```plain
function_name ([expression [, expression ... ]]) [ FILTER ( WHERE filter_clause ) ] OVER window_name
function_name ([expression [, expression ... ]]) [ FILTER ( WHERE filter_clause ) ] OVER ( window_definition )
function_name ( * ) [ FILTER ( WHERE filter_clause ) ] OVER window_name
function_name ( * ) [ FILTER ( WHERE filter_clause ) ] OVER ( window_definition )
```

window_definition: only a column_ref is supported in the `ORDER BY` clause.

```plain
[ existing_window_name ]
[ PARTITION BY expression ]
[ ORDER BY column_ref ]
[ frame_clause ]
```

frame_clause: only `ROWS` is supported in our first version, and `BETWEEN` is necessary.

```plain
ROWS BETWEEN frame_start AND frame_end
```

## Motivation

The feature is a part of the SQL standard and is widely supported by different SQL databases and streaming systems.

The feature is also helpful in fraud detection.

## Design

### Primary key derivation

The output rows count of OverWindow is exact as the input rows, as is the primary key.

### Distribution key

In most window definitions, one partition key is specified, and the data should be exchanged by the partition key.

### BatchOverWindow

The implementation is trivial.

### StreamOverWindow

The design highly depends on [the WatermarkFilter and StreamSort operator](https://github.com/risingwavelabs/rfcs/pull/1).

If the input stream is ordered, then the implementation is trivial.

There are some limitations in stream aggregation:

* If the sort key is specified, then a watermark must exist on the column.
* If no sort key is specified, then the result is undefined but may be helpful; it's roughly the same as the proctime in Flink.
* `FOLLOWING` is **banned**; the max value of `frame_start` and `frame_end` is `CURRENT ROW`.
* Window functions that peek the future rows are **banned**, e.g., `lag`.

## Unresolved questions

### Future records lookup

We can support `FOLLOWING` and `lag` by buffering some records or emitting a `NULL` downstream and updating them later. We can discuss them later.

## Future possibilities

### Two-phase OverWindow

If the `PARTITION BY` clause is not specified, we can only use a singleton to maintain the materialized view, which may be the bottleneck of the whole DAG graph. We can do two-phase optimization like how we do that in [two-phase aggregation](https://singularity-data.quip.com/KtaRA6CspqRK/RFC-2-Phase-Agg-TopN-Operator-in-Streaming).

However, too many window aggregators can't be implemented, e.g., `lead`.

### Multiple windows

In the SQL standard, different windows are allowed in the queries, although there are few cases to use that.

We can support them in the frontend easily.

Assume we have two windows:

```plain
WINDOW w1 AS (PARTITION BY p1 ORDER BY s1)
WINDOW w2 AS (PARTITION BY p2 ORDER BY s3)
```

We can plan the query as two branches with a join to combine them:

![https://viewer.diagrams.net/?tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1#R7VrbcpswEP0aZtqHdAwYgl99adJO2jTjduz0TTEy0IBEhWzjfH2EkQxY2IPdNMiZvNja1ep29mjZxdbMQZReERD737ALQ83ouKlmDjXD0Du6w74yzTrX2KaZKzwSuNyoUIyDJyhGcu0icGFSMaQYhzSIq8oZRgjOaEUHCMGrqtkch9VVY%2BBBSTGegVDWTgKX%2BrnWMS4L%2FTUMPF%2BsrNu9vCcCwpifJPGBi1cllTnSzAHBmOatKB3AMANP4JKP%2B7ynd7sxAhFtMuB%2Baf9e9CfzL451O41%2B%2Fp3e9ZIL7p2ErsWBocvOz0VMqI89jEA4KrR9ghfIhdmsHSYVNjcYx0ypM%2BUfSOmaOxMsKGYqn0Yh74VpQKel9n021SeLS8OUz7wR1kJAlKynZaE0KhOLYRtJjJNR4sAleEFm8AA0BmcbIB6kB%2Bys3C7DrbQA98EVxBFk%2B2EGBIaABssqrwCnp7e1KzzIGtyJRzi09%2B7Qf3WorZRD%2Ba6XIFzwlX7FCSUQRJphh%2Bwc%2FQfCWl7W0qx%2B%2FMiM2Ic1lIkQhixqZg5f%2BQGF4xhs4FqxwF11516Il5BQmB4Ehfc6POjxqG%2BKILgqYqgudH4pfopxLw%2BjjGMbN%2BV09loN2SsmVIS%2BlgT7KJ35ALHN1dHXfWTGG7tOrLdNYqNbZfFWbo%2FFMpznxWK7KYu7SrHYPp3Fhmos7nZekcU3%2Fe%2FX9vRpTu7g9c3tw4%2BJ5U8vLhUi8Q7BTmO1LiqTEq1rT263SWKxyxLukwBlNUltIgE2mUTrQdiyWwzCtU7svTn6mg3p67RKX%2FME%2BrYefXfp%2B6rRtx5Hlfh7El2bpsLi3ZMiSYQuJ29fcYBq6RsgBJk8kDtnGGWLfOCF3se2Ce4oV%2BoJt5dwHuIVUr9o1jtd1bA05KB7XsHCqEnNDmZHigQLQ07Wxgy4WvomRbmRtJ6vbR9w6hTNcvV2ZhzuNuWwqRaHu6dxuPWkbZfD7ZfM4uewt%2F5e%2F9h6Zn%2BV3eS6WK90Ow7t8oh3Si%2BUo0j3oAblvVfj0lCtHBcP7ver0eBqOGdxNeQE%2Fhyvxv98ajCx%2BOfApq%2F0%2Fwtz9Aw%3D](https://user-images.githubusercontent.com/9161438/198952308-a7ab08f2-2858-4e06-a06a-edbbeee72638.png)

The `pk` is the primary key of upstream.

There is a particular optimization for state cleaning; we need to store the `s1` as the prefix of the right-side state table in the join executor and the same on the left side. This can be done quickly by the front end.

