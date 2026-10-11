# 07 · Quorums (and Sloppy Quorums) — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 5, "Quorums for reading and writing", "Limitations of Quorum Consistency" and "Sloppy Quorums and Hinted Handoff" (pp. 179–184).
> Folder guide: [README.md](README.md) · Previous: [06-Leaderless-Replication.md](06-Leaderless-Replication.md) · Next: [08-Detecting-Concurrent-Writes.md](08-Detecting-Concurrent-Writes.md)

## 1. The intuition

With **3 replicas**, if every successful write is on **at least 2**, then **at most 1 replica is stale**. If you **read from 2**, at least **one of them has the latest value**.

## 2. The quorum rule

| Symbol | Meaning |
|---|---|
| **n** | Number of replicas that store each value |
| **w** | Replicas that must **confirm a write** for it to succeed |
| **r** | Replicas that must **answer a read** |

> **Quorum condition: `w + r > n`**: the replicas you **wrote to** and the replicas you **read from** must **overlap in at least one node**, so the read sees the latest value.

```
n = 5, w = 3, r = 3          (3 + 3 = 6 > 5 ✅)

replica:        1    2    3    4    5
write (w=3):    ✔    ✔    ✔
read  (r=3):              ✔    ✔    ✔
                          ▲
                overlap → replica 3 has the latest value
```

- Reads and writes that follow these values are **quorum reads and writes**. Think of **w and r as the minimum votes** needed.
- This kind is called a **strict quorum**, as opposed to a sloppy quorum (section 8).
- In practice, requests are **sent to all n replicas in parallel**. **w and r are just how many responses you wait for.**
- If fewer than w (or r) replicas respond, the write (or read) returns an **error**. It doesn't matter *why* a node failed (crash, full disk, network): **only success counts**.

## 3. Choosing n, w and r

| Configuration | w + r > n? | Node failures tolerated | Notes |
|---|---|---|---|
| **n=3, w=2, r=2** | 4 > 3 ✅ | **1** | Common default |
| **n=5, w=3, r=3** | 6 > 5 ✅ | **2** | More fault tolerance |
| **n=3, w=3, r=1** | 4 > 3 ✅ | Reads: 2, **writes: 0** | **Read-heavy**: very fast reads, but **one dead node stops all writes** |
| **n=3, w=1, r=1** | 2 ≤ 3 ❌ | 2 | Fastest and most available, but **stale reads are likely** |

- **Common choice:** n is **odd** (3 or 5), and **w = r = (n + 1) / 2** rounded up, i.e. a **majority**.
- **w < n** means writes still work with a node down. **r < n** means reads still work with a node down.
- A cluster can have **more than n nodes**. Each value lives on **only n of them**, which allows **partitioning** (Chapter 6).
- **Quorums don't have to be majorities.** All that matters is that the read and write sets **overlap**.

### Choosing w + r ≤ n
- ✅ **Lower latency** and **higher availability**: the system keeps working even when many replicas are unreachable.
- ❌ **More likely to read stale values**, because the read may miss the node with the latest write.

## 4. Limitations: stale reads even when w + r > n

| Edge case | Why it breaks the guarantee |
|---|---|
| **Sloppy quorum** | The w writes may land on **different nodes** from the r reads, so there's **no guaranteed overlap** |
| **Concurrent writes** | It's unclear which came first. Merging is the only safe option. **LWW loses writes** because of clock skew |
| **Write concurrent with a read** | The write may be on **only some** replicas, so the read may return the old **or** the new value |
| **Partially failed write** | The write succeeded on **fewer than w** replicas and was reported as failed, but it is **not rolled back** where it succeeded, so later reads **may still return it** |
| **Node restored from a stale replica** | A node holding the new value fails and is **restored from an old replica**, so the number of nodes with the new value **drops below w** |
| **Unlucky timing** | Even when everything works, there are timing edge cases (linearizability, Chapter 9) |

➡️ Dynamo-style databases are built for workloads that **tolerate eventual consistency**. **w and r adjust the *probability* of stale reads**. They are **not absolute guarantees**.
➡️ You **usually don't get** read-your-writes, monotonic reads or consistent prefix reads (see [02](02-Replication-Lag-and-Stale-Reads.md)). Stronger guarantees need **transactions or consensus**.

## 5. Quorums vs failover

- Quorums tolerate **failed nodes without failover**.
- They also tolerate **slow nodes**, because you return after w or r responses instead of waiting for all n.
- ➡️ That makes leaderless systems attractive for **high availability, low latency, and tolerance of occasional stale reads**.

## 6. Example: Cassandra consistency levels 📘 Beyond the book

Cassandra sets w and r per query with **consistency levels**:

| Level | Meaning |
|---|---|
| `ONE` | Wait for 1 replica |
| `QUORUM` | Wait for a majority of all replicas |
| `LOCAL_QUORUM` | Majority in the **local datacenter** only |
| `EACH_QUORUM` | Majority in **every** datacenter (writes) |
| `ALL` | Wait for every replica |

Writing at `QUORUM` and reading at `QUORUM` satisfies **w + r > n**.

## 7. Monitoring staleness

| | Leader-based | Leaderless |
|---|---|---|
| How to measure lag | **Leader's log position − follower's log position** | **Hard**: writes have **no fixed order** |
| How old can data get? | Bounded by the lag | With **read repair only** (no anti-entropy), a rarely read value can be **ancient** |

- Even if your app tolerates stale reads, **monitor replication health** and **alert** when it falls far behind (network problems, overloaded nodes).
- Research can **predict the percentage of stale reads** from n, w and r, but it isn't common practice yet.
- "Eventual" is vague, so **measure it**.

## 8. Sloppy quorums and hinted handoff

**The problem:** a **network interruption** can cut a client off from many database nodes. They're alive, but to that client they **might as well be dead**, so it **can't reach w or r of the value's home nodes**.

In a **large cluster** (many more nodes than n), the client can still reach **some** nodes, just not the right ones. The trade-off:

1. **Return errors** for every request that can't reach a proper quorum, **or**
2. **Accept writes anyway** on reachable nodes that **aren't** among the value's n home nodes. This is a **sloppy quorum**.

```
Home nodes for key K:  A, B, C   (n = 3, w = 2)
Network cut: the client can only reach C, D, E

Sloppy write: C ✅ + D ✅ (D stores K with a "hint": this belongs to A)
Network heals → D hands K over to A  ← HINTED HANDOFF
```

> 🏠 **Analogy:** you're locked out of your house, so you sleep on the **neighbour's couch** (sloppy quorum). When you find your keys, the neighbour asks you to **go home** (hinted handoff).

| | |
|---|---|
| ✅ **Benefit** | Much higher **write availability**: any w reachable nodes can accept a write |
| ❌ **Cost** | **Not really a quorum.** Even with **w + r > n**, a read may **miss the latest value** until hinted handoff finishes. It's only a **durability** promise: the data is stored on w nodes *somewhere* |
| **Defaults** | **Riak: on.** **Cassandra and Voldemort: off** |

## 9. Conclusion

A **quorum** means writing to n replicas and waiting for **w** acknowledgments, and reading from n and waiting for **r** responses. If **w + r > n**, the read and write sets **overlap**, so at least one replica you read has the latest value. Typical settings are n = 3 or 5 with **w = r = majority**, tolerating 1 or 2 failed nodes **without failover**, while w = n with r = 1 favours reads at the cost of write availability. Choosing **w + r ≤ n** buys latency and availability but risks stale reads. Even strict quorums can return stale data: sloppy quorums, concurrent writes resolved by LWW, reads during writes, partial writes that aren't rolled back, and restores from stale replicas. Quorums tune **probability, not guarantees**, and don't give read-your-writes or monotonic reads. Staleness is **hard to monitor** without a log order. **Sloppy quorums** accept writes on non-home nodes during network cuts and later move them home with **hinted handoff**. That boosts write availability but breaks the overlap guarantee (on by default in Riak, off in Cassandra and Voldemort).

### Interview one-liners
- **Quorum:** "With n replicas, write to w and read from r. If w + r > n, the two sets overlap, so a read sees the latest write. For example, n = 3 with w = r = 2 tolerates one node down."
- **Tuning:** "Bigger w gives more durable writes but lower write availability. Bigger r gives fresher reads but slower reads. w + r ≤ n gives lower latency but stale reads."
- **Limits:** "Quorums aren't strong consistency. Concurrent writes, sloppy quorums and partial failures can still return stale data."
- **Sloppy quorum:** "During a network cut, accept writes on any w reachable nodes, then hand them back to the home nodes later. That's better availability, but no overlap guarantee."

### Self-check questions
1. Define n, w and r. Why does w + r > n give up-to-date reads?
2. How many failures do n = 3/w = 2/r = 2 and n = 5/w = 3/r = 3 tolerate?
3. What are the trade-offs of w = n, r = 1? And of w + r ≤ n?
4. Must quorums be majorities? Why or why not?
5. List five cases where a quorum read can still return stale data.
6. Why is staleness harder to monitor in leaderless systems?
7. Explain sloppy quorums and hinted handoff with the house analogy.
8. Why isn't a sloppy quorum a real quorum? Where is it enabled by default?
9. How do Cassandra's consistency levels relate to w and r?
