# Caching

## Where caches live

| Layer | Examples | Notes |
| --- | --- | --- |
| Client | Browser cache, in-app memory | Fastest, no network at all |
| CDN | CloudFront, Cloudflare | Static content close to users (separate topic) |
| Server-local | In-process memory on each app server | Very fast, but each server has its own copy and they drift apart |
| **Distributed cache** | **Redis**, Memcached | Shared by all servers. **What "the cache" usually means in interviews** |
| Database | Buffer pool | Built in, not something you design |

Redis vs. Memcached (L1 is enough): Redis has richer data structures (lists, sorted sets, hashes), optional persistence, and replication. Memcached is plain key-value and simpler. Default to Redis in interviews.

Why it helps: memory is about 100 ns and SSD about 100 us, and a database query adds its own processing, so a read usually takes several milliseconds. A cache hit costs one network round trip, about 0.5 ms.

## Read and write strategies

| Strategy | Reads | Writes | Notes |
| --- | --- | --- | --- |
| **Cache-aside** (lazy loading) | App checks the cache. On a miss, it reads the DB and fills the cache | App writes the DB, then **deletes** the cache key | **Most common.** Only data that is actually read gets cached. If the cache dies, the system still works, just slower |
| Read-through | The cache itself loads from the DB on a miss | Same as above | Like cache-aside, but the loading logic lives in the cache layer |
| **Write-through** | Same as above | Write cache and DB together, return when both succeed | Cache is always fresh, but writes are slower and the cache fills with data nobody reads |
| **Write-behind** (write-back) | Same as above | Write only the cache, flush to the DB **asynchronously** in batches | Very fast writes, but **data is lost** if the cache dies before flushing |
| Write-around | Same as above | Write only the DB, skip the cache | For data that is rarely read right after being written |

Interview default: **cache-aside.** Consider write-behind only for very high write volume where losing a little data is acceptable (view counts, for example).

## Eviction

- **LRU** (least recently used): evict what has gone unused the longest. **The usual default.**
- **LFU** (least frequently used): evict what is used least often. Good when popular items stay popular for a long time.
- **TTL** (time to live): each key expires after a set time. It is both an eviction tool and a **consistency safety net**: if an invalidation is ever missed, the stale value still disappears after the TTL.

## Invalidation

When data changes in the DB, the common pattern is **update the DB, then delete the cache key**. The next read loads the fresh value.

Delete rather than update the cache: with two concurrent writers, updates to the cache can land in a different order than updates to the DB. Deleting has no "last writer wins" problem.

### Why "update cache, then DB" is worse

1. **The DB write fails**: the cache already has the new value, so readers see data that does not exist in the DB.
2. **Concurrent writers**: writers 1 and 2 can interleave so the cache ends with writer 2's value and the DB with writer 1's. They stay inconsistent until the TTL expires.

### The race that remains with "update DB, then delete cache"

| Time | Reader A | Writer B |
| --- | --- | --- |
| 1 | Cache miss, reads the **old** value from the DB | |
| 2 | | Updates the DB to the new value |
| 3 | | Deletes the cache key |
| 4 | Writes the **old** value into the cache | |

The cache now holds the old value until the TTL expires. The window is small, because a read is usually faster than a write, so a whole write rarely fits between steps 1 and 4. But it can happen.

Mitigations:
- **TTL as a safety net**: the value is stale for at most one TTL
- **Delayed double delete**: delete, then delete again a few hundred milliseconds later
- **Versioning**: store a version with each value, and never let an older version overwrite a newer one

A strong interview answer: "Update the DB, then delete the cache key. There's still a small race that can leave a stale value, so I keep a TTL as a safety net."

## Three classic failures

| Problem | What happens | Fixes |
| --- | --- | --- |
| **Cache stampede** (thundering herd) | A **hot key** expires and many requests miss at once, all hitting the DB | **Request coalescing / single-flight**: one request loads from the DB, the rest wait for its result. **Locking**: only the lock holder loads. **Stale-while-revalidate**: serve the old value and refresh in the background. **No TTL on hot keys**, refreshed by a background job instead |
| **Cache avalanche** | **Many keys expire at the same moment** (for example, they were all loaded together), and the DB is overwhelmed | Add **jitter** to TTLs, such as 1 hour plus or minus 5 minutes, so expirations spread out |
| **Cache penetration** | Many requests ask for **keys that do not exist**. Every one misses and hits the DB. Can be an attack | **Cache the miss**: store "not found" with a short TTL. **Bloom filter**: check whether the key could exist before going to the DB |

## Estimating the effect

**Effective latency** = hit rate x cache latency + (1 - hit rate) x DB latency

Example: 90% hit rate, cache 1 ms, DB 10 ms.
> 0.9 x 1 + 0.1 x 10 = **1.9 ms**, and the DB sees only 10% of reads

Going from 90% to 99% cuts DB reads from 10% to 1%, another 10 times. Each extra nine of hit rate matters.

## Worked example: URL shortener redirect

A URL shortener (bit.ly, TinyURL) maps a short code to a long URL. Visiting `short.ly/abc123` redirects the browser to the long URL. Each link is created once and clicked many times, so it is read-heavy.

Read path with cache-aside:
1. The request reaches an app server through the load balancer.
2. The app server looks up `abc123` in Redis.
3. **Hit**: return `302` with the long URL in the `Location` header.
4. **Miss**: look up the short code (the primary key) in the DB, write the result to the cache with a TTL, then return `302`.
5. **Not in the DB either**: return `404`, and cache the "not found" with a short TTL to prevent cache penetration.

Choices:
- **LRU**: links are hottest right after they are shared and cool down over time, so recency predicts the next click well.
- **Long TTL** (a day or more): the mapping almost never changes, so cached values rarely go stale. The TTL is mostly a safety net, and LRU handles memory. If the product lets users edit the destination, delete the cache key on edit.
- **Click analytics**: do not write to the DB on every click, which would turn every read into a write. Send click events to a message queue and process them in batches.

> "On a redirect, I check Redis first. On a hit, I return a 302 with the long URL. On a miss, I look it up in the database by short code, write it back to the cache, then redirect. If it doesn't exist, I return 404 and cache that too, with a short TTL, to avoid cache penetration. I'd use LRU, because links are hottest right after they're shared and cool down over time. And since the mapping almost never changes, the TTL can be long. It's mostly a safety net."

## Follow-up questions

**A viral link hits 100K QPS and its cache entry expires. What happens?**
A cache stampede: every request misses and goes to the DB at once. Use request coalescing so only one request reloads it, or keep hot keys without a TTL and refresh them in the background.

**Update cache then DB, or update DB then delete cache?**
Update the DB, then delete the cache. The other order can leave values in the cache that never reached the DB. A small race remains, so keep a TTL.

**When would you not use a cache?**
When data changes constantly and must always be fresh, when every read is unique (no reuse, so the hit rate is near zero), or when the DB already handles the load comfortably.

## Parked

- Redis internals (single-threaded event loop, persistence modes)
- Distributed cache sharding and replication (covered with the distributed cache problem)
- Cache warming strategies before launches

## References

- Alex Xu, *System Design Interview: An Insider's Guide*, Vol. 1, Chapter 1
- AWS, "Caching Best Practices" and ElastiCache documentation (caching strategies)
- Facebook, "Scaling Memcache at Facebook" (NSDI 2013), on leases for stampedes and stale sets
