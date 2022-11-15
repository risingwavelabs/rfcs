---
feature: backup-and-restore
authors:
  - "zwang28"
start_date: "2022/11/14"
---

# Backup and Restore

## Motivation
Support backup and restore to protect against data loss, due to user error, software error, disaster scenario, etc.

## Design

A RisingWave cluster's persistent data includes metadata stored in meta store, and SSTable stored in object store. We back up these types of data differently:
- For metadata, we back up a metadata snapshot. See [Metadata Snapshot](#metadata-snapshot).
- For SSTable, we decide **not to replicate object store data** during backup. Instead, we only back up a SSTable manifest, which references to the same SSTable as the running cluster. See [SSTable Maintenance](#sstable-maintenance).
- In addition, to support point-in-time recovery(PITR), we also back up metadata changes. See [Point-in-Time Recovery](#point-in-time-recovery).

![](https://user-images.githubusercontent.com/70626450/201866284-923db172-89ff-423d-95e0-0e93bcde23da.png)

### Metadata Snapshot
We leverage meta store's snapshot read to get a consistent cut of all interested metadata. Specifically with etcd meta store, we read using the same etcd revision during backing up.

### SSTable Maintenance
Now that SSTables are shared between the running cluster and backups, a SSTable shouldn't be deleted until it's not referenced by any backup's SSTable manifest. Specifically, a running cluster is aware of all available backups. When trying to delete stale SSTables, it filters out candidate SSTables by checking all backup's SSTable manifest.

### Point-in-Time Recovery
For metadata, we need capture its change so that we can bring it to any point in time within history retention period. Let's consider 4 categories of metadata respectively:
1. Hummock metadata. It already has write-ahead log.
2. Table catalog. It will be multiple versioned.
3. Monotonic ones, e.g. id generator offset, last barrier. Using a later value in earlier point-in-time restoration doesn't compromise correctness.
4. Other, e.g. user info.

We can coordinate the first 3 categories during restoration to result a consistent state, for example aligning them with epoch. However, the 4th category cannot be handled out-of-box. So I think we need implement a unified metadata WAL.

For SSTable, we need ensure any SSTable referenced to is available within history retention period. It's achieved by pinning SSTables based on SSTable manifest change, similar to how we do in [SSTable Maintenance](#sstable-maintenance) based on SSTable manifest.

## Unresolved questions
If we all agree to implement a unified metadata WAL as required by PITR, we then need another doc to detail its design. Before that, PITR is not supported.

## Alternatives
I've also considered to leverage etcd's inherent features to ease implementation. Specifically, since etcd already supports [disaster recovery](https://etcd.io/docs/v3.5/op-guide/recovery/), we use it directly without intervening RisingWave cluster. To support PITR, we can use [etcd watch](https://etcd.io/docs/v3.5/dev-guide/interacting_v3/#watch-historical-changes-of-keys) to capture data change.  
The downside is, backup and restore is deeply coupled with etcd, which likely requires much refactor if we change the underlying meta store in the future. Besides, because the change log is at etcd KV level, it's meaningless until applied to a snapshot. However meta node does want to extract some metadata change from log directly, e.g. SSTable manifest change,
