# 02 · Consistent Hashing — Revision Notes

> Source: *Designing Data-Intensive Applications*, Chapter 6, the "Consistent Hashing" sidebar (p. 204) and "Partitioning proportionally to nodes" (pp. 212–213).
> ⚠️ The book covers this only briefly and even recommends avoiding the term. Sections marked **📘 Beyond the book** add the standard **hash-ring** explanation (Karger et al., 1997; Amazon Dynamo, 2007) that interviews expect.
> Folder guide: [README.md](README.md) · Previous: [01-Partitioning.md](01-Partitioning.md) · Next: [03-Linearizability.md](03-Linearizability.md)

## 1. What the book says

- **Consistent hashing** (Karger et al.) was designed to spread load across an **internet-wide system of caches**, such as a **CDN**.
- It uses **randomly chosen partition boundaries**, so it needs **no central control or consensus**.
- ⚠️ "Consistent" here has **nothing to do** with replica consistency or ACID consistency. It describes a **rebalancing approach**.
- Kleppmann's view: the original approach **doesn't work very well for databases**, and database docs often use the term **inaccurately**, so it's **better to just say "hash partitioning"**.
- The closest real database version is **Cassandra / Ketama**: a **fixed number of partitions per node**, where a new node **randomly splits existing partitions** and takes half of each (see [01 §7](01-Partitioning.md#7-rebalancing-partitions)).

## 2. The problem it solves: `hash mod N`

```
node = hash(key) mod N

N = 10 → 11 nodes:  a key only stays put if hash mod 10 == hash mod 11
                     → roughly 90% of ALL keys move ❌
```

**Goal:** when a node is **added or removed**, move only about **1/N of the keys**, just the share the new node should own.

## 3. The hash ring 📘 Beyond the book

1. Imagine the hash output space (e.g. **0 … 2³² − 1**) bent into a **circle (ring)**.
2. Hash each **node** (e.g. its IP) onto the ring.
3. Hash each **key** onto the ring and **walk clockwise**: the **first node** you meet owns the key.

```
                     0 / 2^32
                        │
              Node A ●──┼──────────╮
                ╱       │            ╲
     key k3 ○ ╱                        ╲  ○ key k1 → walks clockwise → Node B
             │                          │
     Node D ●                            ● Node B
             │                          │
              ╲                        ╱
                ╲   ○ key k2         ╱
                  ╰──────● Node C ──╯
                  k2 → walks clockwise → Node C
```

### Adding a node
```
Before:  ... Node A ─────────────────────── Node B ...
              keys in (A, B] → B
Add Node X between A and B:
         ... Node A ─────── Node X ──────── Node B ...
              keys in (A, X] → X  (moved from B)
              keys in (X, B] → B  (unchanged)
```
- **Only the keys between X's predecessor and X move**, all of them taken from **one neighbour**. Everything else stays put. ✅

### Removing a node
- Its keys move to the **next node clockwise**. No other keys move. ✅

## 4. The problem: uneven load → virtual nodes 📘 Beyond the book

With only a few nodes, random positions leave **uneven arcs**. Some nodes own far more of the ring than others, and a removed node dumps **all** its load on **one** neighbour.

> **Virtual nodes (vnodes):** each physical node is placed on the ring **many times** (e.g. A#1, A#2, … A#100).

```
Ring: A#1  C#4  B#2  A#7  D#1  B#9  C#2  A#3  D#6  B#5 ...
      (each physical node owns many small, scattered arcs)
```

| Benefit | Why |
|---|---|
| **Even load** | Many small arcs average out |
| **Smooth rebalancing** | A new node takes small slices from **many** nodes, and a removed node's load spreads over **many** nodes |
| **Heterogeneous hardware** | A bigger machine gets **more vnodes**, so it takes more load |

This is exactly the book's **"partitions proportional to nodes"** idea: **Cassandra** used **256 tokens per node** by default when the book was written (newer versions use fewer).

## 5. Replication on the ring 📘 Beyond the book

In **Dynamo-style** systems, a key is stored on its owner **plus the next N − 1 distinct physical nodes clockwise**. This is the **preference list**.

```
key k → owner Node B, replicas: Node C, Node D   (n = 3)
```

That's where the **n** in quorums comes from (see [REPLICATION/07-Quorums.md](../REPLICATION/07-Quorums.md)).

## 6. Where it's used

| System | How |
|---|---|
| **Akamai CDN** (the original Karger et al. use) 📘 | Spread cached content over cache servers |
| **Amazon Dynamo** 📘 | Ring + vnodes + preference lists |
| **Cassandra, Riak** | Token ring with vnodes (Cassandra: "partitions per node") |
| **Ketama** (memcached clients) | Consistent hashing for cache servers |
| **Load balancers and caches** 📘 | Sticky routing that survives servers being added or removed |

## 7. Alternatives 📘 Beyond the book

| Technique | Idea |
|---|---|
| **Rendezvous (HRW) hashing** | For each key, compute `hash(key, node)` for every node and pick the highest. Minimal movement, no ring |
| **Jump consistent hash** (Google) | Tiny, fast, no memory, but nodes must be numbered 0…N−1 (no arbitrary removal) |
| **Fixed partitions + map** (the book's preferred way) | Many partitions, and a lookup table (often in ZooKeeper) maps partitions to nodes |

The book notes that **newer hash functions** achieve the same effect with **less metadata**.

## 8. Comparison

| | `hash mod N` | Consistent hashing (ring + vnodes) | Fixed partitions + map |
|---|---|---|---|
| Keys moved when adding 1 node to N | **~all** ❌ | **~1/(N+1)** ✅ | **~1/(N+1)** ✅ (whole partitions) |
| Central metadata needed | No | No (positions come from hashes) | Yes (partition → node map) |
| Load evenness | Even | Even **with vnodes** | Even (assign partitions) |
| Range queries | ❌ | ❌ | ❌ (hash-based) |

## 9. Conclusion

**Consistent hashing** places both **nodes and keys on a hash ring**, and each key belongs to the **first node clockwise**. Adding or removing a node only moves the keys in **one arc** (about 1/N of the data), whereas `hash mod N` reshuffles almost everything. Plain rings give **uneven arcs**, so real systems use **virtual nodes**: many positions per physical node for even load, smooth rebalancing and support for bigger machines. Dynamo-style stores replicate each key to the **next N − 1 nodes** clockwise. The book's caution is that "consistent" has nothing to do with consistency, the original idea fits CDNs better than databases, and in databases it shows up as **Cassandra's partitions-per-node** scheme, so "hash partitioning" is the clearer term.

### Interview one-liners
- **Consistent hashing:** "Hash nodes and keys onto a ring, and a key goes to the next node clockwise. Adding or removing a node only remaps the neighbouring arc, about 1/N of the keys."
- **Virtual nodes:** "Each server takes many ring positions, which evens out load, spreads rebalancing across many nodes and supports heterogeneous capacity."
- **Why not mod N:** "Changing N remaps nearly every key, which is a massive data movement."

### Self-check questions
1. Why is `hash(key) mod N` bad when N changes?
2. Draw a hash ring and show which node owns a key.
3. What moves when a node is added? When a node is removed?
4. Why do we need virtual nodes? List three benefits.
5. How do Dynamo-style systems pick replicas on the ring?
6. Why does Kleppmann suggest avoiding the term "consistent hashing"?
7. How does Cassandra's "partitions proportional to nodes" relate to consistent hashing?
8. Name two alternatives to ring-based consistent hashing.
