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
  Notice: the specific grammar for this has not designed(e.g. should we expose append-only table to user?) but the situation is the same.

- **Partial Update**
  The ON CONFLICT make users can do partial update on the table.

  ```SQL
    CREATE table t(k int primary key, int a, int b) 
    ON CONFLICT DO UPDATE SET (a,b) = (
      CASE when EXCLUDED.a is not null then EXCLUDED.a else a end,
      CASE when EXCLUDED.b is not null then EXCLUDED.b else b end);
  ```
  
  This feature will be useful when user use wide-table schema data model.

- **Express Reduce Function or UDAF?**
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

#### Syntax and semantics

```SQL
  CREATE TABLE [ IF NOT EXISTS ] table_name (
    col_name data_type [ PRIMARY KEY ],
    ...
    [ PRIMARY KEY (col_name, ... ) ]
  ) ON CONFLICT conflict_action;

  where conflict_target can be one of:
    DO NOTHING
    DO OVERWRITE
    DO UPDATE SET { column_name = expression |
                    ( column_name [, ...] ) = [ ROW ] ( expression[, ...] )
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

- **Defined when `CREATE TABLE` instead of `INSERT`**
  Why:
  1. Different with PG, the conflicts are not only happens in `INSERT` statement but also on the data from external source.
  2. In PG, users can define different behavior on different `INSERT` statement. We can not implement it easily because RW treat DML statement with just one `Materialize` operator as a streaming job.
  
  By the way, for the `UPDATE` statement, we can just override the old value and it can not break the PK constrain after we ban the update on PK columns <https://github.com/risingwavelabs/risingwave/issues/7710>.

- **Not specify `conflict_target`**
  In RW, we only have PK constrain, so we can just do check on PK constrain. Also if the table does not have pk, user can not specify the ON CONFLICT clause.

- **DO OVERWRITE behavior**
  just a Syntactic sugar because it is a usual usage. It has the same meaning with `DO UPDATE SET (a, b, c, ...) = (excluded.a, excluded.b, excluded.c, ...)`

- **Sub-query/sub-SELECT in UPDATE**
  It is too complex and we need implement stream join plan for the `CREATE TABLE`. So I think we need to put it on hold.

### Future possibilities

  We can support the `ON CONFLICT` clause in the Insert statement if we support attach the on conflict description for the chunk in `BatchInsert` executor and `StreamDML` executor to let `Materialize` executor know how to handle the conflict. But the on conflict clause is always needed in create table statement for the table with connector.

## Consensus and Conclusion

- [] We will support `ON CONFLICT` Clause in `CREATE TABLE` with `OVERRIDE` and `DO NOTHING` behavior.
- [] We will supprot `DO UPDATE` in `ON CONFLICT` Clause
