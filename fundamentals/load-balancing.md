# Load Balancing

## What a load balancer does

- **Distributes traffic** across servers, which is what makes horizontal scaling work
- **Runs health checks**, removes failed servers from rotation, and adds them back when they recover
- **Provides one entry point**: clients only know the load balancer's address, so servers can be added or removed without clients noticing
- **Terminates TLS**, so HTTPS encryption and decryption happen once at the load balancer instead of on every server

## L4 vs. L7

| | L4 (transport layer) | L7 (application layer) |
| --- | --- | --- |
| Routes on | IP address and port only. It never reads the request content | HTTP content: URL path, headers, cookies |
| Speed | Very fast, low overhead | Must parse each request, so more CPU |
| Can do | Forward any TCP or UDP traffic: WebSocket, games, database connections | Path-based routing (`/api` and `/static` to different services), cookie-based sticky sessions, rate limiting |
| Examples | AWS NLB | AWS ALB, Nginx, Envoy |

Rule: **if routing depends on what is inside the request, use L7. If you only need speed and generic forwarding, use L4.**

L4 can still use smart algorithms such as least connections. Its limit is that it cannot see the request. With HTTPS the payload is encrypted, so an L4 load balancer that does not terminate TLS cannot read the URL path at all. An L7 load balancer terminates TLS first, then parses HTTP.

> "I'd use an L7 load balancer, because routing by URL path requires reading the HTTP request. An L4 load balancer only sees IPs and ports, so it can't tell `/api` from `/static`."

## Algorithms

| Algorithm | How it works | Good for |
| --- | --- | --- |
| **Round robin** | Each server in turn | Identical servers, requests of similar cost |
| **Weighted round robin** | In turn, but bigger servers get more | Servers with different capacity |
| **Least connections** | Send to the server with the fewest active connections | **Long-lived connections**, or requests whose cost varies a lot |
| **IP hash / consistent hashing** | Hash the client or a key, so the same client always reaches the same server | Stickiness, or better hit rates for per-server local caches |
| **Power of two choices** | Pick two servers at random, send to the less loaded one | Large fleets. Close to least connections at lower cost (L1 is enough) |

Consistent hashing has its own topic later. Its key property: adding or removing a server moves only a small share of keys.

### Example: WebSocket connections that last hours

Use **least connections**. Round robin evens out the number of *new* connections, but connections last for hours and their lengths vary, so the number of *active* connections per server drifts apart.

It matters most after scaling out. Existing WebSocket connections never move, so with round robin a newly added server only gets its equal share of new connections and stays underloaded for a long time. With least connections, the new server has the fewest connections, so new ones go to it first and the fleet rebalances quickly.

## Health checks

- **Active**: the load balancer calls each server's `/health` every few seconds. After N consecutive failures it removes the server, after M consecutive successes it adds it back.
- **Passive**: the load balancer watches real traffic and stops sending to servers with high error rates.

### Pitfall: health checks that are too deep

Scenario: `/health` queries the database, and the database stalls for 10 seconds.

1. Every app server's health check fails at the same moment.
2. The load balancer removes **all** servers from rotation.
3. The whole service returns errors, even though the app servers were fine and the database only had a short hiccup.
4. After the database recovers, each server still needs M successful checks before it gets traffic again, so the outage lasts longer than the hiccup.

A brief problem in one dependency turns into a full outage. This is a **cascading failure**.

Better design:

| Practice | Why |
| --- | --- |
| **Shallow health check** for the load balancer: "is this process up and able to serve requests?" | The load balancer should remove a server only when *that server* is broken |
| **Deep dependency checks** go to monitoring and alerting, not to the load balancer | Dependency problems should page an engineer, not pull every server out of rotation |
| **Graceful degradation** when a dependency is down: return a clear error, serve cached data, or disable one feature | Some service is better than no service |
| **Fail open**: if every backend looks unhealthy, keep sending traffic to all of them | Many managed load balancers (for example AWS ALB) do this, since "all servers broken at once" is more likely a health check problem |

Kubernetes makes the same split with **liveness probes** (should this container be restarted?) and **readiness probes** (should it receive traffic right now?).

## The load balancer is a single point of failure

The load balancer is in series with everything behind it, so it needs redundancy too (see the availability notes):
- **Active-passive pair with a floating IP (virtual IP)**: when the primary fails, the IP moves to the standby (failover)
- **DNS pointing to several load balancers**
- **Managed cloud load balancers** (ALB, NLB) are redundant internally

## Where load balancers sit

> Client, then DNS, then (CDN), then **L7 load balancer**, then app servers, then **internal load balancer**, then internal services

Databases are usually **not** behind a generic load balancer. Writes must go to the leader and only reads can spread across replicas, so the application or a dedicated database proxy handles that routing.

## Follow-up questions

**L4 or L7 for a chat app's WebSocket traffic?**
Either works. L4 is simpler and cheaper for long-lived connections. L7 is needed if you want to route by path or inspect the upgrade request (for example, authenticate it). Use least connections either way.

**How does the load balancer avoid being a single point of failure?**
Run it redundantly: an active-passive pair sharing a floating IP, multiple load balancers behind DNS, or a managed load balancer that is redundant internally.

**What should a health check verify?**
That this server can serve requests. Keep it shallow. Report dependency health to monitoring, not to the load balancer.

## Parked

- Global server load balancing (GSLB) and anycast
- Connection draining during deploys
- Load balancer algorithms in detail (power of two choices, maglev hashing)

## References

- Alex Xu, *System Design Interview: An Insider's Guide*, Vol. 1, Chapter 1
- AWS Elastic Load Balancing documentation (ALB vs. NLB, health checks)
- Kubernetes documentation, "Configure Liveness, Readiness and Startup Probes"
