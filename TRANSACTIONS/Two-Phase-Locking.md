# Two-Phase Locking (2PL) — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 7, "Two-Phase Locking (2PL)" (pp. 257–261).
> Related: [Write-Skew-and-Phantoms.md](Write-Skew-and-Phantoms.md) · [Actual-Serial-Execution.md](Actual-Serial-Execution.md) · [Serializable-Snapshot-Isolation.md](Serializable-Snapshot-Isolation.md)

## 1. What it is

- For about **30 years**, 2PL was the **only widely used algorithm for serializability** in databases.
- Its full name is **strong strict two-phase locking (SS2PL)**.
- It is a **pessimistic concurrency control** method: if anything *might* go wrong, **wait** until it's safe.

> ⚠️ **2PL is not 2PC.** Two-phase *locking* is about isolation inside a database. Two-phase *commit* (Chapter 9) is about committing atomically across several nodes. They are completely different things.

## 2. The core rule: readers and writers block each other

Ordinary row locks (as in read committed) only make **writers wait for writers**. 2PL is much stricter:

- Many transactions may **read** the same object at once, as long as **nobody is writing** to it.
- To **write** (modify or delete), a transaction needs **exclusive access**:
  - A has **read** X and B wants to **write** X → B **waits** until A commits or aborts, so B can't change X behind A's back.
  - A has **written** X and B wants to **read** X → B **waits** until A commits or aborts. Reading an old version is not allowed.

| | Snapshot isolation | 2PL |
|---|---|---|
| Mantra | Readers never block writers, writers never block readers | **Writers block readers, readers block writers** |
| Reads old versions? | Yes (MVCC) | No, it waits instead |
| Prevents lost updates? | Some databases detect them | ✅ Yes |
| Prevents write skew and phantoms? | ❌ No | ✅ Yes, all race conditions |

## 3. Implementation: shared and exclusive locks

Used by the **serializable** level in **MySQL (InnoDB)** and **SQL Server**, and the **repeatable read** level in **DB2**.

Every object in the database has a lock, which can be held in one of two modes:

- **To read**, take the lock in **shared mode**. Many transactions can share it, but they wait if someone holds it in exclusive mode.
- **To write**, take the lock in **exclusive mode**. Wait if anyone holds it in any mode.
- **To read and then write**, **upgrade** the shared lock to exclusive. This works the same as taking an exclusive lock directly.
- **Hold every lock until the transaction ends** (commit or abort).

| Lock already held ↓ · Lock requested → | Shared (read) | Exclusive (write) |
|---|---|---|
| **None** | ✅ Granted | ✅ Granted |
| **Shared** | ✅ Granted | ⏳ Wait |
| **Exclusive** | ⏳ Wait | ⏳ Wait |

## 4. Why it's called "two-phase"

```
 locks held
   ▲
   │          ┌──────────────┐
   │       ┌──┘              │
   │    ┌──┘                 │   ← all released at once
   │ ┌──┘                    │
   └─┴───────────────────────┴──────▶ time
     Phase 1: acquire locks     Phase 2: commit/abort,
     while executing            release every lock
```

1. **Phase 1:** while the transaction runs, it **acquires** locks and never releases any.
2. **Phase 2:** at the end (commit or abort), it **releases all** of them.

## 5. Deadlocks

With so many locks, transactions can easily get stuck waiting for each other:

```
Txn A: holds lock on X, wants Y  ──▶ waits for B
Txn B: holds lock on Y, wants X  ──▶ waits for A
                ⟳  neither can move = deadlock
```

- The database **detects deadlocks automatically** and **aborts one** of the transactions so the others can continue.
- The **application must retry** the aborted transaction.
- Deadlocks can happen with lock-based read committed too, but they happen **much more often under 2PL**. Each retry redoes all the work, so frequent deadlocks waste a lot of effort.

## 6. Performance: the big downside

This is why not everyone has used 2PL since the 1970s: **throughput and query response times are much worse** than with weak isolation.

| Cause | Detail |
|---|---|
| Lock overhead | Acquiring and releasing all those locks |
| **Reduced concurrency** (the main cause) | Anything that *might* cause a race makes one transaction **wait** for the other |
| **No limit on waiting** | Traditional databases allow long, interactive transactions, so a wait can last a long time. **Queues** form behind a popular object |
| **Unstable latency** | Very slow at **high percentiles** (e.g. p99) when there's contention. One slow transaction, or one that locks a lot of data, can make the whole system **grind to a halt** |
| **Deadlock retries** | Aborted transactions must redo all their work |

## 7. Predicate locks: stopping phantoms

Row locks can't lock rows that **don't exist yet** (see [phantoms](Write-Skew-and-Phantoms.md#6-phantoms)). Serializable isolation must prevent phantoms anyway.

A **predicate lock** belongs to **every object matching a search condition**, not to one row:

```sql
SELECT * FROM bookings
  WHERE room_id = 123
    AND end_time   > '2018-01-01 12:00'
    AND start_time < '2018-01-01 13:00';
```

- **Reading with a condition:** take a **shared predicate lock** on that condition. Wait if another transaction holds an exclusive lock on any matching object.
- **Inserting, updating or deleting:** first check whether the **old or new value matches any existing predicate lock**. If another transaction holds one, **wait** until it commits or aborts.

**Key idea:** a predicate lock also covers objects that **might be added in the future**. 2PL + predicate locks = **true serializability**, with no write skew and no phantoms.

Bookings for other rooms, or for room 123 at a different time, can still go ahead concurrently.

## 8. Index-range locks (next-key locking): the practical version

Predicate locks are **slow**: checking every write against many active predicates takes time. So most 2PL databases use **index-range locking**, a simpler **approximation**.

**The safe trick:** lock a **bigger** set of objects than needed. Any write that matches the original predicate also matches the bigger one.

| Exact predicate | Approximation | Where the lock goes |
|---|---|---|
| Room 123, noon–1pm | Room 123, **any time** | The `room_id = 123` entry in the room index |
| Room 123, noon–1pm | **All rooms**, noon–1pm | A range of values in the time index |

- The shared lock is attached to the **index the query used**.
- Another transaction that wants to insert, update or delete a booking for that room or time must **update the same part of the index**. It hits the shared lock and **waits**.
- **No suitable index?** Fall back to a **shared lock on the whole table**. This is safe but slow, because it blocks every writer to that table.

| | Predicate lock | Index-range lock |
|---|---|---|
| Precision | Exact | Locks more than needed |
| Overhead | High | **Low** |
| Used in practice | Rarely | **Most 2PL databases** |

## 9. 2PL is pessimistic concurrency control

> **Pessimistic:** "If anything *might* go wrong (another transaction holds a lock), **wait** until it's safe before doing anything."

- It works like a **mutex** (mutual exclusion) in multi-threaded programming.
- [Actual serial execution](Actual-Serial-Execution.md) is pessimistic **to the extreme**. It's like each transaction holding an exclusive lock on the **whole database** (or partition). It makes up for this by keeping every transaction very fast.
- The **optimistic** alternative is **SSI** → [Serializable-Snapshot-Isolation.md](Serializable-Snapshot-Isolation.md).

## 10. Conclusion

**Two-phase locking** makes transactions serializable by locking. Reads take **shared** locks and writes take **exclusive** locks, and every lock is held **until commit or abort** (phase 1 acquires, phase 2 releases). Unlike snapshot isolation, **writers block readers and readers block writers**, so lost updates, write skew and phantoms are all prevented. Phantoms need **predicate locks**, which in practice are approximated by cheaper **index-range (next-key) locks**, with a whole-table lock as the fallback. The cost is **performance**: less concurrency, queues behind hot rows, unstable p99 latency, and frequent **deadlocks** that force retries. It's the classic **pessimistic** approach, used for serializable isolation in MySQL InnoDB and SQL Server.

### Interview one-liner
> "2PL takes shared locks for reads and exclusive locks for writes and holds them until commit, so readers and writers block each other. Predicate or index-range locks stop phantoms. It's fully serializable but slow, with deadlocks and unpredictable latency."

### Self-check questions
1. How is 2PL different from the row locks used in read committed?
2. Explain shared vs exclusive locks. When does each one block?
3. Why is it called "two-phase"?
4. What is a deadlock, and how does the database handle it?
5. Why is 2PL's performance poor and its latency unstable?
6. What is a predicate lock, and how does it stop phantoms?
7. Why do databases use index-range locks instead? What's the fallback when there's no index?
8. Why is 2PL called pessimistic? Why is serial execution "pessimistic to the extreme"?
9. What's the difference between 2PL and 2PC?
