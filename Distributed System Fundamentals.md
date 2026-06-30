WORK IN PROGRESS : NOT COMPLETE INFO
Originally Created for Key Value Store so terminoligy related to that only.

1. Data Parititoning : WHAT WOULBE BE PARITION KEY?
  2. Hashing: Hash the key to provide the server where data is stored. One hash fxn can be just modulo with N,
     where N is number of serversInefficient as it causes n data movement when new nodes added or removed. 
  3. Consistent Hashing : Will provide only 1/n data movement only , good option for high available systems and replication (Primary choice in this design)
      https://github.com/kanxou/General-System-Design-All-Inclusive/ConsistentHashing.md
  4. Range Parititoning : Specific Range of Keys is set as the parition key. 
     During the insert or get operation, the key value automatically routed to the range it belongs to.
     Problem: 
     1. Data can be skewed if keys in a range dominate and the parititon would be a hotspot and cause performance bottleneck.
     2. In a time-based system, almost all new writes and reads target the most recent range
     3. Failures effect entire range read/write
  5. Directory/Lookup Based Partitioning: Simple Lookup Table to route. To keep the lookup table small we keep partition to server information rather than the each key. So first from key we find parition using a hash which could be used for finding server 
     Common Techniques to achive this:
      1. Dedicated Service : Most commonly used. Client ask which server to query to directory and then routed to that. Data is Visible to client as client reaches out to directory
      2. Router Has the Directory: Router Handles the logic and internally uses its directory for routing. Hide from the client i.e Client rely on Router. Also if multiple clients they will have to 
         implement the retry, failover, fetching , caching etc. But router does this all and now clients simply call client without worrying about anything unlike the Option 1
      3. Each client has directory : Clients cache the directory to avoid extra hop
  
     Parititon Vs Shard : https://github.com/kanxou/General-System-Design-All-Inclusive/Parititon Vs Shard.md
  6. Geographic Parititoning: https://github.com/kanxou/General-System-Design-All-Inclusive/Geographic Parititoning.md

2. Data Replication : Availability, Reliability, Fault Tolerance. 
  Replication Models: 
  1. Leader Follower AKA Master Slave : 
      Only Master receives updates and then propagated to replicas
      Cons : Leader Election Needed
             Leader Failure
      MySQL, PostgreSQL, Redis
  2. Each node can accept writes. Then propagate to rest. In terms of conflict (concurrent writes, writes during partition), need resolution. Used by Apache Cassandra and Dynamo DB.
    Read/Write Quorom can be used for consistency. W+R> N means high consistency , where N is replication factor , W and R and write read quorom of size W and R. W = 1 and R = N, means system optimized for Write
  3. Multi Leader: For Region Based, US Leader , India Leader etc
  
2.2.1 Concistency : Strong Consistenct System means any read operation returns a value corresponding to the result of the most updated write data item. This is achieved by rejecting the writes during the
  network partition if not all replicas agree on this. 
  Eventual Consistency is a weak Consistency where the nodes given enough time become consistent when updates are propagated
  Inconsistency Resolution

  Inconsistency will arise in highly available systems when
    Asynchronous replication → updates are still propagating.
    Network partitions → replicas cannot communicate.
    Leader crashes before replication completes → some replicas never receive the update.
    Concurrent writes in multi-leader systems → different replicas accept different updates.
    Leaderless writes → some replicas receive a write while others don't yet.

  Now we need to do Inconsistency detection and Resolution
  Lets say RF = 3, A B C nodes we have. 
  Initially in all nodes x = 100 lets say .User updates x = 150 and node A receives
  Network partition causes A-B to update but C is disconnected, so C has x = 100 still
  
  Detection: To detect if two nodes have different verions of data
  1 Versioning : Add a version to each write. Problem: Two concurrent updates on two nodes generate same new version
  2. Timestamp : Add a timestamp as well But distributed systems has clock skew problem.
  3. Checksum/Hash
  4. Merkle Tree: Root Node hash uniquely defines the entire dataset. Instead of Comparing each key, we just compare the root node. After a network partition, to compare the two nodes we would beed to compare entire keys
    Step 1 If Db has say keys A, B , C ,D then the leaf node of tree has hash of these
    These are the leaf nodes.
    Step 2: Combine adjacent hashes.
    Step 3 ; Compute the root
              hABCD
              /  \
          hAB        hCD
        /    \      /    \
      hA     hB   hC     hD

    Similar we will have for other network nodes, and we can compare root. 
    If root is different we need to just go to one path which is different to find out the difference
    Usually we keep multiple keys together in buckets , if 16 keys,4 bucket of size 4 are used at bottom level to keep depth of tree finite.
    In real-world systems, the bucket size is quite big. For instance, a possible configuration is one million buckets per one billion keys, so each bucket only contains 1000 keys.
Time reduced for search from n to logn
5. Vector Clock : Detect Concurrent Writes. It is version history of an object
  Each Replica has its own counter , A B C. For one key user:123 instead of storing key value, db stores key value and vector clock. e.g. 
    Key:
      user:123
    Value:
      Alice
    Vector Clock:
      [A:2, B:1, C:0]

      Conceptually just a map Map<NodeId, Long> 
      {
        "A" : 2,
        "B" : 1,
        "C" : 0
      }
  Initial Clock - [A:0 B:0 C:0] of each replica
  ReplicaA Updates - [A:1 B:0 C:0]
  Replica A updates again  - [A:2 B:0 C:0]
  Replica B updates - [A:0 B:1 C:0]

  During replication this clock is compared for inconsistency
  In Dynamo db both versions are provided to application and it can choose to merge etc.
  In cassandra : LWW , if replicas disagree then version with newest timestamp is chosen

6. Read Repair : During read operaton, multiple replicas are consulted, older versions removed
7. Anti Entropy Repair :Run periodically in background, check merkle trees, hashes missing data

Conflict Resolution : LWW, Application Level, Merge( Shopping Carts), CRDT(Topics to cover)

Failure Detection : All-All multicasting is expensive , n^2 messages are sent.
    Gossip protocol can be used. 

Distributed Key-Value Store Notes: Hinted Handoff & Read Repair
Hinted Handoff
Definition

Hinted Handoff is a write-path recovery mechanism used when one or more replicas are temporarily unavailable during a write.

Instead of failing the write, another replica temporarily stores a hint (the missed write) and delivers it when the unavailable replica comes back online.

Why is it needed?

Example:

Replication Factor (RF) = 3

Replicas:
A
B
C

Current data:

A → Gold
B → Gold
C → Gold

Node C goes down.

Client writes:

user123 → Platinum

Coordinator sends the write:

A ✓
B ✓
C ✗ (Unavailable)

Instead of failing:

A stores:

Hint
------------
Target Replica: C
Mutation: user123 → Platinum
Timestamp: 100
TTL: 3 hours

Client write succeeds because the required consistency level is met.

When C comes back

Coordinator (or the node storing the hint):

Checks if C is alive
        ↓
Sends stored mutation
        ↓
C applies the update
        ↓
Hint is deleted

Final state:

A → Platinum
B → Platinum
C → Platinum
What does a Hint contain?
Target replica
Partition key
Clustering key (if applicable)
Mutation (updated columns/value)
Timestamp/version
TTL (expiration)

A hint is not a full copy of the replica.

Advantages
High write availability
Fast recovery after temporary failures
Automatic background synchronization
Low network overhead
Limitations
Works only for temporary outages
Hints expire after a configurable time
Does not repair very old inconsistencies
Cannot repair data corruption

If hints expire, another repair mechanism is required.

Read Repair
Definition

Read Repair is a read-path consistency mechanism.

When replicas return different versions of the same data during a read, the coordinator returns the latest value to the client and updates stale replicas.

Why is it needed?

Suppose:

RF = 3

A → Gold
B → Gold
C → Silver (stale)

Client performs:

GET user123

Coordinator queries replicas.

Responses:

A → Gold (Timestamp 200)
B → Gold (Timestamp 200)
C → Silver (Timestamp 150)

Coordinator detects:

C has an older version.
Read Repair Process
Client Read
      ↓
Coordinator queries replicas
      ↓
Versions compared
      ↓
Latest value selected
      ↓
Latest value returned to client
      ↓
Coordinator sends repair to stale replica
      ↓
Replica updates itself

Final state:

A → Gold
B → Gold
C → Gold
How does the coordinator know which value is latest?

Using version metadata such as:

Timestamp
Version number
Vector Clock (Dynamo-style systems)
Hybrid Logical Clock (HLC)

Example:

Replica A → Version 25
Replica B → Version 25
Replica C → Version 18

25 > 18

Repair C
Advantages
Improves eventual consistency
No manual intervention
Automatically fixes stale replicas encountered during reads
Keeps replicas synchronized over time
Limitations
Only repairs data that is read
Extra network traffic
Slight increase in read latency
Cold (rarely read) data may remain inconsistent
Hinted Handoff vs Read Repair
Feature	Hinted Handoff	Read Repair
Path	Write Path	Read Path
Trigger	Replica unavailable during write	Replica stale during read
Purpose	Replay missed writes	Synchronize stale replicas
Temporary storage	Yes (Hints)	No
Requires client read	No	Yes
Requires node recovery	Yes	No
Best for	Temporary failures	Detecting stale replicas
Overall Consistency Flow
                 Client Write
                      │
                      ▼
             Replica Available?
              ┌────────┴────────┐
             Yes               No
              │                 │
       Write to Replica     Store Hint
              │                 │
              │           Replica Recovers?
              │            ┌─────┴─────┐
              │           Yes         No
              │            │           │
              │      Deliver Hint   Hint Expires
              │                        │
              └──────────────┬─────────┘
                             ▼
                     Client Performs Read
                             │
                             ▼
                  Coordinator Queries Replicas
                             │
                             ▼
                 Versions Compared Across Replicas
                             │
                             ▼
                  Any Replica Stale?
                     ┌────────┴────────┐
                    No                Yes
                     │                 │
              Return Data      Return Latest Data
                                       │
                                       ▼
                              Repair Stale Replica
                                       │
                                       ▼
                            All Replicas Consistent
Where Each Mechanism Fits
Write Path
──────────
Client
   │
Coordinator
   │
Replicas
   │
Node Down?
   │
Hinted Handoff

──────────────────────────────

Read Path
─────────
Client
   │
Coordinator
   │
Reads Multiple Replicas
   │
Version Comparison
   │
Read Repair (if needed)

──────────────────────────────

Background
──────────
Periodic Anti-Entropy Repair
(Merkle Trees)
Interview Summary
Hinted Handoff ensures writes aren't lost when a replica is temporarily unavailable by storing and replaying missed writes later.
Read Repair detects stale replicas during reads and synchronizes them with the latest version.
Hinted Handoff operates during the write path, while Read Repair operates during the read path.
Together with Anti-Entropy Repair (Merkle Trees), they provide eventual consistency in distributed key-value stores.


  
  
  

  
  
  
    
