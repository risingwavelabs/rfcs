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

Currently, for a table with primary key, the later operations will always overwrite the previous ones when the new operation [0017-DML.md](0017-DML.md)
But users' requirement are various here.
I will give some different behavior here. Notice that here the conflict behavior is not just happens when an insert DML statement happening but also in a table with external connector.

- **Overwrite**
  The most important needs of using the Overwrite behavior is to express "Upsert" operation. Some connector can not give the old value of the update operation and we need to overwrite the old value with the same pk in storage.

- **Ignore**
  The Ignore behavior make user can define a "append-only table with primary key" which maintain the primary constrain and also provide a append-only changes to downstream with good performance. This is good to express a table with an At-least-once Delivery connector where a same record can be replayed multiple times to break the primary key constrain.
  Notice: the specific gammar for this has not designed(e.g. should we expose append-only table to user?) but the situation is the same.

- **More... express reduce function or UDAF?**
  In fact, ON CONFLICT Clause can make user be able to do more things. User can even construct a streaming Map-reduce job with that where the primary key define the map fields and ON_CONFLICT clause define the reduce logic.
  For example.

  ```SQL
    CREATE table t(k int primary key, int cnt) 
    ON CONFLICT DO UPDATE SET cnt = t.cnt + EXCLUDED.cnt

    Insert INTO t VALUES (1, 2);
    Insert INTO t VALUES (1, 3);

    dev=# select * from t;
     k | cnt 
    ---+-----
     1 |   5
    (1 row)
  ```

  The most strange part is that the input parameter and return value of the "reduce function" must have the same schema. But it can be easily workaround with constant expressions.
  
### Design

#### Syntax and Semantics

```SQL
  CREATE TABLE [ IF NOT EXISTS ] table_name (
    col_name data_type [ PRIMARY KEY ],
    ...
    [ PRIMARY KEY (col_name, ... ) ]
  ) ON CONFLICT conflict_action;

  where conflict_target can be one of:
    DO NOTHING
    DO OVERWRITE
    DO UPDATE SET { column_name = { expression | DEFAULT } |
                    ( column_name [, ...] ) = [ ROW ] ( { expression | DEFAULT } [, ...] ) |
                  } [, ...]
              [ WHERE condition ]
```

I will compare it with the PG's ON CONFLICT clause(<https://www.postgresql.org/docs/current/sql-insert.html>) in the following text.

```SQL
INSERT INTO table_name [ AS alias ] [ ( column_name [, ...] ) ]
    [ ON CONFLICT [ conflict_target ] conflict_action ]

where conflict_target can be one of:
    ( { index_column_name | ( index_expression ) } [ COLLATE collation ] [ opclass ] [, ...] ) [ WHERE index_predicate ]
    ON CONSTRAINT constraint_name

and conflict_action is one of:
    DO NOTHING
    DO UPDATE SET { column_name = { expression | DEFAULT } |
                    ( column_name [, ...] ) = [ ROW ] ( { expression | DEFAULT } [, ...] ) |
                    ( column_name [, ...] ) = ( sub-SELECT )
                  } [, ...]
              [ WHERE condition ]

```

## Unresolved questions

- Are there some questions that haven't been resolved in the RFC?
- Can they be resolved in some future RFCs?
- Move some meaningful comments to here.

## Alternatives

What other designs have been considered and what is the rationale for not choosing them?

## Future possibilities

Some potential extensions or optimizations can be done in the future based on the RFC.
