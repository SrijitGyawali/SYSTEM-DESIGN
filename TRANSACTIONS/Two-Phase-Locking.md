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
