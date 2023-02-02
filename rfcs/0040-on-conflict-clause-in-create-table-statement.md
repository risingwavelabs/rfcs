---
feature: ON CONFLICT Clause in Create Table Statement
authors:
  - "st1page"
start_date: "2023/2/1"
---

# ON CONFLICT Clause in Create Table Statement

## Summary

allow user to declare the conflict behavior when the newly inserted row break the unique constraint of primary key.

## Motivation

Currently, for a table with primiary key, the later operations will always overwrite the previous ones when the new operation [0017-DML.md](0017-DML.md)
But users' requirement are various here.

#### Overwrite

The most common needs of using the Overwrite behavior is to express "Upsert" operation.  

#### Ignore

### Design

Explain the feature in detail.

## Unresolved questions

* Are there some questions that haven't been resolved in the RFC?
* Can they be resolved in some future RFCs?
* Move some meaningful comments to here.

## Alternatives

What other designs have been considered and what is the rationale for not choosing them?

## Future possibilities

Some potential extensions or optimizations can be done in the future based on the RFC.
