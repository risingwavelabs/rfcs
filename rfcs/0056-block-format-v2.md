---
feature: Block Format V2
authors:
  - "Li0k"
start_date: "2023/02/27"
---

# **Background**

### Key plain

```rust
| `table_id`(4B) | `table_key` |`epoch`(8B)
```

### Block plain

```rust
/// | entries | restart point 0 (4B) | ... | restart point N-1 (4B) | N (4B) 
/// | compression method (1B) | crc32sum (4B) |
```

### Entry plain

```rust
/// For diff len < MAX_KEY_LEN (65536)
/// entry (kv pair): | overlap len (2B) | diff len (2B) | value len(4B) | diff key | value |

/// For diff len >= MAX_KEY_LEN (65536)
/// entry (kv pair): | overlap len (2B) | MAX_KEY_LEN (2B) | diff len (4B) | value len(4B) | diff key | value |

/// [`KeyPrefix`] contains info for prefix compression.
#[derive(Debug)]
pub struct KeyPrefix {
    overlap: usize,
    diff: usize,
    value: usize,
    /// Used for calculating range, won't be encoded.
    offset: usize,
}

impl KeyPrefix {
    pub fn encode(&self, buf: &mut impl BufMut) {
        buf.put_u16(self.overlap as u16);
        buf.put_u16(self.diff as u16);
        buf.put_u32(self.value as u32);
    }
}
```

`BlockBuilder` generates blocks and implements prefix compression on keys via `KeyPrefix`. When we store a key, we drop the prefix shared with the previous string. This helps reduce the space requirement significantly. However, in the user scenario, u16 cannot handle all cases and we need to find a new solution

# **Motivation**

Design a new format that achieves the following two goals as much as possible

1. flexible length encoding
2. achieve better compression

# **Design**

## Flexible encoding

The obvious way is to change the encoding type of length from u16 to u32 so that internal keys larger than 4GB can be handled. This approach is standard and easy to implement, but it is somewhat wasteful in terms of storage. (from 2bytes -> 4bytes) In the existing user scenario, the percentage of large-length keys is not high, so we can consider introducing a more flexible variant type to solve the above problem.

```rust
/// [`KeyPrefix`] contains info for prefix compression.
#[derive(Debug)]
pub struct KeyPrefix {
    overlap: Variant,
    diff: Variant,
    value_len: Variant,
    /// Used for calculating range, won't be encoded.
    offset: usize,
}

impl KeyPrefix {
    pub fn encode(&self, buf: &mut impl BufMut) {
        self.overlap.put_variant(buf);
        self.diff.put_variant(buf);
        buf.put_variant(self.value);
    }
}
```

## compression

```rust
/// Key plain
/// | `table_id`(4B) | `table_key` |`epoch`(8B)
```

For compression, we can find that in the existing key encoding, `TableId` and `Epoch` are two attributes that already exist in the key encoding and are easily duplicated. `TableId` can be reduced by `KeyPrefix`, while `Epoch` is a suffix in encoding and cannot benefit from `KeyPrefix`. Therefore, the above two attributes can be compressed by other means.

### TableId

In the current implementation, the block is already split by table_id, and from a data compression point of view, table_id only needs to be stored in BlockMeta without encoding in the key.

### Epoch

`Epoch`(physical_time(48)|seq(16)) indicates `physical_time` and is localized in the block

In `Epoch`(physical_time(48)|seq(16)) encoding, `physical_time` is in the high 48 bits. In our system, the key is generated with the epoch, which means that the key in the block is localized and the number of epochs involved is not large. To address this issue, we can introduce `Dictionary Encoding` for `Epoch`.

![Untitled](Design%20Block%20Format%20V2%20cc022efbc09d4768b57e384df6f875df/Untitled.png)

minor: we can also remove the epoch of the key that is lower than the watermark in the version because it is the oldest key in the storage and does not need to care about the epoch.

## **Concern**

1. Does the above encoding and compression bring enough benefit and does it depend on the compression algorithm enough?
2. Does complex structure bring complex coding process？