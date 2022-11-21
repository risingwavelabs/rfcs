---
feature: non_deterministic-functions
authors:
  - "st1page"
start_date: "2022/10/24"
---

# nondeterministic-functions

## Summary

We introduce the `MaterializeProject` stream executor to handle non-deterministic function on updatable stream.

## Motivation

In the discussion about the udf, @skyzh raise the issue about the non-deterministic function on updatable stream <https://github.com/risingwavelabs/rfcs/pull/12#issuecomment-1305105614>. This issue also appears on the built-in functions such as `random(),` `proc_time()` or `now()`.

## Design

We will introduce a `MaterializeProject`, it will materialize some **partial** columns of result with the stream key of the input as the primiary key. When an `Insert` operation comes, it will compute the result and materialize some columns. when `Update` or `Delete` comes, it will lookup its state to replace the old value of the operation.
For example, if we have a materialized view with SQL `select random() as rd, v*2 as vv, pk from t` and the pk is the primary key of the table t. The stream processing plan will be like:

```plain
StreamMaterialize(columns:[rd, vv, pk])
  StreamMaterializeProject(
      exprs: [Random(), InputRef(0) * 2, InputRef(1)],
      materialized_cols: [0],
      state_table: [pk, rd]
    )
    StreamScan(fields:[v, pk])
```

The `StreamMaterializeProject` will materialize the first columns of its input which is the `random()` function's result. When we have a serial of changes `Insert(v = 10, pk = 1), Update((v = 10, pk = 1) -> (v = 20, pk = 1)), Delete(v = 20, pk = 1)` the `StreamMaterializeProject` will handle it as following:

1. receive `Insert(v = 10, pk = 1)`
    - comput the result `(rd = 111, v = 20, pk = 1)`
    - insert into state table `(pk = 1, rd = 111)`
    - output `Insert(rd = 111, v = 20, pk = 1)`

2. receive `Update((v = 10, pk = 1) -> (v = 20, pk = 1))`
    - compute the old value's project result `(rd = 55555, v = 20, pk = 1)`
    - get the old values `(pk = 1, rd = 111)` from state table modify the result old value to `(rd = 111, v = 10, pk = 1)`
    - compute the new value `(rd = 444, v = 30, pk = 1)`
    - insert into state table `(pk = 1, rd = 444)`
    - output `Update((rd = 111, v = 20, pk = 1) -> (rd = 444,v = 40, pk = 1))`
3. receive `Delete(v = 20, pk = 1)`
    - compute the old value's project result `(rd = 55555, v = 40, pk = 1)`
    - get the old values `(pk = 1, rd = 444)` from state table modify the result old value to `(rd = 444, v = 40, pk = 1)`
    - delete state table `(pk = 1)`
    - output `Delete(rd = 444,v = 40, pk = 1)`

## Future possibilities

We might can pull-up those `StreamMaterializeProject` to the first `Materialize` executor or other stateful executors. For example, the above example's plan can be

```plain
StreamMaterialize(columns:[rd, vv, pk], flag: check_and_modify)
  StreamProject(exprs: [Random(), InputRef(0) * 2, InputRef(1)],)
    StreamScan(fields:[v, pk])
```
