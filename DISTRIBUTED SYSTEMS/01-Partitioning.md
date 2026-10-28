# 01 · Partitioning (Sharding) — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), **Chapter 6: Partitioning** (pp. 199–217).
> Folder guide: [README.md](README.md) · Next: [02-Consistent-Hashing.md](02-Consistent-Hashing.md)

## 1. What partitioning is and why

> **Partitioning = splitting a large dataset into smaller pieces (partitions) so that each piece can live on a different machine.**

| Database | Calls a partition a… |
|---|---|
| MongoDB, Elasticsearch, SolrCloud | **shard** |
| HBase | **region** |
| Bigtable | **tablet** |
| Cassandra, Riak | **vnode** |
| Couchbase | **vBucket** |

- **Goal: scalability.** Spread the **data** over many disks and the **query load** over many CPUs. In theory, 10 nodes handle 10× the data and 10× the throughput.
- Each record belongs to **exactly one partition**.
- ⚠️ This has **nothing to do with network partitions** (network faults), which is the "P" in CAP.

### Partitioning + replication
Partitions are usually **replicated** too: each partition has copies on several nodes.

```
            Node 1            Node 2            Node 3
          ┌────────┐        ┌────────┐        ┌────────┐
          │ P1 (L) │        │ P1 (F) │        │ P1 (F) │
          │ P2 (F) │        │ P2 (L) │        │ P2 (F) │
          │ P3 (F) │        │ P3 (F) │        │ P3 (L) │
          └────────┘        └────────┘        └────────┘
   L = leader replica of that partition, F = follower replica
```

Each node is the **leader for some partitions** and a **follower for others**. Everything from the replication notes ([../REPLICATION](../REPLICATION/README.md)) applies per partition.

## 2. The core problem: skew and hot spots

- **Skew:** some partitions get **more data or more queries** than others.
- **Hot spot:** a partition with **disproportionately high load**. In the worst case, one node does all the work while 9 sit idle.
- Assigning records **randomly** spreads them evenly, but then you **don't know where a record is**, so every read must ask **every node**. ❌

## 3. Partitioning by key range

Give each partition a **continuous range of keys**, like volumes of a paper encyclopedia (A–B, C–D, …, T–Z).

```
Partition 1: [A ... Bz]   Partition 2: [C ... Ez]   ...   Partition 12: [T ... Z]
```

- Ranges are **not evenly spaced**, because the data isn't. Boundaries must **adapt to the data** (set by an admin or automatically).
- Inside each partition, keys are **kept sorted** (SSTables), so **range scans are easy**.
- **Used by:** Bigtable, HBase, RethinkDB, MongoDB (before 2.4).

| ✅ Pros | ❌ Cons |
|---|---|
| **Efficient range queries** (e.g. all sensor readings for one month) | **Hot spots** with certain access patterns |

**Hot spot example:** the key is a **timestamp**, so all of today's writes go to **today's partition** while the others sit idle.
**Fix:** **prefix the key with the sensor name**, e.g. `sensor42:2024-05-01T10:00`. Writes now spread across sensors, but reading many sensors over a time range needs **one range query per sensor**.

## 4. Partitioning by hash of key

Run every key through a **hash function** and give each partition a **range of hash values**.

```
hash("user_17")  = 0x3A1F...  → partition 1 (hash range 0x0000 – 0x3FFF)
hash("user_18")  = 0xC072...  → partition 4 (hash range 0xC000 – 0xFFFF)
   similar keys → completely different partitions (evenly spread)
```

- A good hash turns **skewed keys into a uniform distribution**. It doesn't need to be cryptographically strong: **MD5** (Cassandra, MongoDB), **Fowler–Noll–Vo** (Voldemort).
- ⚠️ Don't use a language's built-in hash such as Java's `Object.hashCode()` or Ruby's `Object#hash`. The **same key can hash differently in different processes**.
- Boundaries can be **evenly spaced** or **chosen pseudorandomly**. The pseudorandom version is sometimes called **consistent hashing** (see [02](02-Consistent-Hashing.md)).

| ✅ Pros | ❌ Cons |
|---|---|
| **Even distribution**, few hot spots | **Range queries become inefficient**: adjacent keys end up scattered |

- MongoDB (hash sharding) sends range queries to **all partitions**. Riak, Couchbase and Voldemort **don't support** primary-key range queries at all.

### The compromise: compound keys (Cassandra)
- Declare a **compound primary key** `(user_id, update_timestamp)`.
- **Only the first part is hashed** to pick the partition. The other columns are a **sorted concatenated index** inside it.
- So you can **fetch one user's updates in a time range**, sorted, from **one partition**. That's an elegant **one-to-many** model.

### Key range vs hash partitioning

| | Key range | Hash of key |
|---|---|---|
| Distribution | Can be skewed | Even |
| Range queries | ✅ Efficient | ❌ Scatter to all partitions |
| Hot spot risk | High (sequential keys like timestamps) | Low (except for a single hot key) |
| Rebalancing | Usually **dynamic splitting** | Usually a **fixed number of partitions** |
| Examples | HBase, Bigtable, RethinkDB | Cassandra, Riak, Voldemort, MongoDB (hashed) |

## 5. Skewed workloads and relieving hot spots

Hashing **can't fix a single hot key**: identical keys hash identically.
- Example: a **celebrity** with millions of followers posts, causing a storm of writes to **one key** (their user ID or the post ID).
- Most databases **can't fix this automatically**, so the **application** must.

**Technique: split the hot key**
```
"celebrity_42"  →  "celebrity_42_00", "celebrity_42_01", ..., "celebrity_42_99"
writes: pick a random 2-digit suffix → spread over 100 keys → many partitions
reads:  must read all 100 keys and combine ❌ extra work
```
- Only do this for the **few hot keys**, which means you must **track which keys are split**.

## 6. Partitioning and secondary indexes

A secondary index searches **by value** ("all red cars", "all actions by user 123") and **doesn't map neatly to partitions**. There are two approaches:

### (a) Document-partitioned index = local index
Each partition indexes **only its own documents**.

```
Partition 0 (cars 0–499)                  Partition 1 (cars 500–999)
  color:red   → [191, 306]                  color:red   → [768]
  color:black → [235]                       color:silver→ [515, 893]
  make:Honda  → [235, 306]                  make:Honda  → [515]
```

- **Writes:** touch **only one partition**. ✅
- **Reads:** "find red cars" must ask **every partition** and merge the results. This is **scatter/gather**: expensive, and it suffers from **tail latency amplification**. ❌
- **Used by:** MongoDB, Riak, Cassandra, Elasticsearch, SolrCloud, VoltDB.

### (b) Term-partitioned index = global index
One **global** index covering all partitions, **itself partitioned by the term** (e.g. colors a–r on partition 0, s–z on partition 1).

```
Index partition 0 (colors a–r):  color:black → [235]   color:red → [191, 306, 768]
Index partition 1 (colors s–z):  color:silver → [515, 893]   color:yellow → [...]
```

- **Reads:** go to **only the partition holding that term**. ✅
- **Writes:** one document may update **several index partitions** on different nodes. Slower and more complex. ❌
- Keeping it perfectly in sync needs a **distributed transaction**, so updates are usually **asynchronous** (e.g. **DynamoDB global secondary indexes** update "within a fraction of a second" normally).
- **Used by:** DynamoDB GSIs, Riak search, Oracle data warehouse.

| | Local (document-partitioned) | Global (term-partitioned) |
|---|---|---|
| Write cost | 1 partition ✅ | Several partitions ❌ |
| Read cost | **Scatter/gather to all** ❌ | 1 partition ✅ |
| Freshness | Synchronous | Often **asynchronous** (may lag) |

## 7. Rebalancing partitions

**Rebalancing** = moving load from one node to another when:
- query throughput grows (more CPUs),
- data grows (more disks and RAM),
- or a machine fails (others take over).

**Requirements:**
1. Afterwards, load is **shared fairly**.
2. The database keeps **accepting reads and writes during** rebalancing.
3. **Move no more data than necessary**.

### ❌ How NOT to do it: `hash(key) mod N`
```
hash(key) = 123456
10 nodes: 123456 mod 10 = 6
11 nodes: 123456 mod 11 = 3   ← moved!
12 nodes: 123456 mod 12 = 0   ← moved again!
```
When N changes, **almost every key moves**, which makes rebalancing hugely expensive.

### Strategy 1: Fixed number of partitions
Create **many more partitions than nodes** at the start (e.g. **1,000 partitions for 10 nodes**, about 100 each).

```
Before (4 nodes):  N0:[p0 p4 p8 p12]  N1:[p1 p5 p9 p13]  N2:[p2 p6 p10 p14]  N3:[p3 p7 p11 p15]
Add node N4: it steals a few partitions from each existing node
After  (5 nodes):  N0:[p0 p8 p12]  N1:[p1 p5 p13]  N2:[p2 p6 p14]  N3:[p3 p7 p15]  N4:[p4 p9 p10 p11]
```

- **Only whole partitions move.** The key → partition mapping **never changes**, only partition → node.
- Old assignments serve requests while the transfer is in progress.
- More powerful nodes can take **more partitions**.
- **Used by:** Riak, Elasticsearch, Couchbase, Voldemort.
- ⚠️ The partition count is **fixed at setup**, which caps the **maximum number of nodes**. If it's too high there's overhead. If it's too low, partitions become **huge**. It's hard to pick when the data size varies a lot.

### Strategy 2: Dynamic partitioning
- When a partition grows past a size limit (**HBase default: 10 GB**), it **splits in two**. When it shrinks, it **merges** with a neighbour, much like a B-tree.
- ✅ The number of partitions **adapts to the data volume**.
- ⚠️ An empty database starts with **one partition**, so one node takes all the writes at first. The fix is **pre-splitting** (HBase, MongoDB), which requires knowing the key distribution.
- **Used by:** HBase, RethinkDB (key range), MongoDB 2.4+ (key range or hash).

### Strategy 3: Partitioning proportionally to nodes
- A **fixed number of partitions per node** (**Cassandra: 256 per node** by default at the time of writing).
- A new node **randomly picks existing partitions to split** and takes half of each.
- Partition size stays fairly stable as data and nodes grow together.
- Requires **hash partitioning**. It's the closest match to the original **consistent hashing** (see [02](02-Consistent-Hashing.md)).
- **Used by:** Cassandra, Ketama.

| Strategy | Number of partitions | Partition size | Examples |
|---|---|---|---|
| Fixed number | Fixed | Grows with the data | Riak, Elasticsearch, Couchbase, Voldemort |
| Dynamic | Grows with the data | Bounded (split and merge) | HBase, RethinkDB, MongoDB |
| Per node | Grows with the nodes | Fairly stable | Cassandra, Ketama |

### Automatic or manual rebalancing?
- **Fully automatic** is convenient but **unpredictable**: moving data is expensive and can overload the network.
- ⚠️ Combined with **automatic failure detection** it's **dangerous**. An overloaded, slow node is wrongly declared dead, rebalancing piles **more load** onto it, and you get a **cascading failure**.
- ✅ A **human in the loop** is safer. **Couchbase, Riak and Voldemort** suggest an assignment, and an **admin commits it**.

## 8. Request routing (service discovery)

**"I want key `foo`: which IP and port do I connect to?"** Partitions move over time, so somebody must track the assignment.

```
(1) Any node              (2) Routing tier              (3) Partition-aware client
client → any node         client → router → right node   client → right node directly
  (forwards if needed)      (partition-aware LB)
```

| Approach | Who decides | Examples |
|---|---|---|
| **1. Contact any node** | That node forwards the request to the owner | **Cassandra, Riak** (using a **gossip protocol** to share cluster state) |
| **2. Routing tier** | A partition-aware load balancer | **MongoDB** (`mongos` + config servers), Espresso (Helix), Couchbase (`moxi`) |
| **3. Client knows** | The client connects straight to the owner | Partition-aware clients |

**Who keeps the map of partition → node?** Often **ZooKeeper** (see [08](08-ZooKeeper.md)):
```
nodes register in ZooKeeper ──▶ ZooKeeper holds the authoritative partition → node map
routing tier / clients subscribe ──▶ notified on every change
```
- **Uses ZooKeeper:** HBase, SolrCloud, Kafka (historically), and LinkedIn Espresso through Helix.
- **Gossip instead:** Cassandra and Riak. No external dependency, but more complexity in the nodes.
- Finding the router or node IPs themselves changes slowly, so **DNS** is usually enough.

### Parallel query execution
NoSQL stores mostly do single-key reads and writes (plus scatter/gather). **MPP** (massively parallel processing) analytic databases split complex queries with joins and aggregations into **stages that run in parallel** across nodes.

## 9. Conclusion

**Partitioning (sharding)** splits data that's too big for one machine into partitions spread over many nodes, so that data and queries **scale out**. Each partition is usually replicated too. The enemy is **skew and hot spots**. **Key-range partitioning** keeps keys sorted, which makes **range scans** cheap, but sequential keys like timestamps create hot spots. **Hash partitioning** spreads keys evenly but **loses range queries**. Cassandra's **compound keys** combine both, and a single **hot key** must be split by the application. **Secondary indexes** are either **local** (cheap writes, scatter/gather reads) or **global** (cheap reads, expensive and often async writes). **Rebalancing** must stay online and move little data: never `hash mod N`. Instead use a **fixed number of partitions**, **dynamic split and merge**, or **partitions per node**, ideally with a **human approving** moves. **Request routing** (contact any node, a routing tier, or a smart client) relies on **ZooKeeper** or **gossip** to know which node owns which partition.

### Interview one-liners
- **Partitioning:** "Split data across nodes so each holds a subset. It scales storage and throughput, and is usually combined with replication."
- **Range vs hash:** "Range keeps order for scans but risks hot spots. Hash spreads load evenly but kills range queries."
- **Local vs global index:** "Local means cheap writes and scatter/gather reads. Global means single-partition reads and multi-partition, usually async, writes."
- **Rebalancing:** "Never use hash mod N, which moves almost every key. Use many fixed partitions per node and move whole partitions."
- **Routing:** "A coordination service like ZooKeeper, or gossip, keeps the partition-to-node map, and requests are routed by any node, a routing tier or a smart client."

### Self-check questions
1. Why partition? How does it relate to replication?
2. What are skew and hot spots? Why is random assignment a bad idea?
3. Compare key-range and hash partitioning. Give a hot-spot example for key range and its fix.
4. How do Cassandra's compound primary keys give you both even distribution and range scans?
5. How do you handle a single extremely hot key?
6. Compare document-partitioned (local) and term-partitioned (global) secondary indexes.
7. Why is `hash mod N` bad for rebalancing? Use the 123456 example.
8. Compare fixed-number, dynamic, and per-node partitioning.
9. Why can fully automatic rebalancing cause cascading failures?
10. Describe the three request-routing approaches and how ZooKeeper or gossip helps.
