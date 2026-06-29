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


# 3. SSTable (Sorted String Table)

An **SSTable (Sorted String Table)** is an **immutable, sorted, on-disk data structure** that stores data permanently after it has been flushed from the Memtable.

It is the primary storage format used by Cassandra and most LSM-tree databases (ScyllaDB, RocksDB, LevelDB, HBase, etc.).

---

# Why Do We Need SSTables?

The Memtable resides entirely in RAM.

Although RAM provides extremely fast reads and writes, it is:

* Volatile (data is lost on power failure)
* Limited in size
* Expensive compared to disk storage

Eventually, the Memtable must be persisted to disk.

Instead of updating existing files, Cassandra writes the entire Memtable as a brand-new **SSTable**.

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
 Immutable Memtable
       │
       ▼
 Background Flush
       │
       ▼
 SSTable (Disk)
```

---

# Why is it called a "Sorted String Table"?

The name originates from Google's **Bigtable** paper.

* **Sorted** → Keys are stored in sorted order.
* **String** → Historical term referring to key-value records.
* **Table** → A persistent collection of records.

Today, the keys and values are binary serialized objects—not just strings.

---

# Key Characteristics

An SSTable is:

* Immutable
* Sorted by Partition Key
* Sequentially written to disk
* Optimized for fast reads
* Never modified after creation
* Created only by flushing a Memtable

---

# SSTable Lifecycle

```text
          Active Memtable
                 │
      (Threshold Reached)
                 │
                 ▼
       Immutable Memtable
                 │
        Background Flush
                 │
                 ▼
            SSTable-1
```

Every Memtable flush creates a **new SSTable**.

Example:

```text
Flush #1

↓

SSTable-1

Flush #2

↓

SSTable-2

Flush #3

↓

SSTable-3
```

Existing SSTables are **never updated**.

---

# Why are SSTables Immutable?

Suppose we already have:

```text
SSTable-1

user1 → John
user2 → Alice
```

A new write arrives:

```text
PUT(user1, Bob)
```

Cassandra does **not** modify SSTable-1.

Instead:

```text
New Write
      │
      ▼
Memtable
      │
      ▼
Flush
      │
      ▼
SSTable-2

user1 → Bob
```

Disk now contains:

```text
SSTable-1

user1 → John


SSTable-2

user1 → Bob
```

During reads, Cassandra compares timestamps and returns the newest value.

The obsolete version is removed later during **Compaction**.

---

# Why Immutable?

Immutability provides several advantages:

* No random disk writes
* No file locking
* No in-place updates
* Readers never block writers
* Safe concurrent access
* Easy crash recovery
* Efficient sequential writes
* Simpler compaction

---

# SSTable Creation

When the Memtable reaches its configured threshold:

1. It becomes immutable.
2. A new Active Memtable is immediately created.
3. A background thread flushes the immutable Memtable.
4. The Memtable is written sequentially to disk.
5. A brand-new SSTable is created.

Since the Memtable is already sorted, **no sorting is required during the flush.**

---

# One SSTable is NOT One File

One of the biggest misconceptions is thinking that an SSTable is a single file.

It is actually a **collection of multiple files**, each serving a specific purpose.

Example:

```text
users-aa-1-Data.db
users-aa-1-Index.db
users-aa-1-Summary.db
users-aa-1-Filter.db
users-aa-1-CompressionInfo.db
users-aa-1-Statistics.db
users-aa-1-Digest.crc32
users-aa-1-TOC.txt
```

All of these files together represent **one SSTable**.

Think of it like a Java project:

```text
Java Project

├── Main.java
├── Utils.java
├── Config.java
└── pom.xml
```

Multiple files together form one logical project.

Similarly, multiple component files together form one logical SSTable.

---

# High-Level SSTable Structure

```text
SSTable
│
├── Data.db
├── Index.db
├── Summary.db
├── Filter.db
├── CompressionInfo.db
├── Statistics.db
├── Digest.crc32
└── TOC.txt
```

---

# SSTable Components

## 1. Data.db

The primary storage file.

Contains:

* Partition data
* Rows
* Columns
* Cell values
* Timestamps
* TTL
* Tombstones

This is where the actual application data lives.

---

## 2. Index.db

Maps every Partition Key to its location inside **Data.db**.

Conceptually:

```text
Partition Key

↓

Disk Offset
```

Example:

```text
user100 → Offset 0

user200 → Offset 2048

user300 → Offset 4096
```

Allows Cassandra to jump directly into the data file without scanning it.

---

## 3. Summary.db

A sparse in-memory index built from **Index.db**.

Instead of loading the entire Index into memory, Cassandra stores every Nth entry.

Example:

```text
Index.db

user1
user2
user3
...
user1000
...
user2000
```

Summary:

```text
user1

↓

Index Position 0


user1000

↓

Index Position 1000


user2000

↓

Index Position 2000
```

This significantly reduces memory usage.

---

## 4. Filter.db

Stores the **Bloom Filter**.

Used to answer:

> "Could this SSTable contain this partition?"

Possible answers:

```text
NO

or

MAYBE
```

If the Bloom Filter returns **NO**, Cassandra skips the SSTable entirely.

This avoids unnecessary disk reads.

---

## 5. CompressionInfo.db

If Data.db is compressed, Cassandra must know:

* Which compressed block contains the requested partition.
* Where that block starts on disk.

CompressionInfo.db stores this metadata.

---

## 6. Statistics.db

Stores metadata about the SSTable.

Examples include:

* Number of partitions
* Minimum timestamp
* Maximum timestamp
* Tombstone count
* Compression ratio
* Minimum key
* Maximum key

Used by:

* Reads
* Compaction
* Repair
* Query optimization

---

## 7. Digest.crc32

Contains checksums for corruption detection.

During reads:

```text
Read Block

↓

Compute CRC32

↓

Compare

↓

Valid / Corrupted
```

Ensures data integrity.

---

## 8. TOC.txt

Table Of Contents.

Simply lists every component file that belongs to this SSTable.

Example:

```text
Data.db
Index.db
Summary.db
Filter.db
Statistics.db
CompressionInfo.db
Digest.crc32
```

---

# Complete SSTable Layout

```text
SSTable
│
├── Data.db
│      ├── Partition
│      ├── Rows
│      ├── Cells
│      └── Values
│
├── Index.db
│      └── Partition Key → Disk Offset
│
├── Summary.db
│      └── Sparse Index
│
├── Filter.db
│      └── Bloom Filter
│
├── CompressionInfo.db
│      └── Compression Metadata
│
├── Statistics.db
│      └── SSTable Statistics
│
├── Digest.crc32
│      └── Checksums
│
└── TOC.txt
       └── Component List
```

---

# SSTable Read Flow

When Cassandra receives:

```text
GET(user123)
```

The lookup follows approximately this path:

```text
Client
   │
   ▼
Memtable
   │
   ▼
Bloom Filter (Filter.db)
   │
   ▼
Summary.db
   │
   ▼
Index.db
   │
   ▼
CompressionInfo.db
   │
   ▼
Data.db
   │
   ▼
Return Row
```

Notice that Cassandra almost never scans the entire `Data.db`. Every supporting component exists to reduce disk I/O and minimize the amount of data that must be read.

---

# Why Split an SSTable into Multiple Files?

Instead of placing everything into one large file, Cassandra separates responsibilities.

| File                   | Purpose                                            |
| ---------------------- | -------------------------------------------------- |
| **Data.db**            | Stores the actual partition data                   |
| **Index.db**           | Maps Partition Keys to locations inside Data.db    |
| **Summary.db**         | Small in-memory sparse index into Index.db         |
| **Filter.db**          | Bloom Filter used to skip unnecessary SSTables     |
| **CompressionInfo.db** | Maps compressed blocks to disk offsets             |
| **Statistics.db**      | Stores metadata used for reads and compaction      |
| **Digest.crc32**       | Detects corruption using checksums                 |
| **TOC.txt**            | Lists all component files belonging to the SSTable |

---

# Key Takeaways

* SSTables are immutable, sorted files created by flushing Memtables.
* Every Memtable flush creates a brand-new SSTable.
* Existing SSTables are never modified.
* Multiple SSTables may contain different versions of the same key.
* Compaction later merges SSTables and removes obsolete data.
* An SSTable is a logical structure composed of multiple physical files, each optimized for a specific purpose in the read path.



# Data.db

`Data.db` is the **primary file inside an SSTable**. It stores the **actual application data** (partitions, rows, columns, and cell values). Every other SSTable component (`Index.db`, `Summary.db`, `Filter.db`, etc.) exists only to help Cassandra locate data inside `Data.db` efficiently.

---

## Purpose

- Stores the actual user data permanently on disk.
- Written sequentially during a Memtable flush.
- Immutable (never modified after creation).
- Optimized for sequential reads and writes.

---

## What Does Data.db Store?

Conceptually, it stores **Partitions**.

Each Partition contains:

- Partition Key
- Partition Metadata
- Rows
- Columns
- Cell Values
- Timestamps
- TTL
- Tombstones

```text
Data.db

Partition 1
│
├── Row
├── Row
└── Row

Partition 2
│
├── Row
└── Row

# 4. Index.db

`Index.db` is an **on-disk index** that maps every **Partition Key** to its corresponding location (byte offset) inside `Data.db`.

It allows Cassandra to jump directly to a partition without scanning the entire data file.

---

## Why Do We Need Index.db?

Suppose `Data.db` contains:

```text
Offset

0      Partition(user1)
500    Partition(user2)
1200   Partition(user3)
2000   Partition(user4)
```

Without an index, Cassandra would need to sequentially scan the file.

Instead, `Index.db` stores:

```text
user1 → Offset 0

user2 → Offset 500

user3 → Offset 1200

user4 → Offset 2000
```

When searching for `user3`:

```text
Partition Key
      │
      ▼
Index.db
      │
      ▼
Offset = 1200
      │
      ▼
Jump directly into Data.db
```

---

## What Does Index.db Store?

Each index entry contains:

* Partition Key
* Byte Offset inside `Data.db`

It does **not** store:

* Row data
* Column values
* Cell values

---

## How is it Created?

During a Memtable flush:

1. Cassandra writes a partition into `Data.db`.
2. Records the current file offset.
3. Stores:

```text
Partition Key → Data.db Offset
```

No second pass is required.

---

## Why Only Partition Keys?

A partition may contain thousands of rows.

Instead of indexing every row:

```text
10:00 → Offset
10:01 → Offset
10:02 → Offset
...
```

Cassandra indexes only:

```text
chat1 → Offset 5000
```

Once inside the partition, Cassandra navigates using the partition's internal row structure.

---

## Key Characteristics

| Property    | Description                            |
| ----------- | -------------------------------------- |
| Purpose     | Maps Partition Keys to Data.db offsets |
| Stores      | Partition Key + Byte Offset            |
| Mutable     | No                                     |
| Created     | During SSTable creation                |
| Used During | Every read                             |

---

# 5. Summary.db (Sparse Index)

`Summary.db` is a **small in-memory sparse index** built from `Index.db`.

Instead of loading the entire `Index.db` into RAM, Cassandra loads only **every Nth entry**.

It acts as an **index of the index**.

---

## Why Do We Need Summary.db?

Suppose `Index.db` contains:

```text
100 Million Partition Keys
```

Loading the entire file into memory would consume a huge amount of RAM.

Instead Cassandra stores:

```text
chat1

↓

Index Position 0


chat1000

↓

Index Position 1000


chat2000

↓

Index Position 2000
```

Only a small subset of entries.

---

## Read Flow

Suppose we search for:

```sql
chat1850
```

The lookup becomes:

```text
Summary.db

↓

chat1000

↓

chat2000

↓

Target lies between them

↓

Jump into Index.db

↓

Binary Search

↓

Get Data.db Offset
```

Only a small portion of `Index.db` needs to be searched.

---

## Why is it Called a Sparse Index?

Unlike `Index.db`, it does **not** store every Partition Key.

Example:

```text
Index.db

chat1
chat2
chat3
...
chat1000
...
chat2000
```

Summary.db

```text
chat1

chat1000

chat2000

chat3000
```

This significantly reduces memory consumption.

---

## Key Characteristics

| Property        | Description                                    |
| --------------- | ---------------------------------------------- |
| Purpose         | Index of Index.db                              |
| Stores          | Every Nth Partition Key + Position in Index.db |
| Loaded Into RAM | Yes                                            |
| Mutable         | No                                             |
| Used During     | Every read                                     |

---

# 6. Filter.db (Bloom Filter)

`Filter.db` stores the **Bloom Filter** for an SSTable.

A Bloom Filter is a **probabilistic data structure** that quickly determines whether a Partition Key **might exist** inside an SSTable.

It prevents unnecessary disk reads.

---

## Why Do We Need Bloom Filters?

Suppose a node has:

```text
100 SSTables
```

Without Bloom Filters:

```text
Read Request

↓

Open SSTable-1

↓

Open SSTable-2

↓

...

↓

Open SSTable-100
```

Most of those SSTables may not even contain the requested partition.

---

## Solution

Each SSTable has its own Bloom Filter.

```text
Read Request

↓

Bloom Filter

↓

Definitely NOT Present

↓

Skip Entire SSTable
```

or

```text
Bloom Filter

↓

Maybe Present

↓

Continue Reading
```

---

## Internal Structure

A Bloom Filter consists of:

* Bit Array
* Multiple Hash Functions

Example:

```text
Bit Array

0 0 0 0 0 0 0 0 0 0
```

Insert:

```text
user123
```

Hash Functions produce:

```text
2

5

8
```

Set those bits:

```text
0 0 1 0 0 1 0 0 1 0
```

---

## Lookup

Search:

```text
user500
```

Hashes:

```text
2

5

8
```

All bits are 1.

Bloom Filter returns:

```text
Maybe Present
```

Search:

```text
user999
```

Hashes:

```text
1

4

8
```

Bit 4 is 0.

Bloom Filter immediately returns:

```text
Definitely NOT Present
```

No disk access required.

---

## False Positives

Two different keys may hash to the same bit positions.

Example:

```text
user123

↓

2,5,8
```

```text
user500

↓

2,5,8
```

Bloom Filter reports:

```text
Maybe Present
```

even though `user500` was never inserted.

This is called a **False Positive**.

---

## False Negatives?

Never.

If any required bit is 0:

```text
Hash1 → 2 → 1

Hash2 → 5 → 0

Hash3 → 8 → 1
```

The key definitely does not exist.

Therefore:

* False Positives → Possible
* False Negatives → Impossible

---

## Read Flow

```text
Client

↓

Bloom Filter

↓

NO

↓

Skip SSTable


OR


↓

MAYBE

↓

Summary.db

↓

Index.db

↓

Data.db

↓

Read Partition
```

---

## Key Characteristics

| Property        | Description                                                    |
| --------------- | -------------------------------------------------------------- |
| Purpose         | Quickly determine whether an SSTable might contain a partition |
| Data Structure  | Bit Array + Multiple Hash Functions                            |
| Answers         | Definitely NOT Present / Maybe Present                         |
| False Positives | Yes                                                            |
| False Negatives | Never                                                          |
| Built           | During SSTable creation                                        |
| Mutable         | No                                                             |

