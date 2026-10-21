# URL Shortener (bit.ly)

A URL shortener maps a short code to a long URL. Visiting `short.ly/abc1234` redirects the browser to the long URL. Each link is created once and clicked many times, so the system is read-heavy.

Why the product exists: links fit where length is limited (SMS, social posts, QR codes, print), every click passes through the service so it can count clicks (the main business value), and the destination can be changed without reprinting. Companies also run branded shorteners (`amzn.to`, `youtu.be`).

## Core Challenge

Generating short codes that are unique and hard to guess at scale, and serving a read-heavy redirect workload with low latency and high availability.

## 1. Requirements

### Functional
1. Given a long URL, return a unique short URL.
2. Visiting a short URL redirects to the original long URL.
3. Users can optionally choose a custom alias.
4. Users can optionally set an expiration time.

Click counts are a nice-to-have: kept in mind, not designed in depth.

### Non-functional
- **Scale targets**: 100 million new URLs per day, about 10 reads per write
- **Latency**: redirects should be fast, since users are waiting for a page to load
- **Availability over consistency for redirects**: if redirects go down, every link ever shared breaks. A new link taking a second to become visible everywhere is fine
- **Uniqueness**: two different long URLs must never get the same code (hard correctness requirement)
- **Durability**: links last for years
- **Codes are hard to guess**, so nobody can enumerate all links

### Out of scope
- User accounts
- Editing a link's destination (covered as a follow-up)
- Malicious URL detection (covered as a follow-up)

### How to open the interview

> "Before I start designing, I'd like to clarify the requirements. The core features, I assume, are creating a short URL from a long URL, and redirecting users when they visit the short URL. Should users be able to choose a custom alias? Should links expire? And do we need analytics, like click counts?"

State your own assumptions and let the interviewer confirm them, instead of asking "what features do you want?". Naming the out-of-scope items shows you are controlling scope on purpose.

## 2. Core Entities

- **URL mapping**: short code, long URL, expiration
- **Creator**: who created the link (an API key, since accounts are out of scope)
- **Click event**: for click counts

## 3. API

**Create a short URL**

```
POST /v1/urls
Authorization: Bearer <api_key>

{ "long_url": "https://example.com/...", "custom_alias": "my-event", "expires_at": "2027-01-01T00:00:00Z" }
```

`custom_alias` and `expires_at` are optional. Response `201 Created`:

```json
{ "short_url": "https://short.ly/abc1234", "short_code": "abc1234", "long_url": "https://example.com/...", "expires_at": "2027-01-01T00:00:00Z" }
```

Errors: `400` for an invalid URL, `409 Conflict` if the alias is taken, `429` if a client creates too many links (spammers love URL shorteners).

**Redirect**

```
GET /{short_code}
```

`302 Found` with the long URL in the `Location` header, `404` if the code does not exist, `410 Gone` if the link has expired. The redirect path has no `/v1/` prefix, because the short link itself is the product and must stay short.

**Why 302, not 301**: with a 301, browsers cache the redirect and skip our server on later clicks. Then we cannot count clicks, and **expiration stops working**, since a browser that cached the redirect keeps following it after the link expires. A 302 sends every click through us. The cost is more traffic, which the cache absorbs.

**Same long URL submitted twice**: create a new code each time. Different users or campaigns want separate links to track clicks separately, and deduplication would need an extra index on the long URL. If needed later, add a hash of the long URL as a secondary index.

## 4. High-Level Design

```
                        ┌──────────────┐
  Client ──► DNS ──►    │  L7 Load     │
                        │  Balancer    │
                        └──────┬───────┘
                 POST /v1/urls │ GET /{code}
              ┌────────────────┴────────────────┐
              ▼                                 ▼
     ┌─────────────────┐               ┌─────────────────┐
     │  Write Service  │               │ Redirect Service│
     │  (~1K QPS)      │               │  (~10K+ QPS)    │
     └─┬───────┬─────┬─┘               └──┬──────────┬───┘
       │       │     │                    │          │ click events
       │       │     │ ┌──────────────┐   │          ▼
       │       │     └►│ Redis Cache  │◄──┘    ┌────────────┐
       │       │       └──────────────┘        │  Message   │
       │       ▼              │ miss           │  Queue     │
       │ ┌───────────┐        │                └─────┬──────┘
       │ │  Redis    │        │                      ▼
       │ │  Counter  │        │                ┌────────────┐
       │ └───────────┘        │                │ Analytics  │
       ▼                      ▼                │ Worker     │
     ┌─────────────────────────────┐           └────────────┘
     │  Key-value store            │
     │  (DynamoDB / Cassandra)     │
     │  partitioned by short_code  │
     └─────────────────────────────┘
```

- **L7 load balancer**: routes by path, `/v1/urls` to the write service and `/{code}` to the redirect service.
- **Separate write and redirect services**: reads outnumber writes 10 to 1, and redirects need higher availability. Splitting them lets each scale on its own, and a problem on the write side cannot take redirects down. Both are stateless.
- **Redundancy everywhere**: every box runs more than one instance, so there is no single point of failure.

### Write flow
1. `POST /v1/urls` reaches the write service.
2. Validate the URL and check the client's rate limit.
3. Get a short code: the custom alias if given, otherwise the next ID from the counter (see deep dive).
4. **Conditional write** to the database: insert only if the code does not exist.
5. Return `201` with the short URL.

### Redirect flow
1. `GET /abc1234` reaches the redirect service.
2. Check Redis. On a hit, check `expires_at`, then return `302`.
3. On a miss, read from the database, fill the cache, and redirect.
4. Missing code: `404`, cached briefly to prevent cache penetration. Expired code: `410`.
5. Publish a click event to a message queue, asynchronously. An analytics worker updates click counts in batches, so a redirect never waits on an analytics write.

### Data model

| Field | Type | Notes |
| --- | --- | --- |
| `short_code` | string(7), **primary key** | Also holds custom aliases |
| `long_url` | string | Up to about 2 KB |
| `created_at` | timestamp | |
| `expires_at` | timestamp, nullable | |
| `creator_id` | string | API key that created it, for rate limiting and abuse handling |

**Database choice**: the access pattern is insert a row and look one up by `short_code`, with no joins or multi-row transactions. That is a key-value pattern, so a key-value store such as DynamoDB fits, partitioned by `short_code`. Postgres would be fine at first, since 1,000 writes per second is easy for one node, but storage grows to hundreds of terabytes (see estimates), so it would need sharding eventually. A key-value store has sharding and replication built in.

**Custom alias uniqueness**: a conditional write (DynamoDB) or a UNIQUE constraint (SQL). If two users ask for the same alias at once, exactly one write succeeds and the other gets `409`.

**Expiration**: check `expires_at` on every redirect and return `410` when it has passed. For cleanup, use the database's TTL feature or a background job. The read-time check guarantees correctness. Cleanup only frees space and can lag (DynamoDB TTL deletion can take days).

**Cache**: cache-aside with LRU, since links are hottest right after they are shared. A long TTL is fine because mappings rarely change, but cap it at the link's expiration.

## 5. Deep Dives

### Depth plan

| Component | Level | Reason |
| --- | --- | --- |
| Short code generation | L2 | The core challenge: unique and hard to guess |
| Cache | L2 (covered in the design) | Read-heavy workload, see [`../fundamentals/caching.md`](../fundamentals/caching.md) |
| Database | L1 | Simple key lookup |
| Analytics pipeline | L1 | Nice-to-have, not the focus |

### Short code generation

| Option | How | Problems |
| --- | --- | --- |
| A. Hash plus truncate | SHA-256 the long URL, base62-encode, keep 7 characters | Truncation causes collisions, so it needs a unique constraint and a salted retry. Same URL gives the same code unless salted |
| **B. Counter plus base62** | Each new URL gets the next integer, encoded in base62 | Unique by construction. But a single counter is a bottleneck, a single point of failure, and guessable |
| C. Random code plus conditional write | Random 7-character base62 string, insert if not exists, retry on collision | Unguessable and needs no coordination, but retries grow as the space fills |

**Choice: B**, the standard answer in both Hello Interview and Alex Xu, with its three weaknesses fixed:

**1. Bottleneck: counter batching.** Keep the counter in Redis. Each write server calls `INCRBY counter 1000` to reserve a range of 1,000 IDs that only it will use, hands them out from memory, and returns to Redis only when the batch runs out. At 1,000 writes per second, Redis sees about one call per second. If a server crashes, the rest of its batch is skipped, which is harmless with 3.5 trillion codes available.

**2. Single point of failure: replicate, and never reuse IDs.** Run Redis with a replica and automatic failover (for example Redis Sentinel). Redis replication is asynchronous, so after a failover the new primary may not have seen the last increments and could hand out a range again, creating duplicate codes. Two safeguards:
- After a failover, bump the counter forward by a safe margin (a few million) before serving again.
- Keep the conditional write in the database as the final safety net. A duplicate insert fails, and the server takes the next ID.

Rule: **skipping IDs is harmless, reusing them is not.**

**3. Guessable: reversible transformation.** Sequential counters give sequential codes, so anyone could enumerate links. Before encoding, pass the counter through a reversible transformation with a secret key, such as multiplying by a large secret number modulo 62^7, or XOR with a secret key. Because it is a one-to-one mapping, every counter value still produces a different code, but neighboring values produce codes that look unrelated.

**Custom aliases vs. generated codes**: an alias could equal a code the counter produces later. The conditional write catches it (the generated code is taken, so take the next ID). Also keep namespaces apart: generated codes are exactly 7 characters, custom aliases must be at least 8.

**Multiple regions**: give each region a disjoint range, for example a region ID in the top bits of the counter, so regions never coordinate on each write.

**When to reconsider**: option C is a valid alternative. After 10 years the code space is about 10% full, so a random code collides about 10% of the time and needs about 1.1 attempts on average. It is simpler to run (no counter service), and interviewers accept it if the collision math is stated. Hello Interview rates the hash with salted retry, which behaves the same way, as a great solution too.

**Why not Snowflake IDs?** They are 64-bit, about 11 base62 characters, which is too long for a short link.

### Estimates used

| Question | Math | Decision |
| --- | --- | --- |
| How long must codes be? | 100M per day x 365 x 10 years = about 365 billion. 62^6 is about 57 billion (too few), 62^7 is about 3.5 trillion | 7 characters |
| Write load on the counter | 100M per day / 10^5 seconds = about 1,000 writes per second. With batches of 1,000, about 1 Redis call per second | Batching makes one Redis counter enough |
| Read load | About 10,000 redirects per second, more at peak | Cache is the main lever, redirect service scales separately |
| Storage | About 500 B per row x 365 billion = about 180 TB (about 550 TB with 3 replicas) | Needs sharding, so a key-value store fits |
| Cache size | 20% of daily reads: 10^9 x 500 B x 0.2 = about 100 GB | Fits in a few Redis nodes |
| Collisions for option C | 365 billion / 3.5 trillion = about 10% full, so about 1.1 attempts | Option C would also work |

## 6. Trade-offs and Alternatives

| Decision | Chosen | Alternative | Why |
| --- | --- | --- | --- |
| Short code generation | Counter + batching + reversible transformation | Random code + conditional write, hash + salted retry | Unique by construction, no retries |
| Redirect status | 302 | 301 | 301 is cached by browsers, which breaks click counts and expiration |
| Database | Key-value store | Postgres, sharded later | Pure key lookups, and storage reaches hundreds of TB |
| Cache | Cache-aside, LRU, TTL capped at expiration | Write-through | Hot links stay in memory, expired links do not linger |
| Services | Separate write and redirect services | One service | 10:1 traffic gap, redirects need higher availability |
| Analytics | Async through a message queue | Write on every click | A redirect never waits for an analytics write |

## 7. Follow-up Questions

**A link goes viral at 100 times normal traffic. What breaks?**
The hot key concentrates on one Redis node. Add a small in-process cache on each redirect server for the hottest codes, and serve redirects from a CDN or edge so most clicks never reach our servers. If the edge caches redirects, it must report clicks back to keep counts accurate. If the cache entry expires under load, use request coalescing to avoid a cache stampede.

**How do you handle links to malware?**
Check new URLs against a blocklist such as Google Safe Browsing before creating them, rate limit creation per API key, and support disabling a code: delete it from the cache and purge it from the CDN.

**What if users can edit a link's destination?**
Update the database, delete the cache key, and purge the CDN, in that order.

**What happens if the Redis counter fails?**
Failover to the replica, bump the counter forward before serving, and rely on the conditional write to reject any duplicate that slips through. Write servers can keep serving from their current batch during a short outage.

**Same long URL twice: same code or new?**
New code each time, so different users and campaigns can track clicks separately. Deduplication is possible with a hash index on the long URL.

## Parked

- The math of reversible transformations (modular inverses, Feistel networks)
- Redis persistence settings (AOF fsync) and how they affect counter durability
- Detailed analytics pipeline design (stream processing, aggregation windows)
- CDN edge compute for redirects

## References

- Hello Interview, "Design a URL Shortener Like Bit.ly": https://www.hellointerview.com/learn/system-design/problem-breakdowns/bitly
- Alex Xu, *System Design Interview* Vol. 1, Chapter 8, "Design a URL Shortener": https://bytebytego.com/courses/system-design-interview/design-a-url-shortener
- Related notes: [`../fundamentals/caching.md`](../fundamentals/caching.md), [`../fundamentals/api-design.md`](../fundamentals/api-design.md), [`../fundamentals/load-balancing.md`](../fundamentals/load-balancing.md), [`../fundamentals/back-of-the-envelope-estimation.md`](../fundamentals/back-of-the-envelope-estimation.md)
