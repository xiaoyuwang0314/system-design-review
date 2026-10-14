# Depth Guide

Rules for deciding how deep to go, so that deep dives stay focused on what the problem is actually testing.

## 1. Start with the core challenge

Before designing anything, write one sentence describing what makes the problem hard. Deep dives go only into components that serve this sentence. Everything else stays at a high level.

| Problem | Core challenge |
| --- | --- |
| URL shortener | Generating unique short codes at scale, and serving a read-heavy workload with low latency |
| News feed | Balancing fan-out cost against read latency, especially for accounts with millions of followers |
| Chat system | Delivering messages in order and in real time over long-lived connections |
| Ticket booking | Preventing overselling when many users compete for the same seats at once |

## 2. Three depth levels

| Level | What I should be able to do | When to use it |
| --- | --- | --- |
| **L1: Know** | Name it, say why I chose it, give a one-sentence trade-off | Supporting components (load balancer, DNS, generic web tier) |
| **L2: Explain** | Explain how it works, what happens when it fails, and answer 2 to 3 follow-ups | Components tied to the core challenge |
| **L3: Internals** | Walk through algorithms and implementation details | Only when the interviewer asks, or when the component *is* the core challenge |

Example for a URL shortener:

| Component | Level | Reason |
| --- | --- | --- |
| Short code generation | L2 to L3 | This is the core challenge: counter vs. hash vs. Snowflake, collision handling |
| Cache | L2 | Read-heavy workload: cache-aside, LRU, hot keys |
| Database | L1 | Simple access pattern, so a key-value store is enough |
| ZooKeeper consensus internals | Too deep | Knowing Raft details would not change any design decision here |

This table shows the full range of levels for illustration. Under the mid-level target in section 4, a write-up picks one of these components as its main deep dive.

## 3. The stop test

Before going one level deeper, ask:

> Would going deeper change a design decision, or change my answer to a likely follow-up?

If not, stop. Put the topic in the **Parked** section of the write-up and move on.

## 4. Target bar: mid-level

I am preparing for new grad interviews in general, and I target the mid-level bar as a safety margin. See [`research/ng-interview-landscape-2026.md`](research/ng-interview-landscape-2026.md) for the reasoning.

| | What that means for each write-up |
| --- | --- |
| Requirements | Clarify functional and non-functional requirements before drawing anything |
| Estimation | Quick and rough, a few lines. NG interviews rarely weigh it heavily |
| Design | Cover both the read path and the write path, plus basic failure handling |
| Deep dives | **One** component at L2 per problem. A second one only if it is clearly tied to the core challenge |
| Trade-offs | Justify every main choice, and name at least one alternative |

## 5. Hard limits

- **Each deep dive fits in about five minutes of speaking.**
- **A fundamentals topic is done** when I can explain it in two minutes and answer three follow-up questions. Anything beyond that goes to Parked.

## 6. Default levels by topic group

- Foundations, Networking, and Data topics: L2.
- Message queues and event streaming: L2, since many practical designs (log ingestion, notifications, pub/sub) depend on them.
- Other Distributed Systems topics (consensus, distributed transactions, clocks): L1. Go to L2 only after the rest of the roadmap is solid.
