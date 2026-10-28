# 04 · Two-Phase Commit (2PC) and Distributed Transactions — Revision Notes

> Source: *Designing Data-Intensive Applications*, **Chapter 9**, "Atomic Commit and Two-Phase Commit (2PC)" and "Distributed Transactions in Practice" (pp. 354–364).
> Folder guide: [README.md](README.md) · Previous: [03-Linearizability.md](03-Linearizability.md) · Next: [05-Distributed-Consensus.md](05-Distributed-Consensus.md)

## 1. The atomic commit problem

**Atomicity** (the A in ACID) means a transaction **either commits fully or aborts fully**, with no half-finished state.

### On a single node
1. Write the transaction's data to disk (the WAL).
2. Append a **commit record**.

The moment the **disk finishes writing the commit record** is the deciding moment. **One disk controller** makes the commit atomic.

### Across several nodes (partitions, or a global secondary index on another node)
Just sending "commit" to every node **isn't safe**:
- one node finds a **constraint violation** and must abort, while others commit
- a commit request is **lost in the network**, so that node times out and aborts
- a node **crashes** before writing its commit record and rolls back

➡️ Nodes end up **inconsistent**. **A commit must be irrevocable**: once data is committed it's visible and others rely on it (read committed). So **a node may only commit once it's sure everyone will commit.**

## 2. Two-phase commit (2PC)

> ⚠️ **2PC ≠ 2PL.** Two-phase **commit** gives **atomic commit across nodes**. Two-phase **locking** gives **serializable isolation**. They're unrelated.

- A new component: the **coordinator** (transaction manager), often a **library inside the application** (e.g. in a Java EE container), or a separate service. Examples: Narayana, JOTM, BTM, MSDTC.
- The database nodes are **participants**.

```
           Coordinator                     Participant DB1        Participant DB2
               │  ── (app reads/writes on each participant as normal) ──
PHASE 1        │──────── prepare ─────────▶│                      │
(prepare/vote) │──────── prepare ──────────────────────────────▶│
               │◀──────── "yes" ───────────│                      │
               │◀──────── "yes" ────────────────────────────────│
               │ decision = COMMIT, written to coordinator's log on disk  ← COMMIT POINT
PHASE 2        │──────── commit ──────────▶│                      │
(commit/abort) │──────── commit ───────────────────────────────▶│
```

- **All participants say "yes"** → phase 2 sends **commit**.
- **Any participant says "no"** (or a prepare times out) → phase 2 sends **abort** to all.

> 💍 **Marriage analogy:** the minister asks each partner "Do you?" (prepare). Both say "I do" (yes votes). The minister pronounces them married (commit). If either says no, the ceremony is aborted.

## 3. Why it works: a system of promises

1. The app gets a **globally unique transaction ID** from the coordinator.
2. It runs a normal **single-node transaction on each participant**, tagged with that ID. Anyone can abort at this stage.
3. When ready, the coordinator sends **prepare** (with the ID) to all. If any fails or times out → **abort** all.
4. A participant that replies **"yes"** must first make sure it **can definitely commit under all circumstances**: data written to disk, constraints checked. ⭐ **It gives up the right to abort**, without committing yet.
5. The coordinator collects all votes and **decides**, writing the decision to its **log on disk**. This is the **commit point**.
6. It sends commit or abort to everyone, **retrying forever** until each one succeeds. A crashed participant commits when it recovers, because it **promised**.

⭐ **Two points of no return:**
- a participant voting **"yes"**
- the coordinator **deciding**

(On a single node, both happen in one step: writing the commit record.)

## 4. Coordinator failure: the weak spot

- The coordinator crashes **before** sending prepare → participants can **safely abort**.
- The coordinator crashes **after** participants voted "yes" → participants **can't abort or commit on their own**. They must **wait**.

> A participant that has voted "yes" but hasn't heard the outcome is **in doubt** (uncertain).

```
Coordinator decided COMMIT → sent commit to DB2 ✅ → CRASH 💥 before reaching DB1
DB1: voted yes, no decision received → IN DOUBT
   • abort on timeout?   ❌ DB2 already committed → inconsistent
   • commit on its own?  ❌ another participant might have voted no
   → must WAIT for the coordinator to recover
```

- The only way out is for the **coordinator to recover** and read its **log**. Any transaction without a commit record in the log → **abort**.
- So 2PC's commit point is really **a single-node atomic commit on the coordinator**.
- 2PC is a **blocking** atomic commit protocol.

### Three-phase commit (3PC)
- It's **non-blocking** in theory, but assumes **bounded network delay and bounded response times**.
- Real networks have unbounded delays, so 3PC **can't guarantee atomicity** there.
- Non-blocking atomic commit needs a **perfect failure detector**, which doesn't exist (a timeout can't tell a crash from a slow network).
- ➡️ **2PC is still used** despite the blocking problem.

## 5. Distributed transactions in practice

They have a **mixed reputation**: an important safety guarantee, but **operational pain** and **slowness**. For example, **MySQL distributed transactions** are reported to be **over 10× slower** than single-node ones, because of extra **fsyncs** and **network round trips**. Many cloud services don't offer them.

| Type | Meaning | Works well? |
|---|---|---|
| **Database-internal** | All participants run the **same database software** (VoltDB, MySQL Cluster NDB) | ✅ Often well, since they can use custom optimized protocols |
| **Heterogeneous** | **Different technologies** (two databases from different vendors, or a database plus a message broker) | ⚠️ Much harder |

### Exactly-once message processing (the heterogeneous win)
- Atomically commit **"message acknowledged" + "database writes"** together.
- If either fails, **both abort**, and the broker can **safely redeliver**. The message is effectively processed **exactly once**.
- ⚠️ This only works if **every** side effect supports the same atomic commit protocol. An email server without 2PC might **send the email twice**.

### XA transactions
- **X/Open XA (1991)** is the standard for **2PC across heterogeneous systems**.
- **Supported by:**
  - **databases:** PostgreSQL, MySQL, DB2, SQL Server, Oracle
  - **message brokers:** ActiveMQ, HornetQ, MSMQ, IBM MQ
- **It's a C API, not a network protocol.** In Java it's **JTA**, used through **JDBC** and **JMS** drivers.
- The coordinator is usually a **library in the application process** with its log on the **application server's local disk**.
  - If the app server dies, participants are **stuck in doubt** until it **restarts** and reads the log.
  - The database **can't contact the coordinator directly**. Everything goes through its client library.

### Holding locks while in doubt (why "in doubt" is so bad)
- Transactions hold **row locks** (exclusive locks for writes, plus shared locks for reads under 2PL) **until commit or abort**.
- If the coordinator takes **20 minutes** to restart, the locks are held for **20 minutes**. If its **log is lost**, they're held **forever**.
- Other transactions touching those rows **block**, so large parts of the app may become **unavailable**.

### Recovering from coordinator failure
- **Orphaned in-doubt transactions** do happen (a lost or corrupted log, software bugs).
- They survive **database restarts**, because correct 2PC must keep their locks.
- The only fix is **manual**: an admin checks every participant and applies the same outcome, usually **under pressure during an outage**.
- **Heuristic decisions** are an emergency escape hatch: a participant decides on its own. "Heuristic" is a **euphemism for probably breaking atomicity**. Use it only in catastrophes.

## 6. Limitations of distributed transactions

| Limitation | Detail |
|---|---|
| **The coordinator is a database** | It stores transaction outcomes. If it's **not replicated**, it's a **single point of failure**, and many coordinators **aren't highly available** by default |
| **Stateful application servers** | The coordinator's log lives on the app server, so app servers are **no longer stateless**. The log is as critical as the databases |
| **Lowest common denominator** | XA **can't detect deadlocks across systems** and **doesn't work with SSI** |
| **Amplifies failures** | **Every** participant must respond, so if **any** part is broken the transaction fails. That runs counter to fault tolerance |

> 📘 *Beyond the book:* a common alternative in microservices is the **Saga pattern**: a chain of **local transactions**, each with a **compensating action** to undo it if a later step fails. It gives **eventual consistency without locks or a blocking coordinator**. The book discusses similar alternatives in Chapters 11 and 12.

## 7. 2PC vs consensus (preview)

| | **2PC** | **Fault-tolerant consensus** (Raft, Paxos, Zab) |
|---|---|---|
| Who decides | A **fixed coordinator** (not elected) | An **elected leader** |
| Votes needed | **All** participants must say yes | A **majority** |
| Coordinator or leader dies | **Blocks** (in doubt) | **Elects a new leader** and continues |
| Satisfies "termination"? | ❌ No | ✅ Yes (if a majority is alive) |

➡️ 2PC is **a kind of consensus algorithm, but not a very good one**. See [05](05-Distributed-Consensus.md).

## 8. Conclusion

**Atomic commit** across nodes needs every participant to agree to commit or abort, because a commit **can't be revoked** once visible. **Two-phase commit** does this with a **coordinator**. In **phase 1 (prepare)**, each participant makes sure it *can* commit and votes "yes", **giving up its right to abort**. The coordinator then **writes its decision to disk (the commit point)**, and in **phase 2** it sends commit or abort, **retrying forever**. Its fatal flaw is **coordinator failure**: participants that voted "yes" are **in doubt**, holding **locks** until the coordinator recovers (or forever, if its log is lost, needing **manual or heuristic** resolution). 3PC can't fix this on real networks. In practice, **database-internal** distributed transactions can work well, while **heterogeneous XA** transactions enable **exactly-once processing** but bring **single points of failure, stateful app servers, no cross-system deadlock detection and failure amplification**. Fault-tolerant **consensus** improves on 2PC by **electing** the leader and needing only a **majority**.

### Interview one-liners
- **2PC:** "The coordinator asks every participant to prepare. If all vote yes, it logs commit and tells everyone to commit. Otherwise it aborts. A yes vote is a promise that you can commit."
- **The problem:** "If the coordinator dies after the votes, participants are in doubt and hold their locks until it recovers. 2PC is blocking."
- **XA:** "The standard C API for 2PC across heterogeneous databases and message brokers. It enables exactly-once processing but is operationally painful."
- **2PC vs consensus:** "2PC needs all participants and a fixed coordinator. Consensus needs a majority and elects a new leader on failure."

### Self-check questions
1. Why can't you just send "commit" to every node in a distributed transaction?
2. Describe both phases of 2PC with a diagram.
3. What are the two "points of no return"? Why do they guarantee atomicity?
4. What does "in doubt" mean? Why can't a participant just time out?
5. How does the coordinator recover? What is the commit point?
6. Why doesn't 3PC solve the problem in practice?
7. Database-internal vs heterogeneous distributed transactions?
8. How do distributed transactions enable exactly-once message processing?
9. What is XA? Where does its coordinator usually live, and why is that a problem?
10. Why is holding locks while in doubt so damaging? What are heuristic decisions?
11. List four limitations of distributed transactions.
12. 2PC vs 2PL vs consensus: what's the difference?
