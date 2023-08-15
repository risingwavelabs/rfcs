# Arrangement Backfill

## Background

We previously introduced [Backfill](0013-use-backfill-to-let-mv-on-mv-stream-again.md),
to enable Materialized View Creation to progress smoothly.

In the existing implementation, Backfill and Upstream MV are coupled together.
With the introduction of Arrangement Backfill,
we can decouple them to allow Backfill to scale independently.

A scenario where this could help:
Upstream has many updates, and backfill cannot keep up.
With Arrangement Backfill, we can scale-out backfill, and it can catch up.

This RFC has 2 parts. First, the conceptual design, and second the implementation design.

## Conceptual Design of ArrangementBackfill

The design of Arrangement Backfill is largely the same as the No Shuffle Backfill (existing implementation).
The main changes are to allow Arrangement Backfill to scale independently.

Similar to existing Backfill, we have two streams we read from, Upstream Updates and Snapshot stream.
The snapshot stream is refreshed on every barrier.
In Backfill, we do a non-checkpoint read on the upstream storage table to get the snapshot data.

Because Arrangement Backfill can be on a different CN than the Upstream MV executor,
they may not have the same shared buffer.
This means we can only read from checkpointed data.

We need some way to get the non-checkpointed data, i.e. data residing in upstream's shared buffer.
To do this, we can buffer all upstream messages from the upstream mv between two checkpoint barriers.
Then, we perform a merged read on the buffer and the checkpoint data in the storage.

When a checkpoint barrier comes, discard the buffer (We need to keep the buffer until it can be read from the storage.).

The other difference is that Backfill's state now needs to be per vnode.
Previously Backfill had a single `current_pos`, which indicated backfill's progress.
Because we want to support scale-out however, we need to track progress per vnode, so we know where to resume from.

## Implementation Design of ArrangementBackfill

For the frontend component, we can support ArrangementBackfill as an experimental feature first, just disable it unless some session variable is set.

Only after [Test plan](https://www.notion.so/Test-plan-a49c0b6166634147898d9f055bfb6cde?pvs=21) is finished, we consider it stable.

It’s implementation will be similar to the `Backfill` executor.

Key differences:

- It will read from state_table instead of storage table.
- The state_table interface will need to be modified to support ordered reads from all the table’s vnodes.
- This state_table also needs to use a local_state_store with `replicated` property. **See  [Scenario - Motivation for LocalStateStore replication](https://www.notion.so/Scenario-Motivation-for-LocalStateStore-replication-5fc32752d4934aaaab6f739519169a42?pvs=21).**
- Updates need to be inserted into this state table.
- We need to flush this state table on barrier as well.
- Note we do not need to support iter by epoch interface in state table. This is because our iteration pattern is the same, we will always read from latest committed epoch for snapshot read. That is supported by state table.

## Scenario - Motivation for LocalStateStore replication

> 💡 The following is an example of **dropped updates.** It can happen if we don’t have some mechanism to buffer messages outside of the current batch `(current_pos, batch_size]`.
> 

Suppose batch size = 1.

At epoch 4, we have the following upstream committed state, i.e. persisted to s3 (3 snapshots):

| v1 (PK) | v2  |
|---------|-----|
| 1       | 2   |
| 2       | 4   |
| 3       | 6   |

Executor processes the first snapshot.

- This means downstream to backfill has received `+ (1, 2)`.
- This means **current_pos = `1`.**

Next, we receive a chunk of updates from upstream executor.

Chunk A

| U-  | 1   | 2   |
|-----|-----|-----|
| U+  | 1   | 3   |
| U-  | 2   | 4   |
| U+  | 2   | 5   |

The entire chunk is buffered.

Next, we receive a non-checkpoint barrier:

Barrier

| prev_epoch | cur_epoch | is_checkpoint |
|------------|-----------|---------------|
| 4          | 5         | f             |

Now all data has been flushed to `shared buffer` in upstream.

Backfill will have to do the following:

1. Yield the buffered updates to downstream.
    
    `mark_chunk` will ensure we only process updates to `v1 = 1` for Chunk A.
    
    That is because for  `v1 = 2, 2 > current_pos = 1`. Which means it will be ignored.
    
2. Read a new snapshot from the **latest committed epoch**, with the current_pos as the excluded lower bound.
    
    → **This means we read from epoch 4, (1,2]**
    
    That would be:

    | v1 (PK) | v2  |
    |---------|-----|
    | 2       | 4   |
        
    Whereas, the state in upstream would be:
    
    | v1 (PK) | v2          |     |
    |---------|-------------|-----|
    | 1       | 3 (updated) |     |
    | 2       | 5 (updated) |     |
    | 3       | 6           |     |

    The above tables show the **main problem** we are trying to solve with the temporary storage.
    
    Typically at this point, we would just read from storage, and get `(2, 5)`.
    
    But because we can’t do that, we get `(2, 4)` which is outdated.
    
    Additionally because [we ignored the `(2, 5)` update as well](https://www.notion.so/Arrangement-Backfill-31d4d0044726451c9cc77dc84a0f8bb2?pvs=21) when we processed the stream chunks, this effectively means `2, 5` has been dropped, and we lost the update.
    
    **Here’s the solution:**
    
    We initialize `ArrangementBackfill` **with a State Table**.
    The state table is has a replica of upstream’s `local_state_store`. Everything else should be the same.
    When we receive updates from upstream, we write + flush them to state table **on barrier** 
    (this is an implementation detail, we could write them before barrier too, but that makes it more complicated).
    Then, when snapshot read happens, state table will merge the local state store’s `ReadVersion` with persisted data from upstream.
    So we will see `(2, 5)`
    Next, when checkpoint barrier received, it will get the state table to do async cleanup of shared buffer.

3. Finally yield the barrier downstream.

## Storage Component - `LocalHummockStorage`

It should configure a flag which indicates that it should not upload any imm when flushing.

It should be initialized to read from Upstream’s storage component. This can be done by just reusing upstream `table_id`.

Additionally, when we have `ArrangementBackfill` in the same CN as upstream `mview` executor, we will have 2 copies of the same read version.
To avoid reading both when doing global state store read (happens when reading via `StorageTable`),
we just ignore any replicated versions when reading via GlobalStatestore (i.e.  `HummockStateStore`).

## Test plan

- Similar to backfill executor, run the same tests.
- Additionally, we need to test 3 scenarios for correctness:
    - Different CN
    - Same CN
    - Scale out
- Finally, we need to benchmark and compare the two performance in same CN.
    - This can be done using mview-on-mview nexmark queries.
- Only after these are done, we can consider it stable.

## Discussions

- Cleanup of shared buffer
    
    **Noel:** Here’s a scenario:
    
    1. upstream received checkpoint barrier.
    2. upstream starts committing data to s3.
    3. upstream then forward checkpoint barrier downstream to arrangement backfill.
    4. arrangement backfill needs to wait (asynchronously) for all executors to finish committing to s3, only then it can discard its shared buffer.
    5. That is because if it discards the shared buffer, and upstream not finished persisting to s3, then the buffered updates will be lost.
    
    Seems like step 4 will need some changes as well, since we will only get the arrangement backfill state table to discard its shared buffer AFTER all executor finish committing to s3. This is different from current design, where cleaning up of state table can be done independently. Is this correct?
    
    **Dylan Chen:** This step has already been handled by the storage, because in the normal case (no arrangement backfill) the local state storage still need to wait the commit message from meta to decide whether to discard its data.
    
- Noel: Could we use replicate executor + backfill instead?
    
    Originally this is the thought, since we could use replicate executor with delta join as well.
    
    But this is harder than expected.
    
    **Reason 1:** We want to stop replication the moment upstream is finished to save memory. It is much more complex to do this since backfill executor needs to communicate this to replicate executor.
    
    **Reason 2:** For storage layer, need to replicate into a local state store. Since backfill and replicate executor are in separate executors, if backfill reads from replicate executor, it needs to do so via storage table. If replicate executor and the upstream mview executor are in the same CN, their read versions will both be read by storage table. This means we will have duplicate results, which is inconsistent behavior.
    
    If we embed the replication in arrangement backfill however, it can just read its own read version via `state_table`, avoiding this issue.
    

- Noel: When to write and flush to buffer? We have to avoid writes while there’s an immutable handle on state_table.
    1. When barrier comes we can write + flush.
    2. This is okay because we read state from ***latest committed epoch.**
    3. i.e. if we write + flush, then read, our snapshot would indeed be from latest committed epoch.

## Archived

## Implementation (Stubbing uploads)

IIUC the hummock_event_handler is responsible for handling sync epoch events.

Specifically uploading on checkpoint:

```rust

if is_checkpoint {
self.uploader.start_sync_epoch(epoch);
} else {
```

- A simple first idea: Just create a new uploader whose start_sync_epoch does not actually run the upload tasks, but just drops them.
- This can be done by creating a new `HummockUploader` trait.
- Then just parameterize the hummock state store on this trait.
- Q: Does `start_sync_epoch` contain any cleanup logic of shared buffer? Or update any state of state table? If so, this approach may not work.
- Alternative #1, is it possible to completely do away with `uploader`? Maybe not. It is used to synchronize epoch which has been sealed, which is maybe needed to see when we can clear the shared buffer.
- Alternative #2, use the `ReadOnlyObjectStore` trait and stub the `upload` function. **Probably can go with this.**

Archived because it is more complicated than necessary. See above section on storage to see a simpler solution.

## Implementation (access to upstream persistence layer)

Just provide read-only access to upstream’s ObjectStore, via `ReadOnlyObjectStore` wrapper.

```rust
struct ReadOnlyObjectStore(Box<dyn ObjectStore>);

impl ObjectStore for ReadOnlyObjectStore<T> {
    async fn upload(...) -> ObjectResult<()> { Ok() }
    async fn streaming_upload(...) -> ObjectResult<BoxedStreamingUploader> {
      Ok(/* TODO */)
    }
    async fn read(...) -> ObjectResult<Bytes> {
        self.inner.read(...)
    }
    // ...
}
```

Archived because we can just use upstream state store, don't need to create a new one.