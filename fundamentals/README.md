# Fundamentals

Core concepts that show up across most system design interviews. One file per topic. Check a box once the topic has a write-up I am happy with.

## Foundations
- [ ] [Back-of-the-envelope estimation](back-of-the-envelope-estimation.md) (QPS, storage, bandwidth, latency numbers)
- [ ] [Scalability: vertical vs. horizontal scaling](scalability.md)
- [ ] [Availability, reliability, and SLAs](availability-reliability-sla.md)
- [ ] [CAP theorem and PACELC](cap-theorem-pacelc.md)
- [ ] [Consistency models](consistency-models.md) (strong, eventual, causal, read-your-writes)

## Networking and Traffic
- [ ] DNS
- [ ] Load balancing (L4 vs. L7, algorithms, health checks)
- [ ] CDN
- [ ] API design (REST, gRPC, GraphQL, pagination, idempotency)
- [ ] Real-time communication (polling, long polling, WebSocket, SSE)
- [ ] Rate limiting

## Data
- [ ] SQL vs. NoSQL and how to choose
- [ ] Indexing (B-tree, LSM tree, inverted index)
- [ ] Replication (leader-follower, multi-leader, leaderless)
- [ ] Sharding and partitioning
- [ ] Consistent hashing
- [ ] Transactions and isolation levels
- [ ] Caching (strategies, eviction, invalidation, cache stampede)
- [ ] Blob and object storage

## Distributed Systems
- [ ] Message queues and event streaming (Kafka, SQS, delivery semantics)
- [ ] Consensus (Raft, Paxos, leader election)
- [ ] Distributed transactions (2PC, saga)
- [ ] Unique ID generation
- [ ] Clocks and ordering (Lamport, vector clocks)
- [ ] Bloom filters, HyperLogLog, and other probabilistic structures

## Operations
- [ ] Monitoring, logging, and tracing
- [ ] Microservices vs. monolith
- [ ] Security basics (authN, authZ, encryption)
