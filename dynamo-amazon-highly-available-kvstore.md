# Dynamo: Amazon's Highly Available Key-value Store

**Giuseppe DeCandia, Deniz Hastorun, Madan Jampani, Gunavardhan Kakulapati, Avinash Lakshman, Alex Pilchin, Swaminathan Sivasubramanian, Peter Vosshall, Werner Vogels**  
*Amazon.com, 2007*

**Original Source:** [Local PDF](sources/dynamo-amazons-highly-available-key-value-store.pdf) | [Online](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)

## Overview

Dynamo is Amazon's highly available key-value storage system that powers core services requiring an "always-on" experience. This paper introduced techniques that became foundational for NoSQL databases, including consistent hashing, vector clocks, and quorum-based replication. It demonstrates how to build a system that sacrifices consistency for availability and partition tolerance, following the CAP theorem.

## Key Design Requirements

### Business Requirements
- **Always Writable**: The system must accept writes even during network partitions and server failures (shopping cart must always be available)
- **Highly Available**: 99.9% of requests must be served within 300ms
- **Scale Incrementally**: Must handle Amazon's peak holiday load (tens of millions of customers)
- **Cost-Effective**: Commodity hardware, no special infrastructure

### Technical Trade-offs
- **Consistency vs Availability**: Chose eventual consistency to maximize availability
- **Simple Key-Value Model**: No complex queries, just get(key) and put(key, value)
- **No Hierarchical Namespaces**: Single flat keyspace per table
- **Small Objects**: Typically less than 1MB

## Core Techniques

### 1. Consistent Hashing with Virtual Nodes

Traditional hashing would require rehashing all keys when nodes are added/removed. Dynamo uses:
- **Hash Ring**: Keys and nodes are hashed to points on a ring (0 to 2^128)
- **Virtual Nodes**: Each physical node is assigned multiple "tokens" (virtual nodes) on the ring
- **Benefits**:
  - Load spreads evenly when nodes join/leave
  - Heterogeneous hardware handled by assigning more virtual nodes to powerful machines
  - Incremental scaling without massive data movement

### 2. Replication for Availability

- **N-Replica Strategy**: Each key is replicated on N nodes
- **Preference List**: Coordinator node + N-1 clockwise successors on ring
- **Hinted Handoff**: If a replica is down, another node temporarily holds the data with a "hint" about the intended recipient
- **Read/Write Quorums**:
  - W = minimum nodes that must acknowledge a write
  - R = minimum nodes that must respond to a read
  - W + R > N guarantees reading latest write (strong consistency)
  - Amazon typically uses N=3, W=2, R=2

### 3. Vector Clocks for Versioning

Problem: In eventually consistent systems, concurrent updates create conflicts.

Solution: Vector clocks track causality between versions:
- Each node maintains a counter
- Updates increment the node's counter
- Versions include all counters: [(node1, count1), (node2, count2), ...]
- Can determine if versions are concurrent or one descends from another

Example:
```
Client writes v1 → [(NodeA, 1)]
NodeA replicates to NodeB
Client updates at NodeA → [(NodeA, 2)]
Client updates at NodeB → [(NodeA, 1), (NodeB, 1)]
Result: Two concurrent versions that must be reconciled
```

### 4. Sloppy Quorum and Hinted Handoff

Traditional quorum systems become unavailable if too many nodes fail. Dynamo's approach:
- **Sloppy Quorum**: First N *reachable* nodes from preference list
- **Hinted Handoff**: Temporary replica holders forward data when original node recovers
- **Trade-off**: Higher availability but temporary inconsistency

### 5. Anti-Entropy with Merkle Trees

Background process to detect and repair inconsistencies:
- **Merkle Trees**: Hash trees where each leaf is hash of a key range
- **Comparison**: Nodes exchange tree roots; if different, traverse down to find differences
- **Efficiency**: Only transfer differing key ranges, not entire dataset
- **Challenge**: Tree rebuilding when nodes join/leave

### 6. Gossip Protocol for Membership

Nodes discover cluster state without central coordination:
- **Random Gossip**: Nodes periodically exchange membership information
- **Eventually Consistent View**: All nodes converge on same cluster view
- **Failure Detection**: Nodes not gossiping are considered failed
- **Manual Intervention**: Node additions/removals initiated by administrator

## Implementation Details

### API
- **get(key)**: Returns object(s) and context (vector clock)
- **put(key, context, object)**: Context required for versioning

### Request Coordination
1. **Client Request**: Can contact any node (load balancer or client library routing)
2. **Coordinator Selection**: First node in preference list for the key
3. **Read Path**:
   - Coordinator sends read requests to N nodes
   - Waits for R responses
   - Returns all versions that are causally unrelated
4. **Write Path**:
   - Coordinator generates new vector clock
   - Sends to N nodes in preference list
   - Returns success after W acknowledgments

### Conflict Resolution
- **Last-Write-Wins**: Simple but loses data
- **Application-Level**: Shopping cart merges all items from conflicting versions
- **Business Logic**: Critical for correctness in eventual consistency

## Performance Characteristics

### Latencies (Amazon's Production)
- 99.9th percentile: ~200-300ms
- Average: ~10-20ms
- Variance reduced by bounding replica set

### Load Distribution
- Virtual nodes: 10-30% better load distribution than physical nodes
- Typical: 100-200 virtual nodes per physical node

### Availability
- During network partitions: System remains available
- Node failures: Transparent to clients
- Data center failures: Cross-DC replication (not detailed in paper)

## Lessons Learned

### What Worked
1. **Incremental Scalability**: Add one node at a time
2. **Symmetry**: Every node can serve any request
3. **Decentralization**: No single point of failure
4. **Heterogeneity**: Different hardware capabilities handled gracefully

### Challenges
1. **Vector Clock Size**: Can grow with client count (addressed by limiting clock size)
2. **Merkle Tree Maintenance**: Expensive when membership changes
3. **Load Imbalance**: Initial random token assignment sometimes created hotspots
4. **Operational Complexity**: Many tunable parameters (N, R, W)

## Impact on Industry

Dynamo's techniques influenced numerous systems:

### Direct Descendants
- **Apache Cassandra**: Combined Dynamo's distribution with BigTable's data model
- **Riak**: Commercial implementation closely following Dynamo
- **Voldemort**: LinkedIn's implementation

### Concepts Adopted Widely
- **Consistent Hashing**: Redis Cluster, MongoDB
- **Vector Clocks**: CouchDB, Riak
- **Hinted Handoff**: Cassandra, Riak
- **Anti-Entropy**: Most eventually consistent systems

### Evolution of Ideas
- **CRDTs**: Conflict-free replicated data types for automatic conflict resolution
- **Hybrid Consistency**: Systems offering both strong and eventual consistency
- **Multi-Region**: Global distribution patterns

## Practical Applications

### When to Use Dynamo-Style Systems
✅ **Good Fit**:
- Shopping carts, session stores
- User preferences, game state
- Write-heavy workloads
- Multi-region deployments
- "Always-on" requirements

❌ **Poor Fit**:
- Strong consistency requirements (banking)
- Complex queries (analytics)
- Large objects (media storage)
- Transactions across keys

### Modern Considerations

Since 2007, the landscape has evolved:
- **DynamoDB**: Amazon's managed service inspired by Dynamo
- **Consistency Options**: Many systems now offer tunable consistency
- **Global Tables**: Multi-region with automatic conflict resolution
- **Serverless**: Pay-per-request pricing models

## Key Takeaways

1. **CAP Trade-offs Are Real**: You cannot have perfect consistency, availability, and partition tolerance
2. **Eventual Consistency Is Complex**: Requires careful application design
3. **Simple Interface, Complex Implementation**: get/put API hides sophisticated distributed systems
4. **Operational Characteristics Matter**: P99 latency often more important than average
5. **One Size Doesn't Fit All**: Dynamo optimizes for specific use cases

## Critical Analysis

### Strengths
- Proven at massive scale (Amazon's shopping cart)
- Clear design rationale for each decision
- Influential techniques still relevant today

### Limitations
- Eventually consistent model pushes complexity to applications
- No built-in support for multi-key transactions
- Operational complexity (tuning N, R, W values)
- Limited query capabilities

### Modern Alternatives
- **Strong Consistency**: Spanner, CockroachDB (using consensus protocols)
- **Multi-Model**: Azure Cosmos DB (multiple consistency levels)
- **Specialized**: Redis (in-memory), ScyllaDB (C++ rewrite of Cassandra)

## Conclusion

Dynamo demonstrated that highly available, scalable systems could be built on commodity hardware by carefully choosing trade-offs. Its "always writable" design philosophy and eventual consistency model challenged traditional database assumptions and paved the way for the NoSQL movement. While not every system needs Dynamo's extreme availability, understanding its techniques is essential for building distributed systems.

The paper's lasting contribution isn't just the specific techniques but the systematic approach to designing for failure, the careful consideration of operational requirements, and the recognition that different applications need different consistency guarantees. In a world where distributed systems are the norm, Dynamo's lessons remain remarkably relevant.