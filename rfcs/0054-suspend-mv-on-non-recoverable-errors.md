---
feature: Suspend MV on Non-Recoverable Errors
authors:
  - "Patrick Huang"
start_date: "2023/2/24"
---
# RFC: Suspend MV on Non-Recoverable Errors

# Motivation

Currently we have two error handling mechanisms on MV:

1. Ignore and tolerate the compute error. Relevant issue: https://github.com/risingwavelabs/risingwave/issues/4625)
2. Report error to meta and trigger a full recovery.

However, when there are non-recoverable errors caused by malform records ingested by the user, the current error handling mechanisms are insufficient:

1. Even if these messages can be skip or fixed via filling in default values, correctness can be affected, especially when sinking to external systems, since the damage may hardly get reverted.
2. Recovery won’t help here. We may end up with re-consuming the malform records and trigger full recovery over and over again. All MVs in the cluster will be affected and no progress can be made.

This RFC proposes that we should suspend the affected MV when encountering non-recovery errors. Non-recoverable errors by definition cannot be fixed automatically so manual intervention and re-creating the MV are unavoidable. The goals of this RFC are:

- Reduce the radius of impact when non-recoverable errors happen. In other words, non-relevant MVs should not be affected.
- Improve observability when non-recoverable errors happen. In other words, MV progress and necessary states should be kept to help user/SRE identify the issue.

# Design

## Non-Recoverable Errors

- Errors caused by malform records ingested by the user source. Examples: record too large, record parsing error, record format/type mismatch.
- Errors caused by failure in sanity check. Examples: state table insert/update/delete sanity check fails.

## Suspend MV on Non-Recoverable Errors

Error MV and all its downstream MVs are suspended and removed from the streaming graph for checkpointing. Suspended MV and its internal states are still queryrable (at least for SRE). You can treat the suspended MV as a dropped but not GCed MV.

Detail steps on MV suspension:

1. Actor reports non-recoverable errors to meta node.
2. Meta node identifies the MV of the failed actor as well as all its downstream MVs.
3. Meta node suspends these MVs by removing them from the streaming graph.
4. A suspended MV will be deleted manually or automatically TTLed.
    
    

### DISUCSSION: How to suspend the MV?

1. Trigger a recovery: all ongoing concurrent checkpoint fails and actors of the suspended MV won’t get rebuilt during recovery.
2. No recovery triggered: remove the suspended MV from the streaming graph immediately or in the next checkpoint. (Is this possible? e.g. failure in downstream MVs may cause backpressure in upstream and can we preempt backpressure?) 

Personally prefer a > b.

### DISCUSSION: What happen when user queries the suspended MV?

1. Ban user query and throw an error.
2. Allow users to query but remind them there is an error and the MV will not longer be updated.

Personally prefer a > b. 

## Future possibilities

- We can store the error message, stack trace as well as other detailed information of the suspended MV in a user queryable system table.
-