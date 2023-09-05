---
feature: expr-opaque-context
authors:
  - "TennyZhuang"
start_date: "2023/09/06"
---

# RFC: Opaque context in expression evaluation

## Motivation

### Why we want to introduce a context?

When implementing some util functions in pg, we found that they heavily depends on some context:

* `current_database`/`current_user`/`pg_postmaster_start_time`/...: depends on session variables
* `pg_is_in_recovery`: depends on session states
* `pg_get_userbyid`/`obj_description`/...: depends on catalog cache
* `pg_cancel_backend`: depends on meta client

Currently, we implement them all in binder, and only literals can be argument to these function calls. Some complicated queries can't be supported:

```sql
SELECT
  idx.schemaname,
  idx.tablename,
  idx.indexname,
  obj_description(
    format('%s.%s',
           quote_ident(idx.schemaname),
           quote_ident(idx.indexname)
          )::regclass
  ) AS comment
FROM pg_indexes AS idx;
```

It is acceptable for this query to be executed only in local execution mode.

### Why it should be an opaque context?

In brief, we don't want to introduce the cyclic dependencies between `frontend` and `expr`.

When we want to introduce an extra context argument to `Expression::eval` method, what should the signature be?

```rust
async fn eval(&self, ctx: ?, input: &DataChunk) -> Result<ArrayRef>;
```

If we define a concrete FrontendContext here, it should be like this:

```rust
async fn eval(&self, ctx: Option<FrontendContext>, input: &DataChunk) -> Result<ArrayRef>;
```

There are several drawbacks:

1. **The `expr` crate depends on the `frontend` crate.**
2. It's hard for us to provide more context from other crates. e.g. the backend may provide a part of context.
3. Override some context variables is a little hard. We have to modify the variable, and don't forget to recover it when exiting context.

The key point is that the call stack is in the ABA form: `frontend::run_query` -> `executor::exec` -> `expr::eval` -> `frontend::current_schema`

And the `context` should be opaque for `executor` and `expr`, but concrate for `frontend`.

## Design

We'll utilize the [`provide_any`](https://github.com/nrc/provide-any) crate, although it's rejected by std.

In `expr`:

```rust
pub trait Context: Provider {}
// We can also provide some utilities such as `ChainedContext` here.
pub trait Expression {
  fn eval(&self, ctx: &dyn Context>, chunk: &DataChunk);
  // ...
}
```

In `frontend`:

```rust
pub struct FrontendContext {
  meta_client: MetaClient,
  session_variables: SessionVariables,
  catalog: Catalog,
  tz: TimeZone,
}
impl Provider for FrontendContext {
  fn provide<'a>(&'a self, demand: &mut Demand<'a>) {
    demand
      .provide_ref::<MetaClient>(&self.meta_client)
      .provide_ref::<SessionVariables>(&self.session_variables)
      .provide_ref::<Catalog>(&self.catalog)
      .provide_value::<Tz>(self.tz);
  }
}
impl Context for FrontendContext {}
```

In `frontend/expr`:

```rust
struct PgGetUserByIdExpression;
// TODO: we can simplify the codes by another #[function] macro.
impl expr::Expression for PgGetUserByIdExpression {
  fn eval(&self, ctx: &dyn Context>, chunk: &DataChunk) {
    let Ok(catalog) = request_ref::<Catalog>(ctx) else {
      return Err("Not supported in the current context");
    };
    // retrieve the ids from chunk.
    Ok(res)
  }
}
// TODO: We should also consider how to construct the expression without depending on frontend.
```

In `executor`:

```rust
pub struct BackendContext {
  tz: TimeZone,
}
impl Provider for BackendContext {
  fn provide<'a>(&'a self, demand: &mut Demand<'a>) {
    // We can only provide a subset of context.
    demand
      .provide_value::<Tz>(self.tz);
  }
}
```

We can also dispatch some different logics based on the context:

```rust
pub trait Reporter {}
// In frontend
pub struct NoticeReporter;
impl Reporter for NoticeReporter {}
demand.provide_ref<dyn Reporter>(&self.notice_reporter);
// In backend
pub struct ErrorTableReporter;
impl Reporter for ErrorTableReporter {}
demand.provide_ref<dyn Reporter>(&self.error_table_reporter);
```

## Related work

The pattern was widely used in golang. `context` is a part of std lib.

There are also a hook [`useContext`](https://react.dev/reference/react/useContext) in react, which is excatly the same pattern as the RFC proposed.

## Alternatives

### task_local

tokio's [`task_local` macro](https://docs.rs/tokio/latest/tokio/macro.task_local.html) can achieve the same result even simpler, and the only cost is the extensive use of global variables.

## Further questions

Can the pattern be used for contexts in other module?
