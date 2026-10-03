# Read Committed Isolation — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 7, "Weak Isolation Levels".
> Related: [ACID.md](ACID.md) · [Snapshot-Isolation.md](Snapshot-Isolation.md)

## 1. What it is

The most basic useful isolation level. It makes **two guarantees**:

1. **No dirty reads:** you only **see** data that has been **committed**.
2. **No dirty writes:** you only **overwrite** data that has been **committed**.

It is the **default** in PostgreSQL, Oracle, SQL Server and many others.

## 2. No dirty reads

A **dirty read** means reading another transaction's **uncommitted** change.

```
Txn A: UPDATE x = 3   (not committed yet)
Txn B: SELECT x  → sees 2 (old committed value) ✅, NOT 3
Txn A: COMMIT
Txn B: SELECT x  → now sees 3
```

**Why it matters:**
- A transaction that updates several rows (an email row and an unread-counter row) must not be **seen half-done**.
- If Txn A **aborts**, anyone who read its value would have read data that **never existed**.

## 3. No dirty writes

A **dirty write** means overwriting another transaction's **uncommitted** value.

```
Car sale: Alice and Bob both try to buy car #1234.
Txn Alice: UPDATE listings SET buyer='Alice'    UPDATE invoices SET recipient='Alice'
Txn Bob:   UPDATE listings SET buyer='Bob'      UPDATE invoices SET recipient='Bob'
Without protection → listing says Bob, invoice says Alice ❌
```

With read committed, the second writer **waits** until the first commits or aborts.

## 4. How it's implemented

| Guarantee | Mechanism |
|---|---|
| **No dirty writes** | **Row-level locks**. A writer locks the row and holds the lock **until commit or abort**. Other writers wait |
| **No dirty reads** | **No read locks.** The database keeps **two values** for each row being written: the **old committed value** and the **new uncommitted** one. Readers get the old value until the writer commits |

Why not use read locks? One long write would **block every reader**, which would hurt response times badly.

## 5. What read committed does NOT prevent

### Read skew (non-repeatable read)
Alice has ₹500 in each of two accounts, so ₹1000 in total. A transfer of ₹100 runs between them while she reads:

```
Alice reads Account 1 → 500
        Transfer: Acc1 = 600, Acc2 = 400, COMMIT
Alice reads Account 2 → 400
Alice sees a total of 900 ❌   (₹100 seems to have vanished)
```

Every value she saw was committed, but they came from **different points in time**.

This breaks:
- **Backups** (a mix of old and new data)
- **Analytics queries** and **integrity checks**

### Also not prevented
- **Lost updates:** two read-modify-write cycles where one overwrites the other.
- **Write skew** and **phantoms**.

➡️ These need **snapshot isolation** or **serializable**.

## 6. Conclusion

**Read committed** guarantees you never **read** or **overwrite** uncommitted data. It uses **row-level write locks** held until commit, and keeps the **old committed value** to serve readers without read locks. It's fast and the default in most databases. However, a transaction can still see **different committed values at different times** (read skew), which breaks backups and analytic queries. It also doesn't stop lost updates or write skew.

### Interview one-liner
> "Read committed means no dirty reads and no dirty writes. It uses row locks for writers and returns the old committed value to readers. It still allows read skew, lost updates and write skew."

### Self-check questions
1. What are the two guarantees of read committed?
2. Why don't databases use read locks to prevent dirty reads?
3. Walk through the Alice ₹900 read-skew example.
4. Which anomalies does read committed still allow?
