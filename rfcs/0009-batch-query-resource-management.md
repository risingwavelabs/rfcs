---
feature: batch_resource_management
authors:
  - "Renjie Liu"
start_date: "2022/11/03"
---

# Batch Query Resource Management

## Summary

We will introduce coarse grained resource management for batch query engine:

1. Limit number of concurrent distributed queries in each frontend node.
2. Use `catch_unwind` to prevent oom aborting process.

## Motivation

When user executes complex sql in distributed mode in our system with huge amount of data, our compute node will abort(not panic) when running out of memory. We want to reduce process failure and impact to other users as much as possible.

### Non Goal

There exists many different fine grained resource management strategies in industry, and the goal of this rfc is not to bring them into our system.

## Design

To avoid aborting in allocation failure, we should catch oom error and return an error. Here is an example code:

```rust
#[cfg(test)]
mod tests {
    use std::alloc::set_alloc_error_hook;

    use futures::FutureExt;
    use tokio::spawn;

    #[tokio::test]
    async fn run_oom() {
        set_alloc_error_hook(|_| {
            panic!("Allocation error!");
        });

        let f = async {
            let x = Vec::<u128>::with_capacity(99999999999999999);
            println!("x's len is {:?}", x.len());
        }
        .catch_unwind();

        let ret = f.await;
        assert!(!ret.is_err());
    }
}
```

We set an oom error hook to panic rather than abort, and use unwind to catch this error. For details please refer to [this rfc][1] and [this pr][2].

To further reduce the chance of query failure, we need to limit the number of concurrent distributed queries running in cluster. To implement this mechanism, we have two assumptions:

1. Load balancer works well, so each frontend node receives almost same amount of queries.
2. Enforcing this limitation is not mandatory, just best effort is enough.

Based on above assumptions, we will introduct a cluster wide `max_concurrent_distributed_queries` config, which limits number of concurrent distributed queries in cluster. Meta node will divide this value according to number of frontend nodes, and sync to each frontend node. Frontend node will limit this concurrent queries according to this value.

## Unresolved questions

1. Although [this rfc][1] is approved, it's not in stable rust. There is no guarantee that all collections is aware of this behavior(panic of abort), and this may be unsound.

2. Currently the default behavior of panic is aborting, and this is used for health check in streaming engine. There are two ways to solve this issue:

* Add true heartbeat mechanism for streaming health check.
* Tokio runtime has provided [an unstable api](https://docs.rs/tokio/latest/tokio/runtime/struct.Builder.html#method.unhandled_panic) to customize unhandled panic, e.g. we can stop a runtime when panic happens. We can use this to stop streaming runtime when panic happens.

# Alternatives

For memory limitation, we may use other techniques, such as `try_reserve` method in collection api, or reserves an estimated size before true allocation happends. There are some problems with this approach:

1. We will need much more effort than proposed approach.
2. The estimation maybe difficult and inaccurate when collection alloc is not falliable.
3. Since currently we don't want to work on fine grained resource management, there is nothing we can do when we catch the error except throwing away.
4. The proposed approach is the server profile in [this rfc][1], which is similar to our current requirement.

## Future possibilities

Fine grained resource management.

[1]: <https://rust-lang.github.io/rfcs/2116-alloc-me-maybe.html#additional-background-how-collections-handle-allocation-now>
[2]: <https://github.com/rust-lang/rust/issues/51245>
