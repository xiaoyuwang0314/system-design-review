# Roadmap

I am a new grad preparing for NG interviews in general, with no specific interview scheduled. The goal is to reach a level where I can handle a design round at any time, then stay there with light, regular practice.

I target the mid-level bar as a safety margin above NG expectations. The reasoning, based on interview reports from the past year, is in [`research/ng-interview-landscape-2026.md`](research/ng-interview-landscape-2026.md). Depth targets follow [`DEPTH-GUIDE.md`](DEPTH-GUIDE.md).

The roadmap is organized by phases instead of dates. I move to the next phase only after meeting its exit criteria. Mock feedback decides what to revisit.

## Phase 1: Foundations

Build the vocabulary and reasoning that every problem relies on.

- [ ] Back-of-the-envelope estimation
- [ ] Scalability: vertical vs. horizontal scaling
- [ ] Availability, reliability, and SLAs
- [ ] CAP theorem and PACELC
- [ ] Consistency models
- [ ] SQL vs. NoSQL and how to choose

**Exit criteria**
- [ ] Given a prompt, produce rough QPS and storage estimates in a few minutes
- [ ] Explain CAP and the main consistency models in two minutes each, with a real system as an example

## Phase 2: Building blocks and warm-up problems

Alternate between learning a building block and solving a problem that uses it.

| Learn | Then solve |
| --- | --- |
| Load balancing, API design, caching | URL shortener |
| Blob and object storage | Pastebin |
| Rate limiting | Rate limiter |
| Unique ID generation | Unique ID generator |
| Replication, sharding, consistent hashing | Key-value store |

Also cover along the way: DNS, CDN, indexing.

**Exit criteria**
- [ ] Solve one warm-up problem alone in 45 minutes, then write it up with the problem template
- [ ] Complete the first mock interview and log it

## Phase 3: Core problems and regular mocks

Practice the patterns that appear most often, under time pressure.

| Learn | Then solve |
| --- | --- |
| Fan-out strategies (review caching) | News feed |
| Real-time communication | Chat system |
| Message queues and event streaming | Notification system |
| Tries and precomputation (review indexing) | Search autocomplete |
| Task scheduling and deduplication | Web crawler |
| Review caching and replication | Distributed cache |
| Review message queues | Distributed message queue |

Mix in the Component and API Design problems from [`problems/README.md`](problems/README.md) along the way. They are short, so one can fit between two Core problems. In-memory pub/sub and log ingestion fit well right after message queues.

Run a mock about every two problems.

**Exit criteria**
- [ ] Pass a 45-minute mock on a Core problem I have not practiced before
- [ ] Pass a mock on a Component and API Design problem I have not practiced before
- [ ] For every Core problem, state the core challenge and the main trade-off from memory

## Phase 4: Stay ready

Once Phase 3 is done, switch to maintenance.

- One mock every one to two weeks, rotating through problems
- Revisit any topic that comes up as weak in mock feedback
- Stretch goals, in any order:
  - [ ] Advanced problems from [`problems/README.md`](problems/README.md)
  - [ ] Distributed Systems topics at L2 (consensus, distributed transactions, clocks)
  - [ ] Monitoring, microservices vs. monolith, security basics

When a real interview gets scheduled, go back over the core challenge of every practiced problem and add a few mocks targeted at that company's product area.
