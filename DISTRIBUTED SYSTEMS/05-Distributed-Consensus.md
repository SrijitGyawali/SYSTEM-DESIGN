# 05 · Distributed Consensus — Revision Notes

> Source: *Designing Data-Intensive Applications*, **Chapter 9**, "Ordering Guarantees" and "Fault-Tolerant Consensus" (pp. 339–352, 364–369).
> Folder guide: [README.md](README.md) · Previous: [04-Two-Phase-Commit.md](04-Two-Phase-Commit.md) · Next: [06-Raft-Leader-Election.md](06-Raft-Leader-Election.md)

## 1. What consensus is

> **Consensus = getting several nodes to agree on something**, in a way that's **final** and survives failures.

It sounds simple, but **many broken systems** were built by people who thought it was easy.

**Where nodes must agree:**

| Problem | What they agree on |
|---|---|
| **Leader election** | Who the **one** leader is, to avoid **split brain** |
| **Atomic commit** | **Commit or abort** a distributed transaction (all the same way) |
| **Uniqueness / locks** | Who gets the **last seat**, the **username** or the **lock** |
| **Total order broadcast** | The **same order** of messages on every node |

## 2. Background: ordering, causality and total order

### Causality
- **Cause comes before effect**: a question before its answer, a row created before it's updated.
- Causality is a **partial order**. Some events are **concurrent** (neither caused the other), so history **branches and merges**, like Git.
- **Linearizability** is a **total order**: one single timeline. Linearizability **implies** causality.
- 📌 **Causal consistency** is the **strongest model that doesn't slow down with network delays and stays available during network failures**.

### Lamport timestamps
- Each node keeps a counter and stamps events with **(counter, nodeID)**.
- Every node and client tracks the **maximum counter it has seen** and sends it with every request. On seeing a bigger one, it **jumps its own counter up** to match.
- This gives a **total order consistent with causality**: compare counters first, then node IDs.

```
Node 1: (1,N1) ──▶ (2,N1) ───────────────────────▶ (6,N1)   (jumped after seeing 5)
Node 2:          (1,N2) ──▶ (5,N2) ── sends max=5 ──┘
```

### ⚠️ Timestamp ordering isn't enough
- **Example:** two users register the **same username** at the same time.
- Lamport timestamps tell you which came first **only after you've collected every operation**.
- A node can't decide **right now** whether someone else is claiming the name at this moment.
- ➡️ You need to know when the order is **final**. That's **total order broadcast**.

### Total order broadcast (atomic broadcast)
A protocol for delivering messages to all nodes with two safety properties:
1. **Reliable delivery:** if one node gets a message, **all** get it (nothing lost).
2. **Totally ordered delivery:** every node gets messages in the **same order**.

- It's equivalent to building a **shared append-only log**.
- The order is **fixed when a message is delivered**. No message can be slipped in earlier later on.

**Uses:**
- **State machine replication:** same writes in the same order means identical replicas.
- **Serializable transactions** as ordered stored procedures.
- **Lock services with fencing tokens:** the log position is a monotonically increasing token, which ZooKeeper calls the **zxid**.

### The equivalence (a deep result)
> **Linearizable compare-and-set (or increment-and-get) ⇔ total order broadcast ⇔ consensus.** Solve one and you can build the others.

- **Linearizable storage from total order broadcast:** to claim a username, **append** "I claim X" to the log, **wait for it to come back**, and check whether **your message is the first** for X. Writes are linearizable. Reads need extra steps: sequence the read through the log, ZooKeeper's `sync()`, or etcd's quorum read.
- **Total order broadcast from linearizable storage:** use **increment-and-get** on a linearizable counter as a **gap-free sequence number** for every message.

## 3. The formal definition of consensus

One or more nodes **propose** values, and the algorithm **decides** one of them. It must satisfy:

| Property | Meaning | Type |
|---|---|---|
| **Uniform agreement** | **No two nodes decide differently** | Safety |
| **Integrity** | **No node decides twice** (no changing your mind) | Safety |
| **Validity** | A decided value was **proposed by some node** (rules out "always decide null") | Safety |
| **Termination** | Every node that doesn't crash **eventually decides** | **Liveness** (fault tolerance) |

- Without termination it's easy: hard-code a **"dictator"** node. But if it dies, nothing can be decided. **That's exactly 2PC's problem.**
- Consensus assumes a crashed node **may never come back** ("buried under 30 feet of mud"), so any algorithm that **waits for recovery** (like 2PC) **fails termination**.
- ⭐ **Termination needs a majority of nodes working.** Safety (agreement, integrity, validity) is kept **even if a majority fails**: the system may **stop**, but it never makes a **wrong** decision.
- Most algorithms assume **no Byzantine faults** (lying nodes). Byzantine-tolerant consensus needs **fewer than 1/3 faulty nodes**.

### The FLP impossibility result
- **Fischer, Lynch and Paterson** proved that **no algorithm can always reach consensus if a node may crash**.
- But that's in the **asynchronous model**: no clocks, no timeouts.
- With **timeouts** (even unreliable ones), or even just **random numbers**, consensus **becomes solvable**.
- ➡️ Consensus is possible **in practice**.

## 4. Consensus algorithms

**The best-known ones: Viewstamped Replication (VSR), Paxos, Raft, Zab.**
- They're similar but not identical.
- Most decide on a **sequence of values**, so they're really **total order broadcast** algorithms. That's more efficient than one value at a time (for Paxos, this optimization is **Multi-Paxos**).
- Total order broadcast is **repeated rounds of consensus**:
  - **agreement** → same order everywhere
  - **integrity** → no duplicates
  - **validity** → nothing fabricated
  - **termination** → nothing lost

| Algorithm | Used in |
|---|---|
| **Raft** | **etcd** (Kubernetes), Consul 📘, CockroachDB 📘, TiKV 📘, Kafka KRaft 📘 |
| **Zab** | **ZooKeeper** |
| **Paxos / Multi-Paxos** | Google Chubby 📘, Spanner 📘 |
| **Viewstamped Replication** | Research and early systems |

📘 = examples beyond the book.

> ⚠️ Kleppmann's advice: **don't implement consensus yourself**. It's notoriously hard. Use ZooKeeper or etcd.

## 5. Single-leader replication vs consensus

- Single-leader replication is **total order broadcast** (the leader orders all writes). Why didn't Chapter 5 need consensus?
- **Manually chosen leader** = a **dictatorship**. It works, but it **fails termination**: humans must step in when the leader dies.
- **Automatic failover** needs everyone to **agree on the new leader**, which needs **consensus**. But consensus algorithms use a leader…
- 🐔🥚 *"To elect a leader we first need a leader."* How is this resolved?

## 6. Epoch numbers and quorums: how it's resolved

Consensus protocols **don't guarantee a unique leader at all times**. They guarantee **a unique leader within each epoch**.

| Algorithm | Name for an epoch |
|---|---|
| Paxos | **Ballot number** |
| Viewstamped Replication | **View number** |
| Raft | **Term number** |

- When the leader seems dead, an **election** starts with a **higher epoch number**. Epochs are **totally ordered and increasing**.
- If two leaders conflict, **the higher epoch wins**.

**Two rounds of voting:**
1. **Elect a leader:** a quorum votes for it in epoch e.
2. **Decide each proposal:** the leader sends the value, and a **quorum** must accept it. A node accepts **only if it hasn't seen a higher epoch**.

⭐ **The two quorums must overlap.** If a proposal gets a quorum and none of those nodes knows a higher epoch, then **no newer leader exists**, and the leader can safely decide.

```
5 nodes. Election quorum (epoch 7): {A, B, C}.  Proposal quorum: {C, D, E}.
Overlap = C. If a newer epoch 8 existed, C would know → proposal rejected.
```

> "**The truth is defined by the majority.**" A node can't trust its own belief that it's the leader. It must get **votes**.

## 7. Consensus vs two-phase commit

| | 2PC | Consensus |
|---|---|---|
| Leader or coordinator | **Not elected**, fixed | **Elected**, replaceable |
| Votes needed | **Every** participant | **Majority** |
| After failure | **Blocks** (in doubt) | A **recovery process** brings nodes to a consistent state under a new leader |
| Termination | ❌ | ✅ (with a majority alive) |

## 8. Limitations of consensus

| Limitation | Detail |
|---|---|
| **Synchronous replication cost** | Voting before deciding is **synchronous replication**. Many databases prefer **async** for speed, accepting possible data loss |
| **Needs a strict majority** | **3 nodes tolerate 1 failure, 5 tolerate 2.** During a network split, only the **majority side** makes progress |
| **Fixed membership** | Most algorithms assume a **fixed set of voters**. **Dynamic membership** extensions exist but are **less well understood** |
| **Timeout-based failure detection** | With variable delays (especially across regions), nodes **falsely suspect the leader**. **Frequent elections** wreck performance |
| **Sensitive to bad links** | **Raft edge case:** one consistently unreliable link can make **leadership bounce** between two nodes forever, so there's no progress. This is still an open research problem |

## 9. Conclusion

**Consensus** gets nodes to **agree irrevocably** despite failures. It underlies **leader election, atomic commit, locks, uniqueness and total ordering**. Formally it needs **agreement, integrity, validity** (safety) and **termination** (liveness), and termination requires a **working majority**. FLP says it's impossible without timeouts, but **timeouts and randomness make it practical**. **Lamport timestamps** give a causal total order but can't decide things **in the moment**. **Total order broadcast** can, and it's **equivalent** to consensus and to linearizable compare-and-set. Real algorithms (**Paxos, Raft, Zab, VSR**) implement total order broadcast with a **leader per epoch** (ballot, view or term). Each decision needs a **quorum that overlaps the election quorum**, so a stale leader can't win. Unlike **2PC**, they **elect** leaders and need only a **majority**, so they don't block. The costs are **synchronous voting, majority requirements (3 nodes for 1 failure), fixed membership, timeout sensitivity and edge cases with flaky links**. **Use ZooKeeper or etcd rather than building your own.**

### Interview one-liners
- **Consensus:** "Nodes agree on a value such that no two decide differently, no one changes their mind, the value was proposed, and every live node eventually decides. It needs a majority alive."
- **FLP:** "Deterministic consensus is impossible in a fully asynchronous system with crashes. Timeouts make it practical."
- **Total order broadcast:** "Every node delivers the same messages in the same order. It's equivalent to consensus and to linearizable compare-and-set."
- **Epochs:** "A unique leader per term or ballot. Decisions need a quorum that overlaps the election quorum, so a stale leader can't commit."
- **vs 2PC:** "Consensus elects its leader and needs a majority. 2PC has a fixed coordinator, needs everyone, and blocks."

### Self-check questions
1. Give four situations that need consensus.
2. Causality vs total order: why is causal consistency attractive?
3. How do Lamport timestamps work? Why can't they enforce unique usernames?
4. Define total order broadcast. How do you build linearizable compare-and-set from it?
5. State the four properties of consensus. Which is liveness?
6. Why does 2PC fail the termination property?
7. What does FLP say, and why can we still build consensus systems?
8. How do epoch numbers and overlapping quorums solve "to elect a leader you need a leader"?
9. How many nodes do you need to tolerate 2 failures?
10. List five limitations of consensus.
