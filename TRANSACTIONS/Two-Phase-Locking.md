# Two-Phase Locking (2PL) — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 7, "Two-Phase Locking (2PL)" (pp. 257–261).
> Related: [Write-Skew-and-Phantoms.md](Write-Skew-and-Phantoms.md) · [Actual-Serial-Execution.md](Actual-Serial-Execution.md) · [Serializable-Snapshot-Isolation.md](Serializable-Snapshot-Isolation.md)

## 1. What it is

- For about **30 years**, 2PL was the **only widely used algorithm for serializability** in databases.
- Its full name is **strong strict two-phase locking (SS2PL)**.
- It is a **pessimistic concurrency control** method: if anything *might* go wrong, **wait** until it's safe.

> ⚠️ **2PL is not 2PC.** Two-phase *locking* is about isolation inside a database. Two-phase *commit* (Chapter 9) is about committing atomically across several nodes. They are completely different things.
