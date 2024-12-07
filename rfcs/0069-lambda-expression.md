---
feature: lambda_expression
authors:
  - "st1page"
start_date: "2023/08/01"
---

# Lambda expression

## Background

To better support user's semi-structure data, We have supported `Array` and `Struct` types compatible with PG. But in some user's requirement, We found we can not support them easily.

### Introduction

```SQL
CREATE TABLE t(
  arr_i int[],
  arr_json jsonb[],
  k int primary key
);
```

Consider we have the table definition and want to get the sum of the `arr_i` and sum of `arr_json`'s field `x`. In PG user can write the correlated subquery like this.

```SQL
dev=# explain (verbose) 
    select *, 
        (select sum(i) from unnest(arr_i) i) as sum_i, 
        (select sum((i->'x')::int) from unnest(arr_json) i) as sum_x 
    from t;
                                        QUERY PLAN                                        
------------------------------------------------------------------------------------------
 Seq Scan on public.t  (cost=0.00..294.75 rows=850 width=84)
   Output: t.arr_i, t.arr_json, t.k, (SubPlan 1), (SubPlan 2)
   SubPlan 1
     ->  Aggregate  (cost=0.13..0.14 rows=1 width=8)
           Output: sum(i.i)
           ->  Function Scan on pg_catalog.unnest i  (cost=0.00..0.10 rows=10 width=4)
                 Output: i.i
                 Function Call: unnest(t.arr_i)
   SubPlan 2
     ->  Aggregate  (cost=0.18..0.19 rows=1 width=8)
           Output: sum(((i_1.i -> 'x'::text))::integer)
           ->  Function Scan on pg_catalog.unnest i_1  (cost=0.00..0.10 rows=10 width=32)
                 Output: i_1.i
                 Function Call: unnest(t.arr_json)
```

The correlated column is in the table function `Unnest` and it is hard to do the sub-query decorrelation. Unfortunately, we do not implement `Apply` operator in streaming execution.
Their are some equivalent query that I think could be decorrelated plan. But they all are complex and with high cost.

```SQL
  select * from t t1 
    join (
      select k, sum(i)
      from (select k, unnest(arr_i) i from t) u group by k
    ) t2 on t1.k = t2.k
    join (
      select k, sum((i->'x')::int) 
      from (select k, unnest(arr_json) i from t) u group by k
    ) t3 on t1.k = t3.k;
  /* Or */
  select distinct on(k)
    k, arr_i, arr_json,
    sum(i) over (partition by k),
    sum((j->'x')::int) over (partition by k)
  from ( select *, unnest(arr_i) i, unnest(arr_json) j from t) foo;
```

In investigation of other SQL systems, there are 3 levels' enhancement on this kind of query, this RFC will be concerned with the three levels.

### level 1: array aggregate expression

It is a kind of SQL **scalar expression** which accept a array type and return the array's aggregation result, such as `array_sum`, `array_max`. It can solve some part of the issue but can not handle the complex nested datatype in the array.

```SQL
select *, array_sum(arr_i) from t;
```

### level 2: scalar lambda expression

Implement lambda expression function to make user can define there `transform` logic for array's each elements.
It can be used with the array aggregate expression together.

```SQL
select *,
  array_sum(arr_i),
  array_sum(transform(arr_json, v -> (v->'x')::int))
from t;
```

### level 3: aggregate lambda expression

Based on the level 2, these system support the `reduce` to make user use lambda function implement the aggregation logic.

```SQL
select *,
  reduce(arr_i, 0, (s, v) -> s + v),
  reduce(arr_json, 0, (s, v) -> s + (v->'x')::int)),
from t;
```

Btw, It often can be used to express UDAF too.

## Investigation

 Array expressions support. (All of them supports the `array_size/cardinality`, `unnest/flatten/explode` which is not listed in the table.)

|            | array_cnt | array_sum | array_cum_sum | array_avg |  array_max/min   | array_sort | array_distinct | array_zip | Unnest<br>with idx |
| ---------- | :-------: | :-------: | :-----------: | :-------: | :--------------: | :--------: | :------------: | :-------: | :----------------: |
| Databricks |           |           |               |           |        Y         |     Y      |       Y        |     Y     |         Y          |
| Trino      |           |           |               |           |        Y         |     Y      |       Y        |     Y     |         Y          |
| PrestoDB   |     Y     |     Y     |       Y       |     Y     |        Y         |     Y      |       Y        |     Y     |         Y          |
| Starrocks  |     Y     |     Y     |       Y       |     Y     |        Y         |     Y      |       Y        |     Y     |                    |
| Doris      |     Y     |     Y     |       Y       |     Y     |        Y         |     Y      |       Y        |     Y     |                    |
| ClickHouse |     Y     |     Y     |       Y       |     Y     |        Y         |     Y      |       Y        |     Y     |                    |
| DuckDB     |     Y     |     Y     |       Y       |     Y     |        Y         |     Y      |       Y        |           |                    |
| Vertica    |     Y     |     Y     |               |     Y     |        Y         |            |                |           |         Y          |
| FlinkSQL   |           |           |               |     Y     | only MAX, no MIN |            |                |           |                    |
| Snowflake  |           |           |               |           |                  |            |       Y        |           |         Y          |



Array expressions with lambda expression paramter

|            | transform | transform<br>with idx | filter | filter<br>with idx | sort cmp | reduce | UDAF  | Lambda Limitations                                                                                                     |
| ---------- | --------- | --------------------- | ------ | ------------------ | -------- | ------ | :---: | ---------------------------------------------------------------------------------------------------------------------- |
| Databricks | Y         | Y                     | Y      | Y                  | Y        | Y      |   Y   | No subquery, No SQL UDF                                                                                                |
| Trino      | Y         |                       | Y      |                    | Y        | Y      |   Y   | No subquery, No Aggregations                                                                                           |
| PrestoDB   | Y         |                       | Y      |                    | Y        | Y      |   Y   | No subquery, No Aggregations                                                                                           |
| Starrocks  | Y         |                       | Y      |                    | Y        |        |       |                                                                                                                        |
| Doris      | Y         |                       | Y      |                    | Y        |        |       |                                                                                                                        |
| ClickHouse | Y         | Y                     | Y      | Y                  | Y        |        |       |                                                                                                                        |
| DuckDB     | Y         |                       | Y      |                    |          |        |       |                                                                                                                        |
| Vertica    | Y         | Y                     | Y      | Y                  | Y        |        |       | arg name [limitations](https://docs.vertica.com/23.3.x/en/sql-reference/language-elements/lambda-functions/#arguments) |
| FlinkSQL   |           |                       |        |                    |          |        |       |                                                                                                                        |
| Snowflake  |           |                       |        |                    |          |        |       |                                                                                                                        |

  - No capture in lambda function, I think No system support refer column or sub query in the lambda function
  - Snowflake do not anything about it and fall back to the `unnest/flatten/explode` https://stackoverflow.com/questions/62807878/how-to-get-maximum-value-from-an-array-column-in-snowflake
  - [to confirm] Accroding to the doc, FlinkSQL only support flatten the array in join clause https://nightlies.apache.org/flink/flink-docs-master/docs/dev/table/sql/queries/joins/#array-expansion
  - There are some Syntactic sugar. 
    - Trino/PrestoDB/Starrocks: support pass multiple arrays in `unnest/flatten/explode` so that the arrays will do zip first.
    - ClickHousr/Starrocks: `array_sum(lambda_function, arr1,arr2...) = array_sum(array_map(lambda_function, arr1,arr2...))`
  - ClickHouse support any aggregation function on array directly, e.g. `SELECT arrayReduce('max', [1, 2, 3]);`
  
### Refers
  - Databricks
    - https://docs.databricks.com/sql/language-manual/sql-ref-lambda-functions.html
    - https://docs.databricks.com/sql/language-manual/sql-ref-functions-builtin-alpha.html
  - Trino
    - https://trino.io/docs/current/functions/lambda.html
    - https://trino.io/docs/current/functions/array.html?highlight=array#reduce
    - https://trino.io/docs/current/functions/aggregate.html?highlight=lambda+expressions#lambda-aggregate-functions
    - https://trino.io/docs/current/functions/array.html?highlight=array#array_max
  - PrestoDB
    - https://prestodb.io/blog/2020/03/02/presto-lambda
    - https://prestodb.io/docs/current/functions/lambda.html
    - https://prestodb.io/docs/current/functions/aggregate.html#reduce_agg
  - Starrocks
    - https://docs.starrocks.io/en-us/3.0/sql-reference/sql-functions/Lambda_expression
  - Doris
    - https://doris.apache.org/docs/dev/sql-manual/sql-functions/array-functions/array_max
    - https://doris.apache.org/search?q=lambda
  - ClickHouse
    - https://clickhouse.com/docs/en/sql-reference/functions/array-functions
    - https://clickhouse.com/docs/en/sql-reference/functions#higher-order-functions---operator-and-lambdaparams-expr-function
  - DuckDB
    - https://duckdb.org/docs/sql/functions/nested#list-aggregates
    - https://duckdb.org/docs/sql/functions/nested#lambda-functions
  - Vertica
    - https://docs.vertica.com/23.3.x/en/sql-reference/language-elements/lambda-functions/
  - FlinkSQL
    - https://nightlies.apache.org/flink/flink-docs-master/docs/dev/table/functions/systemfunctions/

## Design
[TODO]
We can support missing array function's at first and they has been implemented for aggregators so we can reuse the code.
For the lambda expression, it is like a anonymous SQL UDF. We can implement the SQL UDF first and design lambda expression later.

## Unresolved questions
[TODO]

- Are there some questions that haven't been resolved in the RFC?
- Can they be resolved in some future RFCs?
- Move some meaningful comments to here.

## Alternatives
[TODO]
What other designs have been considered and what is the rationale for not choosing them?

## Future possibilities
[TODO]
Some potential extensions or optimizations can be done in the future based on the RFC.
