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
  
  Detection: 
              
  
  
    
