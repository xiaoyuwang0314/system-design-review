# Back-of-the-Envelope Estimation

## Why it matters

Estimation is a tool for making design decisions. The point is to answer questions like "does the data fit on one machine?", "do I need a cache?", or "how many servers?". The exact numbers do not matter, the order of magnitude does.

### When to estimate

Following Hello Interview, do the math **only when it drives a decision**:
- Put scale targets (users, requests per day, data size) in the non-functional requirements.
- Calculate inside the design at the moment a number changes a choice: how long a short code must be, whether data fits on one machine, how many connection servers are needed.
- If the interviewer asks for upfront estimates (Alex Xu's style), do them in two to three minutes.

Either way, always state what the numbers imply for the design.

## Numbers to remember

### Powers of ten

| Unit | Bytes | Shortcut |
| --- | --- | --- |
| KB | 10^3 | a short text record |
| MB | 10^6 | a photo |
| GB | 10^9 | fits in memory easily |
| TB | 10^12 | fits on one large disk |
| PB | 10^15 | needs a distributed storage system |

### Time

| Period | Seconds | Round to |
| --- | --- | --- |
| Day | 86,400 | **10^5** |
| Month | about 2.6 million | 2.5 x 10^6 |
| Year | about 31.5 million | 3 x 10^7 |

### Requests per day to QPS

Divide by 10^5 (seconds per day, rounded).

| Per day | Average QPS |
| --- | --- |
| 1 million | about 10 |
| 100 million | about 1,000 |
| 1 billion | about 10,000 |

**Peak QPS** is usually 2 to 3 times the average. Use up to 10 times for spiky traffic (ticket sales, live events).

### Data sizes

| Item | Size |
| --- | --- |
| ASCII character | 1 B (UTF-8 can be up to 4 B) |
| Integer | 4 B |
| Long integer or timestamp | 8 B |
| UUID | 16 B |
| Typical metadata row | 100 B to 1 KB |
| Photo | 100 KB to a few MB |
| One minute of compressed video | tens of MB |

### Latency (orders of magnitude)

| Operation | Time |
| --- | --- |
| Main memory reference | 100 ns |
| SSD random read | 100 us |
| Round trip within a data center | 0.5 ms |
| Read 1 MB sequentially from SSD | 1 ms |
| HDD seek | 10 ms |
| Round trip across continents | 100 to 150 ms |

Takeaways: memory is about 1,000 times faster than SSD, and SSD is about 100 times faster than an HDD seek. A cross-continent round trip is expensive, which is why CDNs and regional deployments exist.

### Rough capacity per machine

These are rules of thumb. Real numbers vary a lot with the workload, so state them as assumptions.

| Component | Rough capacity |
| --- | --- |
| Application server | 1,000s of QPS for simple requests |
| Redis node | about 100,000 operations per second |
| Relational database node | 1,000s to 10,000 writes per second, more for reads |
| Server memory | 64 to 256 GB |
| Server disk | a few TB to tens of TB |
| Network card | 10 Gbps, about 1 GB/s |

## Formulas

- **QPS** = DAU x actions per user per day / 10^5
- **Peak QPS** = QPS x 2 to 3
- **Storage** = new records per day x record size x days of retention x replication factor (usually 3)
- **Bandwidth** = QPS x payload size
- **Cache size** = about 20% of the data read per day (the 80/20 rule: a small share of items gets most of the reads)
- **Servers** = peak QPS / QPS per server
- **Concurrent requests or connections** (Little's law) = QPS x average time each request or connection is held

## Process

1. **State assumptions out loud.** DAU, actions per user, read-to-write ratio, record size, retention.
2. **Round aggressively.** 86,400 becomes 10^5. 3.65 becomes 4 if it helps.
3. **Compute only what drives decisions.** Usually QPS (read and write), storage, and sometimes bandwidth or cache size.
4. **Sanity-check.** Does the number look reasonable compared with a known system?
5. **State the implication.** Compare each number with the per-machine capacity table above. The gap tells you what to do: "20,000 read QPS is too much for one database, so I will add a cache."

## Worked example 1: URL shortener (read-heavy)

Assumptions:
- 100 million new short URLs per day
- Read-to-write ratio of 10:1
- 500 B per record (short code, long URL, user ID, timestamps)
- Keep data for 10 years

| Quantity | Calculation | Result |
| --- | --- | --- |
| Write QPS | 10^8 / 10^5 | about 1,000 |
| Read QPS | 1,000 x 10 | about 10,000 (peak 20,000 to 30,000) |
| Storage per day | 10^8 x 500 B | 50 GB |
| Storage for 10 years | 50 GB x 365 x 10 | about 180 TB (about 550 TB with 3 replicas) |
| Cache | 20% of daily reads: 10^9 reads x 500 B x 0.2 | about 100 GB |

Implications:
- **Reads dominate**, so caching is the main lever.
- **The cache fits** in a few Redis nodes.
- **Storage grows to hundreds of TB**, so the database must be sharded eventually. The access pattern is a simple lookup by key, so a key-value store fits well.
- **Write QPS is moderate**, so short code generation does not need an exotic design.

## Worked example 2: Chat app (write-heavy)

Assumptions:
- 500 million DAU
- 40 messages sent per user per day
- 100 B per message, including metadata
- Keep messages for 5 years
- About 10% of DAU online at any moment, each holding one WebSocket connection
- One server holds about 100,000 connections

| Quantity | Calculation | Result |
| --- | --- | --- |
| Messages per day | 5 x 10^8 x 40 | 2 x 10^10 |
| Write QPS | 2 x 10^10 / 10^5 | 2 x 10^5 (peak about 6 x 10^5) |
| Storage per day | 2 x 10^10 x 100 B | 2 TB |
| Storage for 5 years | 2 TB x 365 x 5 | about 3.65 PB (about 11 PB with 3 replicas) |
| Concurrent connections | 5 x 10^8 x 10% | 5 x 10^7 |
| Connection servers | 5 x 10^7 / 10^5 | 500 (deploy more for evening peaks and failover) |

Compare with single-machine capacity:

| | Needed | One relational database | Gap |
| --- | --- | --- | --- |
| Writes | 6 x 10^5 per second at peak | 10^3 to 10^4 per second | 60 to 600 times |
| Storage | about 11 PB | a few TB to tens of TB | 1,000 times or more |

Implications:
- **One database cannot hold this.** The data must be sharded across many machines.
- **The access pattern decides the store.** Writes append a message to a conversation, and reads fetch the latest N messages of one conversation. There are no joins or cross-conversation transactions. A wide-column store such as Cassandra or HBase fits: LSM-tree storage makes writes cheap, and it scales horizontally with built-in sharding and replication.
- **Partition by `conversation_id`, cluster by `message_id`.** All messages of one conversation live in one partition, sorted by time, so "latest 50 messages" is a single range scan. Partitioning by sender `user_id` would instead scatter a conversation across every participant's partition, so each read would have to query many partitions and merge the results (scatter-gather).
- **Storage, not QPS, drives the node count.** At about 20,000 writes per second per node, writes need about 30 nodes. At about 10 TB per node, storage needs about 1,100 nodes. This suggests tiering: keep recent messages in the hot store and move old ones to cheaper cold storage such as object storage.

Rule worth remembering: **choose the partition key based on the most common read.**

## Common mistakes

- Spending more than a few minutes, or chasing precise numbers
- Forgetting peak traffic
- Forgetting replication when estimating storage
- Going from per-day to multi-year totals without multiplying by 365. I made this mistake in the chat app example and was off by more than 100 times
- Stopping at the numbers without comparing them with single-machine capacity, which is where the design decision comes from
- Computing numbers but never saying what they mean for the design
- Silent assumptions. The interviewer cannot correct an assumption they never heard

## Follow-up questions

**What changes if traffic grows 10 times?**
Recompute the bottleneck. In the URL shortener, read QPS reaches 100,000 or more, so the cache tier grows and database read replicas matter more. Storage grows to petabytes, so sharding becomes urgent instead of eventual.

**Why multiply storage by 3?**
Most distributed storage systems keep three replicas for durability and availability. Losing one machine, or even two, does not lose data.

**How do you know how much one server can handle?**
In an interview, state a rule-of-thumb assumption and move on. In practice, you load test, because capacity depends on request cost, payload size, and hardware.

## Parked

- Queueing theory and how utilization affects latency
- Tail latency (p99) math across many service calls
- Exact cost estimation for cloud resources

## References

- Jeff Dean, "Numbers Everyone Should Know" (latency numbers, widely reproduced)
- Alex Xu, *System Design Interview: An Insider's Guide*, Vol. 1, Chapter 2
