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
