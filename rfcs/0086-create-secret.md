---
feature: New Syntax Create Secret
authors:
  - "TabVersion"
start_date: "2024/03/18"
---

# Introducing SECRETs for Secure Storage of Sensitive Information

## Background

This RFC proposes a new syntax and implementation for securely storing and referencing sensitive information,
such as database passwords and API keys, in SQL statements using a SECRET catalog.

The primary goals are to enhance the security of production environments by encrypting sensitive data, applying access controls, 
and preventing abuse of API keys by developers.

## Proposed Syntax

```sql
-- create secret
create secret <secret-name> from <expr>
```

We always store SECRETs as bytea type, which maximally ensures that the information we store can adapt to different usage requirements.

```sql
-- use secret
create table t ( ... )
with (
    ...
    properties.sasl.password = s-<secret-name>
)
```

### Usage Scenarios

There are two main scenarios for using SECRETs:

* Direct value usage (e.g., `properties.sasl.password`): The SECRET needs to be correctly decrypted in the meta and passed along with the table fragment to the connector.
* File path usage (e.g., `properties.ssl.certificate.location`): The Kafka SDK requires a file path, so a temporary file needs to be written on the CN, and the path is passed to the Kafka SDK.

## Implementation

SECRETs are always stored as bytea type in SQL to ensure flexibility in adapting to different usage requirements.
If a user needs to use a certificate, its contents are also stored as bytea type and distributed to Compute Nodes (CNs) as needed.

To control storage consumption on meta nodes, the maximum length of a single SECRET is limited to 1MB.

### Security Considerations

* Access Control: Administrators can create SECRETs and allow other developers to reference them without exposing the actual API keys or sensitive information. 
  This prevents abuse of API keys by developers.
* Encryption: Storing SECRETs in meta nodes is unavoidable, but encryption will be applied to protect the sensitive information. 
  SECRETs will never be displayed in plain text, even to administrators.
* Updating SECRETs: In the future, an ALTER SECRET command will be supported to allow overwriting existing SECRETs without revealing the original value.

### Backward Compatibility

The proposed changes do not break backward compatibility, as the new syntax and implementation are additive.

### Performance Impact
The performance impact is expected to be minimal, as the decryption and file writing operations are lightweight.
The storage consumption on meta nodes is controlled by limiting the maximum length of a single SECRET.
