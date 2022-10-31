---
feature: over_window
authors:
  - "TennyZhuang"
start_date: "2022/10/27"
---

# Over Window

## Summary

We will introduce the Over Window (OverWindow) to RisingWave in the RFC.

Another well-known name of OverWindow is [Window Function][window_function_wiki]. Unfortunately, the name is highly ambiguous in streaming.

Flink named the feature Over Aggregation][over_aggregation_flink], but we think Over Window can describe the part more concisely.

[window_function_wiki]: https://en.wikipedia.org/wiki/Window_function_(SQL)#:~:text=In%20SQL%2C%20a%20window%20function,single%20value%20for%20multiple%20rows.

## Definitions

Window function call:

```plain
function_name ([expression [, expression ... ]]) [ FILTER ( WHERE filter_clause ) ] OVER window_name
function_name ([expression [, expression ... ]]) [ FILTER ( WHERE filter_clause ) ] OVER ( window_definition )
function_name ( * ) [ FILTER ( WHERE filter_clause ) ] OVER window_name
function_name ( * ) [ FILTER ( WHERE filter_clause ) ] OVER ( window_definition )
```

Window definition:

```plain
[ existing_window_name ]
[ PARTITION BY expression [, ...] ]
[ ORDER BY expression [ ASC | DESC | USING operator ] [ NULLS { FIRST | LAST } ] [, ...] ]
[ frame_clause ]
```

frame_cause:

```plain
 RANGE | ROWS | GROUPS } frame_start [ frame_exclusion ]
{ RANGE | ROWS | GROUPS } BETWEEN frame_start AND frame_end [ frame_exclusion ]
```

We can simplify that in our first version:

```plain
[ existing_window_name ]
[ PARTITION BY expression ]
[ ORDER BY column_ref ]
[ frame_clause ]
```

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
* `FOLLOWING` is **banned; the max value of `frame_start` and `frame_end` is `CURRENT ROW`.
* Window functions that peek the future rows are **banned**, e.g., `lag`.

### Multiple windows

In the SQL standard, different windows are allowed in the queries, although there are few cases to use that.

We can support them in the frontend easily.

Assume we have two windows:

```plain
WINDOW w1 AS (PARTITION BY p1 ORDER BY s1)
WINDOW w2 AS (PARTITION BY p2 ORDER BY s3)
```

We can plan the query as two branches with a join to combine them:

![https://viewer.diagrams.net/?tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1&title=over-window-multiple.drawio#R7Vlbs5owEP41zLQP7UAQ1FcvvU07nY7t6DlvKeRAeiChIQj21zdAEBC11DmnwY4vmv2yS5Ldb5eNauY8zN4yGPmfqIsCDehuppkLDQBDNybiK0d2JWKbZgl4DLtSqQZW%2BBeqLCWaYBfFLUVOacBx1AYdSghyeAuDjNG0rfZAg%2FaqEfRQB1g5MOiia%2Bxyv0QnYFzj7xD2%2FGplw56WMyGslOVJYh%2B6NG1A5lIz54xSXo7CbI6C3HmVX0q7Nydm9xtjiPA%2BBndb%2Bz6ZrR%2FeT6zPm%2FDrz82XafxKRifmu%2BrAyBXnlyJl3KceJTBY1uiM0YS4KH%2BqLqRa5yOlkQANAf5AnO9kMGHCqYB8HgZyFmWYbxrju%2FxRry0pLTL55ELYVQLhbLdpCg2rXKzNCqmyK8%2BXH%2Bqk2yQU04Q56IyvgKQfZB7iZ%2FSsfXBFViAaIrEfYcdQADnetvcBJT29vV4dQTGQQfyLgE5vAX3ygNoqAyo3uYVBIlf6FsWcIRhqwA7EtmffmRh5%2BUizZtGjUBIf1qJLhCAQVTMPeOpjjlYRLLyTisLdDqdcETGOsvNO7jpFGoxk0ZNV36yKYFrXUKPC%2FEb9nOjP5cauH1VkyhOy1%2BrJ3lOR%2Bjf0tTpuX2aOD4lwwDH6uo9CudDTI0M1iQ9ZvJfVsbjrzitnsd2XxSOVLLYvZzEYGotHumoWG%2FatSemdIEZ16%2FlThhhK%2B85qm40UWWOSX3iOdimwaFOUV3jLHlqFN8a33OifG%2BZ15IZ5QW4of28c5sYA3hvT%2F637Mfo28eXPaMoI3G07P1BMjtIXE4KEPO9OOpTki7yQV9SXqgk%2Btod2Sa1%2BLG34eUFTMvzr%2FhQMzZWgW3OvvFaA3o2g0gs%2F6DaCK%2BG4o%2ByN63tSrLwX3L%2FfBtMLgu6189o5POrLYVMph0eXcVh5z3bI4efs2YRY%2F3lVzDX%2BAjSXvwE%3D](https://user-images.githubusercontent.com/9161438/198935912-7771b230-736f-4b9e-8ded-0aa8073d2bee.png)

The `pk` is the primary key of upstream.

There is a particular optimization for state cleaning; we need to store the `s1` as the prefix of the right-side state table in the join executor and the same on the left side. This can be done quickly by the front end.

## Unresolved questions

### Future records lookup

We can support `FOLLOWING` and `lag` by buffering some records or emitting a `NULL` downstream and updating them later. We can discuss them later.

## Future possibilities

### Two-phase OverWindow

If the `PARTITION BY` clause is not specified, we can only use a singleton to maintain the materialized view, which may be the bottleneck of the whole DAG graph. We can do two-phase optimization like how we do that in [two-phase aggregation](https://singularity-data.quip.com/KtaRA6CspqRK/RFC-2-Phase-Agg-TopN-Operator-in-Streaming).

However, too many window aggregators can't be implemented, e.g., `lead`.
