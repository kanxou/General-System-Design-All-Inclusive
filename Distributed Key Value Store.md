# Cassandra Local Write Path

Reference:
https://github.com/kanxou/General-System-Design-All-Inclusive/blob/main/Distributed%20System%20Fundamentals.md

---

# How Cassandra Does It

## Key Components

### 0. Mutation

Unlike a traditional application that thinks in terms of "set this variable," Cassandra thinks in terms of **Mutations**.

A client request such as:

```text
PUT(user123, "John")
```

is internally converted into a Mutation object:

```java
Mutation {
    key = "user123"
    value = "John"
    timestamp = 1719581011000
    ttl = null
}
```

A Mutation contains much more than just the key and value. It also stores metadata required for replication and conflict resolution, including:

* Partition Key
* Timestamp
* TTL (if specified)
* Deletion/Tombstone information
* Table metadata
* Other internal metadata

### Multiple Versions

Suppose another request arrives:

```text
PUT(user123, "Alice")
```

This also becomes a completely new Mutation.

```java
Mutation {
    key = "user123"
    value = "Alice"
    timestamp = 1719581025000
}
```

**Important:**

* The old value is **not immediately overwritten on disk.**
* Cassandra follows an **append-only** storage model.
* Multiple versions of the same key can temporarily exist simultaneously.
* During reads, the mutation with the **latest timestamp** wins.
* Older versions are removed later during **Compaction**.

---

# 1. Commit Log (Write-Ahead Log / WAL)

The **Commit Log** is the first durable write.

Before updating the Memtable, Cassandra appends the mutation to the Commit Log to guarantee durability.

Initially:

```text
CommitLog-1.log

(empty)
```

After several writes:

```text
Offset      Record

0           PUT user123 John
48          PUT user555 Alice
96          DELETE item900
```

## Characteristics

* Append-only
* Sequential disk writes
* Never modified in-place
* Extremely fast due to sequential I/O

Internally it simply performs:

```text
append()
append()
append()
append()
```

There is **no searching**, **no sorting**, and **no rewriting**.

---

# Commit Log Segments

The Commit Log is divided into multiple **Segments**.

A Segment is a physical file on disk.

By default, a segment size is approximately **32 MB** (configurable).

Example:

```text
CommitLog-1.log
CommitLog-2.log
CommitLog-3.log
```

When one segment becomes full, Cassandra creates a new segment.

Old segments are deleted only after every mutation contained within them has been safely flushed into SSTables.

---

# High-Level Commit Log Structure

```text
Commit Log
│
├── Segment 1
├── Segment 2
├── Segment 3
└── ...
```

Each Segment consists of:

```text
+-------------------------------------------------------------+
| Segment Header                                               |
+-------------------------------------------------------------+
| Frame 1                                                      |
+-------------------------------------------------------------+
| Frame 2                                                      |
+-------------------------------------------------------------+
| Frame 3                                                      |
+-------------------------------------------------------------+
```

---

# Segment Header

Every segment begins with a descriptor/header.

Typical contents include:

### Format Version

Specifies the serialization format used by this Commit Log version.

This allows newer Cassandra versions to correctly interpret older log files.

---

### Segment ID

A unique, monotonically increasing **64-bit integer** identifying the Commit Log segment.

Example:

```text
Segment ID = 348912381
```

---

### Parameters Block

Stores Commit Log configuration information such as:

* Compression type
* Encryption settings
* Other serialization parameters

---

# Frames

A **Frame** is the fundamental physical storage unit inside a Commit Log segment.

Instead of writing every Mutation directly to disk one-by-one, Cassandra groups multiple mutations into a Frame.

Therefore:

```text
Segment
    ├── Frame
    │      ├── Mutation
    │      ├── Mutation
    │      ├── Mutation
    │      └── Mutation
    │
    ├── Frame
    │      ├── Mutation
    │      ├── Mutation
    │      └── Mutation
    │
    └── Frame
```

Think of it as:

* **Segment** = Physical file on disk.
* **Frame** = Storage block inside that file.
* **Mutation** = Individual write record.

---

# Mutation Record Layout

Each Mutation stored inside a Frame contains several fields.

## 1. Size Descriptor & Metadata

```text
Mutation Length (4 bytes)
Timestamp (8 bytes)
```

The size allows Cassandra to know exactly where the next Mutation begins.

---

## 2. Serialized Mutation Payload

The payload contains the actual write information.

Typical fields include:

* Keyspace Name
* Partition Key (byte array)
* Table (Column Family) ID
* Row modifications
* Column Definition ID
* Cell Value (BLOB)
* Cell Flags

  * TTL
  * Tombstone
  * Expiration metadata

Conceptually:

```text
Mutation
│
├── Keyspace
├── Partition Key
├── Table ID
├── Row Updates
│      ├── Column A
│      ├── Column B
│      └── Column C
└── Metadata
```

---

# Record Trailer

Every Mutation ends with a **CRC32 checksum**.

```text
+----------------------------+
| Serialized Mutation Data   |
+----------------------------+
| CRC32 (4 Bytes)            |
+----------------------------+
```

The checksum is computed over the serialized mutation bytes.

---

# Why CRC32?

Suppose the server crashes while writing the Commit Log.

Possible result:

```text
PUT user123 John XXXXXXXX
```

The write may be only partially written.

During startup, Cassandra sequentially scans the Commit Log.

For every Mutation it:

1. Reads the serialized payload.
2. Computes a new CRC32 checksum.
3. Compares it with the stored checksum.

If they differ:

* The record is considered corrupted.
* Cassandra stops processing that portion of the Commit Log.
* This prevents replaying partially written or corrupted mutations and protects data integrity.

---

# Commit Log Hierarchy

```text
Commit Log
│
├── Segment
│     │
│     ├── Header
│     │      ├── Version
│     │      ├── Segment ID
│     │      └── Parameters
│     │
│     ├── Frame
│     │      ├── Mutation
│     │      ├── Mutation
│     │      └── Mutation
│     │
│     ├── Frame
│     │      └── ...
│     │
│     └── ...
│
├── Segment
└── Segment
```
