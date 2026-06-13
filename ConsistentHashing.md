## Why Do We Need Consistent Hashing?

Suppose we have 4 cache servers:

```java
server = hash(key) % 4;
```

Example:

```java
hash("user123") % 4 = 2
```

Store on Server 2.

### Problem

Add a new server:

```java
server = hash(key) % 5;
```

Now:

```java
hash("user123") % 5 = 4
```

The same key maps to a different server.

This happens for most keys.

### Consequences

* Massive data movement
* Cache miss storms
* Database overload
* Network traffic spikes

With modulo hashing, adding/removing a server remaps almost every key.

---

# Core Idea

Instead of mapping servers into an array, map both:

* Servers
* Keys

onto a circular hash ring.

```text
0 -----------------> 2^32 - 1
^                   |
|                   |
+-------------------+
```

The ring wraps around.

---

# Step 1: Place Servers On Ring

Hash server identifiers.

```text
hash(ServerA) = 100
hash(ServerB) = 350
hash(ServerC) = 700
hash(ServerD) = 900
```

Ring:

```text
100(A) -> 350(B) -> 700(C) -> 900(D) -> back to 100(A)
```

---

# Step 2: Place Keys On Ring

Hash keys too.

```text
hash(user1) = 250
hash(user2) = 600
hash(user3) = 950
```

---

# Assignment Rule

A key belongs to the first server encountered while moving clockwise.

### Example

```text
user1 = 250
```

Clockwise:

```text
250 -> 350(B)
```

Stored on:

```text
ServerB
```

---

```text
user2 = 600
```

Clockwise:

```text
600 -> 700(C)
```

Stored on:

```text
ServerC
```

---

```text
user3 = 950
```

Clockwise:

```text
950 -> wrap -> 100(A)
```

Stored on:

```text
ServerA
```

---

# What Happens When A New Server Is Added?

Before:

```text
100(A)
500(C)
900(D)
```

Ownership:

```text
(100,500] -> C
(500,900] -> D
(900,100] -> A
```

Add:

```text
300(B)
```

Now:

```text
100(A)
300(B)
500(C)
900(D)
```

Ownership:

```text
(100,300] -> B
(300,500] -> C
```

Only:

```text
(100,300]
```

moves from C to B.

Everything else remains unchanged.

---

# Important Interview Point

Keys are NOT rehashed.

Example:

```text
hash(user123) = 250
```

Before:

```text
250 -> C
```

After B is inserted:

```text
250 -> B
```

The hash value remains:

```text
250
```

Only ownership changes.

---

# Why Only 1/N Data Moves?

Suppose:

```text
100 servers
```

Each owns approximately:

```text
1/100
```

of the ring.

When a new server joins:

```text
≈ 1% data moves
```

instead of:

```text
≈ 100%
```

with modulo hashing.

---

# What Happens When A Server Dies?

Suppose:

```text
A=100
B=300
C=500
D=900
```

C crashes.

Only:

```text
(300,500]
```

moves.

Everything else stays exactly where it was.

---

# Major Problem: Uneven Distribution

Suppose:

```text
A=100
B=110
C=120
D=900
```

D owns most of the ring.

Result:

* Uneven traffic
* Uneven storage
* Hotspots

---

# Virtual Nodes (VNodes)

Solution:

Instead of one position per server:

```text
A=100
B=300
C=700
```

Use many positions.

```text
A#1
A#2
A#3

B#1
B#2
B#3

C#1
C#2
C#3
```

Example:

```text
A#1=100
A#2=350
A#3=800

B#1=200
B#2=500
B#3=900

C#1=150
C#2=600
C#3=750
```

Benefits:

* Better load balancing
* Better distribution
* Easier scaling
* Smaller rebalancing operations

Production systems almost always use virtual nodes.

---

# Replication

Store data on multiple nodes.

Example:

Replication Factor = 3

```text
Primary -> B
Replica1 -> C
Replica2 -> D
```

If B fails:

```text
C already has the data
```

---

# Where Does The Ring Live?

## Option 1: Every Node Knows The Ring

Each node stores:

```text
A
B
C
D
```

membership information.

Updates distributed using:

* ZooKeeper
* etcd
* Consul

Pros:

* No central bottleneck
* Fast lookups

Cons:

* Membership synchronization required

---

## Option 2: Coordinator

```text
Client
  |
Coordinator
  |
Server
```

Pros:

* Simpler

Cons:

* Extra network hop
* Potential bottleneck

---

# Time Complexity

Using:

```java
TreeMap<Long, Node>
```

Lookup:

```java
ceilingEntry()
```

Complexities:

```text
Lookup      O(log N)
Insert      O(log N)
Delete      O(log N)
```

---

# Why Use MD5?

We need a uniformly distributed position on the ring.

```java
hash(ServerA)
hash(ServerB)
hash(Key123)
```

MD5 provides a deterministic, well-distributed hash.

---

# Why Convert MD5 To Long?

MD5 returns:

```java
byte[]
```

Example:

```java
[-22, -74, -13, 12, ...]
```

But the ring requires:

```java
Long
```

because:

* TreeMap needs sortable keys
* Numeric comparisons are faster
* Range calculations become easy

We are NOT improving the hash.

We are simply converting MD5 output into a numeric token.

---

# Understanding This Code

```java
for (int i = 0; i < 8; i++) {
    hash = (hash << 8)
            | (digest[i] & 0xFF);
}
```

Purpose:

Build a 64-bit number from 8 bytes.

---

## Why `<< 8`?

Move existing bits left by one byte.

Example:

```java
0x12 << 8
```

Binary:

```text
00010010
```

becomes:

```text
00010010 00000000
```

Result:

```text
0x1200
```

This creates room for the next byte.

---

## Why `|` ?

Insert the next byte.

Example:

```java
0x1200 | 0x34
```

Result:

```text
0x1234
```

---

# Why `digest[i] & 0xFF`?

Java bytes are signed.

Range:

```text
-128 to 127
```

MD5 bytes should represent:

```text
0 to 255
```

Example:

```java
byte b = (byte)0xEA;
```

Java sees:

```text
-22
```

But:

```java
b & 0xFF
```

becomes:

```text
234
```

which is the actual byte value.

Without masking:

```java
hash |= digest[i];
```

negative bytes would corrupt the final hash.

---

# Why Not Store Hashes As Strings?

Example:

```java
"EAB6F30C..."
```

Possible, but not ideal.

Problems:

* More memory
* Slower comparisons
* Harder range calculations

Numeric tokens are:

* Smaller
* Faster
* Easier to sort

Therefore production systems usually use:

```java
long
```

or

```java
BigInteger
```

instead of strings.

---

# Common Implementation

```java
private long generateHash(String key) {

    MessageDigest md =
            MessageDigest.getInstance("MD5");

    byte[] digest =
            md.digest(key.getBytes(StandardCharsets.UTF_8));

    long hash = 0;

    for (int i = 0; i < 8; i++) {
        hash <<= 8;
        hash |= (digest[i] & 0xFF);
    }

    return hash & Long.MAX_VALUE;
}
```

---

# Ring Lookup

```java
Long token = generateHash(key);

Long nextToken = ring.ceilingKey(token);

if (nextToken == null) {
    nextToken = ring.firstKey();
}

return ring.get(nextToken);
```

This finds the first server clockwise.

---

# Interview Summary

1. Normal hashing uses `hash(key) % N`.
2. Adding/removing servers remaps almost all keys.
3. Consistent hashing maps servers and keys onto a ring.
4. A key belongs to the first server clockwise.
5. Only neighboring partitions move when membership changes.
6. Data movement becomes approximately `1/N`.
7. Virtual nodes improve distribution and balancing.
8. Replication provides fault tolerance.
9. Rings are usually maintained via ZooKeeper, etcd, or gossip protocols.
10. Lookups are typically O(log N) using TreeMap or similar ordered structures.
11. MD5 output is converted into numeric tokens for efficient ordering.
12. `digest[i] & 0xFF` converts signed bytes into unsigned values.
13. `<< 8` creates room for the next byte while constructing the hash token.

If you understand everything in this document, you are well-prepared to discuss consistent hashing in most backend and distributed systems interviews.
