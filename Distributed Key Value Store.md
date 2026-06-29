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

* Keyspace Name - like DB Name
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

# 2. Memtable

The **Memtable** is an in-memory, sorted data structure that stores the latest write operations (**INSERT**, **UPDATE**, **DELETE**) before they are permanently flushed to disk as an SSTable.

It acts as a **write buffer** between the Commit Log and SSTables.

---

# Why Do We Need a Memtable?

Traditional **B-Tree** databases update data directly on disk.

Every write typically requires:

1. Finding the correct page.
2. Loading the page into memory.
3. Modifying it.
4. Writing the page back.

This results in **random disk I/O**, which is relatively slow.

```text
Write
   │
Find Disk Page
   │
Modify Page
   │
Random Disk Write
```

LSM-based databases (Cassandra, RocksDB, ScyllaDB, LevelDB) avoid this problem.

Instead, writes are first accumulated in an in-memory buffer called the **Memtable**.

Memory writes are several orders of magnitude faster than random disk writes.

---

# Memtable in the Write Path

After a Mutation has been successfully appended to the Commit Log (WAL), the same Mutation is inserted into the **Active Memtable**.

```text
Incoming Write
      │
      ▼
Commit Log (Durability)
      │
      ▼
Active Memtable (RAM)
      │
      ▼
Client ACK
```

The Commit Log guarantees **durability**, while the Memtable provides **fast writes and reads**.

---

# Memtable Lifecycle

A Memtable goes through three stages during its lifetime.

```text
                 Incoming Write
                       │
                       ▼
              Active Memtable
                       │
        (Size Threshold Reached)
                       │
                       ▼
            Immutable Memtable
                       │
             Background Flush
                       │
                       ▼
                  SSTable
```

### Stage 1 - Active Memtable

* Accepts all new writes.
* Maintains data in sorted order.
* Supports fast lookups.
* Lives entirely in RAM.

---

### Stage 2 - Immutable Memtable

Once the configured memory threshold is reached (for example, 32 MB, 256 MB, etc.), Cassandra:

* Marks the Memtable as **Immutable (Frozen)**.
* Stops accepting new writes.
* Creates a brand-new Active Memtable immediately.

```text
                 RAM

 Active Memtable (New)
          ▲
          │
     New Writes


 Immutable Memtable
          │
          ▼
   Waiting for Flush
```

This ensures writes continue without interruption.

---

### Stage 3 - Flush

A background thread sequentially writes the immutable Memtable to disk.

This process is called a **Flush**.

The flush creates a brand-new **SSTable**.

**Every Memtable flush creates a new SSTable.**

Existing SSTables are **never modified**.

---

# Why Create a New SSTable?

Suppose we already have:

```text
SSTable-1

user1
user2
user3
```

A new write arrives:

```text
user0
```

Appending it would break the sorted order.

```text
user1
user2
user3
user0
```

Instead, Cassandra creates another SSTable.

```text
SSTable-2

user0
```

Later, **Compaction** merges the two SSTables into a larger sorted SSTable.

---

# Internal Data Structures

A Memtable is not simply an array or a HashMap.

It must support:

* Fast insertions
* Fast lookups
* Sorted iteration
* Concurrent writes

Depending on the storage engine, different data structures are used.

| Data Structure | Used By                                        | Advantages                                                             |
| -------------- | ---------------------------------------------- | ---------------------------------------------------------------------- |
| Skip List      | RocksDB, LevelDB, Cassandra (earlier versions) | Lock-friendly, sorted, expected O(log n) operations                    |
| Trie           | Cassandra 5.0+                                 | Better memory efficiency, faster prefix lookups, lower memory overhead |
| Red-Black Tree | Some LSM implementations                       | Balanced tree with guaranteed O(log n) operations                      |
| B-Tree         | Some storage engines                           | Cache-friendly memory layout                                           |

---

# Skip List

A **Skip List** is a probabilistic multi-level linked list.

Instead of traversing every node one-by-one like a normal linked list, higher levels contain **express lanes** that allow searches to skip large portions of the list.

Example:

```text
Level 3

A -------------------------- H

Level 2

A -------- D -------- H

Level 1

A --- B --- C --- D --- E --- F --- G --- H
```

Searching for **G**:

```text
A
↓

Jump to D
↓

Jump to H (too far)

↓

Move back to D

↓

Traverse

E → F → G
```

Average complexity:

| Operation | Skip List         |
| --------- | ----------------- |
| Search    | O(log n)          |
| Insert    | O(log n) expected |
| Delete    | O(log n) expected |

Unlike balanced trees, Skip Lists are simpler to implement and support high concurrency.

---

# Trie (Cassandra 5.0)

Modern Cassandra versions use **Trie-indexed Memtables** by default.

A Trie stores keys character-by-character.

Example:

```text
          root
         /    \
        a      b
       /        \
      p          o
     /            \
    p              b
```

Advantages:

* Better memory utilization
* Faster lookups for common prefixes
* Reduced memory fragmentation
* Improved cache locality

---

# What Does the Memtable Actually Store?

Conceptually, a Memtable stores the **latest state** of every key (or partition in Cassandra).

For a simple Key-Value Store:

```text
user1 → John

user2 → Alice

user3 → Bob
```

For Cassandra:

```text
Partition Key

↓

Partition

↓

Rows

↓

Columns
```

A Memtable therefore stores much richer structures than simple key-value pairs.

---

# Data Representation

Every entry inside the Memtable contains more than just a key and value.

Conceptually:

| Component       | Description                                     | Example           |
| --------------- | ----------------------------------------------- | ----------------- |
| User Key        | Application-level key                           | `user123`         |
| Timestamp       | Version used for conflict resolution            | `1719694400`      |
| Value           | Serialized application data                     | `{name: "Alice"}` |
| Tombstone Flag  | Indicates whether the entry represents a delete | `true` / `false`  |
| TTL             | Time-to-Live (optional)                         | `3600 seconds`    |
| Expiration Time | Absolute expiration timestamp                   | `1719698000`      |

Notice that the Memtable stores the **latest version** of a key.

Suppose:

```text
PUT(user1, John)
```

followed by

```text
PUT(user1, Alice)
```

The Memtable contains:

```text
user1

↓

Alice
```

The older version may still exist inside an older SSTable.

During reads, Cassandra compares timestamps and returns the newest value.

---

# Memory Layout (Conceptual)

```text
                RAM

+------------------------------------------+
| Active Memtable                          |
|------------------------------------------|
| user100 → Alice                          |
| user200 → Bob                            |
| user300 → Charlie                        |
+------------------------------------------+

             │ Flush Trigger
             ▼

+------------------------------------------+
| Immutable Memtable                       |
+------------------------------------------+

             │ Background Flush
             ▼

               Disk

+------------------------------------------+
| SSTable                                  |
+------------------------------------------+
```

---

# Key Characteristics

* Entirely in memory.
* Stores the latest writes.
* Always maintained in sorted order.
* Supports efficient reads before data reaches disk.
* Never flushed in-place.
* Once frozen, it becomes immutable.
* Every flush creates a brand-new SSTable.
* Existing SSTables are never modified.
* Background flushing ensures writes never pause.




