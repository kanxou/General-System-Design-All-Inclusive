
## Problem Statement

In distributed systems, we need a mechanism to generate unique identifiers across multiple independent servers.

An ideal ID generation system should provide:

| Property          | Description                       |
| ----------------- | --------------------------------- |
| Uniqueness        | No duplicate IDs                  |
| Scalability       | Works across many servers         |
| High Throughput   | Can generate millions of IDs      |
| Ordering          | IDs roughly reflect creation time |
| Small Size        | Efficient storage and indexing    |
| Availability      | Continues working during failures |
| Low Coordination  | Minimal central dependency        |
| Human Readability | Easy to inspect and debug         |

There are two broad approaches:

1. Centralized ID generation
2. Decentralized (serverless) ID generation

---

# 1. Centralized ID Generation

## Flickr's Ticket Server Architecture

Before Twitter Snowflake existed, Flickr solved distributed ID generation using dedicated ticket servers.

### Why Didn't Flickr Use GUIDs?

At that time, GUIDs (UUIDs) were considered too large.

UUID:

* 128 bits
* 16 bytes

Large primary keys increase:

* Index size
* Memory consumption
* Disk I/O

Flickr used MySQL and wanted primary key indexes small enough to fit entirely in RAM.

If indexes don't fit in memory:

* More disk reads occur
* More disk writes occur
* Query performance degrades

### Solution

Flickr created dedicated servers whose only responsibility was handing out IDs.

Example:

**Ticket Server 1**

```text
1, 3, 5, 7, 9...
```

**Ticket Server 2**

```text
2, 4, 6, 8, 10...
```

Uniqueness was guaranteed while keeping IDs compact.

### Pros

* Simple
* Small integer IDs
* Sequential ordering

### Cons

* Central dependency
* Single point of operational complexity
* Limited scalability

---

# 2. Range-Based ID Generation

Instead of requesting every ID from a central service, servers request a range of IDs.

### Example

Server A:

```text
1 - 1,000,000
```

Server B:

```text
1,000,001 - 2,000,000
```

Server C:

```text
2,000,001 - 3,000,000
```

Each server generates IDs locally until its range is exhausted.

### Pros

* Massive throughput
* Compact 64-bit IDs
* Excellent database performance
* Minimal coordination

### Cons

#### ID Gaps

Suppose Server A receives:

```text
1 - 1,000,000
```

but crashes after generating:

```text
1 - 500,000
```

The remaining IDs are lost forever.

#### Latency Spikes

When a range is exhausted:

1. Application pauses
2. Requests a new range
3. Waits for allocation

This causes occasional latency spikes.

---

# 3. Multi-Master Database Coordination

## Increment With Offsets

Multiple database servers independently generate IDs using different starting values.

### Example

Database Server 1

```text
Starts at 1
Increment by 3

1, 4, 7, 10...
```

Database Server 2

```text
Starts at 2
Increment by 3

2, 5, 8, 11...
```

Database Server 3

```text
Starts at 3
Increment by 3

3, 6, 9, 12...
```

### Pros

* Simple
* Sequential IDs on each server
* Easy to understand

### Cons

#### Poor Horizontal Scaling

Adding a new database server changes the increment strategy.

Example:

```text
Old increment = 3
New increment = 4
```

Operationally difficult.

#### No True Global Ordering

Consider actual insertion order:

```text
Server 2:
2, 5, 8

Server 1:
1, 4

Server 3:
3, 6
```

Observed creation order:

```text
2, 5, 8, 1, 4, 3, 6
```

Numerically sorted:

```text
1, 2, 3, 4, 5, 6, 8
```

The sorted sequence incorrectly suggests chronological ordering.

---

# 4. Key-Value Store Counter Strategy

## Redis INCR

ID generation is delegated to a centralized Redis cluster.

Applications send:

```text
INCR
```

Redis atomically increments a counter and returns the next value.

### Example

```text
1001
1002
1003
1004
```

### Pros

* Extremely fast
* In-memory operation
* Sequential IDs
* Can generate hundreds of thousands of IDs per second

### Cons

#### Network Bottleneck

Every ID request requires:

```text
Application -> Redis -> Application
```

As traffic increases:

* Network latency increases
* Redis becomes a critical dependency

---

# 5. UUID

## Why UUID Exists

UUIDs allow independent systems to generate unique IDs without coordination.

No server communication is required.

---

## UUID Storage

UUID size:

```text
128 bits = 16 bytes
```

### Why Does UUID Often Consume 36 Bytes?

Humans cannot easily read raw binary:

```text
100101010101010101...
```

Therefore UUIDs are converted to hexadecimal text.

Example:

```text
550e8400-e29b-41d4-a716-446655440000
```

Storage breakdown:

```text
16 bytes
= 32 nibbles
= 32 hexadecimal characters
+ 4 hyphens
= 36 characters
```

If stored as CHAR(36) or VARCHAR(36):

```text
36 bytes
```

Important:

UUIDs do NOT have to be stored as text.

Modern databases often use:

```text
UUID type
BINARY(16)
```

which stores them in only:

```text
16 bytes
```

---

# UUID v4

## Structure

128-bit random identifier.

Most bits are random.

### Pros

* No infrastructure
* No coordination
* Ubiquitous support
* Extremely low collision probability

### Cons

#### Index Fragmentation

Relational databases use B-Tree indexes.

### Sequential Inserts

```text
1
2
3
4
5
```

New insert:

```text
6
```

Database simply appends.

---

### Random Inserts

```text
1
5
9
13
17
```

New insert:

```text
8
```

Database must:

1. Find insertion location
2. Split pages
3. Move data
4. Update pointers

This causes:

* More CPU usage
* More disk writes
* More cache misses

UUID v4 heavily fragments B-Tree indexes.

#### Storage Bloat

UUIDs remain large:

```text
16 bytes binary
36 bytes text
```

Larger indexes reduce cache efficiency.

---

# UUID v7

## Structure

```text
48-bit Unix Timestamp
74 bits Randomness
```

Timestamp comes first.

### Pros

#### Database Friendly

IDs increase over time.

New records naturally move toward the end of indexes.

Result:

* Fewer page splits
* Less fragmentation
* Better cache locality

Often called:

```text
B-Tree Friendly
```

#### No Coordination

Each server generates IDs independently.

### Cons

#### Large Size

Still:

```text
128 bits
16 bytes
```

#### Predictability

Creation time can be approximately inferred.

---

# 6. Twitter Snowflake

## Overview

Snowflake produces compact, sortable 64-bit IDs.

Bit layout:

```text
0 | 41-bit Timestamp | 10-bit Machine ID | 12-bit Sequence
```

### Timestamp (41 bits)

```text
2^41 milliseconds
≈ 69 years
```

### Machine ID (10 bits)

```text
2^10 = 1024 servers
```

### Sequence (12 bits)

```text
2^12 = 4096 IDs per millisecond
```

### Throughput

Per server:

```text
4096 IDs/ms
≈ 4.1 million IDs/sec
```

---

## Why Sequence Number Is Required

Suppose a server generates multiple IDs in the same millisecond.

Without a sequence:

```text
Timestamp = Same
Machine ID = Same
```

All IDs would collide.

Sequence guarantees uniqueness within the same millisecond.

---

## Advantages

* Compact 64-bit integer
* Excellent database performance
* Time ordered
* Minimal index fragmentation
* Very high throughput

---

## Challenges

### Coordination Infrastructure

Each server must receive a unique machine ID.

Common coordinators:

* ZooKeeper
* Consul

Without coordination:

```text
Two servers
Same Machine ID
Same Timestamp
Same Sequence
```

Potential collision.

---

### Clock Dependency

Snowflake assumes clocks always move forward.

Example:

```text
ID generated at 1000ms
```

Clock moves backward:

```text
999ms
```

Now a newly generated ID may be smaller than a previously generated ID.

Possible solutions:

1. Stop generating IDs until time catches up
2. Use logical clocks
3. Alert and fail the node

---

# Modern Alternatives

## ULID

Structure:

```text
Timestamp + Randomness
```

Size:

```text
128 bits
```

### Pros

* Sortable
* Human friendly
* No coordination

### Cons

* Larger than Snowflake

---

## KSUID

Structure:

```text
Timestamp + Randomness
```

Size:

```text
160 bits
```

### Pros

* Sortable
* Globally unique
* No coordination

### Cons

* Even larger than UUID

---

# Storage Size Comparison

| Type          | Size          |
| ------------- | ------------- |
| BIGINT        | 8 bytes       |
| Snowflake     | 8 bytes       |
| UUID (Binary) | 16 bytes      |
| UUID (Text)   | 36 bytes      |
| ULID          | 26 characters |
| KSUID         | 27 characters |

---

# Comparison Matrix

| Strategy             | Coordination            | Size   | Ordering     | Scalability | Main Drawback           |
| -------------------- | ----------------------- | ------ | ------------ | ----------- | ----------------------- |
| Flickr Ticket Server | Centralized             | Small  | Sequential   | Moderate    | Central dependency      |
| Range Allocation     | Partial                 | Small  | Sequential   | High        | Gaps and latency spikes |
| Multi-Master Offset  | Partial                 | Small  | Local only   | Poor        | Difficult scaling       |
| Redis INCR           | Centralized             | Small  | Sequential   | Moderate    | Network bottleneck      |
| UUID v4              | None                    | Large  | None         | Excellent   | Index fragmentation     |
| UUID v7              | None                    | Large  | Time ordered | Excellent   | Larger size             |
| Snowflake            | Machine ID coordination | Small  | Time ordered | Excellent   | Clock dependency        |
| ULID                 | None                    | Large  | Time ordered | Excellent   | Larger size             |
| KSUID                | None                    | Larger | Time ordered | Excellent   | Storage overhead        |

---

# Which One Should You Choose?

### UUID v4

Use when:

* Simplicity matters
* Database scale is moderate
* No ordering requirements

---

### UUID v7

Use when:

* No coordination desired
* Multiple services generate IDs
* Good database performance required

This is becoming the default recommendation for many modern systems.

---

### Snowflake

Use when:

* Massive scale
* Compact IDs matter
* Strong ordering requirements exist
* Infrastructure management is acceptable

Common choice in high-scale distributed systems.

---

# Key Takeaways

1. UUID v4 solves uniqueness but hurts database indexes.
2. UUID v7 fixes most database problems while keeping decentralization.
3. Snowflake offers the best storage efficiency and ordering.
4. Redis INCR is simple but introduces a centralized bottleneck.
5. Range allocation provides excellent throughput but creates gaps.
6. Multi-master offsets work but scale poorly.
7. The "best" ID strategy depends on trade-offs between coordination, ordering, storage size, and operational complexity.
