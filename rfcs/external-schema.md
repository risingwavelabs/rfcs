---
feature: Define External Schema
authors:
  - "TabVersion"
start_date: "2024/03/30"
---

# Define External Schema

## Background

在之前的实现中，我们保存了访问外部表的方式，例如 schema file 的位置和对应的访问密钥，没有将 schema 信息存在元数据中，而是在使用让组件来独立拉取 schema 信息。
这样造成了我们在处理 schema 信息时，需要在多个地方进行处理，不利于统一管理也可能造成 schema 不一致的情况。

除了使用 schema registry 外，用户也常常使用文件的方式来保存 schema 信息，我们花费了很多努力来让集群可以访问到这些文件，包括访问本地的文件系统和 s3 等云存储。
这对系统带来了很多额外的复杂度，但是对用户体验上乏善可陈。并且我们并不感知用户的 schema 文件是否发生了变化，这可能导致我们的数据处理出现问题。

所以我建议我们应该在 meta store 中保存 schema 信息，这样我们可以有效管理 schema 的版本，保证分发的 schema 信息是一致的。
请注意，这篇 RFC 中只处理 schema 的定义（作为各个组件独立从远端拉取文件的替代），不处理 schema 的版本管理。

## Proposed Syntax

* 以文本的形式保存 schema 信息，允许定义 json 或者 avro schema，我们可以轻量的编译这些 schema 信息。

```sql
create external schema <schema-name>
    as <contents in bytes>
```



```sql
create external schema <schema-name> 
```
