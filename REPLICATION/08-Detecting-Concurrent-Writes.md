# 08 · Detecting Concurrent Writes (LWW, Happens-Before, Version Vectors) — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 5, "Detecting Concurrent Writes" (pp. 184–191).
> Folder guide: [README.md](README.md) · Previous: [07-Quorums.md](07-Quorums.md)

## 1. The problem: replicas see writes in different orders

Dynamo-style databases let several clients write the **same key at the same time**, so conflicts happen **even with strict quorums**. They can also appear during **read repair** and **hinted handoff**.

```
Clients A and B both write key X to a 3-node datastore at the same time:

Node 1:  receives A ............... (never gets B: transient outage)   → X = A
Node 2:  receives A, then B                                            → X = B
Node 3:  receives B, then A                                            → X = A

If each node just overwrites → nodes disagree FOREVER ❌
```

To be **eventually consistent**, the replicas must **converge** on the same value. Most databases handle this **poorly**, so **you, the developer, need to understand it** to avoid losing data.

## 2. Last write wins (LWW)

**Idea:** each replica keeps only the **"most recent"** value and discards older ones.

- ⚠️ "Recent" is **misleading**. Neither client knew about the other, so **neither happened "first"**. The writes are **concurrent**, and their order is **undefined**.
- LWW **forces an arbitrary order**: attach a **timestamp** to each write and keep the **biggest**.

| | |
|---|---|
| Used by | **Cassandra** (the **only** conflict resolution it supports), **Riak** (optional) |
| ✅ Achieves | **Eventual convergence**, cheaply |
| ❌ Costs | **Durability.** Of several concurrent writes, **all reported successful** (written to w replicas), **only one survives** and the rest are **silently discarded** |
| ❌ Also | It can drop **non-concurrent** writes because of **clock skew** (Chapter 8) |
| OK for | **Caching**, where losing a write is acceptable |
| Safe usage | **Write each key only once and treat it as immutable**, e.g. in Cassandra use a **UUID as the key**, so each write has a unique key |

## 3. The "happens-before" relationship and concurrency

> **A happens before B** if B **knows about** A, **depends on** A, or **builds on** A.
> **A and B are concurrent** if **neither happens before the other**, meaning neither knows about the other.

For any two operations A and B there are **three possibilities**:

| Case | What to do |
|---|---|
| A happened before B | B **overwrites** A |
| B happened before A | A **overwrites** B |
| A and B are **concurrent** | **Conflict**: it must be **resolved or merged** |

Examples:
- An **insert, then an increment of the inserted value**: **not concurrent**. The increment builds on the insert (it's **causally dependent**).
- **Two clients writing X without knowing about each other**: **concurrent**.

> **Concurrency is not about clock time.** Two operations are concurrent if they're **unaware of each other**, even if they happened seconds apart. For example, a **slow network** may have stopped one from learning about the other. The book compares this to **relativity**: events can only affect each other if information can travel between them.

## 4. Capturing happens-before: the shopping cart algorithm (one replica)

Two clients add items to the **same cart** concurrently. The server gives each write a **version number**.

| Step | Client | Sends | Based on version | Server keeps afterwards (siblings) |
|---|---|---|---|---|
| 1 | Client 1 | `[milk]` | — | **v1:** `[milk]` |
| 2 | Client 2 | `[eggs]` | — (didn't see milk) | **v1:** `[milk]`, **v2:** `[eggs]` (concurrent, keep both) |
| 3 | Client 1 | `[milk, flour]` | v1 | **v2:** `[eggs]`, **v3:** `[milk, flour]` (v1 overwritten) |
| 4 | Client 2 | `[eggs, milk, ham]` (merged what it got) | v2 | **v3:** `[milk, flour]`, **v4:** `[eggs, milk, ham]` |
| 5 | Client 1 | `[milk, flour, eggs, bacon]` (merged) | v3 | **v4:** `[eggs, milk, ham]`, **v5:** `[milk, flour, eggs, bacon]` |

**No write is lost**, and older versions are eventually overwritten.

### The server's rules
1. Keep a **version number per key**, **increment it on every write**, and store it with the value.
2. On **read**, return **all values not yet overwritten** plus the **latest version number**. **A client must read before it writes.**
3. On **write**, the client sends the **version number from its last read** and **merges all the values** it received.
4. The server **overwrites every value with that version number or lower** (they were merged into the new value) and **keeps values with higher version numbers** (they're **concurrent** with this write).

➡️ A write **without** a version number is concurrent with everything. It **overwrites nothing** and is just added as another value.
➡️ The server never needs to **look inside the value**, only at version numbers, so the value can be any data structure.

## 5. Merging concurrently written values (siblings)

- Riak calls concurrent values **siblings**. Merging them is **the same problem** as multi-leader conflict resolution (see [04](04-Handling-Write-Conflicts.md)).
- Picking one by timestamp (LWW) **loses data**, so do something smarter in application code.
- **Shopping cart:** take the **union**: `[milk, flour, eggs, bacon]` ∪ `[eggs, milk, ham]` = **`[milk, flour, eggs, bacon, ham]`**.
- ⚠️ **Removals break union.** If an item was removed in only one sibling, the union **brings it back**. That's the Amazon cart bug.
  - **Fix:** don't delete the item. Leave a **tombstone** (a deletion marker with a version number), the same idea as in log compaction ([Hash-Indexes.md](../DATABASE%20INDEXES/Hash-Indexes.md)).
- Merging by hand is **error-prone**. **CRDTs** do it automatically, **including deletions** (Riak data types). See [05-CRDTs.md](05-CRDTs.md).

## 6. Version vectors (multiple replicas)

With **several replicas accepting writes**, a single version number isn't enough. Use **a version number per replica, per key**:

- Each replica **increments its own counter** when it processes a write.
- It also **tracks the counters it has seen from every other replica**.
- The collection of all of them is a **version vector**. It tells the database what to **overwrite** and what to **keep as siblings**.

```
How to compare two version vectors (standard rule):

V1 = {A:2, B:1}   V2 = {A:2, B:3}   → every slot V1 ≤ V2 → V1 happened before V2 → V2 overwrites V1
V1 = {A:3, B:1}   V2 = {A:2, B:2}   → neither ≤ the other → CONCURRENT → keep both as siblings
```

- **Riak 2.0** uses **dotted version vectors** and encodes them as a string called the **causal context**.
- Version vectors are **sent to the client on read** and **sent back on write**.
- It's **safe to read from one replica and write to another**. That may create siblings, but **no data is lost** if siblings are merged correctly.

> **Version vector vs vector clock:** they're often confused and not quite the same. **To compare the state of replicas, version vectors are the right structure.**

## 7. Ways to handle concurrent writes, compared

| Approach | Detects concurrency? | Loses data? | Who merges | Example |
|---|---|---|---|---|
| **LWW (timestamps)** | ❌ Forces an order | ❌ **Yes**, silently | Nobody: the loser is dropped | Cassandra |
| **Version number** (one replica) | ✅ Yes | ✅ No | Client merges siblings | Single-node cart example |
| **Version vectors** (many replicas) | ✅ Yes | ✅ No | Client merges siblings | Riak (dotted version vectors) |
| **CRDTs** | ✅ Built in | ✅ No | **Automatic** | Riak data types |

## 8. Conclusion

In leaderless (and multi-leader) systems, replicas receive concurrent writes **in different orders**, so blindly overwriting leaves them **permanently inconsistent**. **Last write wins** forces convergence by keeping the largest timestamp, but it **silently drops writes that were reported successful** (and can drop non-concurrent ones because of clock skew). Use it only for caches, or for keys written once (UUIDs). The right model is **happens-before**: B overwrites A only if B **knew about** A. Otherwise they're **concurrent**, whatever the clock says, and must be merged. On one replica, a **version number per key** captures this: clients read before writing, send back the version they saw, and the server overwrites lower versions and keeps higher ones as **siblings**. Applications then **merge siblings** (e.g. union for a cart), using **tombstones** so deletions survive, or let **CRDTs** do it. With many replicas, a **version vector** (a counter per replica) tells overwrites apart from concurrent writes. Riak uses **dotted version vectors** sent as a **causal context**.

### Interview one-liners
- **LWW:** "Keep the write with the highest timestamp. It converges, but it silently loses concurrent writes. Only safe for caches or immutable keys."
- **Concurrent:** "Two writes are concurrent if neither knew about the other, regardless of wall-clock time."
- **Version vectors:** "One counter per replica. If one vector is less than or equal to the other in every slot, it's an overwrite. Otherwise the writes are concurrent and both are kept as siblings."
- **Siblings:** "Concurrent values the application must merge, with tombstones for deletes, or that a CRDT merges automatically."

### Self-check questions
1. Walk through the three-node example where replicas end up permanently inconsistent.
2. How does LWW work? Why does it lose data, and when is it safe?
3. Define "happens-before" and "concurrent". Why isn't concurrency about clock time?
4. Walk through the five shopping cart steps and the server's versions.
5. State the server's four rules for version numbers.
6. What happens to a write sent without a version number?
7. How do you merge cart siblings, and why are tombstones needed?
8. Why do multiple replicas need version vectors instead of one version number?
9. Compare V1 = {A:3, B:1} with V2 = {A:2, B:2}. What's the relationship?
10. Version vector vs vector clock: which does the book recommend for comparing replicas?
