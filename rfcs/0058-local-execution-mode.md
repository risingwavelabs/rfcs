---
feature: local_execution_mode
authors:
  - "Renjie Liu"
start_date: "2022/04/01"
---

# Design: Local execution mode for online serving queries

## Background

The query engine of risingwave is expected to support two kinds of query: highly concurrent point query and adhoc query.
The characteristics of these two different kinds of queries are summarized as followings:

|	|Point Query	|Adhoc query	|
|---	|---	|---	|
|Latency	|several ms	|ms to minutes	|
|QPS	|1000 ~ 100000	|100 ~ 1000	|
|SQL	|Simple	|Arbitrary complex	|
|Result Set	|Small	|Small, Medium, Large	|
|Use Case	|Dashboard	|Engineers, Data Scientist	|

In [Distributed Query Processing Engine Design](./0057-distribution-query-engine.md) we described the distributed query processing engine for complex adhoc query, but it can’t meet the latency/qps requirement of point query, and in this article we introduce local execution mode for point query.

## Design

![](./images/0058-local-execution-mode/design.png)


## Example 1

```sql
select a from t where b in (1, 2, 3, 4)
```

Let’s use above sql as example:

![](./images/0058-local-execution-mode/example-1.png)

The key changes from distributed mode:

1. The exchange executor will be executed by directly local query execution, not by the distributed scheduler. This means that we no longer have async execution/monitoring, etc.
2. The rpc is issued by the exchange executor directly, not by the scheduler.

## Example2: 

```sql
SELECT pk, t1.a, t1.fk, t2.b FROM t1, t2 WHERE t1.fk = t2.pk AND t1.pk = 114514
```

Following is the plan and execution of the above SQL in local mode:

![](./images/0058-local-execution-mode/example-2.png)

As explained above, the lookup join/exchange phase will be executed directly on frontend. The pushdown(filter/table, both the build and probe side) will be issued by executors rather than the scheduler.

### Optimization/Scheduling

The overall process will be quite similar to distributed processing, but with a little difference:

1. We only use a heuristic optimizer for it, and only a limited set of rules will be applied.
2. No scheduler will involve, and the physical plan is executed in the current thread(coroutine) immediately.

### Monitoring/Management

Local execution mode will not go through query management mentioned in  [Batch Query Manager Design](https://singularity-data.quip.com/XfFGAGsdU4T2) to reduce latency as much as possible.

### How to switch between local/distributed execution mode

As mentioned in first paragraph, the main use case for local execution mode is determined(dashboard/reporting), so currently we just expose a session configuration (`query_mode`) to user. In future, we may use the optimizer to determine it, but it depends on the requirement.

### RPC execution in local mode

In distributed mode we have several steps to execute compute task and fetch results:

![](./images/0058-local-execution-mode/rpc.png)

There are some problems with above process in local mode:

1. We need at least two rpcs to fetch task execution result, this increases query overhead
2. We have task lifecycle management apis, this is unnecessary for local mode.
3. We may need to add several new apis for task monitoring/failure detection


For local mode we will add a new rpc api:

```protobuf
rpc Execute(ExecuteRequest) returns (ExecuteResponse)

message ExecuteRequest {
 batch_plan.PlanFragment plan = 2;
 uint64 epoch = 3;
}

message ExecuteResponse {
  common.Status status = 1;
  data.DataChunk record_batch = 2;
}
```

This is quite similar to distributed execution api, but with some differences:

1. Response is returned directly in rpc call, without the need of another call
2. No task lifecycle management/monitoring is involved, and if it fails, we just remove the task and return error in response directly.

## Conclusion

1. We should keep query_mode so that user can enable this when necessary.
2. We should keep optimizer/execution as concise as possible so that we don’t need to maintain two code paths.
