# 06 · Leaderless Replication — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 5, "Leaderless Replication" (pp. 177–184).
> Folder guide: [README.md](README.md) · Previous: [05-CRDTs.md](05-CRDTs.md) · Next: [07-Quorums.md](07-Quorums.md)

## 1. What it is

> **Leaderless replication:** there is **no leader**. **Any replica accepts writes directly** from clients.

- Some of the **earliest** replicated systems were leaderless. The idea was forgotten during the relational era and **revived by Amazon's Dynamo**.
- **Dynamo-style** open-source databases: **Riak, Cassandra, Voldemort**.
- ⚠️ **DynamoDB ≠ Dynamo.** Amazon's hosted **DynamoDB** used a **different, single-leader** design at the time the book was written.
- Writes go either **straight from the client to several replicas**, or through a **coordinator node**. Unlike a leader, the coordinator **does not enforce any order of writes**, which has big consequences.

## 2. Writing when a node is down: no failover

```
n = 3 replicas, replica 3 is down (e.g. rebooting for an update)

User 1234: set(key, value) v7 ──▶ replica 1  ✅ ok
                              ──▶ replica 2  ✅ ok
                              ──▶ replica 3  ❌ (down, misses the write)
2 of 3 ok's ⇒ the write is SUCCESSFUL. The client ignores the missing one.
```

- With a leader, you might need a **failover**. With leaderless replication, **failover doesn't exist**.
- When replica 3 comes **back online**, it is **missing writes**, so reading only from it would give **stale** values.
- **Fix:** **reads also go to several replicas in parallel**. The client may get different answers and uses **version numbers** to pick the **newest**.

```
User 2345: get(key) ──▶ replica 1 → v7 ✅
                    ──▶ replica 2 → v7 ✅
                    ──▶ replica 3 → v6 (stale)  → client keeps v7
```

## 3. How a stale replica catches up

| Mechanism | How it works | Good for | Weakness |
|---|---|---|---|
| **Read repair** | During a parallel read, the client **notices a stale replica** and **writes the newer value back** to it (v7 → replica 3) | **Frequently read** values | **Rarely read** values stay stale |
| **Anti-entropy process** | A **background process** keeps **comparing replicas** and copying missing data | Everything, eventually | **No ordering** of writes (unlike a replication log) and possibly a **long delay** |

- Not every system has both. **Voldemort** had **no anti-entropy**.
- Without anti-entropy, rarely read values may exist on **fewer replicas**, which **reduces durability**.
- 📘 *Beyond the book:* Dynamo and Cassandra use **Merkle trees** (hash trees) in anti-entropy, so replicas can find differing key ranges by comparing hashes instead of all the data.

## 4. How many replicas must agree? Quorums

How many "ok" responses do we need for writes and reads? The rule **w + r > n** guarantees the read set **overlaps** the write set. Full details are in **[07-Quorums.md](07-Quorums.md)**:
- choosing n, w and r
- the limitations
- monitoring staleness
- sloppy quorums and hinted handoff

## 5. Multi-datacenter operation

Leaderless replication suits multiple datacenters, because it's built to tolerate **concurrent writes, network interruptions and latency spikes**.

| Database | How it spans datacenters |
|---|---|
| **Cassandra, Voldemort** | **n counts replicas in all datacenters**, and you configure how many go in each. Every write is sent to **all** replicas, but the client usually **waits only for a quorum in its local datacenter**, so it isn't slowed by the cross-datacenter link. Remote writes are often **asynchronous** |
| **Riak** | Client-to-node traffic stays **inside one datacenter** (n = replicas per datacenter). **Cross-datacenter replication** between clusters runs **asynchronously in the background**, similar to multi-leader |

## 6. Concurrent writes are unavoidable

With no leader to order writes, several clients can write the **same key concurrently**, and replicas may receive the writes in **different orders**. Conflicts can also come from **read repair** or **hinted handoff**. How to detect and resolve them (LWW, version numbers, version vectors, siblings) is covered in **[08-Detecting-Concurrent-Writes.md](08-Detecting-Concurrent-Writes.md)**.

## 7. Leader-based vs leaderless

| | Leader-based (single or multi) | Leaderless (Dynamo-style) |
|---|---|---|
| Who accepts writes | The leader(s) | **Any replica** |
| Write order | Set by the leader | **No fixed order** |
| Node failure | **Failover** needed for the leader | **No failover**: just need w acknowledgments |
| Catching up | Follower replays the **replication log** | **Read repair + anti-entropy** |
| Slow nodes | A slow leader slows everything | Tolerated: wait for only w or r responses |
| Monitoring lag | Easy (log positions) | **Hard** (no order) |
| Consistency | Can offer read-your-writes and so on | Usually **eventual**: tunable but no strong guarantees |
| Examples | PostgreSQL, MySQL, MongoDB, CouchDB | **Cassandra, Riak, Voldemort** |

## 8. Conclusion

**Leaderless (Dynamo-style) replication** drops the leader: clients (or a coordinator that doesn't order writes) send each write to **several replicas in parallel** and treat it as successful after **w** acknowledgments, so a down node needs **no failover**. Reads also go to several replicas, and **version numbers** pick the newest value. Stale replicas catch up through **read repair** (fixing stale copies during reads, best for hot keys) and an **anti-entropy** background process (copying missing data without ordering, needed for rarely read keys). How many replicas must answer is set by **quorums** (`w + r > n`). Cassandra and Voldemort count n across datacenters and wait for a **local** quorum, while Riak replicates between datacenter clusters asynchronously. Because there's no leader to order writes, **concurrent writes conflict**, and that has to be handled with LWW, version vectors or CRDTs. Riak, Cassandra and Voldemort follow this model. It gives **high availability and low latency** but only **eventual consistency**.

### Interview one-liners
- **Leaderless:** "Any replica accepts writes. The client writes to n replicas and waits for w acknowledgments, and reads from r replicas taking the newest version. There's no failover."
- **Read repair vs anti-entropy:** "Read repair fixes stale replicas during reads, which covers hot keys. Anti-entropy is a background comparison that covers everything else."
- **Multi-datacenter:** "Cassandra waits for a local-datacenter quorum and replicates to remote datacenters asynchronously."

### Self-check questions
1. How does leaderless replication differ from leader-based? Name three Dynamo-style databases.
2. Why is there no failover in leaderless replication?
3. How does a client avoid reading stale data from a node that just came back?
4. Compare read repair and anti-entropy. Why do you need anti-entropy for rarely read keys?
5. How do Cassandra and Riak handle multiple datacenters differently?
6. Why do concurrent writes occur even with strict quorums?
7. Why is DynamoDB not the same as Dynamo?
