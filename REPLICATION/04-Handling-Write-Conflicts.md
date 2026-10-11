# 04 · Handling Write Conflicts — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 5, "Handling Write Conflicts" (pp. 171–175).
> Folder guide: [README.md](README.md) · Previous: [03-Multi-Leader-Replication.md](03-Multi-Leader-Replication.md) · Next: [05-CRDTs.md](05-CRDTs.md)

## 1. The problem

The **biggest problem with multi-leader replication** is **write conflicts**. (Leaderless databases have them too, see [08](08-Detecting-Concurrent-Writes.md).)

```
Wiki page title = "A"

User 1 ──"title = B"──▶ LEADER 1  ✅ ok           LEADER 2 ◀──"title = C"── User 2  ✅ ok
                           │                         │
                           └──── async replication ──┘
                                       ⚡ CONFLICT: B or C?
```

Both writes **succeed locally**. The conflict is only found **later**, when the changes replicate. This **can't happen with a single leader**.

## 2. Synchronous vs asynchronous conflict detection

| | Single-leader | Multi-leader |
|---|---|---|
| Second writer | **Blocks** and waits, or is **aborted** so the user retries | **Both writes succeed** |
| When the conflict is found | Immediately | **Later, asynchronously**, when it may be **too late to ask the user** |

- You *could* detect conflicts synchronously by waiting for replication to every replica before confirming the write.
- But that throws away the **main benefit** of multi-leader (each leader accepting writes independently). If you want synchronous detection, **just use single-leader**.

## 3. Strategy 1: Conflict avoidance (the most recommended)

> **If all writes to a given record go through the same leader, conflicts can't happen.**

- Many multi-leader implementations handle conflicts **badly**, so avoiding them is the **most common advice**.
- **Example:** route each user to their **"home" datacenter** (e.g. the nearest one) and use its leader for reads and writes. From that user's point of view it's **single-leader**.
- ❌ **It breaks down** when a record's leader has to change:
  - a **datacenter fails** and traffic is rerouted
  - a **user moves** and is now closer to another datacenter

  Then concurrent writes on different leaders become possible again.

## 4. Strategy 2: Converge toward a consistent state

- A single leader applies writes **in order**, so the **last write sets the final value**.
- Multi-leader has **no defined order**: Leader 1 sees B then C (final **C**), while Leader 2 sees C then B (final **B**). ❌ **The replicas disagree forever.**
- Every replication scheme must make **every replica end up with the same value** once all changes have replicated. This is called **convergent** conflict resolution.

| Method | How it works | Result |
|---|---|---|
| **Highest unique ID wins** | Give each write an ID (timestamp, random number, UUID, hash) and keep the highest. With a **timestamp**, this is **Last Write Wins (LWW)** | Popular but **dangerously prone to data loss** (see [08](08-Detecting-Concurrent-Writes.md#2-last-write-wins-lww)) |
| **Highest replica ID wins** | Writes from a higher-numbered replica always beat lower ones | **Also loses data** |
| **Merge the values** | e.g. sort and join them: title = **"B/C"** | Nothing lost, but may be odd |
| **Record the conflict** | Keep **all versions** in an explicit structure and let application code (or the **user**) resolve it later | Nothing lost, but more work |

## 5. Strategy 3: Custom conflict resolution logic

The best resolution depends on the **application**, so most tools let you **write your own conflict handler**:

| | **On write** | **On read** |
|---|---|---|
| When it runs | As soon as the database **detects a conflict** in the replication log | The **next time the data is read** |
| How | The database calls your handler (e.g. **Bucardo** lets you write a snippet of **Perl**) | **All conflicting versions are stored** and returned to the application, which resolves them (automatically or by **asking the user**) and **writes the result back** |
| Limits | Runs in the **background**, must be **fast**, and **can't ask the user** | More flexible |
| Example | Bucardo | **CouchDB** |

⚠️ Conflict resolution usually works **per row or document**, **not per transaction**. If a transaction made several writes, **each write is resolved separately**.

## 6. Automatic conflict resolution

Custom handlers get **complicated and buggy**.

> **Amazon shopping cart bug:** the conflict handler kept **items added** to the cart but **not items removed**, so **removed items reappeared** in customers' carts.

Research into **automatic** conflict resolution:

| Approach | Idea | Used in |
|---|---|---|
| **CRDTs** (Conflict-free Replicated Data Types) | Data structures (sets, maps, ordered lists, counters …) that **many users can edit concurrently**, and which **merge automatically and sensibly**. They use **two-way merges** | **Riak 2.0** → see [05-CRDTs.md](05-CRDTs.md) |
| **Mergeable persistent data structures** | Track **history explicitly, like Git**, and use a **three-way merge** | Research |
| **Operational transformation (OT)** | Designed for concurrent editing of an **ordered list**, such as the characters in a document | **Etherpad, Google Docs** |

These are **still young** in databases but likely to spread. They could make multi-leader sync **much simpler**.

## 7. What counts as a conflict?

- **Obvious:** two writes set the **same field of the same record** to different values (the wiki title).
- **Subtle:** a **meeting room booking** system must not allow **overlapping bookings for the same room**. Even if the app **checks availability first**, two bookings made on **two different leaders** can still conflict.
  - This is like **write skew** in transactions (see [Write-Skew-and-Phantoms.md](../TRANSACTIONS/Write-Skew-and-Phantoms.md)), but across replicas.
- There's **no quick, ready-made answer**. The book returns to this in Chapters 7 and 12.

## 8. Conclusion

In multi-leader systems, two leaders can accept **conflicting writes** to the same data, and the conflict is only found **asynchronously**, too late to ask the user. Detecting it synchronously would cancel out multi-leader's benefits. The best strategy is **conflict avoidance**: route all writes for a record to one "home" leader. That breaks when the leader has to change. Otherwise the replicas must **converge**: **LWW or highest ID wins** (simple, but **loses data**), **merge the values**, or **keep every version** for later resolution. Applications can plug in **custom handlers** that run **on write** (Bucardo, fast and in the background) or **on read** (CouchDB, returning all versions to the app). Resolution is **per row or document, not per transaction**. Hand-written logic is error-prone, as Amazon's cart bug showed, which motivates **automatic resolution**: **CRDTs**, **mergeable persistent data structures** and **operational transformation**. Conflicts can also be **subtle**, like overlapping room bookings.

### Interview one-liners
- **Why conflicts happen:** "Two leaders accept writes to the same data independently, so the conflict only shows up asynchronously during replication."
- **Best strategy:** "Avoid conflicts by routing every write for a record through the same leader, such as the user's home region."
- **Convergence:** "All replicas must end up with the same value. LWW does that cheaply but silently drops writes. Merging or keeping siblings doesn't lose data."
- **On write vs on read:** "On-write handlers run in the background as soon as a conflict is detected. On-read keeps every version and lets the application resolve it when the data is next read."

### Self-check questions
1. Why can't conflicts happen with a single leader?
2. Why is synchronous conflict detection pointless in multi-leader replication?
3. How does conflict avoidance work, and when does it break down?
4. What does "convergent" mean? Why would replicas otherwise disagree?
5. Name four ways to converge, and say which ones lose data.
6. Compare on-write and on-read conflict resolution, with an example tool for each.
7. Why is resolution per row or document a problem for transactions?
8. Explain the Amazon shopping cart bug.
9. Compare CRDTs, mergeable persistent data structures and operational transformation.
10. Give an example of a subtle conflict.
