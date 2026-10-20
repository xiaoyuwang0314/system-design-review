# SQL vs. NoSQL and How to Choose

## Relational databases (SQL)

PostgreSQL, MySQL. Key properties:
- **Fixed schema**: tables with typed columns
- **Joins** across tables
- **ACID transactions**
  - **Atomicity**: a group of operations either all succeed or none happen
  - **Consistency**: constraints hold before and after (for example, a balance cannot go negative)
  - **Isolation**: concurrent transactions do not interfere with each other
  - **Durability**: once committed, data is not lost
- Strong support for secondary indexes and ad-hoc queries, and decades of maturity

## The four NoSQL families

| Type | Examples | Data shape | Good for |
| --- | --- | --- | --- |
| **Key-value** | Redis, DynamoDB | One value per key | Sessions, caches, shopping carts |
| **Document** | MongoDB | JSON documents, each with its own structure, nested data read together | Product catalogs, user profiles, content management |
| **Wide-column** | Cassandra, HBase, ScyllaDB | Partition key plus clustering key, rows sorted within a partition | Chat messages, IoT and time-series data, activity logs. Very high write volume |
| **Graph** | Neo4j | Nodes and relationships | Friends of friends, recommendations, fraud detection |

## How to choose

1. **What are the access patterns?** Simple lookups by key, known in advance, point toward NoSQL. Flexible queries and joins point toward SQL.
2. **Do you need transactions across rows or entities?** Money and inventory do, so use SQL.
3. **How big is it?** If writes or storage exceed one machine and the access pattern is simple, consider wide-column or key-value.
4. **How much does the schema vary?** If records have very different fields, consider a document store.

Default: **"I'd start with a relational database like Postgres unless there's a specific reason not to."** Then use the estimation numbers or the access pattern to justify switching.

## Two common myths

- **"SQL doesn't scale."** Sharded MySQL is common. YouTube and Slack use Vitess to shard MySQL, and Slack stores its messages this way. NewSQL databases (Spanner, CockroachDB) offer SQL, horizontal scaling, and strong consistency, at the cost of higher latency.
- **"NoSQL has no transactions."** MongoDB supports multi-document transactions, and DynamoDB has transactions too. The real difference is in the defaults and in which operations are cheap.

## Data modeling works in opposite directions

- **SQL**: model the entities, normalize to remove duplication, and join at query time.
- **NoSQL**: **start from the queries, then design tables around them.** Duplicate data (denormalize) so that each query reads a single partition.

This is the same rule as in the estimation notes: choose the partition key based on the most common read.

## One system, several databases

A chat app might use:
- **Postgres** for users and accounts, which need transactions and uniqueness constraints
- **Cassandra** for messages, which have huge write volume and are read per conversation
- **Redis** for presence and sessions, which need speed and can be rebuilt if lost

This is called **polyglot persistence**.

## Worked example: e-commerce

| Data | Choice | Why |
| --- | --- | --- |
| Orders and payments | **SQL** (Postgres) | Placing an order touches several tables at once (order, payment, inventory), so it needs ACID transactions. Money must never be half-written |
| Product catalog | **Document store** (MongoDB) | Electronics and clothing have very different attributes. Each product is read as one whole document. Postgres with a JSONB column is a reasonable alternative that keeps everything in one database |
| Shopping cart | **Key-value store** (DynamoDB, or Redis with persistence) | Small, one per user, always read and written as a whole by `user_id`. Users expect the cart to survive, so it needs durable storage rather than a pure cache |

Why not Cassandra for the cart? Wide-column stores shine with huge write volumes and time-ordered data inside a partition. A cart has neither, so Cassandra adds operational cost without a benefit.

## Worked example: "Why not MySQL for chat messages?"

> "At our scale, peak writes are around 600,000 per second and storage grows to about 11 PB with replication. A single MySQL node handles thousands of writes per second and a few terabytes, so we would need hundreds of shards and would have to run that sharding ourselves. The access pattern is simple: append a message and read the latest messages of one conversation, with no joins. A wide-column store like Cassandra handles that pattern with built-in sharding and replication, so it's a better fit."

When MySQL is fine for chat:
- **Small scale.** A team chat with 10,000 users sending 40 messages a day is 400,000 messages a day, about 5 writes per second and about 40 MB a day. One MySQL instance with a replica handles this easily.
- **Large scale with a sharding layer.** Slack stores messages in MySQL sharded with Vitess. Choosing it is an operational trade-off, not a hard limit.

## Worked example: a query that does not match the partition key

Messages are partitioned by `conversation_id`. A new feature asks: "show all messages a user sent in the past week."

**Problem**: a user's messages are spread across every conversation they are in, so the query must hit many partitions on many nodes (scatter-gather), or scan everything.

**Changing the partition key to `user_id` is wrong.** It would break the main read, loading a conversation, which would then become the scatter-gather query instead.

Options:

| Option | How | Trade-off |
| --- | --- | --- |
| Second table, denormalized | Also write each message to `messages_by_user`, partitioned by `sender_id` and clustered by time | Every message is written twice, and the two tables can briefly disagree (eventual consistency) |
| Search index | Stream messages asynchronously into a search system such as Elasticsearch | Another system to run, and results lag slightly. Required anyway if the feature needs full-text search |

Use the requirement to cut the cost: the feature only needs **the past week**, so give the second table a **7-day TTL**. Writes double, but storage for it stays at about 7 days of messages instead of 5 years.

## Follow-up questions

**When must you use SQL?**
When you need transactions across multiple rows or entities (money, inventory, bookings), strong uniqueness constraints, or flexible queries and joins that you cannot predict in advance.

**Why is Cassandra good for chat?**
Very high write throughput (LSM-tree storage), built-in sharding and replication, and a data model where one conversation is one partition sorted by time, so "latest 50 messages" is a single range scan.

**A new query does not fit your partition key. What do you do?**
Do not change the primary key, since it serves the main read. Add a denormalized table keyed for the new query, or feed a search index. Use the feature's requirements, such as a time window, to limit the extra storage.

## Parked

- Isolation levels in detail (covered in the transactions topic)
- How LSM trees and B-trees work internally (covered in the indexing topic)
- Vitess and how sharding layers route queries

## References

- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 2 (Data Models and Query Languages)
- Slack Engineering, "Scaling Datastores at Slack with Vitess" (2020)
