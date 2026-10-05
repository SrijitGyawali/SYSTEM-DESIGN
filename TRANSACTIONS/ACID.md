# ACID Properties — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 7, "Transactions".

**Transaction** = a group of operations that behaves as **one unit of work**.
Running example: transfer ₹500 from Ram to Sita.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE name = 'Ram';
UPDATE accounts SET balance = balance + 500 WHERE name = 'Sita';
COMMIT;
```

## A — Atomicity: all or nothing
- If the database crashes after debiting Ram but before crediting Sita, the **whole transaction rolls back**.
- **How:** WAL / undo log, then roll back unfinished transactions after a crash.
- **Why it matters:** a failed transaction is **safe to retry**.
- DDIA: a better name would be **"abortability"**.

## C — Consistency: the rules always hold
- Invariants stay true: total money stays the same, and `balance >= 0`.
- **How:** constraints (`NOT NULL`, `UNIQUE`, `FOREIGN KEY`, `CHECK`) **plus application logic**.
- DDIA: consistency is mostly the **application's** job. The database gives you A, I and D as tools.
- ⚠️ This is not the "C" in CAP (replicas agreeing).

## I — Isolation: concurrent transactions don't interfere
- The result should look as if transactions ran **one after another** (serializability).

| Problem | Meaning |
|---|---|
| Dirty read | Reading **uncommitted** data |
| Dirty write | Overwriting **uncommitted** data |
| Non-repeatable read | The same row read twice gives **different values** |
| Phantom read | The same query run twice returns **new rows** |
| Lost update | Two read-modify-writes, and one **overwrites** the other |
| Write skew | Each transaction is valid alone, but together they **break a rule** (e.g. two doctors both go off call) |

| Isolation level | Prevents | Default in |
|---|---|---|
| Read Uncommitted | Dirty writes | (rarely used) |
| Read Committed | + Dirty reads | PostgreSQL, Oracle |
| Repeatable Read / Snapshot Isolation | + Non-repeatable reads (uses MVCC) | MySQL InnoDB |
| Serializable | **Everything** | (slowest, safest) |

- **How:** locks (2PL), **MVCC** snapshots (readers don't block writers), **SSI** (optimistic: abort on conflict).
- Stronger isolation is **safer but slower**.
- Deep dives: [Read committed](Read-Committed-Isolation.md) · [Snapshot isolation](Snapshot-Isolation.md) · [Write skew & phantoms](Write-Skew-and-Phantoms.md) · [Actual serial execution](Actual-Serial-Execution.md) · [Two-phase locking](Two-Phase-Locking.md) · [SSI](Serializable-Snapshot-Isolation.md)

## D — Durability: committed data survives
- Once the database says "COMMIT OK", a crash or power loss won't lose the data.
- **How:** WAL + `fsync` on one machine, **replication** across machines, and backups.
- DDIA: **perfect durability doesn't exist**. It's about reducing risk.

## Summary table

| Property | Simple meaning | Protects against | Implemented with |
|---|---|---|---|
| **Atomicity** | All or nothing | Crashes partway through | WAL / undo log, rollback |
| **Consistency** | Rules always hold | Invalid data | Constraints + application logic |
| **Isolation** | No interference | Race conditions | Locks, MVCC, SSI |
| **Durability** | Survives crashes | Data loss | WAL + fsync, replication |

## ACID vs BASE

| | ACID | BASE (Basically Available, Soft state, Eventual consistency) |
|---|---|---|
| Focus | Correctness | Availability and scale |
| Examples | PostgreSQL, MySQL, Oracle | Cassandra, DynamoDB, Riak |
| Use for | Money, orders, inventory | Feeds, likes, analytics |

## Conclusion

ACID is a promise that a group of operations behaves safely as **one unit**. **Atomicity** means a failed transaction is fully undone, so it can be retried safely. **Consistency** means your data rules always hold, mostly through the application plus database constraints. **Isolation** means concurrent transactions don't see each other's half-done work. Databases offer levels from Read Committed up to Serializable, trading safety for speed. **Durability** means committed data survives crashes through the WAL and replication, though never with 100% certainty. Choose ACID databases when correctness matters (payments, orders), and BASE systems when availability and scale matter more than instant consistency.

**Remember:** **A**ll or nothing · **C**orrect rules hold · **I**ndependent transactions · **D**ata survives.

### Self-check questions
1. What happens to a half-finished transaction after a crash? What makes this possible?
2. Why does DDIA say consistency belongs to the application?
3. What's the difference between a dirty read, a non-repeatable read and a phantom read?
4. What is write skew? Which isolation level prevents it?
5. How does MVCC let readers avoid blocking writers?
6. Why is perfect durability impossible?
7. When would you choose BASE over ACID?
