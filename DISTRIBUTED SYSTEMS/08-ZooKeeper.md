# 08 · ZooKeeper (and Coordination Services) — Revision Notes

> Source: *Designing Data-Intensive Applications*, **Chapter 9**, "Membership and Coordination Services" (pp. 370–373), plus Chapter 6 "Request Routing" and Chapter 9 on locks and fencing tokens.
> Sections marked **📘 Beyond the book** add ZooKeeper's data model and recipes (from the ZooKeeper docs and paper) that interviews expect.
> Folder guide: [README.md](README.md) · Previous: [07-Raft-Log-Replication.md](07-Raft-Log-Replication.md)

## 1. What ZooKeeper is

> **ZooKeeper** is a **coordination service**: a small, highly reliable, **consensus-replicated key-value store** that distributed systems use to **agree on things** such as who the leader is, who holds a lock, which node owns which partition, and who is alive.

- It's described as a "distributed key-value store" or a "coordination and configuration service". Its API looks like a database: read and write keys, list keys.
- **Why use consensus at all?** Because the data it holds must be **agreed on by everyone**.
- Modelled after **Google's Chubby** lock service.
- You rarely use it **directly**. You rely on it **through** other systems: **HBase, Hadoop YARN, OpenStack Nova, Kafka** (historically), SolrCloud.
- **Similar systems:** **etcd** (uses **Raft**, and is the backbone of **Kubernetes**) and **Consul**.

**What it's NOT for:**
- ❌ Not a general-purpose database. It holds **small data that fits in memory** (still written to disk for durability).
- ❌ Not for **fast-changing runtime state**. Its data changes over **minutes or hours** ("10.1.1.23 is leader for partition 7"), not thousands of times a second. For replicating application state, use tools like **Apache BookKeeper**.

## 2. How it's built

- A small, fixed **ensemble**, usually **3 or 5 servers**, so a majority survives 1 or 2 failures.
- Data is replicated to all servers with **fault-tolerant total order broadcast** using the **Zab** protocol (ZooKeeper Atomic Broadcast).
- 📘 **One leader handles all writes** and followers replicate them. **Reads are served locally by any server**, so they're fast but **may be stale**.
- 📘 **Observers** are non-voting servers that receive the log but don't vote. They add **read capacity** without slowing votes. This is DDIA's "**read-only caching replicas**".
- ⭐ **Outsourcing coordination:** the app may grow to **thousands of nodes**, but the **majority votes run only among 3 to 5 ZooKeeper servers**. Consensus, operation ordering and failure detection are **delegated** to an external service.

## 3. The four key features (DDIA)

| Feature | What it gives you | Needs consensus? |
|---|---|---|
| **Linearizable atomic operations** | **Compare-and-set**, so you can build a **lock**: if several nodes try at once, **exactly one succeeds**, even during failures. Locks are usually **leases** (they expire if the client dies) | ✅ **Yes**, the only one that truly needs it |
| **Total ordering of operations** | Every operation gets a monotonically increasing **transaction ID (`zxid`)** and version (`cversion`). These are **fencing tokens** that stop a paused old lock holder from corrupting data | — |
| **Failure detection** | Clients keep a **session** with **heartbeats**. If heartbeats stop for longer than the **session timeout**, the session is dead and its **ephemeral nodes** (and locks) are **released automatically** | — |
| **Change notifications (watches)** | Clients **watch** keys and get **notified** when they change, for example a node joins, or a node dies and its ephemeral node disappears. **No polling** needed | — |

> The **combination** of these features is what makes ZooKeeper useful for coordination.

### Fencing tokens: why total ordering matters
```
Client 1 gets lock (token 33) → long GC pause 💤 … lease expires
Client 2 gets lock (token 34) → writes to storage with token 34 ✅
Client 1 wakes up, still thinks it holds the lock → writes with token 33
Storage: "33 < 34, reject" ✅  (the zxid from ZooKeeper is the token)
```

> ⚠️ ZooKeeper's **writes are linearizable**, but **reads may be stale** by default. Call **`sync()`** before a read when you need it to be linearizable.

## 4. Data model 📘 Beyond the book

A **hierarchical namespace like a filesystem**. Each node is a **znode** that holds small data (≤ 1 MB, typically much less) and can have children.

```
/
├── app1
│   ├── config            (persistent: "db_url=...")
│   ├── leader            (ephemeral: "node-3")
│   └── workers
│       ├── worker-0000000001   (ephemeral + sequential)
│       └── worker-0000000002
└── locks
    └── resource-x
        ├── lock-0000000007     (ephemeral + sequential)
        └── lock-0000000008
```

| znode type | Behaviour |
|---|---|
| **Persistent** | Stays until explicitly deleted |
| **Ephemeral** | **Deleted automatically when the creating session dies**, which is the basis of failure detection |
| **Sequential** | ZooKeeper appends a **monotonically increasing counter** to the name (`-0000000007`) |
| **Ephemeral + sequential** | Both. The building block for **locks and leader election** |

- **Watches** are **one-time triggers**: after firing, you must **re-register** them.
- **Guarantees:** **linearizable writes**, **FIFO order per client**, and every client sees updates in the **same order** (sequential consistency).

## 5. What ZooKeeper is used for

### (a) Leader election and allocating work (DDIA)
- Several instances of a service, **exactly one** of which must be leader (single-leader databases, **job schedulers**), and **someone else takes over** if it fails.
- **Assigning partitions to nodes** and **rebalancing** when nodes join or fail (see [01 §8](01-Partitioning.md#8-request-routing-service-discovery)).
- Done with **atomic operations, ephemeral nodes and notifications**. Libraries like **Apache Curator** provide recipes. It's still hard, but **far better than writing your own consensus**.

📘 **The leader election recipe:**
```
1. Each candidate creates an EPHEMERAL + SEQUENTIAL znode: /election/n_0000000005
2. List children. The LOWEST sequence number is the leader.
3. Every non-leader WATCHES only the znode just before it (not the leader),
   which avoids the "herd effect" of everyone waking up at once
4. Leader dies → its session expires → its ephemeral znode vanishes
   → the next node gets notified → it re-checks and becomes leader
```
A **distributed lock** works the same way: the lowest sequence number holds the lock.

### (b) Service discovery (DDIA)
- Services **register their IP and port** on startup, and others **look them up**.
- Used with ZooKeeper, **etcd** and **Consul**.
- ⚠️ Service discovery **doesn't strictly need consensus**. **DNS** works well and is not linearizable, and staleness is fine. But **leader election does**, so it makes sense to reuse the consensus system to tell services **who the leader is**. Read-only replicas (observers) serve those non-linearizable reads.

### (c) Membership services (DDIA)
- Decide which nodes are **currently alive members** of the cluster. This research dates back to the 1980s (e.g. **air traffic control**).
- You **can't reliably detect failure** over a network, but **failure detection plus consensus** lets nodes **agree** on who's alive.
- A node may be **wrongly declared dead**, but **agreement** is what matters. For example, "leader = the lowest-numbered member" only works if everyone agrees on the member list.

### (d) Configuration and metadata 📘
- Small shared config (feature flags, connection strings) that everyone **watches** for changes.
- **Partition maps** for routing tiers: HBase, SolrCloud, Kafka (historically), and LinkedIn Espresso via Helix.

## 6. ZooKeeper vs etcd vs Consul 📘 Beyond the book

| | **ZooKeeper** | **etcd** | **Consul** |
|---|---|---|---|
| Consensus | **Zab** | **Raft** | **Raft** |
| Data model | Hierarchical znodes | Flat key-value (with prefixes) | Key-value + service catalog |
| Failure detection | Sessions + ephemeral nodes | **Leases** (TTL) | Health checks + sessions |
| Notifications | One-time watches | Streaming watches | Blocking queries / watches |
| Known for | Hadoop ecosystem, HBase, (older) Kafka | **Kubernetes** | Service discovery and mesh |

> 📘 **Kafka moved away from ZooKeeper:** **KRaft** mode (Kafka's own **Raft**-based metadata quorum) replaced it. ZooKeeper support was **removed in Kafka 4.0**. It's a good example of a system moving consensus **in-house**.

## 7. Conclusion

**ZooKeeper** (like etcd and Consul) is a **coordination service**: a small, in-memory, **consensus-replicated** store (Zab, on a fixed **3 or 5 server** ensemble) that distributed systems use to **agree on shared facts**. It isn't a general database. It provides **linearizable atomic operations** (compare-and-set locks and leases, the only feature that truly needs consensus), **total ordering** (zxid as **fencing tokens**), **failure detection** (sessions and heartbeats that delete **ephemeral nodes**), and **change notifications** (watches). Combined, these power **leader election and work allocation** (ephemeral sequential znodes: lowest wins, each watches its predecessor), **partition assignment and request routing**, **service discovery** (which doesn't strictly need consensus, but can reuse the leader info) and **membership services**. Writes are linearizable, but reads may be stale unless you call `sync()`. It lets thousands of app nodes **outsource consensus** to a few servers instead of implementing it themselves, though systems like Kafka (KRaft) now embed Raft directly.

### Interview one-liners
- **What is ZooKeeper:** "A consensus-backed coordination service (Zab, 3 to 5 servers) for small, slowly changing shared state: leader election, locks, config, membership and partition maps."
- **Key primitives:** "Linearizable compare-and-set, zxid ordering for fencing tokens, sessions with ephemeral nodes for failure detection, and watches for notifications."
- **Leader election recipe:** "Create an ephemeral sequential znode. The lowest number is leader. Everyone else watches their predecessor, and when the leader dies its znode disappears and the next one takes over."
- **Caveat:** "Writes are linearizable, but reads can be stale unless you call sync(). Don't use it as a general database."

### Self-check questions
1. What is ZooKeeper for, and what is it NOT for?
2. Why does ZooKeeper use consensus? Which protocol?
3. List DDIA's four key features. Which one actually requires consensus?
4. How do zxids work as fencing tokens? Walk through the GC-pause example.
5. What are ephemeral and sequential znodes?
6. Describe the leader election recipe and how it avoids the herd effect.
7. How does ZooKeeper help with partition assignment and request routing?
8. Does service discovery need consensus? Why might you still use ZooKeeper for it?
9. What is a membership service, and why is agreement more important than accuracy?
10. Are ZooKeeper reads linearizable? How do you make them so?
11. Compare ZooKeeper, etcd and Consul. Why did Kafka move to KRaft?
