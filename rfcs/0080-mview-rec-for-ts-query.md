---
feature: mview_rec_for_ts_query
authors:
  - "TennyZhuang"
start_date: "2023/11/15"
---

# PreRFC:  Minimal mview recommendation for speeding up Time-Series queries

## Motivation

Currently, RisingWave is not user-friendly for Time-Series Data Analysis in certain scenarios. In Time-Series Databases (TSDB), they usually use Pre-Aggregation to accelerate time series queries. We typically consider Incremental Materialized View to be a superset of Pre-Aggregation, but in fact, this is not the case.

A typical query for Time-Series data analysis:

```sql
SELECT region, az, hostname,
    ROUND(AVG(measure_value::double), 2) AS avg_cpu_utilization,
    ROUND(APPROX_PERCENTILE(measure_value::double, 0.99), 2) AS p99_cpu_utilization
FROM TUMBLE(DevOps, time, interval '1' minute)
WHERE measure_name = 'cpu_utilization'
    AND $__time_filter(window_start)
GROUP BY region, hostname, az
```

The query calculates the average CPU utilization and the p99 CPU utilization within **a specified time range**. To speedup the query,  the data are binned by 1 minute. The results are grouped by region, hostname, and availability zone (az).

Due to the arbitrariness of `$__time_filter`, we encounter some difficulties when trying to accelerate this query with a materialized view.

In fact, that is feasible. The definition of a materialized view should be like this:

```sql
CREATE MATERIALIZED VIEW cpu_utilization_intermediate AS
SELECT region, az, hostname,
    window_start,
    SUM(mesuare_value::double) AS sum_cpu_utilization,
    COUNT(1) AS record_cnt,
    APPROX_PERCENTILE_PARTIAL(0.99, measure_value::double) AS p99_cpu_utilization_intermediate
FROM TUMBLE(DevOps, time, interval '1' minute)
WHERE measure_name = 'cpu_utilization'
GROUP BY window_start, region, hostname, az
```

And it should be used in this way:

```sql
SELECT region, az, hostname,
    ROUND(sum_cpu_utilization / record_cnt, 2) AS avg_cpu_utilization,
    ROUND(APPROX_PERCENTILE_GLOBAL(0.99, p99_cpu_utilization_intermediate), 2) AS p99_cpu_utilization
FROM cpu_utilization_intermediate
WHERE $__time_filter(window_start)
GROUP BY region, hostname, az;
```

`APPROX_PERCENTILE_{PARTIAL|GLOBAL}` are two internal functions, which are combined through a two-phase aggregation to calculate the approximate percentile result.

We will find that manually splitting this query is very unfriendly to users, they need:

1. Deeply understand two-phase aggregation and split aggregate functions.
2. Split the query and expression tree into two parts: pre-aggregation and post-aggregation. The calculation before aggregation is defined in the mview, while the calculation after aggregation is called in an arbitrary query.

The root reason is that the definition of materialized view is unaware of the queries on top of it.

We should provide users with a suitable syntax to declare their intention of conducting Time-Series queries on a specific column. At the same time, we should also assist users in building materialized views more easily.

I would like to propose some proposals, but I do not have a clear preference.

## Designs

### Materialized View Recommendation and Selection

This is a very ideal solution, where users directly submit their parameterized serving queries to us instead of the definition of materialized views.

```sql
CREATE FUNCTION cpu_metrics(
  timestamptz,
  timestamptz,
  OUT region VARCHAR,
  OUT az VARCHAR,
  OUT hostname VARCHAR,
  OUT avg_cpu_utilization DOUBLE PRECISION,
  OUT p99_cpu_utilization DOUBLE PRECISION)
RETURNS SETOF record AS $$
  SELECT region, az, hostname,
    ROUND(AVG(measure_value::double), 2) AS avg_cpu_utilization,
    ROUND(APPROX_PERCENTILE(measure_value::double, 0.99), 2) AS p99_cpu_utilization
  FROM TUMBLE(DevOps, time, interval '1' minute)
  WHERE measure_name = 'cpu_utilization'
    AND window_start BETWEEN $1 AND $2
  GROUP BY region, hostname, az
$$ LANGUAGE SQL;

EXPLAIN CREATE MATERIALIZED VIEW FOR cpu_metrics;
CREATE MATERIALIZED VIEW FOR cpu_metrics;
```

In the above SQL, a function with two input parameters is created, and then we create materialized views for that. Please note that we can discuss the syntax later, e.g. a sugar like `CREATE MATERIALIZED FUNCTION` also looks acceptable for me.

For the materialized view selection, we'll provide several guarantees:

1. If the function is called directly, it'll always be speedup by the mviews.
2. If the query is **eactly** match the function pattern, it'll always be speedup by the mviews.
3. Otherwise, no guarentee, we'll try our best :(

The potential of MView recommendation is very significant. For example, if we obtain more information than the definition of MView, we can know whether MView serves point queries or range queries, which is very helpful for our storage and compaction strategy.

## Further Probabilities

### Rollup the window

When a user creates a one-minute tumble window, we can actually automatically create five-minute, one-hour, or even one-day tumble windows at the same time. By borrowing ideas from skip lists, we can serve queries over a large range more quickly.
