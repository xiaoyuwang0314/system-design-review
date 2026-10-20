# CAP Theorem and PACELC

## The three letters

| Letter | Meaning |
| --- | --- |
| **C**onsistency | Every read returns the most recent write, as if there were only one copy of the data. The precise term is **linearizability** |
| **A**vailability | Every request to a node that has not failed gets a non-error response, possibly with stale data |
| **P**artition tolerance | The system keeps working when the network between nodes drops messages |

## What CAP actually says

Network partitions will happen in any distributed system, so P is not optional. The real question is: **during a partition, do you give up consistency or availability?**

- **CP**: reject some requests or return errors, so nobody gets wrong data
- **AP**: keep answering, possibly with stale data, and reconcile after the partition heals

"Pick two of three" is misleading. A distributed system without partitions does not exist, so "CA" is not a real option.

## Choose per feature, not per system

| Feature | Choice | Why |
| --- | --- | --- |
| Booking the last concert seat, charging a card | CP | Selling one seat twice or charging twice costs far more than asking the user to retry |
| Like counts, view counts, feeds | AP | Nobody notices a count that is a few seconds behind, but everyone notices an error page |

One flow can mix both. In ticket booking, the seat map can be AP: it may show a seat as free when it was just taken. The actual reservation must be CP. During a partition, the side that cannot confirm the seat with a majority rejects the booking ("please try again") instead of selling the seat twice.

What users see during a partition:
- **Ticket booking (CP)**: a booking attempt fails or asks them to retry. No seat is oversold.
- **View counter (AP)**: the page loads normally. The count is slightly off and catches up later.

Common labels, with a caveat: ZooKeeper, etcd, and Spanner lean CP. Cassandra and DynamoDB lean AP by default but let you tune consistency per request. In interviews, say "AP by default, tunable to stronger consistency" rather than treating the label as fixed.

## PACELC

- **If Partition**: choose **A**vailability or **C**onsistency
- **Else** (normal operation): choose **L**atency or **C**onsistency

Even with a healthy network, strong consistency means waiting for replicas to confirm, which adds latency. So the trade-off exists all the time, not only during partitions.

| System | PACELC | Meaning |
| --- | --- | --- |
| Cassandra, DynamoDB | PA/EL | Available during partitions, low latency otherwise |
| Spanner | PC/EC | Consistent during partitions, and consistent otherwise at the cost of latency |

## Follow-up questions

**Is system X CP or AP?**
Answer per operation, not per product. Then explain what the user sees during a partition, because that is what the label means in practice.

**Why is "CA" not a real choice?**
It describes a system where partitions never happen, which only exists on a single machine. Once data lives on more than one node, partitions are possible and you must decide what happens when one occurs.

**If there is no partition, is there still a trade-off?**
Yes, latency vs. consistency (the "ELC" part of PACELC). Waiting for replicas makes writes slower.

## Parked

- Formal definitions and the proof of CAP
- How Spanner uses TrueTime to get external consistency

## References

- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 9 (Consistency and Consensus)
- Daniel Abadi, "Consistency Tradeoffs in Modern Distributed Database System Design" (2012), which introduced PACELC
