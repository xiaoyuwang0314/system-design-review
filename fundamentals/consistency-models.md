# Consistency Models

## From strongest to weakest

| Model | Guarantee | Typical use |
| --- | --- | --- |
| **Strong** (linearizable) | Everyone reads the latest value | Account balances, seat inventory |
| **Causal** | Events that depend on each other are seen in the same order by everyone | A reply never appears before the comment it answers |
| **Read-your-writes** | A user always sees their **own** writes immediately. Others may lag | After editing a profile, the user sees the new version |
| **Monotonic reads** | A user never sees data go back in time | Refreshing never shows fewer comments than before |
| **Eventual** | If writes stop, all replicas converge to the same value | Like counts, view counts |

Interview habit: do not declare "strong consistency" for the whole system. Pick the **weakest model each feature can tolerate**. Weaker models are faster and more available.

## Read-your-writes

Scenario: a user edits their bio and refreshes, but sees the old bio. The database is one leader with asynchronous read replicas. The write reached the leader, and the read went to a replica that had not caught up yet. This violates **read-your-writes**.

(It is not a monotonic reads violation unless the user had already seen the new bio and then saw the old one.)

Fixes:

| Fix | How | Limitation |
| --- | --- | --- |
| Read from the leader after a write | For a short window after a user writes (for example one minute), send that user's reads to the leader | Extra load on the leader |
| Read only from caught-up replicas | Record the timestamp of the user's last write, and read only from a replica that has replicated past it | Needs replication position tracking |
| Optimistic update on the client | The client shows the value it just submitted without waiting for a fresh read | Only fixes that device; another device may still see the old value |

Making the user wait until the write finishes does not help. The write already finished on the leader. The stale data comes from the replica on the read path.

## Monotonic reads

Send all reads from one user to the same replica, for example by hashing the user ID. That user's view may be behind, but it never moves backward.

## Quorum reads and writes

Each piece of data has **N** replicas. A write succeeds after **W** replicas confirm it. A read asks **R** replicas and keeps the value with the newest version.

If **R + W > N**, every read set overlaps every write set in at least one replica, and that replica has the latest value.

Example with N = 3 (replicas A, B, C):

| Setting | What happens | Stale reads possible? |
| --- | --- | --- |
| W = 1, R = 1 (1 + 1 = 2, not > 3) | The write lands on A only. A read asks only B, which has not seen it | **Yes**. Fast, but only eventually consistent |
| W = 2, R = 2 (2 + 2 = 4 > 3) | The write lands on A and B. Any 2 of 3 replicas must include A or B | **No**. The read always sees the latest version |
| W = 3, R = 1 (3 + 1 = 4 > 3) | Every replica has the write, so any single replica is up to date | **No**. Fast reads, but one down replica blocks all writes |

Tuning W and R trades latency and availability against consistency: the same trade-off as CAP and PACELC, adjusted per request.

## Follow-up questions

**How do you get read-your-writes with asynchronous replicas?**
Route the user's reads to the leader for a short window after they write, or read only from replicas that have caught up to the user's last write.

**Why does R + W > N give up-to-date reads?**
By the pigeonhole principle, the R replicas you read and the W replicas that took the write cannot be completely separate, so at least one replica in the read has the latest version.

**Which model does a social feed need?**
Mostly eventual. Add read-your-writes so a user sees their own new post immediately, and causal ordering for comment threads.

## Parked

- Edge cases where quorums are still not linearizable (sloppy quorums, concurrent writes)
- Conflict resolution: last-write-wins, vector clocks, CRDTs
- Sequential consistency and other formal models

## References

- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 5 (Replication) and Chapter 9 (Consistency and Consensus)
- Werner Vogels, "Eventually Consistent" (2008)
