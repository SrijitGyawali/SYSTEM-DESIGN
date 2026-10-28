# Distributed Systems — Folder Guide & Summary of Everything

> Sources: *Designing Data-Intensive Applications* (Martin Kleppmann), **Chapter 6 (Partitioning)** and **Chapter 9 (Consistency and Consensus)**, plus the **Raft paper** (Ongaro & Ousterhout, 2014) for the Raft files.
> 📄 **Everything in one PDF:** [Distributed-Systems-Complete-Notes.pdf](Distributed-Systems-Complete-Notes.pdf)
> Sections marked **📘** go beyond the book.

## 📚 Reading order

| # | File | What it covers | Source |
|---|---|---|---|
| 01 | [Partitioning](01-Partitioning.md) | Key-range vs hash, hot spots, secondary indexes, rebalancing, request routing | DDIA Ch. 6 |
| 02 | [Consistent Hashing](02-Consistent-Hashing.md) | Hash ring, virtual nodes, replication on the ring, mod-N problem | DDIA Ch. 6 + 📘 |
| 03 | [Linearizability](03-Linearizability.md) | Recency guarantee, vs serializability, use cases, CAP, cost | DDIA Ch. 9 |
| 04 | [Two-Phase Commit](04-Two-Phase-Commit.md) | Atomic commit, 2PC, coordinator failure, XA, in-doubt locks | DDIA Ch. 9 |
| 05 | [Distributed Consensus](05-Distributed-Consensus.md) | Ordering, total order broadcast, consensus properties, FLP, epochs and quorums | DDIA Ch. 9 |
| 06 | [Raft: Leader Election](06-Raft-Leader-Election.md) | States, terms, randomized timeouts, RequestVote, election restriction | 📘 Raft paper |
| 07 | [Raft: Log Replication (Writes)](07-Raft-Log-Replication.md) | AppendEntries, commit rule, log repair, safety, linearizable reads | 📘 Raft paper |
| 08 | [ZooKeeper](08-ZooKeeper.md) | Coordination services, znodes, fencing tokens, recipes, etcd and Consul | DDIA Ch. 9 + 📘 |

**Why this order:**
1. **Partitioning** and **consistent hashing** spread the data out.
2. Spreading data raises the question of **consistency** (linearizability).
3. Writing across partitions needs **atomic commit** (2PC).
4. 2PC's weaknesses lead to **consensus**.
5. **Raft** shows how consensus actually works.
6. **ZooKeeper** packages consensus as a service.

## 1. The big picture

```
            ┌─────────────── data too big / too much load ───────────────┐
            ▼                                                            │
   PARTITIONING (01) ── how to place keys? ──▶ CONSISTENT HASHING (02)   │
            │                                                            │
            │ many nodes + replicas → "which value is current?"          │
            ▼                                                            │
   LINEARIZABILITY (03) ── needs agreement ──┐                           │
            │                                 │                          │
   multi-partition write → TWO-PHASE COMMIT (04) ── blocks if coordinator dies
            │                                 │
            └──────────────▶ CONSENSUS (05) ◀─┘  majority + elected leader
                                  │
                     ┌────────────┴─────────────┐
                     ▼                          ▼
          RAFT ELECTION (06)          RAFT LOG REPLICATION (07)
                     └────────────┬─────────────┘
                                  ▼
                      ZOOKEEPER / etcd (08) = consensus as a service
                      (used for leader election, locks, partition maps)
```

## 2. Partitioning in one table

| Topic | Key point |
|---|---|
| Key-range partitioning | Sorted, so **range scans** are cheap, but **hot spots** appear (timestamps) |
| Hash partitioning | **Even load**, but **no range queries**. Cassandra compound keys are the compromise |
| Hot key | The application splits it (random suffix), and reads must merge |
| Local secondary index | 1-partition writes, **scatter/gather reads** |
| Global secondary index | 1-partition reads, multi-partition, **async** writes |
| Rebalancing | ❌ `mod N`. ✅ Fixed partitions, dynamic split and merge, or partitions per node |
| Routing | Any node / routing tier / smart client, with **ZooKeeper** or **gossip** holding the map |

## 3. Consistent hashing in one line

Nodes and keys sit on a **ring**, and a key goes to the **next node clockwise**. Adding or removing a node moves only **about 1/N of the keys**. **Virtual nodes** even out the load.

## 4. Linearizability and CAP in one line

Behave like **one copy with atomic operations**: after a write, **everyone** sees it. It's needed for **locks, leader election and uniqueness**. During a partition, choose **consistency or availability**, and it's **slow even without faults**.

## 5. 2PC vs consensus

| | **2PC** | **Consensus (Raft, Zab, Paxos)** |
|---|---|---|
| Goal | Atomic commit across nodes | Agree on values or an ordered log |
| Leader | Fixed **coordinator** | **Elected** per term or epoch |
| Votes | **All** must say yes | **Majority** |
| On leader failure | **Blocks** (in doubt, locks held) | New election, continues |
| Termination | ❌ | ✅ (majority alive) |

## 6. Consensus essentials

- **Properties:** uniform agreement, integrity, validity (**safety**) + termination (**liveness**, needs a majority).
- **FLP:** impossible without timeouts. **Timeouts and randomness** make it practical.
- **Equivalences:** linearizable compare-and-set ⇔ total order broadcast ⇔ consensus.
- **Epochs + overlapping quorums:** a unique leader per term, and stale leaders can't commit.
- **Costs:** synchronous voting, **2f + 1 nodes to tolerate f failures**, fixed membership, timeout sensitivity.

## 7. Raft on one page

| | Leader election | Log replication (writes) |
|---|---|---|
| Trigger | Follower hears no heartbeat within a **randomized timeout** (about 150–300 ms) | Client sends a command to the **leader** |
| Action | Term + 1 → candidate → vote for self → **RequestVote** | Append to own log → **AppendEntries** to followers |
| Success | Votes from a **majority** → leader → heartbeats | Stored on a **majority** → **committed** → apply → reply |
| Safety rule | **1 vote per term** + vote only for a candidate with an **up-to-date log** | **prevLogIndex/prevLogTerm** check. Commit only **current-term** entries by counting (no-op at start of term) |
| Guarantees | Election Safety | Log Matching, Leader Completeness, State Machine Safety |

## 8. ZooKeeper on one page

- **What:** a consensus-replicated (**Zab**) coordination store on **3 or 5 servers** for **small, slowly changing** data.
- **Primitives:**
  - **linearizable compare-and-set** (locks and leases)
  - **zxid ordering** (fencing tokens)
  - **sessions + ephemeral znodes** (failure detection)
  - **watches** (notifications)
- **Recipes:** leader election and locks (lowest **ephemeral sequential** znode wins, others watch their predecessor), partition maps, service discovery, membership, config.
- **Caveats:** reads may be stale (use `sync()`). Not a general database. Alternatives: **etcd** (Raft, Kubernetes) and **Consul**. Kafka replaced it with **KRaft**.

## 9. 🎯 Interview cheat sheet

| Question | Short answer |
|---|---|
| What is partitioning / sharding? | Splitting data across nodes so each holds a subset, for scale. Usually combined with replication |
| Range vs hash partitioning? | Range: ordered scans, hot spots. Hash: even load, no range queries |
| How do you fix a hot key? | Split it with random suffixes at the application level, and merge on read |
| Local vs global secondary index? | Local: cheap writes, scatter/gather reads. Global: cheap reads, costly async writes |
| Why not `hash mod N`? | Changing N moves almost every key |
| What is consistent hashing? | A ring of nodes and keys where a key goes to the next node clockwise. Adding a node moves only about 1/N of keys. Virtual nodes even out the load |
| What is linearizability? | The illusion of a single copy. Once written, every read sees it |
| Linearizability vs serializability? | Recency on single objects vs transaction isolation. Both together is strict serializability |
| What does CAP really say? | During a network partition, choose linearizability or availability |
| What is 2PC? | Prepare (vote) and commit phases run by a coordinator, giving atomic commit across nodes |
| Main problem with 2PC? | Coordinator crash means participants are in doubt, block and hold locks |
| What is consensus? | Agreement + integrity + validity + termination. It needs a majority |
| What is FLP? | No deterministic consensus in an async system with crashes. Timeouts fix it in practice |
| How many nodes to tolerate f failures? | **2f + 1** (3 nodes for 1, 5 nodes for 2) |
| How does Raft elect a leader? | Randomized timeout → candidate (term + 1) → RequestVote → majority wins. One vote per term, up-to-date log check |
| How does a Raft write work? | Leader appends → AppendEntries → majority stored → committed → applied → reply |
| How are Raft logs kept consistent? | prevLogIndex/prevLogTerm check, then the leader backs up and overwrites the follower's uncommitted conflicting entries |
| What is ZooKeeper? | Consensus-backed coordination for leader election, locks, config, membership and partition maps |
| What is a fencing token? | An increasing number (zxid) sent with each write, so storage can reject a stale lock holder |

## 10. Conclusion: everything in one paragraph

When data outgrows one machine we **partition** it, by **key range** (good scans, hot-spot risk) or by **hash** (even load, no scans), handle **secondary indexes** locally or globally, **rebalance** without `mod N` (fixed, dynamic or per-node partitions, or **consistent hashing** with virtual nodes), and **route** requests using a coordination service or gossip. With data spread over many replicas, **linearizability** gives the simple illusion of one copy, which locks, leader election and uniqueness need. It costs **availability during partitions (CAP)** and **latency all the time**. Writes spanning partitions need **atomic commit**. **Two-phase commit** gives it, but a **coordinator crash leaves participants in doubt** and holding locks. **Fault-tolerant consensus** fixes this: it **elects** a leader per **epoch or term** and needs only a **majority**, so it **terminates** despite failures (FLP is sidestepped with timeouts). It's equivalent to **total order broadcast** and **linearizable compare-and-set**. **Raft** makes consensus understandable. **Randomized election timeouts**, **one vote per term** and an **up-to-date log check** elect a single safe leader. **AppendEntries** with a **consistency check**, **majority commit** and a **current-term commit rule** keep every log identical and committed writes durable. Finally, **ZooKeeper** (and etcd and Consul) packages consensus as a **service**: linearizable locks, zxid **fencing tokens**, **ephemeral nodes** for failure detection and **watches**. Real systems use it for leader election, partition maps, service discovery and membership instead of implementing consensus themselves.
