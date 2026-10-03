# Snapshot Isolation — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 7, "Snapshot Isolation and Repeatable Read".
> Related: [ACID.md](ACID.md) · [Read-Committed-Isolation.md](Read-Committed-Isolation.md)

## 1. What it is

Each transaction reads from a **consistent snapshot** of the database **as it was when the transaction started**. Even if other transactions commit changes meanwhile, **you keep seeing the old snapshot**.

It fixes **read skew**, the problem that read committed leaves open:

```
Alice starts → snapshot: Acc1 = 500, Acc2 = 500
        Transfer: Acc1 = 600, Acc2 = 400, COMMIT
Alice reads Acc1 → 500, Acc2 → 500 → total 1000 ✅
```

## 2. Why it's useful

- **Backups:** copy the whole database from one consistent point in time.
- **Long analytics queries** and **integrity checks** see a stable view.
- **Key principle:** **readers never block writers, and writers never block readers.**

## 3. How it's implemented: MVCC (Multi-Version Concurrency Control)

The database keeps **several committed versions of each row**, not just two.

- Each transaction gets an increasing **transaction ID (txid)**.
- Each row version is tagged with:
  - `created_by` = the txid that inserted it
  - `deleted_by` = the txid that deleted it (empty if still alive)
- An **update** = delete the old version + create a new version.

```
Row: Acc1
 version 1: balance=500  created_by=3   deleted_by=13
 version 2: balance=600  created_by=13  deleted_by=—
```

### Visibility rules: what can a transaction see?
A row version is **visible** if:
1. The transaction that **created** it had **committed before** this transaction started, **and**
2. It is **not deleted**, **or** the deleting transaction had **not committed** when this transaction started.

Writes from **transactions in progress**, **aborted transactions**, or **later txids** are **ignored**.

**Writes still use locks:** writers block other writers on the same row, as in read committed.

**Garbage collection** removes old versions that no transaction can see any more (e.g. PostgreSQL's `VACUUM`).

### Read committed vs snapshot isolation (MVCC view)
| | Read Committed | Snapshot Isolation |
|---|---|---|
| Snapshot taken | For **each query** | Once for the **whole transaction** |
| Versions kept | Old + new | **Many** versions |

## 4. Indexes with MVCC
- **Option 1:** the index points to **all versions** of a row, and the query filters out invisible ones.
- **Option 2:** **append-only / copy-on-write B-trees** (CouchDB, LMDB). Each write creates a new tree root, and each root is a consistent snapshot. (See [B-Tree.md](../DATABASE%20INDEXES/B-Tree.md).)

## 5. Confusing names ⚠️

The SQL standard has no "snapshot isolation", so databases name it differently:

| Database | Calls it |
|---|---|
| PostgreSQL | **Repeatable Read** |
| MySQL InnoDB | **Repeatable Read** |
| Oracle | **Serializable** (but it's really snapshot isolation!) |

"Repeatable read" means different things in different databases.

## 6. What snapshot isolation does NOT fully prevent

| Anomaly | Status |
|---|---|
| Dirty reads and writes | ✅ Prevented |
| Read skew | ✅ Prevented |
| **Lost updates** | ⚠️ PostgreSQL, Oracle and SQL Server **detect and abort** them. **MySQL InnoDB does not** |
| **Write skew** | ❌ **Not prevented** (e.g. two doctors both go off call, leaving none on duty) |
| **Phantoms causing write skew** | ❌ Not prevented |

➡️ Fixes: `SELECT ... FOR UPDATE` (explicit locks), atomic updates (`SET x = x + 1`), or the **Serializable** isolation level.

## 7. Conclusion

**Snapshot isolation** gives every transaction a **frozen, consistent view** of the database from the moment it started, which fixes **read skew**. It's implemented with **MVCC**: several committed versions of each row, tagged with transaction IDs, plus visibility rules. **Readers never block writers**, which is ideal for backups and long analytics queries. Writes still take row locks. It is called "Repeatable Read" in PostgreSQL and MySQL and "Serializable" in Oracle. It still allows **write skew** and phantoms, so you need explicit locks or true serializability for those cases.

### Interview one-liner
> "Snapshot isolation means each transaction reads a consistent snapshot from its start time. It uses MVCC, keeping several row versions tagged with transaction IDs, so readers never block writers. It prevents read skew but not write skew."

### Self-check questions
1. How does snapshot isolation fix the Alice ₹900 problem?
2. What do `created_by` and `deleted_by` mean in MVCC?
3. State the visibility rules.
4. How does MVCC differ between read committed and snapshot isolation?
5. Why is "repeatable read" a confusing name?
6. Which anomalies does snapshot isolation still allow, and how do you fix them?
