# Replication — Folder Guide & Summary of Everything

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), **Chapter 5: Replication** (pp. 151–192).
> Start here: read the files in order, then use this page as a **one-page revision sheet** before interviews.

## 📚 Reading order

| # | File | What it covers |
|---|---|---|
| 01 | [Single-Leader Replication](01-Single-Leader-Replication.md) | Leaders and followers, sync vs async, new followers, failover, replication log formats |
| 02 | [Replication Lag & Stale Reads](02-Replication-Lag-and-Stale-Reads.md) | Eventual consistency, read-your-writes, monotonic reads, consistent prefix reads |
| 03 | [Multi-Leader Replication](03-Multi-Leader-Replication.md) | Multi-datacenter, offline clients, collaborative editing, topologies |
| 04 | [Handling Write Conflicts](04-Handling-Write-Conflicts.md) | Conflict avoidance, convergence, custom handlers, automatic resolution |
| 05 | [CRDTs](05-CRDTs.md) | Conflict-free replicated data types: counters, sets, OR-Set, limitations |
| 06 | [Leaderless Replication](06-Leaderless-Replication.md) | Dynamo-style, writing with nodes down, read repair, anti-entropy |
| 07 | [Quorums](07-Quorums.md) | w + r > n, tuning, limitations, staleness, sloppy quorums, hinted handoff |
| 08 | [Detecting Concurrent Writes](08-Detecting-Concurrent-Writes.md) | Last write wins, happens-before, siblings, version vectors |

## 1. Why replicate?

| Goal | Meaning |
|---|---|
| **High availability** | Keep running when a machine, or a whole datacenter, goes down |
| **Disconnected operation** | Keep working through a network interruption |
| **Latency** | Put data geographically close to users |
| **Scalability** | Serve more reads than one machine can |

All the difficulty is in **handling changes** to the replicated data.

## 2. The three approaches at a glance

```
SINGLE-LEADER                 MULTI-LEADER                    LEADERLESS
 writes ─▶ Leader              writes ─▶ L1 ◀──▶ L2 ◀─ writes   writes ─▶ R1, R2, R3 (any)
             │                            │        │            reads  ◀─ R1, R2, R3 (several)
         ┌───┴───┐                     followers followers      (quorum: w + r > n)
         F1      F2   (read-only)
```

| | **Single-leader** | **Multi-leader** | **Leaderless** |
|---|---|---|---|
| Writes go to | One leader | Any of several leaders | Several replicas in parallel |
| Reads from | Leader or followers (may be stale) | Any node | Several replicas in parallel |
| Write conflicts | ❌ None | ✅ Yes, must resolve | ✅ Yes, must resolve |
| Node failure | Failover (risky) | Other leaders continue | **No failover**: w acknowledgments are enough |
| Catching up | Replay the replication log | Replication log between leaders | Read repair + anti-entropy |
| Consistency | Can be strong (reads from the leader) | Weak | Weak and tunable (eventual) |
| Complexity | **Low**, the most popular | High | Medium to high |
| Best for | Most applications | Multiple regions, offline clients, collaborative editing | High availability, low latency, tolerating stale reads |
| Examples | PostgreSQL, MySQL, MongoDB, Kafka | CouchDB, BDR, Tungsten, GoldenGate | Cassandra, Riak, Voldemort |

## 3. Synchronous vs asynchronous

| | Synchronous | Asynchronous |
|---|---|---|
| Leader waits for the follower? | Yes | No |
| Durability | Data guaranteed on the follower | **Confirmed writes can be lost** on failover |
| Availability | One slow or dead follower **blocks writes** | Leader keeps going |
| In practice | **Semi-sync**: 1 sync follower, the rest async | Very common, especially across regions |

## 4. Replication log formats

**Statement-based** (non-determinism problems) → **WAL shipping** (tied to the storage engine, upgrades need downtime) → **Logical / row-based** (decoupled, enables **CDC**) → **Trigger-based** (flexible, more overhead).

## 5. Failover dangers (single-leader)

**Lost async writes** · **reused keys** causing inconsistency with other systems (the GitHub incident) · **split brain**, with **STONITH** as the safety catch · **timeouts** that are too short (needless failovers) or too long (slow recovery).

## 6. Replication-lag guarantees

| Guarantee | Promise | Typical fix |
|---|---|---|
| **Eventual consistency** | Replicas converge *eventually* (no time bound) | — |
| **Read-after-write** | You always see **your own** writes | Read your own data from the leader, or track your last-write position |
| **Monotonic reads** | **Time never goes backward** for one user | Pin each user to one replica (hash of user ID) |
| **Consistent prefix reads** | **Cause before effect** (question before answer) | Causally related writes in the same partition, or track causality |

## 7. Conflict handling, summarized

1. **Avoid** conflicts: route each record's writes through one "home" leader.
2. **Converge**: LWW or highest ID (**loses data**), **merge** values, or **keep all versions**.
3. **Custom logic**: **on write** (Bucardo) or **on read** (CouchDB). Resolution is per row or document, not per transaction.
4. **Automatic**: **CRDTs** (two-way merge), **mergeable persistent data structures** (three-way merge, like Git), **operational transformation** (Google Docs).

## 8. Quorums, summarized

- **w + r > n** means the read set overlaps the write set. **n = 3, w = r = 2** tolerates 1 failure. **n = 5, w = r = 3** tolerates 2.
- **w + r ≤ n**: faster and more available, but **staler**.
- **Even strict quorums can return stale data:** sloppy quorums, concurrent writes with LWW, reads during writes, partial writes that aren't rolled back, restores from stale replicas.
- **Sloppy quorum + hinted handoff**: write to any reachable nodes during a network cut and move the data home later. Better availability, **no overlap guarantee** (on by default in Riak, off in Cassandra and Voldemort).

## 9. Concurrent writes, summarized

- **Concurrent = neither knew about the other**. It's not about clock time.
- **LWW**: converges, but **silently drops writes**. Only for caches or write-once keys (UUIDs).
- **Version number per key** (one replica) or **version vector** (many replicas) tells **overwrites** apart from **concurrent writes**, and concurrent values are kept as **siblings**.
- **Merge siblings** (union for a cart, plus **tombstones** for deletes), or use **CRDTs**.

## 10. 🎯 Interview cheat sheet

| Question | Short answer |
|---|---|
| Why replicate? | Availability, lower latency, read scaling, disconnected operation |
| Single vs multi vs leaderless? | One write node (simple, no conflicts) · several write nodes (regions, offline, needs conflict resolution) · any node with quorums (high availability, eventual consistency) |
| Sync vs async? | Sync is durable but blocks. Async is fast but can lose confirmed writes. Semi-sync in practice |
| What is failover, and what can go wrong? | Promote a follower to leader. Risks: lost writes, split brain, bad timeouts, key reuse |
| Statement vs WAL vs logical replication? | Statements break on non-determinism. WAL is tied to the storage engine. Logical is decoupled and enables CDC |
| What is replication lag? | Delay before a follower applies the leader's writes, usually milliseconds, sometimes minutes |
| Read-your-writes? | A user always sees their own updates. Read their data from the leader |
| Monotonic reads? | No going back in time. Pin each user to one replica |
| Consistent prefix? | Writes seen in causal order. Mainly a sharding issue |
| When use multi-leader? | Multiple datacenters, offline clients, collaborative editing. Not inside one datacenter |
| How resolve conflicts? | Avoid · LWW (lossy) · merge · keep versions · custom handler · CRDTs |
| What is a CRDT? | A data type with a commutative, associative, idempotent merge, so replicas converge automatically |
| What is a quorum? | Write to w and read from r of n replicas. w + r > n guarantees overlap |
| Sloppy quorum? | During a partition, accept writes on non-home nodes, then hinted handoff moves them home |
| Read repair vs anti-entropy? | Fix stale replicas during reads vs background comparison |
| Why is LWW dangerous? | Concurrent writes reported as successful are silently discarded, and clock skew can drop even non-concurrent writes |
| What are version vectors? | One counter per replica. They tell causally ordered writes apart from concurrent ones (siblings) |

## 11. Conclusion: the whole chapter in one paragraph

**Replication** keeps copies of the same data on several machines for **availability, low latency, read scaling and offline work**, and all the difficulty lies in **propagating changes**. **Single-leader** replication sends every write through one leader that streams a **replication log** (statement, WAL, logical or trigger-based) to followers. It's simple and conflict-free, but **failover** is risky (lost async writes, split brain, bad timeouts), and reading from **asynchronous followers** exposes **replication lag**. That makes guarantees like **read-after-write, monotonic reads and consistent prefix reads** important. **Multi-leader** replication lets several leaders accept writes, which suits **multiple datacenters, offline devices and collaborative editing**. The price is **write conflicts**, handled by avoidance, convergence (LWW, merge, keep versions), custom handlers or **CRDTs**, plus fragile **topologies**. **Leaderless** (Dynamo-style) replication writes to and reads from several replicas using **quorums (w + r > n)**, with **no failover**. It repairs stale replicas with **read repair and anti-entropy** and boosts availability with **sloppy quorums and hinted handoff**, but it gives only **eventual consistency**, with edge cases even under strict quorums. Because multi-leader and leaderless systems accept **concurrent writes**, databases must detect them using **happens-before**: **version numbers or version vectors** separate overwrites from concurrent **siblings**, which are then merged (tombstones, CRDTs), while **last write wins** converges at the cost of **silently losing data**.
