# Serializable Snapshot Isolation (SSI) — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 7, "Serializable Snapshot Isolation (SSI)" (pp. 261–266).
> Related: [Snapshot-Isolation.md](Snapshot-Isolation.md) · [Write-Skew-and-Phantoms.md](Write-Skew-and-Phantoms.md) · [Two-Phase-Locking.md](Two-Phase-Locking.md) · [Actual-Serial-Execution.md](Actual-Serial-Execution.md)

## 1. Why SSI exists

Before SSI, the options looked bleak:

| Option | Problem |
|---|---|
| [Two-phase locking](Two-Phase-Locking.md) | Serializable, but **slow** |
| [Actual serial execution](Actual-Serial-Execution.md) | Serializable, but **doesn't scale** (one CPU core) |
| Weak isolation (read committed, snapshot isolation) | Fast, but allows **race conditions** (lost updates, write skew, phantoms) |

**SSI** gives **full serializability** with only a **small performance cost** compared to snapshot isolation.

- First described in **2008**, and the subject of Michael Cahill's PhD thesis.
- Used by the **serializable** level in **PostgreSQL (since 9.1)**, and by **FoundationDB** (a distributed database with a similar algorithm).
- It's still young, but it may be fast enough to become the **new default**.

## 2. Pessimistic vs optimistic concurrency control

| | **Pessimistic** (2PL, serial execution) | **Optimistic** (SSI) |
|---|---|---|
| Attitude | "Something might go wrong, so **wait**" | "It's probably fine, so **carry on**" |
| When a conflict is possible | **Block** until it's safe | **Keep running** |
| When it checks | Before each read or write (locks) | **At commit time** |
| If isolation was violated | Can't happen, because it waited | **Abort and retry** |
| Result | Only safe transactions run | Only transactions that ran serializably may **commit** |

## 3. When optimistic concurrency works well (and badly)

It's an old idea whose pros and cons have been debated for a long time.

- ❌ **High contention** (many transactions touching the same objects) leads to **many aborts**. If the system is already near its maximum throughput, the retries add even more load and make performance **worse**.
- ✅ With **spare capacity and low contention**, optimistic techniques usually **perform better** than pessimistic ones.
- **Reduce contention with commutative atomic operations.** If many transactions increment the same counter, the order doesn't matter, so the increments don't conflict (as long as the transaction doesn't also read the counter).

## 4. SSI = snapshot isolation + conflict detection

- All reads in a transaction come from a **consistent snapshot** using MVCC, exactly like [snapshot isolation](Snapshot-Isolation.md). This is the main difference from older optimistic techniques.
- On top of that, SSI adds an **algorithm that detects serialization conflicts** among writes and decides **which transactions to abort**.

## 5. The core problem: decisions based on an outdated premise

Recall the [write skew pattern](Write-Skew-and-Phantoms.md#5-the-pattern-behind-all-write-skew): **read → decide → write**.

- The read result is a **premise**, a fact that was true when the transaction started (e.g. "there are currently 2 doctors on call").
- By commit time, another transaction may have changed the data, so the **premise may no longer be true**.
- The database doesn't know how your code uses a query result. To be safe, it must **assume that any change to the premise may make the transaction's writes invalid**.

➡️ SSI must detect when a transaction **may have acted on an outdated premise** and **abort** it.

There are two cases to detect:
1. **Stale MVCC read:** another transaction's uncommitted write happened **before** the read, and the read ignored it.
2. **Write after read:** another transaction writes to the data **after** it was read.

## 6. Case 1: detecting stale MVCC reads

```
Txn 42: UPDATE Alice SET on_call = false     (not committed yet)
Txn 43: SELECT on-call doctors → sees Alice on_call = true
                                  (MVCC ignores 42's uncommitted write)
Txn 42: COMMIT ✅
Txn 43: UPDATE Bob SET on_call = false
Txn 43: COMMIT? → the write it ignored has now committed
                → its premise is false → ABORT ❌
```

- The database **tracks** whenever a transaction **ignores another transaction's writes** because of MVCC visibility rules.
- At **commit**, it checks whether any of those ignored writes **have since committed**. If so, the transaction **aborts**.

### Why wait until commit instead of aborting right away?

- If transaction 43 is **read-only**, there's no risk of write skew, so it doesn't need to abort. At read time, the database doesn't know yet whether 43 will write later.
- Transaction 42 might still **abort**, or still be **uncommitted** when 43 commits, so the read may turn out not to be stale after all.
- Avoiding unnecessary aborts keeps snapshot isolation's main strength: **long-running reads from a consistent snapshot**.

## 7. Case 2: detecting writes that affect prior reads

This uses something like [index-range locks](Two-Phase-Locking.md#8-index-range-locks-next-key-locking-the-practical-version), except these locks **don't block**. They act as **tripwires**.

```
Txn 42: SELECT on-call WHERE shift_id = 1234  → index notes "42 read shift 1234"
Txn 43: SELECT on-call WHERE shift_id = 1234  → index notes "43 read shift 1234"
Txn 42: UPDATE Alice → sees 43 read this data → tells 43 "your read may be outdated"
Txn 43: UPDATE Bob   → sees 42 read this data → tells 42 "your read may be outdated"
Txn 42: COMMIT ✅   (43's write hasn't committed yet, so it doesn't count)
Txn 43: COMMIT → 42's conflicting write has already committed → ABORT ❌
```

- Reads are recorded on the **index entry** (e.g. `shift_id = 1234`), or at **table level** if there's no index.
- The database only needs to remember this until the transaction **and all transactions running at the same time** have finished.
- When a transaction **writes**, it looks in the index for other transactions that recently **read** the affected data. Instead of blocking, it **notifies** them that their data may be out of date.
- The **first transaction to commit wins**. The later one aborts and must retry.
