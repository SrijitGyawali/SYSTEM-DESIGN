# 06 · Raft — Leader Election — Revision Notes

> 📘 **Source:** DDIA (Chapter 9) only **names** Raft as a consensus algorithm (used by etcd), with its **term numbers** and an edge case with unreliable links. These notes follow the **Raft paper**: *"In Search of an Understandable Consensus Algorithm"*, Diego Ongaro & John Ousterhout, USENIX ATC 2014, and Ongaro's PhD thesis.
> Folder guide: [README.md](README.md) · Previous: [05-Distributed-Consensus.md](05-Distributed-Consensus.md) · Next: [07-Raft-Log-Replication.md](07-Raft-Log-Replication.md)

## 1. What Raft is

- A **consensus algorithm** for managing a **replicated log**, which is **total order broadcast** (see [05](05-Distributed-Consensus.md)).
- It was designed to be **understandable**, because Paxos is famously hard. It splits the problem into three parts:
  1. **Leader election**: this file.
  2. **Log replication** (writes): [07](07-Raft-Log-Replication.md).
  3. **Safety**: covered in both.
- **Strong leader:** log entries flow **only from leader to followers**.
- A cluster is usually **5 servers**, which **tolerates 2 failures** (it needs a majority of 3).

**Used by:** etcd (Kubernetes), Consul, CockroachDB, TiKV, and Kafka KRaft mode.

## 2. Server states

Every server is in **exactly one** state:

| State | Role |
|---|---|
| **Follower** | **Passive**: only responds to the leader and candidates. Everyone **starts here** |
| **Candidate** | Trying to **become leader** in an election |
| **Leader** | Handles **all client requests** and **replicates the log**. At most **one per term** |

```
                 times out,                      receives votes from
   starts up      starts election                majority of servers
  ──────────▶ FOLLOWER ─────────────▶ CANDIDATE ─────────────────────▶ LEADER
                ▲   ▲                  │    │ ⟲ times out, new election     │
                │   └──────────────────┘    │   (term + 1)                  │
                │  discovers current leader │                               │
                │  or a higher term         │                               │
                └───────────────────────────┴───────────────────────────────┘
                              discovers a server with a higher term
```

## 3. Terms: Raft's epoch numbers

- Time is divided into **terms** numbered **1, 2, 3, …**. They're the **epoch numbers** from DDIA.
- Each term **starts with an election**. If a candidate wins, it **leads for the rest of the term**.
- A term can end with **no leader** (a **split vote**), and then a new term starts right away.
- ⭐ **At most one leader per term.**

```
term:   │   1   │   2   │ 3 │   4   ...
        │elect│ normal  │elect│normal│elect(split vote, no leader)│elect│ normal ...
```

**Rules every server follows:**
- It stores `currentTerm` and includes it in **every message**.
- It sees a **higher term** → updates `currentTerm` and **immediately becomes a follower**. A stale leader steps down.
- It gets a request with a **stale (lower) term** → **rejects** it.

➡️ Terms work as a **logical clock** that detects **stale leaders**.

## 4. Heartbeats and election timeouts

- The leader sends periodic **heartbeats**: **empty `AppendEntries` RPCs**.
- If a follower hears **nothing** for its **election timeout**, it assumes the leader is dead and **starts an election**.
- ⭐ **Election timeouts are randomized** (the paper suggests **150–300 ms**). Usually **one server times out first**, wins, and sends heartbeats before anyone else times out. This **avoids repeated split votes**.

## 5. How an election works

When a follower's timeout fires:
1. **Increment `currentTerm`.**
2. Become a **candidate**.
3. **Vote for itself.**
4. Send **`RequestVote` RPCs** to every other server **in parallel**.

### `RequestVote` RPC
| Arguments | Meaning |
|---|---|
| `term` | Candidate's term |
| `candidateId` | Who's asking |
| `lastLogIndex` | Index of the candidate's last log entry |
| `lastLogTerm` | Term of the candidate's last log entry |

| Reply | Meaning |
|---|---|
| `term` | Voter's current term (so the candidate can update itself) |
| `voteGranted` | true or false |

### Three possible outcomes
| Outcome | What happens |
|---|---|
| **1. It wins** (votes from a **majority**) | Becomes **leader** and immediately sends **heartbeats** to establish authority and stop new elections |
| **2. Another server wins** | It receives an `AppendEntries` from a leader whose **term ≥ its own** and **steps back to follower** |
| **3. No one wins** (split vote) | It **times out again** (randomized), **increments the term**, and starts a **new election** |

```
5 servers, S1 times out first (term 4 → 5)
S1: votes for itself ─ RequestVote(term 5) ─▶ S2 ✅  S3 ✅  S4 ❌(down)  S5 ✅
S1 has 4/5 votes ≥ majority (3) → LEADER for term 5 → sends heartbeats
```

## 6. Voting rules (what makes it safe)

1. ⭐ **One vote per term:** each server votes for **at most one candidate per term**, **first come, first served**. Its vote (`votedFor`) is **saved to disk** so a restart can't make it vote twice. Combined with "majority needed", this means **two leaders can never win the same term** (**Election Safety**).

2. ⭐ **Election restriction (up-to-date log):** a server **refuses** to vote for a candidate whose log is **less up to date** than its own:
   - Compare the **term of the last entry**: the **higher** term is more up to date.
   - If the last terms are **equal**, the **longer** log is more up to date.

   Why: a candidate needs a majority, and every **committed** entry is on a majority. The two majorities **overlap**, so the winner's log must contain **every committed entry**. This is **Leader Completeness**: a new leader never lacks committed data.

```
Voter's log:      [t1][t1][t2][t3]            last term = 3
Candidate A log:  [t1][t1][t2]                last term = 2 → REJECT (less up to date)
Candidate B log:  [t1][t1][t2][t3][t3]        last term = 3, longer → GRANT
```

## 7. What each server saves to disk

| Persistent (on disk, before replying) | Volatile |
|---|---|
| `currentTerm`: latest term seen | `commitIndex`: highest entry known to be committed |
| `votedFor`: candidate voted for this term | `lastApplied`: highest entry applied to the state machine |
| `log[]`: the entries (command + term) | Leader only: `nextIndex[]`, `matchIndex[]` (see [07](07-Raft-Log-Replication.md)) |

## 8. Timing requirement

> **broadcastTime ≪ electionTimeout ≪ MTBF**

| Quantity | Typical value |
|---|---|
| **broadcastTime** (time to send RPCs to every server and get replies) | 0.5–20 ms |
| **electionTimeout** | 10–500 ms |
| **MTBF** (mean time between failures of one server) | Months |

If elections take a big share of the time, the system spends its time electing instead of working. DDIA warns about exactly this with **variable network delays**.

## 9. Known problems and fixes 📘

| Problem | Fix (Ongaro's thesis and real implementations) |
|---|---|
| **A partitioned node keeps bumping its term.** It can't win, but when it rejoins, its **high term forces the healthy leader to step down** | **Pre-vote:** before raising its term, a candidate checks whether it *could* win. If not, it doesn't disrupt anyone |
| **Leadership bouncing on one flaky link** (the DDIA edge case) | **Pre-vote + CheckQuorum:** a leader steps down if it can't reach a majority, and followers ignore vote requests while they still hear a live leader |
| **Removed servers disrupting elections** | Ignore `RequestVote` while within the minimum election timeout of hearing from a leader |

## 10. Conclusion

**Raft** elects a **single strong leader per term**. Servers are **followers, candidates or leaders**. **Terms** are increasing epoch numbers carried on every message, and a server that sees a higher term **steps down**, which neutralizes stale leaders. Leaders send **heartbeats**. A follower that hears nothing for its **randomized election timeout** (about 150–300 ms) **increments its term, votes for itself and sends `RequestVote`**. A **majority** makes it leader. Hearing from a legitimate leader makes it a follower again, and a **split vote** triggers a new term. Safety comes from **one persisted vote per term** (so at most one leader per term) and the **up-to-date log restriction** (compare the last term, then length), which guarantees **the new leader has every committed entry**. Elections must be rare relative to work (**broadcastTime ≪ electionTimeout ≪ MTBF**), and **pre-vote and CheckQuorum** fix disruptive or flaky-link cases.

### Interview one-liners
- **Election:** "A follower times out after a randomized interval, increments its term, votes for itself and requests votes. A majority makes it leader for that term."
- **Why randomized timeouts:** "So usually only one server times out first and wins, which avoids repeated split votes."
- **Safety:** "One durable vote per term means at most one leader per term. Voters only accept candidates whose log is at least as up to date, so the leader has every committed entry."
- **Terms:** "A logical clock. A higher term always wins, so stale leaders step down immediately."

### Self-check questions
1. What are the three server states and their transitions?
2. What is a term, and how does it relate to DDIA's epoch numbers?
3. What happens when a server sees a higher term? A lower one?
4. Why are election timeouts randomized?
5. Walk through an election: what does a candidate do, and what are the three outcomes?
6. What arguments does `RequestVote` carry, and why `lastLogIndex` and `lastLogTerm`?
7. Explain the up-to-date log check with an example.
8. Why must `votedFor` and `currentTerm` be saved to disk?
9. State the timing requirement and why it matters.
10. What problem does pre-vote solve?
