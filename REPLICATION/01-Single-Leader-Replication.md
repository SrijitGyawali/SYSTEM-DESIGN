# 01 · Single-Leader Replication — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 5, "Leaders and Followers" (pp. 151–161).
> Folder guide: [README.md](README.md) · Next: [02-Replication-Lag-and-Stale-Reads.md](02-Replication-Lag-and-Stale-Reads.md)

## 1. What replication is and why we do it

> **Replication = keeping a copy of the same data on several machines connected by a network.**

| Reason | What it gives you |
|---|---|
| Keep data **close to users** | Lower **latency** |
| Keep working when parts fail | Higher **availability** |
| Spread reads over many machines | Higher **read throughput** |
| Keep working during a network cut | **Disconnected operation** |

- Copying data that never changes is easy. **All the difficulty is in handling changes.**
- There are three main approaches: **single-leader**, **multi-leader** and **leaderless**. Almost every distributed database uses one of them.
- This chapter assumes every machine holds the **whole dataset**. Splitting data across machines is partitioning (Chapter 6).

## 2. How leader-based replication works

Also called **active/passive**, **master–slave** or **primary–replica** replication.

```
                writes only
   client ─────────────────────▶  LEADER  (master / primary)
                                    │
                    replication log │ (change stream, same order)
                  ┌─────────────────┼─────────────────┐
                  ▼                 ▼                 ▼
             FOLLOWER 1        FOLLOWER 2        FOLLOWER 3   (read replicas / secondaries)

   client ◀──── reads from the leader OR any follower
```

1. One replica is the **leader**. **All writes go to the leader**, which writes them to its own storage first.
2. The others are **followers**. The leader sends each change to them as a **replication log** (change stream), and they apply the writes **in the same order**.
3. **Reads** can go to the leader or any follower. Followers are **read-only** for clients.

| Role | Other names |
|---|---|
| Leader | Master, primary |
| Follower | Read replica, slave, secondary, hot standby |

In PostgreSQL, a **hot standby** accepts reads and a **warm standby** only applies changes. The book treats them the same.

**Used by:**
- **Relational databases:** PostgreSQL (9.0+), MySQL, Oracle Data Guard, SQL Server AlwaysOn.
- **Others:** MongoDB, RethinkDB, Espresso, the **Kafka** and **RabbitMQ** message brokers, and the DRBD block device.

## 3. Synchronous vs asynchronous replication

```
time ──────────────────────────────────────────────────────────▶
client  ──update──▶ leader
                      ├──▶ follower 1 (SYNC)  ──"ok"──▶ leader ──"success"──▶ client
                      └──▶ follower 2 (ASYNC)   ...applies it later, the leader doesn't wait
```

| | **Synchronous** follower | **Asynchronous** follower |
|---|---|---|
| Does the leader wait for it? | ✅ Yes, before telling the client "success" | ❌ No |
| Up-to-date copy guaranteed? | ✅ Yes, so it's safe if the leader dies | ❌ May lag behind |
| If that follower is down or slow | ❌ **Writes are blocked** | ✅ Leader keeps accepting writes |
| Write latency | Higher | Lower |

- **All followers synchronous?** Impractical: **one dead node stops all writes**.
- **Semi-synchronous** (the practical setup): **one** follower is synchronous and the rest are asynchronous. If the synchronous one gets slow, an asynchronous one is promoted to synchronous. This keeps an up-to-date copy on **at least 2 nodes**.
- **Fully asynchronous** (very common):
  - ✅ The leader keeps going even if every follower falls behind.
  - ❌ If the leader dies, **writes already confirmed to the client but not yet replicated are lost**, so durability is weaker.
  - It's used anyway, especially with **many followers** or followers in **other regions**.
- Lag is normally **under a second**, but it can reach **minutes** when a follower is recovering, the system is near capacity, or the network has problems.

> Research: **chain replication** (used in Microsoft Azure Storage) is a synchronous variant that doesn't lose data and still performs well. Replication consistency is closely tied to **consensus** (Chapter 9).

## 4. Setting up a new follower (without downtime)

Copying the data files isn't enough, because the data keeps changing during the copy. Locking the database would hurt availability. Instead:

```
1. snapshot leader  ──▶ 2. copy snapshot ──▶ 3. follower asks for every change
   (no global lock)        to new node          since the snapshot's log position
                                                       │
                                                       ▼
                                               4. "caught up", then keeps streaming
```

1. Take a **consistent snapshot** of the leader, ideally without locking (backups need this too; for MySQL a third-party tool like `innobackupex`).
2. **Copy** the snapshot to the new follower.
3. The follower requests **every change since the snapshot**. The snapshot must be tied to an exact **position in the replication log**: the **log sequence number (LSN)** in PostgreSQL, or the **binlog coordinates** in MySQL.
4. Once it has applied that backlog, it has **caught up** and keeps streaming changes.

## 5. Handling node outages

Goal: **reboot any node** (e.g. for a kernel security patch) **without downtime**.

### Follower failure: catch-up recovery
- Each follower keeps a **log of the changes it has received**.
- After a crash or network cut, it knows the **last transaction it processed**. It asks the leader for everything after that point, applies it, and is **caught up**. This part is easy.

### Leader failure: failover (the hard part)

**Failover** means:
- **promoting a follower** to be the new leader,
- **pointing clients** at the new leader for writes,
- **making the other followers** consume changes from the new leader.

It can be **manual** (an administrator does it) or **automatic**:

1. **Detect** that the leader has failed. There's no foolproof method, so systems use a **timeout**: if the leader doesn't respond for, say, **30 seconds**, it's assumed dead.
2. **Choose a new leader**, either by **election** (a majority of the remaining replicas) or by a **controller node**. The best candidate is the **most up-to-date replica**. Getting every node to agree is a **consensus** problem (Chapter 9).
3. **Reconfigure** the system: clients write to the new leader (request routing). If the **old leader comes back**, it must **step down** to follower.

### What can go wrong during failover

| Problem | What happens |
|---|---|
| **Lost writes** | With async replication, the new leader may be missing the old leader's latest writes. Usually they're **discarded**, which breaks durability |
| **Inconsistency with other systems** | **GitHub incident:** an out-of-date MySQL follower was promoted. Its **auto-increment counter lagged**, so it **reused primary keys** that **Redis** also used. Private data was shown to the **wrong users** |
| **Split brain** | **Two nodes both think they're the leader** and both accept writes, so data is lost or corrupted. The safety catch is **fencing / STONITH** ("Shoot The Other Node In The Head"), but a bad design can **shut down both** |
| **Choosing the timeout** | **Too long** means slow recovery. **Too short** means **unnecessary failovers** during load spikes or network glitches, which make things worse |

➡️ There are no easy fixes, so **many ops teams do failover manually** even when automatic failover is supported.

## 6. How the replication log is implemented

| Method | What is sent | Used by | Main problem |
|---|---|---|---|
| **Statement-based** | The SQL statements (`INSERT`, `UPDATE`, `DELETE`) | MySQL < 5.1, VoltDB | Non-deterministic statements give **different results** on each replica |
| **WAL shipping** | The storage engine's **write-ahead log** (bytes changed in disk blocks) | PostgreSQL, Oracle | **Tied to the storage engine**, so versions must match and **upgrades need downtime** |
| **Logical (row-based) log** | Row-level changes: inserted, deleted and updated rows | MySQL binlog (row-based) | Few problems: it's **decoupled** from the storage engine |
| **Trigger-based** | Application code (triggers) copies changes to another table or system | Databus (Oracle), Bucardo (Postgres) | **More overhead and bugs**, but very **flexible** |

### Statement-based replication
The leader sends every write **statement**, and each follower **runs it again**. It breaks when:
- A statement uses a **non-deterministic function** like `NOW()` or `RAND()`, which gives a different value on each replica.
- A statement uses an **auto-increment** column or depends on existing data (`UPDATE … WHERE …`), so it must run in **exactly the same order** everywhere. That's hard with concurrent transactions.
- A statement has **side effects** (triggers, stored procedures, user-defined functions) that may differ on each replica.

The workaround is for the leader to replace `NOW()` with a fixed value, but there are too many edge cases. MySQL now **switches to row-based** replication when a statement is non-deterministic. **VoltDB** uses statement-based replication safely by requiring **deterministic** transactions.

### Write-ahead log (WAL) shipping
- Storage engines already keep an **append-only log** of every write: the **LSM-tree's log** or the **B-tree's WAL**.
- The leader **sends that same log** to followers, which rebuild **exactly the same data structures**.
- ❌ The log is **very low-level** (which bytes in which disk blocks), so leader and followers usually **can't run different database versions**.
- That matters because a **zero-downtime upgrade** works by upgrading followers first, then failing over to one of them. WAL shipping usually prevents this, so **upgrades need downtime**.

### Logical (row-based) log replication
Uses a **separate log format** for replication, independent of the storage engine:
- **Insert:** the new values of all columns.
- **Delete:** enough to identify the row, usually the **primary key** (or all old values if there's no key).
- **Update:** the row's identity plus the **new values** of the changed columns.
- **Transaction:** one record per changed row, then a **commit** record.

✅ **Backward compatible**: leader and followers can run **different versions or even different storage engines**.
✅ **Easy for outside systems to read**, which is **change data capture (CDC)**: streaming changes into a data warehouse, search index or cache.

### Trigger-based replication
Moves replication into the **application layer** when you need flexibility:
- replicate only **part of the data**
- replicate **between different kinds of database**
- apply **custom conflict resolution**

Tools like **Oracle GoldenGate** read the database log. Alternatively, a **trigger** writes each change to a separate table, and an **external process** reads it and replicates it (Databus for Oracle, Bucardo for Postgres).

## 7. Conclusion

In **single-leader replication**, all writes go to **one leader**, which streams changes to **followers** that apply them in the same order. Reads can go anywhere, but reads from followers may be stale. Replication can be **synchronous** (safe but blocks if the follower is down), **asynchronous** (fast, but confirmed writes can be lost on failover), or **semi-synchronous** (one sync follower, the usual compromise). New followers start from a **snapshot plus the log from its position** (LSN or binlog coordinates). Follower failure is easy (**catch-up**), but leader failure needs **failover**, which risks **lost writes, split brain, bad timeouts** and problems like the GitHub key-reuse incident. The log can be **statement-based** (non-determinism issues), **WAL shipping** (tied to the storage engine), **logical/row-based** (decoupled, enables CDC) or **trigger-based** (flexible but heavier). Single-leader is the most popular approach because it's simple and has **no write conflicts**.

### Interview one-liners
- **Leader-based replication:** "All writes go to one leader, which ships a replication log to followers. Followers apply changes in order and serve reads."
- **Sync vs async:** "Sync is durable but blocks if a follower is down. Async is fast but can lose confirmed writes on failover. In practice, semi-sync means one sync follower."
- **Failover risks:** "Lost async writes, split brain (two leaders), and timeouts that are too short or too long. That's why many teams fail over manually."
- **Logical log:** "Row-level change records decoupled from the storage engine. They allow rolling upgrades and change data capture."

### Self-check questions
1. Give four reasons to replicate data.
2. Walk through how writes and reads work in leader-based replication.
3. Compare synchronous, asynchronous and semi-synchronous replication.
4. Why is fully asynchronous replication still popular despite weaker durability?
5. How do you add a new follower without downtime? What are the LSN and binlog coordinates?
6. How does a follower recover from a crash?
7. List the three steps of automatic failover.
8. Explain split brain and STONITH. What happened in the GitHub incident?
9. Why is choosing the failover timeout hard?
10. Compare statement-based, WAL shipping, logical and trigger-based replication.
11. Why does WAL shipping make zero-downtime upgrades hard?
12. What is change data capture, and which log format enables it?
