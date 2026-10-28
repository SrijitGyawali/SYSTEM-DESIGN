# 07 · Raft — Log Replication (How Writes Work) — Revision Notes

> 📘 **Source:** the **Raft paper** (Ongaro & Ousterhout, 2014) and Ongaro's thesis. DDIA covers the ideas behind it: total order broadcast, quorums, epochs and linearizable reads ([05](05-Distributed-Consensus.md), [03](03-Linearizability.md)).
> Folder guide: [README.md](README.md) · Previous: [06-Raft-Leader-Election.md](06-Raft-Leader-Election.md) · Next: [08-ZooKeeper.md](08-ZooKeeper.md)

## 1. The idea: a replicated state machine

```
 client ──"SET x=5"──▶ LEADER
                          │ 1. append to own log
                          │ 2. AppendEntries to followers (parallel)
                          ▼
   Leader log:   [1:t1 SET a=1][2:t1 SET b=2][3:t2 SET x=5]
   Follower 1:   [1:t1 SET a=1][2:t1 SET b=2][3:t2 SET x=5]  ✅ ack
   Follower 2:   [1:t1 SET a=1][2:t1 SET b=2][3:t2 SET x=5]  ✅ ack
   Follower 3:   [1:t1 SET a=1][2:t1 SET b=2]                (slow)
   Follower 4:   (down)
                          │ 3. majority (3 of 5) has entry 3 → COMMITTED
                          │ 4. apply to state machine → x = 5
 client ◀────── "OK" ─────┘ 5. reply to client
```

- Each **log entry** = **(index, term, command)**.
- Every server applies the **same commands in the same order**, so every **state machine is identical**. That's **total order broadcast**.

## 2. The write path step by step

1. A **client sends a command to the leader**. Followers **redirect** clients to the leader.
2. The leader **appends** the entry to **its own log**.
3. It sends **`AppendEntries`** RPCs to all followers **in parallel**.
4. Once the entry is stored on a **majority** of servers (leader included), it is **committed**.
5. The leader **applies** it to its state machine and **returns the result to the client**.
6. The leader includes its **`commitIndex`** in later `AppendEntries` (and heartbeats), so followers learn what's committed and **apply it too**.

- Slow, crashed or lossy followers: the leader **retries `AppendEntries` indefinitely** until every follower has every entry.
- The **write latency** is **one round trip to a majority**. That's the "synchronous replication" cost DDIA mentions.

### `AppendEntries` RPC

| Arguments | Meaning |
|---|---|
| `term` | Leader's term |
| `leaderId` | So followers can redirect clients |
| `prevLogIndex` | Index of the entry **just before** the new ones |
| `prevLogTerm` | Term of that entry |
| `entries[]` | New entries (**empty = heartbeat**) |
| `leaderCommit` | Leader's `commitIndex` |

| Reply | Meaning |
|---|---|
| `term` | For the leader to update itself (and step down if it's stale) |
| `success` | true if the follower had an entry matching `prevLogIndex` and `prevLogTerm` |

## 3. Keeping logs consistent: the Log Matching Property

> ⭐ **If two logs have an entry with the same index and term, then (1) the entries hold the same command, and (2) the logs are identical in all earlier entries.**

1. A leader creates **at most one entry per index in a term**, and entries **never move**.
2. **The consistency check:**
   - Every `AppendEntries` carries `prevLogIndex` and `prevLogTerm`.
   - A follower **rejects** it if it has no entry at `prevLogIndex` with term `prevLogTerm`.
   - So each accepted append proves the logs **match up to that point**, by induction.

## 4. Repairing a follower's log

After crashes and leader changes, a follower may be **missing** entries or have **extra, uncommitted** entries from old leaders.

The leader keeps a **`nextIndex[f]`** for each follower: the next entry to send. It starts just after the leader's last entry.

```
Leader (term 6):  idx: 1  2  3  4  5  6  7
                  trm: 1  1  1  4  4  5  6
Follower:         idx: 1  2  3  4  5  6
                  trm: 1  1  1  2  2  2      ← stale entries from an old term-2 leader

AppendEntries(prevLogIndex=7, prevLogTerm=6) → follower has no entry 7 → REJECT
leader: nextIndex-- → try prevLogIndex=6 (term 5) → follower has term 2 there → REJECT
leader: nextIndex-- → try prevLogIndex=5 (term 4) → REJECT
leader: nextIndex-- → try prevLogIndex=4 (term 4) → REJECT
leader: nextIndex-- → try prevLogIndex=3 (term 1) → MATCH ✅
follower DELETES its conflicting entries 4–6 and appends the leader's 4–7
```

- ⭐ **The leader never overwrites or deletes its own log** (**Leader Append-Only**). It forces followers to **match it**.
- Only **uncommitted** entries are ever thrown away. Committed entries are safe (see §5).
- *Optimization:* the follower returns the **conflicting term and its first index**, so the leader can skip a whole term per round trip.

## 5. When is an entry committed? (the subtle rule)

- An entry from the **current term** is **committed once it's on a majority**.
- ⚠️ A leader **does not commit entries from earlier terms by counting replicas**. An old entry on a majority could still be **overwritten** by a future leader that never saw it (the famous "Figure 8" scenario in the paper).
- Instead, older entries are committed **indirectly**: when an entry from the **current term** is committed, **every entry before it** is committed too (Log Matching).
- That's why a new leader immediately appends a **no-op entry** at the start of its term: committing it also commits everything earlier.

## 6. Raft's safety guarantees

| Property | Guarantee | Comes from |
|---|---|---|
| **Election Safety** | At most **one leader per term** | One vote per term + majority |
| **Leader Append-Only** | A leader **never overwrites or deletes** its log entries | Design rule |
| **Log Matching** | Same index and term → same entry **and** identical prefix | Consistency check |
| **Leader Completeness** | A committed entry is in the log of **every future leader** | Election restriction ([06 §6](06-Raft-Leader-Election.md#6-voting-rules-what-makes-it-safe)) + commit rule |
| **State Machine Safety** | No two servers apply **different commands at the same index** | All of the above |

## 7. Reads and clients

### Linearizable reads (DDIA: reads can be stale)
A leader might have been **replaced without knowing it**, which would give a **stale read**. The fixes:
- **Read through the log:** append the read as an entry and answer it when committed. Simple but slow.
- **ReadIndex:** the leader records its `commitIndex`, **confirms it's still leader** with a round of heartbeats to a majority, waits until it has applied up to that index, then answers. (DDIA's etcd "quorum read" works this way, and ZooKeeper's `sync()` is similar.)
- **Lease-based reads:** the leader assumes it's still leader for a while after heartbeats. Faster, but **relies on bounded clock drift**.
- A new leader must first commit its **no-op entry**, so it knows what's committed.

### Exactly-once commands
If a leader crashes after committing but before replying, the **client retries**, and the command could run **twice**. The fix is for clients to tag commands with **unique serial numbers**, and the state machine **ignores duplicates**.

## 8. Practical extensions

| Topic | Idea |
|---|---|
| **Log compaction / snapshots** | The log grows forever, so periodically **snapshot the state machine** and **discard the log prefix**. A far-behind follower gets an **`InstallSnapshot`** RPC |
| **Membership changes** | Add or remove servers safely with **joint consensus** (old and new configurations overlap), or by changing **one server at a time** (Ongaro's thesis). DDIA notes dynamic membership is less well understood |
| **Batching and pipelining** | Send many entries per RPC and don't wait for each round trip, which gives higher throughput |

## 9. Raft vs single-leader database replication

| | Single-leader DB (Chapter 5) | Raft |
|---|---|---|
| Leader election | Manual, or ad hoc automatic failover | **Built in**, majority vote, terms |
| Split brain | Possible (needs fencing / STONITH) | **Prevented** (one leader per term, stale terms rejected) |
| Commit | Often **async**, so failover can **lose committed writes** | Committed = **on a majority**, so **never lost** while a majority survives |
| Linearizable | Maybe | ✅ Yes (with ReadIndex or log reads) |

## 10. Conclusion

In **Raft log replication**, every write goes to the **leader**, which **appends** an entry (index, term, command) and sends **`AppendEntries`** to followers in parallel. Once the entry is on a **majority**, it's **committed**, applied to the state machine and acknowledged to the client, and followers learn it via `leaderCommit`. The **consistency check** (`prevLogIndex` and `prevLogTerm`) enforces the **Log Matching Property**. When a follower diverges, the leader **backs up `nextIndex`** until the logs match and then **overwrites the follower's uncommitted extras**, never its own log. A leader only commits **current-term** entries by counting, so older entries commit **indirectly**, which is why it starts each term with a **no-op**. Together with the election restriction, this gives **Election Safety, Log Matching, Leader Completeness and State Machine Safety**: every server applies the same commands in the same order, and **committed writes are never lost**. **Linearizable reads** need a log read, **ReadIndex** or leases. **Client serial numbers** give exactly-once, and **snapshots and membership changes** keep it practical. This is why etcd, Consul, CockroachDB and Kafka KRaft use it.

### Interview one-liners
- **Raft write:** "The client sends to the leader, which appends to its log and replicates with AppendEntries. Once a majority stores it, the entry is committed and applied, and the leader replies."
- **Consistency check:** "AppendEntries includes the previous entry's index and term. A follower rejects a mismatch, so logs that agree at one point agree on everything before it."
- **Repair:** "The leader decrements nextIndex until the logs match, then the follower drops its conflicting uncommitted entries and copies the leader's."
- **Subtle rule:** "A leader only commits entries from its own term by counting replicas. Earlier ones commit indirectly, hence the no-op at the start of each term."
- **Reads:** "To be linearizable, the leader must confirm it's still leader (ReadIndex) or put the read through the log."

### Self-check questions
1. Walk through a Raft write from client to reply.
2. What's in a log entry? What's in an `AppendEntries` RPC?
3. When is an entry committed? How do followers find out?
4. State the Log Matching Property and how the consistency check enforces it.
5. Walk through repairing a follower with stale entries from an old term.
6. Why does the leader never delete its own entries?
7. Why can't a leader commit an old-term entry just by counting replicas? What's the no-op for?
8. List Raft's five safety properties.
9. How do you make reads linearizable in Raft?
10. How do client serial numbers give exactly-once semantics?
11. What are snapshots and joint consensus for?
12. How does Raft improve on ordinary single-leader replication?
