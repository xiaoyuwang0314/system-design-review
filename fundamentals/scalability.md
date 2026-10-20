# Scalability: Vertical vs. Horizontal

## Two ways to scale

| | Vertical scaling (scale up) | Horizontal scaling (scale out) |
| --- | --- | --- |
| How | Move to a bigger machine: more CPU, memory, faster disks | Add more identical machines behind a load balancer |
| Pros | No code changes, no distributed system complexity, consistency is trivial | Capacity grows roughly linearly, and one machine failing does not take the service down |
| Cons | Hardware ceiling, cost grows faster than capacity, single point of failure, downtime to upgrade | More moving parts, services must be stateless, data consistency gets harder |

Vertical scaling goes further than people expect. Cloud machines come with terabytes of memory and hundreds of cores, and many companies ran on a few large database servers for years. In an interview, start simple, then use the estimation numbers to show where the simple design stops working.

## Stateless services

A service is **stateless** when a server keeps no per-user data between requests, so any server can handle any request. This is what makes horizontal scaling of the application tier easy.

If a server keeps user sessions in its own memory, it is stateful. When the load balancer sends the next request to a different server, the session is missing and the user has to log in again.

Ways to fix it:

| Option | How it works | Downside |
| --- | --- | --- |
| Shared session store | Sessions live in Redis or a database that every server reads | One more component to run and keep available |
| Client-side token (e.g. JWT) | The signed token carries the session data, sent with every request | Hard to revoke a token before it expires; tokens add size to every request |
| Sticky sessions | The load balancer routes a user to the same server every time | Uneven load, and sessions are lost when that server dies |

The first two make servers truly stateless. Sticky sessions keep the state on the server and only work around the problem.

## The data tier is the hard part

Stateless application servers are easy to add. The database holds state, so it is much harder to scale. Escalate based on what is actually the bottleneck:

| Problem | Technique | Cost |
| --- | --- | --- |
| Database is generally overloaded | Scale the database vertically first | Hardware ceiling |
| Too many reads | **Cache** repeated reads | Invalidation, possible stale data |
| Too many reads | **Read replicas** spread reads across copies | Replication lag, so replicas may return stale data |
| Too many writes, or data too large for one machine | **Sharding** splits data across machines | Cross-shard queries are hard, and adding machines means moving data |

Rule: **cache and read replicas fix reads. Sharding fixes writes and storage.** Sharding comes last because it is the most expensive and the hardest to undo.

Read replicas do not help with writes. Every write still goes to the leader and is then copied to every replica, so adding replicas adds replication work instead of reducing write load.

## Find the bottleneck

Scale the component that breaks first, then look for the next one. Estimation tells you which one it is: compare each number with what one machine can handle.

| Example | Numbers | First move on the data tier |
| --- | --- | --- |
| URL shortener | About 10,000 read QPS, 1,000 write QPS, reads follow 80/20 | **Cache** first, since a small set of hot links gets most reads. Add read replicas for what the cache misses |
| Chat app | About 600,000 peak write QPS, about 11 PB | **Shard** first. The bottleneck is writes and storage, which cache and replicas do not fix |

## When "just buy a bigger machine" is a good answer

- The estimated load fits on one machine with room to grow, for example an internal tool with a few hundred QPS and a few GB of data
- Early-stage products, where shipping fast matters more than handling scale that does not exist yet
- Workloads that need strong consistency or multi-row transactions, which are much simpler on a single node

Even then, vertical scaling does not remove the single point of failure. Keep a standby replica for failover. Capacity and availability are separate problems.

## Follow-up questions

**Why should services be stateless?**
So any server can handle any request. Then you can add or remove servers freely, the load balancer can spread traffic evenly, and a server failure does not lose user state.

**When is vertical scaling enough?**
When the estimated load fits comfortably on one machine, as in the list above. It is the simplest design, and simplicity is worth a lot.

**The database is overloaded. What do you do, in order?**
Find out whether reads or writes are the problem. For reads, add a cache, then read replicas. For writes or storage, shard. Try vertical scaling first if there is headroom, since it costs nothing in complexity.

## Parked

- Auto-scaling policies and metrics
- Database proxies and connection pooling
- Specific sharding strategies (covered in the sharding topic)

## References

- Alex Xu, *System Design Interview: An Insider's Guide*, Vol. 1, Chapter 1
- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 1
