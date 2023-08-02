---
feature: user_defined_aggregate_functions
authors:
  - "Runji Wang"
start_date: "2023/08/02"
---

# User Defined Aggregate Functions (UDAF)

## Summary

This RFC proposes the user interface and internal design of user defined aggregate functions (UDAF).

## User Interfaces

This section describes how users create UDAFs in Python, Java, and SQL through examples.

### Python API

Similar to scalar functions and table functions, we provide an `@udaf` decorator to define aggregate functions. The difference is that below the decorator, we define a class instead of a function. The class is an **accumulator** that accumulates the input rows and computes the aggregate result. It can also optionally retracts a row and merges with another accumulator.

```python
from risingwave.udf import udaf

# The aggregate function is defined as a class.
# Specify the schema of the aggregate function in the `udaf` decorator.
@udaf(input_types=['BIGINT', 'INT'], result_type='BIGINT')
class WeightedAvg:
    # The internal state of the accumulator is defined as fields.
    # They will be serialized into bytes before sending to kernel,
    # and deserialized from bytes after receiving from kernel.
    sum: int
    count: int

    # Initialize the accumulator.
    def __init__(self):
        self.sum = 0
        self.count = 0

    # Get the aggregate result.
    # The return value should match `result_type`.
    def get_value(self) -> int:
        if self.count == 0:
            return None
        else:
            return self.sum / self.count

    # Accumulate a row.
    # The arguments should match `input_types`.
    def accumulate(self, value: int, weight: int):
        self.sum += value * weight
        self.count += weight

    # Retract a row.
    # This method is optional. If not defined, the function is append-only and not retractable.
    def retract(self, value: int, weight: int):
        self.sum -= value * weight
        self.count -= weight

    # Merge with another accumulator.
    # This method is optional. If defined, the function can be optimized with two-phase aggregation.
    def merge(self, other: WeightedAvg):
        self.count += other.count
        self.sum += other.sum
```

#### Alternative

In Flink, the accumulator is a separate variable passed into functions as an argument.

```python
class WeightedAvg(AggregateFunction):
    def get_accumulator_type(self):
        return 'ROW<f0 BIGINT, f1 BIGINT>'

    def create_accumulator(self):
        # Row(sum, count)
        return Row(0, 0)

    def get_value(self, accumulator):
        # ...

    def accumulate(self, accumulator, value, weight):
        accumulator[0] += value * weight
        accumulator[1] += weight
```

While in this proposal, the accumulator is the class itself. Users don't have to define the accumulator type explicitly.

Reference: [UDAF Python API in Flink](https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/dev/python/table/udfs/python_udfs/#aggregate-functions)

### Java API

The Java API is similar to Python. One of the differences is that users don't need to specify the input and output types, because we can infer them from the function signatures.

```java
import com.risingwave.functions.AggregateFunction;

// The aggregate function is defined as a class, which implements the `AggregateFunction` interface.
public static class WeightedAvg implements AggregateFunction {
    // The internal state of the accumulator is defined as class fields.
    public long sum = 0;
    public int count = 0;

    // Create a new accumulator.
    public WeightedAvg() {}

    // Get the aggregate result.
    // The result type is inferred from the signature. (BIGINT)
    // If a Java type can not infer to an unique SQL type, it should be annotated with `@DataTypeHint`.
    public Long getValue() {
        if (count == 0) {
            return null;
        } else {
            return sum / count;
        }
    }

    // Accumulate a row.
    // The input types are inferred from the signatures. (BIGINT, INT)
    // If a Java type can not infer to an unique SQL type, it should be annotated with `@DataTypeHint`.
    public void accumulate(long iValue, int iWeight) {
        sum += iValue * iWeight;
        count += iWeight;
    }

    // Retract a row. (optional)
    // The function signature should match `accumulate`.
    public void retract(long iValue, int iWeight) {
        sum -= iValue * iWeight;
        count -= iWeight;
    }

    // Merge with another accumulator. (optional)
    public void merge(WeightedAvg a) {
        count += a.count;
        sum += a.sum;
    }
}
```

Reference: [UDAF Java API in Flink](https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/dev/table/functions/udfs/#aggregate-functions)

### SQL API

After setting up the UDF server, users can register the function through `create aggregate` in RisingWave.

```sql
create aggregate weighted_avg(value bigint, weight int) returns bigint
as 'weighted_avg' using link 'http://localhost:8815';
```

The full syntax is:

```sql
CREATE [ OR REPLACE ] AGGREGATE name ( [ argname ] arg_data_type [ , ... ] )
[ RETURNS return_data_type ] [ APPEND ONLY ]
[ LANGUAGE language ] [ AS identifier ] [ USING LINK link ];
```

The `RETURNS` clause is optional if the function has exactly one input argument.
In this case the return type is same as the input type. For example:

```sql
create aggregate max(int) as ...;
```

The `APPEND ONLY` clause is required if the function doesn't have `retract` method.

The remaining clauses are defined the same as for `create function` statement.

## Design

### Dataflow

User defined functions are running in a separate process called UDF server. The RisingWave kernel communicates with the UDF server through Arrow Flight RPC to exchange data.

To avoid trouble in fault tolerance, the UDF server should be **stateless** even if the aggregate function is stateful. This means the aggregate state should be maintained by the kernel. However, exchanging the state in each RPC call (batch aggregation) is not efficient, especially when the state is large. On the other hand, kernel doesn't need to know the state or the aggregate result before a barrier arrives. Therefore, we **create a streaming RPC `doExchange` to call the aggregate function, and sync the state every time a barrier arrives**. When the connection to the UDF server is broken, the executor raises an error and may retry or pause the stream.

```mermaid
sequenceDiagram
    participant UP as Upstream<br/>Downstream
    participant AG as Agg Executor
    participant CP as UDF Server
    UP ->> AG: init(epoch)
    Note over AG: load state<br/>from state table
    AG ->>+ CP: doExchange(fid, state)
    CP -->> AG: output @ epoch
    Note right of CP: decode state,<br/>create accumulator
    loop each epoch
        loop each chunk
            UP ->> AG: input chunk
            AG ->> CP: input chunk
            Note right of CP: accumulate,<br/>retract
        end
        UP ->> AG: barrier(epoch+1)
        AG ->> CP: finish
        Note right of CP: encode state,<br/>get output
        CP -->> AG: state + output @ epoch+1
        Note over AG: store state<br/>into state table
        AG ->> UP: - output @ epoch<br/>+ output @ epoch+1
        AG ->> UP: barrier(epoch+1)
    end
    deactivate CP
```

### State Storage

The state of UDAF is managed by the compute node as a single encoded BYTEA value.

Currently, each aggregate operator has a **result table** to store the aggregate result. For most of our built-in aggregate functions, they have the same output as their state, so the result table is actually being used as the state table. However, for general UDAFs, their state may not be the same as their output. Such functions are not supported for now.

Therefore, we propose to **transform the result table into state table**. The content of the table remains the same for existing functions. But for new functions whose state is different from output, only the state is stored. The output can be computed from the state when needed.

For example, given the input:

| Epoch | op   | id (pk) | v0   | w0   | v1    |
| ----- | ---- | ------- | ---- | ---- | ----- |
| 1     | +    | 0       | 1    | 2    | false |
| 2     | U-   | 0       | 1    | 2    | false |
| 2     | U+   | 0       | 2    | 1    | true  |

The new **state table** (derived from old result table) of the agg operator would be like:

| Epoch | id   | sum(v0) | bool_and(v1)        | weighted_avg(v0, w0) | max(v0)*   |
| ----- | ---- | ------- | ------------------- | -------------------- | ---------- |
| 1     | 0    | sum = 1 | false = 1, true = 0 | encode(1,2) = b'XXX' | output = 1 |
| 2     | 0    | sum = 2 | false = 0, true = 1 | encode(2,1) = b'YYY' | output = 2 |

* For **append-only** aggregate functions (e.g. max, min, first, last, string_agg...), their states are all input values maintained in seperate "materialized input" tables. For backward compatibility, their values in the state table are still aggregate results.

The output would be:

| Epoch | op   | id   | sum(v0) | bool_and(v1) | weighted_avg(v0, w0) | max(v0) |
| ----- | ---- | ---- | ------- | ------------ | -------------------- | ------- |
| 1     | +    | 0    | 1       | false        | 1                    | 1       |
| 2     | U-   | 0    | 1       | false        | 1                    | 1       |
| 2     | U+   | 0    | 2       | true         | 2                    | 2       |

## Unresolved questions

## Alternatives

## Future possibilities
