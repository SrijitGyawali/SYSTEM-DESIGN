# 03 · Linearizability (and CAP) — Revision Notes

> Source: *Designing Data-Intensive Applications*, **Chapter 9**, "Consistency Guarantees" and "Linearizability" (pp. 321–338).
> Folder guide: [README.md](README.md) · Previous: [02-Consistent-Hashing.md](02-Consistent-Hashing.md) · Next: [04-Two-Phase-Commit.md](04-Two-Phase-Commit.md)

## 1. Why we need something stronger than eventual consistency

- Most replicated databases offer **eventual consistency**, which would be better named **convergence**: stop writing, wait, and replicas eventually agree.
- It's **very weak**: until then, reads can return **anything**, even an old value right after you wrote a new one.
- It's hard for developers because a database **doesn't behave like a variable** in a normal program.
- **Stronger models are easier to use correctly, but cost performance and fault tolerance.**

⚠️ **Distributed consistency ≠ transaction isolation.**
- Isolation is about **races between concurrent transactions**.
- Consistency models are about **coordinating replicas despite delays and faults**.

## 2. What linearizability is

> **Linearizability:** make the system **look as if there is only one copy of the data**, and **every operation on it is atomic**.

- Other names: **atomic consistency, strong consistency, immediate consistency, external consistency**.
- **It's a recency guarantee:** once a write completes, **every** later read, from **any** client, sees that value or a newer one. **No stale replica or cache values.**

### The Alice and Bob example (not linearizable)
```
World Cup final ends.
Alice refreshes → replica 1 (up to date) → "Germany won!" → she tells Bob
Bob refreshes AFTER hearing Alice → replica 2 (lagging) → "game still on" ❌
```
Bob's request **started after Alice's finished**, so it must not return **older** data.

## 3. What makes a system linearizable

Think of a **register** x: one key, row or document. Each operation takes effect **atomically at some instant between its request and its response**.

```
time ───────────────────────────────────────────────────────────▶
Client C:        |──────── write(x, 1) ────────|
Client A: |read→0|     |read→0 or 1|      |read→ MUST be 1|
Client B:                   |read→1|  |read→ MUST be 1 (A or B already saw 1)|
```

**Rules:**
1. A read that **finishes before** the write starts returns the **old** value.
2. A read that **starts after** the write finishes returns the **new** value.
3. A read **concurrent** with the write may return either, **but**:
4. ⭐ **Once any read has returned the new value, every later read (on any client) must also return the new value.** The value **flips atomically** at one point and **never flips back**.

- Joining each operation's "effective instant" must give a valid **sequential order** that always **moves forward in time**.
- **compare-and-set** `cas(x, old, new)` succeeds only if x still equals `old`. This is how linearizable systems build locks.
- You can **test** linearizability (expensively) by recording request and response times and checking whether a valid order exists.

### Linearizability vs serializability ⚠️

| | **Serializability** | **Linearizability** |
|---|---|---|
| About | **Transactions** (multi-object) — isolation | **Single object** (register) — recency |
| Guarantee | The result equals **some** serial order (which may differ from real time) | Operations appear in **real-time order**, and reads see the **latest** value |
| Prevents write skew? | ✅ | ❌ (no grouping of operations) |

- **Both together = strict serializability** (strong one-copy serializability).
- **2PL and actual serial execution** are typically linearizable.
- **SSI is NOT linearizable**: it reads from a snapshot, so it deliberately ignores newer writes.

## 4. When you need linearizability

| Use case | Why |
|---|---|
| **Locking and leader election** | Every node must **agree who holds the lock** or who is leader, otherwise you get **split brain**. **ZooKeeper / etcd** provide this with consensus. Oracle RAC uses linearizable per-page locks |
| **Uniqueness constraints** | Usernames, file paths, **no overselling stock**, **no double-booked seats**, **balance ≥ 0**. All need **one up-to-date value** everyone agrees on (like a lock or CAS on the username) |
| **Cross-channel timing dependencies** | Two communication paths can race (see below) |

> Note: ZooKeeper and etcd **writes** are linearizable, but **reads may be stale** by default. Use **`sync()`** (ZooKeeper) or a **quorum read** (etcd) for linearizable reads.

### Cross-channel race (image resizer)
```
1. web server writes full-size photo → file storage (replicating...)
2. web server sends "resize photo 123" → message queue  (FAST)
3. resizer reads the queue → fetches photo 123 from file storage
   → storage replica not updated yet → gets OLD or NO image ❌
```
There are **two channels** (storage and queue). Without linearizable storage, the queue can **overtake** replication.

## 5. Implementing linearizable systems

| Replication method | Linearizable? | Why |
|---|---|---|
| **Single-leader** | ⚠️ **Potentially** | If you read from the leader or synchronous followers. But a node may **wrongly think it's the leader**, and async failover can **lose committed writes** |
| **Consensus algorithms** (Raft, Zab) | ✅ **Yes** | Single-leader-like, with protection against **split brain and stale replicas**. **ZooKeeper and etcd** work this way |
| **Multi-leader** | ❌ No | Concurrent writes on several nodes, replicated asynchronously, so **conflicts** |
| **Leaderless** (Dynamo-style) | ❌ **Probably not** | **LWW with clocks** breaks it, **sloppy quorums** break it, and even **strict quorums** can fail (below) |

### Why strict quorums still aren't linearizable
```
n = 3, w = 3, r = 2. Writer sets x = 1 (arrives at replicas at different times)
Client A reads replicas {1, 2} → sees 1 on replica 1 → returns 1
Client B (starts AFTER A finished) reads replicas {2, 3} → both still 0 → returns 0 ❌
```
- It can be fixed with **synchronous read repair** plus **reading before writing**, at a **performance cost**.
- Even then, **compare-and-set** needs **consensus**.
- **Assume Dynamo-style systems are not linearizable.**

## 6. The cost of linearizability and the CAP theorem

**Two datacenters, and the link between them breaks:**

| Setup | What happens during the network cut |
|---|---|
| **Multi-leader** | Each datacenter keeps working, and writes queue up and sync later. ✅ **Available**, ❌ not linearizable |
| **Single-leader** | Clients in the **follower** datacenter can't reach the leader, so **no writes and no linearizable reads**. ❌ **Unavailable** there |

### CAP, stated properly
- **Network partitions will happen.** They're a fault, **not a choice**.
- So **CAP really means: when partitioned, be either Consistent (linearizable) or Available.**
  - **Need linearizability:** disconnected replicas must **wait or return errors**, so they're **unavailable**.
  - **Don't need linearizability:** each replica can serve requests independently, so it's **available but not linearizable**.
- "Pick 2 of 3" is **misleading**, because you can't opt out of partitions.
- CAP was useful as a **rule of thumb** that sparked the **NoSQL** movement. But formally it covers **only linearizability and only network partitions**, not delays, dead nodes or other trade-offs, so it has **little practical value** today. **Avoid CP/AP labels**.

> 📘 *Beyond the book:* **PACELC** extends CAP: during a **P**artition choose **A** or **C**, **E**lse (normal operation) choose **L**atency or **C**onsistency. That captures the point below.

### Linearizability is slow, all the time
- Even **multi-core CPU RAM isn't linearizable** (each core has caches and store buffers) unless you use memory barriers. The reason is **performance, not fault tolerance**.
- Many databases drop linearizability **for speed**.
- **Attiya & Welch:** linearizable response time is **at least proportional to the uncertainty of network delays**. **No faster algorithm exists.** Weaker models can be much faster.

## 7. Conclusion

**Linearizability** makes a replicated system behave like **one copy of the data with atomic operations**. Once a write completes, or any read has seen the new value, **every later read everywhere** sees it. It's a **single-object recency guarantee**, distinct from **serializability** (multi-object transaction isolation), and combining them gives **strict serializability**. You need it for **locks and leader election**, **uniqueness constraints** (usernames, seats, stock), and avoiding **cross-channel races**. **Consensus-based** systems like ZooKeeper and etcd provide it. Single-leader can, if you really know who the leader is. Multi-leader and Dynamo-style quorums **don't**. The price is the **CAP trade-off**: during a network partition a linearizable system must make some replicas **unavailable**. More importantly, linearizability is **slow all the time**, since response time is bounded below by network delay uncertainty. That's why many systems choose weaker models.

### Interview one-liners
- **Linearizability:** "The system behaves as if there's a single copy and operations are atomic. After a write completes, every read sees it, so it's a recency guarantee."
- **vs serializability:** "Serializability is transaction isolation in some serial order. Linearizability is real-time recency on single objects. Both together is strict serializability."
- **CAP:** "Partitions are inevitable, so during one you choose linearizability or availability. The bigger everyday cost is latency."
- **Quorums:** "Even w + r > n isn't linearizable with variable network delays."

### Self-check questions
1. Why is eventual consistency hard for developers?
2. Define linearizability using the Alice and Bob example.
3. What does "once any read returns the new value, all later reads must" mean?
4. Linearizability vs serializability: what's strict serializability? Is SSI linearizable?
5. Give three use cases that require linearizability.
6. Explain the image-resizer cross-channel race.
7. Which replication methods can be linearizable, and why?
8. Show why a strict quorum read can still be non-linearizable.
9. State CAP correctly. Why is "pick 2 of 3" misleading?
10. Why is linearizability slow even without network faults?
