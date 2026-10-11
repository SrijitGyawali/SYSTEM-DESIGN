# 05 · CRDTs (Conflict-free Replicated Data Types) — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 5, "Automatic Conflict Resolution" (p. 174) and "Merging concurrently written values" (pp. 190–191).
> ⚠️ The book only **introduces** CRDTs in a few paragraphs. Sections marked **📘 Beyond the book** add standard CRDT material (Shapiro et al., 2011) that interviewers commonly ask about.
> Folder guide: [README.md](README.md) · Previous: [04-Handling-Write-Conflicts.md](04-Handling-Write-Conflicts.md) · Next: [06-Leaderless-Replication.md](06-Leaderless-Replication.md)

## 1. What the book says

- **CRDTs** are a family of data structures (**sets, maps, ordered lists, counters** …) that **many users can edit concurrently**, and which **automatically resolve conflicts in sensible ways**.
- They use **two-way merges** (merge state A with state B). Mergeable persistent data structures use **three-way merges** (A, B and their common ancestor), like Git.
- Riak 2.0 implements some CRDTs as **Riak data types**, which merge **siblings** automatically, **including deletions**.
- They're the fix for problems like **Amazon's cart**, where removed items came back after a naive union merge.

## 2. The core idea 📘 Beyond the book

> **Each replica updates its own copy without talking to anyone. When replicas exchange state, a merge function combines them, and every replica that has seen the same updates ends up in the same state, whatever the order.**

- **No coordination, no locks, no leader**: writes are always **local and available**.
- The guarantee is called **strong eventual consistency**: replicas that have received the **same set of updates** are in the **same state**.
- Perfect for **multi-leader**, **leaderless**, **offline-first** and **collaborative** apps.

## 3. Why the merge always converges 📘 Beyond the book

The merge function `⊔` must be:

| Property | Formula | Why it matters |
|---|---|---|
| **Commutative** | `a ⊔ b = b ⊔ a` | **Order** of merging doesn't matter |
| **Associative** | `(a ⊔ b) ⊔ c = a ⊔ (b ⊔ c)` | **Grouping** doesn't matter |
| **Idempotent** | `a ⊔ a = a` | **Duplicates** don't matter (receiving the same state twice is harmless) |

Because messages over a network can be **reordered, grouped or duplicated**, these three properties make the result the **same no matter what**. (Mathematically, states form a **join-semilattice** and only ever "grow".)

## 4. Two families of CRDT 📘 Beyond the book

| | **State-based (CvRDT)** | **Operation-based (CmRDT)** |
|---|---|---|
| What's sent | The **whole state** (or just the changes: **delta-state CRDTs**) | Each **operation** (e.g. "increment", "add x") |
| Requirement | Merge is commutative, associative and idempotent | Concurrent operations must **commute**, and delivery must be **exactly once, in causal order** |
| Network needs | Very relaxed (any order, duplicates OK) | Needs a reliable causal broadcast |
| Message size | Bigger | Smaller |

## 5. Common CRDTs, with examples 📘 Beyond the book

### G-Counter (grow-only counter)
Each node has **its own slot** and only increments its own slot.
- **Merge** = **element-wise max**.
- **Value** = **sum** of the slots.

```
Node A increments 3×:  {A:3, B:0, C:0}
Node B increments 2×:  {A:0, B:2, C:0}
Node C increments 1×:  {A:0, B:0, C:1}

merge (max per slot) → {A:3, B:2, C:1}   value = 3 + 2 + 1 = 6 ✅ on every node
```

Why **max** and not **sum**? Merging the same state twice must not **double count**, so the merge has to be **idempotent**.

### PN-Counter (increment and decrement)
Two G-Counters: **P** for increments and **N** for decrements. **Value = sum(P) − sum(N)**. Example use: likes and unlikes, or stock levels.

### Sets

| CRDT | Rule | Catch |
|---|---|---|
| **G-Set** (grow-only) | Add only. **Merge = union** | Can't remove anything |
| **2P-Set** (two-phase) | An **add set** plus a **remove set** (tombstones). Present = in add set and not in remove set | Once removed, an item **can never be added again** |
| **LWW-Element-Set** | Each add or remove has a **timestamp**, and the latest wins | Same **clock and data-loss** risks as LWW |
| **OR-Set** (observed-remove) | Each **add gets a unique tag**. A **remove deletes only the tags it has seen**. A concurrent add with a new tag **survives** ("**add wins**") | Stores tags and tombstones (metadata) |

### OR-Set walkthrough: fixing the Amazon cart

```
Replica 1: add(milk)          → { milk#a }
   ...replicates to Replica 2  → Replica 2 has { milk#a }
Replica 2: remove(milk)       → removes the tag it SAW: #a  → tombstone {a}
Replica 1 (concurrently): add(milk) again → { milk#a, milk#b }

merge → milk#a is removed (tombstoned), milk#b survives → cart = { milk }
```

- An item removed with **no concurrent re-add** stays **removed**, unlike the naive union merge that brought Amazon's items back.
- A **concurrent add** wins, so the user's latest intention isn't lost.

### Registers (single values)
- **LWW-Register:** keeps the value with the **highest timestamp**. Simple, but **loses concurrent writes**.
- **MV-Register** (multi-value): **keeps all concurrent values** (as **siblings**, tracked with version vectors) for the application to merge. This is what Riak does without data types (see [08](08-Detecting-Concurrent-Writes.md)).

### Sequence (text) CRDTs
- For **collaborative editing**: each character gets a **unique, ordered ID**, so concurrent inserts and deletes merge without OT's central transformation.
- Algorithms: **RGA, Logoot, LSEQ**. Libraries: **Automerge, Yjs**.

## 6. CRDTs vs other ways of resolving conflicts

| | **LWW** | **Siblings + app merge** | **CRDTs** | **Operational Transformation** |
|---|---|---|---|---|
| Data loss | ❌ Yes | ✅ No (if merged well) | ✅ No | ✅ No |
| Who merges | Database (by timestamp) | **Your code** (error-prone) | **The data type** (automatic) | A transformation algorithm, usually via a **central server** |
| Coordination needed | No | No | **No** | Usually a server orders operations |
| Example | Cassandra | Riak siblings, CouchDB | Riak data types, Redis Enterprise, Automerge, Yjs | Google Docs, Etherpad |

## 7. Limitations 📘 Beyond the book

- ⚠️ **Metadata overhead:** per-node slots, unique tags and **tombstones** grow over time, and **garbage-collecting** them safely is hard.
- ⚠️ **Only for supported types:** counters, sets, maps, registers and sequences. Arbitrary business logic doesn't fit.
- ⚠️ **No global invariants:** CRDTs **can't enforce** "balance ≥ 0", "unique username" or "no double booking". Those need **coordination** (transactions or consensus).
- ⚠️ **Semantics are a choice:** add-wins vs remove-wins can **surprise** users.

## 8. Where CRDTs are used

| System | Use |
|---|---|
| **Riak 2.0+** (in the book) | Data types: counters, sets, maps, registers, flags |
| **Redis Enterprise** Active-Active 📘 | Multi-region writes to Redis data types |
| **Automerge, Yjs** 📘 | Local-first and collaborative editing apps |
| **Akka Distributed Data** 📘 | Replicated in-memory state across a cluster |

## 9. Conclusion

**CRDTs** are data types (counters, sets, maps, registers, sequences) that let **every replica update locally without coordination** and still **converge automatically**. Their **merge** is **commutative, associative and idempotent**, so message order, grouping and duplicates don't matter, which is called **strong eventual consistency**. A **G-Counter** merges per-node counts with **max** and sums them, and a **PN-Counter** pairs two G-Counters. An **OR-Set** tags every add, so removes only delete what they've seen, which fixes Amazon's "removed items reappear" bug. Compared with **LWW** they **don't lose data**, and compared with hand-written **sibling merging** they're **automatic**. The costs are **metadata and tombstones**, a **limited set of data types**, and **no global invariants** like uniqueness or "balance ≥ 0". They're used in Riak data types, Redis Enterprise Active-Active and collaborative editors like Automerge and Yjs.

### Interview one-liners
- **CRDT:** "A replicated data type whose merge is commutative, associative and idempotent, so replicas can update independently and always converge with no coordination and no lost writes."
- **G-Counter:** "One slot per node. Increment your own slot, merge with element-wise max, and the value is the sum."
- **OR-Set:** "Each add gets a unique tag, and a remove only deletes the tags it has observed, so concurrent adds win and removed items don't come back."
- **Limitation:** "CRDTs can't enforce global constraints like uniqueness. You still need consensus or transactions for those."

### Self-check questions
1. What problem do CRDTs solve in multi-leader and leaderless systems?
2. What three properties must a CRDT merge have, and why each one?
3. State-based vs operation-based CRDTs: what's the difference?
4. Walk through a G-Counter merge. Why max instead of sum?
5. How does a PN-Counter support decrements?
6. Compare G-Set, 2P-Set, LWW-Element-Set and OR-Set.
7. How does an OR-Set fix the Amazon shopping cart bug?
8. How do CRDTs compare with LWW and with operational transformation?
9. Name three limitations of CRDTs.
