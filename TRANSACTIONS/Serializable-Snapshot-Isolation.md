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
