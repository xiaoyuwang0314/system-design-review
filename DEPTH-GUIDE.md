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

## 3. The stop test

Before going one level deeper, ask:

> Would going deeper change a design decision, or change my answer to a likely follow-up?

If not, stop. Put the topic in the **Parked** section of the write-up and move on.

## 4. Hard limits

- **Two or three deep dives per problem.** Each should fit in about five minutes of speaking, which matches the time available in a real 45-minute interview.
- **A fundamentals topic is done** when I can explain it in two minutes and answer three follow-up questions. Anything beyond that goes to Parked.

## 5. New grad calibration

For new grad interviews, the bar is a clear requirements discussion, a reasonable high-level design, and sensible trade-offs. Interviewers rarely expect L3 on distributed systems internals.

- Foundations and Data topics: aim for L2.
- Distributed Systems topics (consensus, distributed transactions, clocks): L1 is enough by default. Go to L2 only after the rest of the roadmap is solid.
