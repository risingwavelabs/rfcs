---
feature: Use udf as source
authors:
  - "Gio Gutierrez <@bakjos>"
start_date: "2024/04/19"
---

# Use UDF as a source

With the amount of sources and how different is consumed by the clients, using UDFs could help to support more sources, and easily integrate with different types of APIs and protocols:

- Http long pooling (Internal APIs)
- WebSockets/SSE (GraphQL/Rest)
- Grpc Endpoints

## Design

The UDFs are already supporting tables, and use yield to generate the values through streaming futures <https://github.com/risingwavelabs/risingwave/blob/ed75502880af4fad364b9821ab85e7026188063d/src/expr/core/src/table_function/user_defined.rs#L93-L132>. This functionality could be translated to tables/sources to emit records when a new value is received from the UDF

## Proposed Syntax

```sql
CREATE [TABLE|SOURCE ] .. AS <function_name> (param1, param2) [WITH (..)]
```

Table/Source schema can be copied from the UDF return type or could be defined with the `CREATE TABLE` syntax, and it will be required to support updates and deletes, by defining the primary key. This [PoC](https://github.com/risingwavelabs/risingwave/pull/16388) implements it by using an operation field that can be returned by the UDF and have the values `insert`, `delete`, `update_delete` and `update_insert`.

## Unresolved questions

## Future possibilities

- Add support for async wasm UDFs
- Define a way to provide splits for the UDFs
- Add support for local state (to keep things as the auth tokens)
