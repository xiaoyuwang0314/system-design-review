# Alignment with Standard Prep Resources

Checked on 2026-10-20 to make sure the framework, depth, and topic list in this repo match widely accepted interview prep material.

## Resources compared

| Resource | Role | Access |
| --- | --- | --- |
| **Hello Interview** (HI), "System Design in a Hurry" and problem breakdowns | Most current, explicit expectations per level, options rated Bad / Good / Great | Read in full |
| **Alex Xu**, *System Design Interview* Vol. 1 (ByteByteGo) | The classic book. Its chapters map almost one to one onto our upcoming problems | Read in full (framework and URL shortener chapters) |
| donnemartin/system-design-primer | Free topic index, not a full framework | Read in full |
| *Designing Data-Intensive Applications* | Deep reference for replication, partitioning, consistency | Reference only |
| Grokking (DesignGurus), Exponent | Other popular courses | Search summaries only |

The most common pairing in community discussions is Alex Xu plus Hello Interview, though there is no hard count behind that.

## Frameworks

| Source | Steps | Estimation |
| --- | --- | --- |
| Hello Interview | Requirements (about 5 min), Core Entities (about 2 min), API (about 5 min), optional Data Flow, High-Level Design (10 to 15 min), Deep Dives (about 10 min) | Only when it directly influences the design. Scale targets go in the non-functional requirements, and math happens inside the design when needed |
| Alex Xu | Scope the problem (3 to 10 min), high-level design and buy-in (10 to 15 min), deep dive (10 to 25 min), wrap-up (3 to 5 min) | Used to check the design against scale, after checking with the interviewer |
| System Design Primer | Use cases and constraints, high-level design, core components, scale the design | "You might be asked to do some estimates by hand" |
| Grokking | Seven steps, with estimation as step 3 | Upfront |

## Level expectations (Hello Interview)

- Most entry-level roles do not have a system design interview at all.
- Every level must complete a full design that satisfies the requirements.
- Mid-level: covers the basics well, without great depth. In deep dives, the interviewer may jump in and guide.
- Senior: finds the weak spots and leads the deep dives without prompting.

## URL shortener comparison

| Topic | Hello Interview | Alex Xu | Our first draft |
| --- | --- | --- | --- |
| ID generation | Hash plus UNIQUE constraint plus salted retry, or a counter plus base62. Final design: Redis `INCR` with counter batching | Base62 conversion of a unique ID from a distributed ID generator | Random 7-character code plus conditional write |
| Guessable codes | Reversible transformation such as XOR with a secret key | Flagged as a concern | Avoided with random codes |
| 301 vs. 302 | 302, since browsers do not cache it | 302 if analytics matter | 302 |
| Expired links | 410 Gone | Out of scope | 410 Gone |
| Database | Most databases work. About 500 GB fits Postgres, shard later | Relational table | Key-value store |
| Cache TTL | Should match or be shorter than the link's expiration | Not specified | Long TTL |
| Extras | CDN or edge redirects, read/write split, counter failover, disjoint counter ranges per region | Rate limiting, stateless web tier, replication and sharding, analytics | Click events through a message queue |

## Decisions

- **Spine**: Hello Interview is the primary framework and level bar. Alex Xu Vol. 1 is the secondary source for the problem list and estimation method. DDIA is the deep reference.
- **Estimation**: scale targets go in the non-functional requirements. Do the math only when it drives a decision (for example, short code length or whether sharding is needed). The estimation notes stay as a backup skill, since some interviewers still ask for upfront numbers.
- **Problem template**: add a short Core Entities step before the API, following Hello Interview.
- **URL shortener ID generation**: a counter with batching is the primary answer, matching both sources. Random code plus conditional write stays as the alternative, with its collision math.
- **Roadmap**: add a patterns and key technologies phase before the core problems: Scaling Reads, Scaling Writes, Realtime Updates, plus Redis, Kafka, Postgres, DynamoDB, Cassandra, and Elasticsearch.

## Sources

- Hello Interview, Delivery framework: https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery
- Hello Interview, Introduction: https://www.hellointerview.com/learn/system-design/in-a-hurry/introduction
- Hello Interview, Core concepts: https://www.hellointerview.com/learn/system-design/in-a-hurry/core-concepts
- Hello Interview, Bit.ly breakdown: https://www.hellointerview.com/learn/system-design/problem-breakdowns/bitly
- ByteByteGo, A framework for system design interviews: https://bytebytego.com/courses/system-design-interview/a-framework-for-system-design-interviews
- ByteByteGo, Design a URL shortener: https://bytebytego.com/courses/system-design-interview/design-a-url-shortener
- System Design Primer: https://github.com/donnemartin/system-design-primer
- Blind, best resources for system design prep (2025-04): https://www.teamblind.com/post/best-resources-for-system-design-prep-wcunzr80
