---
feature: over_window_on_watermark
authors:
  - "TennyZhuang"
start_date: "2022/10/27"
---

# Over Window on watermark

## Summary

We'll introduce the OverWindow to RisingWave in Our RFC. Another well-known name for this is ["Window Function"][window_function_wiki]. Unfortunately, the name is highly ambiguous when applied to streaming algorithms; We think calling it such might cause confusion among users who do not specialize in SQL standard.

Flink named the feature [Over Aggregation][over_aggregation_flink], but we think OverWindow is more concise.

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
