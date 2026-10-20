# Availability, Reliability, and SLAs

## The nines

| Availability | Max downtime per year | Max downtime per month |
| --- | --- | --- |
| 99% (two nines) | 3.65 days | 7.2 hours |
| 99.9% (three nines) | 8.8 hours | 43 minutes |
| 99.99% (four nines) | 52 minutes | 4.3 minutes |
| 99.999% (five nines) | 5 minutes | 26 seconds |

Each extra nine cuts allowed downtime by 10 times and costs much more. At four nines there are about four minutes per month, which is too short for a human to notice and react. Failure recovery has to be automatic.

## SLI, SLO, SLA

| Term | Meaning | Example |
| --- | --- | --- |
| **SLI** (indicator) | What you measure | Share of successful requests, p99 latency |
| **SLO** (objective) | The internal target | 99.9% of requests succeed over 30 days |
| **SLA** (agreement) | The contract with customers, with penalties | Below 99.9%, customers get service credits |

The SLA is usually looser than the SLO to leave a buffer. The **error budget** is 1 minus the SLO, the amount of failure allowed. When it runs out, the team pauses feature launches and works on reliability.

## Combining components

**In series** (a request must pass through every component): multiply. Availability always goes down.

> LB 99.99% x app 99.9% x DB 99.9% = about 99.79%

**In parallel** (redundant copies, any one is enough): 1 minus the chance that all fail at once. Availability goes up sharply.

> Two servers at 99% each: 1 - (0.01 x 0.01) = 99.99%

Worked example: LB (99.99%), then an app tier of 3 servers at 99% each (any one is enough), then one DB (99.9%).

| Component | Availability |
| --- | --- |
| LB | 99.99% |
| App tier | 1 - 0.01^3 = 99.9999% |
| DB | 99.9% |
| **End to end** | 0.9999 x 0.999999 x 0.999 = **about 99.89%** |

The DB is the weakest link: it is in series and has no redundancy. Three app servers at only 99% each are more available together than one DB at 99.9%. The fix is a standby DB with automatic failover.

Rule: **find components in the request path with no redundancy (single points of failure) and add redundancy there first.**

## Redundancy and failover

| | Active-passive | Active-active |
| --- | --- | --- |
| How | The primary serves all traffic, a standby takes over when it fails | All nodes serve traffic |
| Pros | Simple, no write conflicts | No failover delay, no idle hardware |
| Cons | Failover takes time, the standby sits idle | If several nodes accept writes, conflicts must be resolved. Each node needs spare capacity to absorb a failed node's traffic |

Health checks and heartbeats detect failed nodes and trigger failover automatically.

### What can go wrong in database failover

Setup: active-passive database with **asynchronous replication**. The primary fails and the standby is promoted.

1. **Data loss.** The primary had acknowledged some writes to clients but had not yet copied them to the standby (replication lag). Those writes are gone after failover, even though users were told they succeeded.
2. **Split brain.** The old primary was only slow or cut off from the network, not dead. It comes back still believing it is the primary. Now two nodes accept writes and the data diverges.
3. **Downtime during detection.** The system must wait long enough to be sure the primary is dead. Too short a timeout causes false failovers. Too long means more downtime.

Mitigations:

| Problem | Mitigation | Cost |
| --- | --- | --- |
| Data loss | Synchronous (or semi-synchronous) replication: acknowledge a write only after the standby has it | Higher write latency, and writes stall if the standby is down |
| Split brain | Fencing: cut the old primary off (revoke its access, or reject writes tagged with an old leader epoch). Use a consensus system (ZooKeeper, etcd) to elect the leader | More components and complexity |

This is the same trade-off as CAP and consistency models: waiting for replicas buys safety at the cost of latency and availability.

## Availability, reliability, durability

- **Availability**: can I use it right now? Does a request get a response?
- **Reliability**: does it keep working correctly over time? A system that is always up but returns wrong data is available but not reliable.
- **Durability**: once data is written, will it stay? Durability failures are permanent. Availability failures are usually temporary.

Example: AWS S3 promises 99.999999999% (11 nines) durability and 99.99% availability.
- **Durability**: AWS describes it this way: if you store 10 million objects, you can expect to lose one object about once every 10,000 years. Data is copied across multiple devices and data centers.
- **Availability**: for up to about 52 minutes a year, you might be unable to read or write. The data is still safe, just temporarily unreachable.

Durability is set much higher because losing data is permanent and unrecoverable, while an outage ends.

## Follow-up questions

**What is the single point of failure in your design?**
Walk the request path. Any component in series with no redundancy is one: often the primary database, sometimes a single load balancer or a single region.

**How would you reach 99.99%?**
Remove single points of failure with redundancy in every tier, automate failover, deploy across multiple availability zones, and roll out changes gradually so a bad deploy does not take everything down.

**Why not use synchronous replication everywhere?**
Every write waits for the replica, so write latency goes up. If the replica is down or slow, writes stall. Many systems use semi-synchronous replication: wait for one replica, not all.

## Parked

- Consensus algorithms in detail (Raft, Paxos)
- Graceful degradation and load shedding
- Multi-region disaster recovery, RPO and RTO

## References

- Google, *Site Reliability Engineering*, Chapter 4 (Service Level Objectives)
- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 5 (Replication, "Handling Node Outages")
- AWS, Amazon S3 FAQs (durability and availability)
