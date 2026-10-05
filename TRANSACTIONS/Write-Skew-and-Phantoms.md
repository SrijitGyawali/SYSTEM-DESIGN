# Write Skew and Phantoms — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 7, "Write Skew and Phantoms" (pp. 246–251).
> Related: [ACID.md](ACID.md) · [Read-Committed-Isolation.md](Read-Committed-Isolation.md) · [Snapshot-Isolation.md](Snapshot-Isolation.md) · [Actual-Serial-Execution.md](Actual-Serial-Execution.md)

## 1. The doctors on-call example

Rule: the hospital **must always have at least one doctor on call**. A doctor can go off call only if a colleague stays on call.

Alice and Bob are both on call, both feel sick, and both click "go off call" at **the same moment**:

```
           Alice's transaction                 Bob's transaction
           ───────────────────                 ─────────────────
 check:    SELECT COUNT(*) on call → 2         SELECT COUNT(*) on call → 2
 decide:   2 ≥ 2, safe to leave ✅             2 ≥ 2, safe to leave ✅
 write:    UPDATE Alice on_call = false        UPDATE Bob on_call = false
           COMMIT                              COMMIT

 Result: 0 doctors on call ❌  (the rule is broken)
```

- Both ran under **snapshot isolation**, so both saw the same snapshot with 2 doctors.
- If they had run **one after another**, the second would have seen 1 doctor and been refused.

## 2. What is write skew?

> **Write skew:** two transactions **read the same data**, then each **writes to a different object** based on what it read. Each transaction is fine on its own, but together they break a rule.

- It's **not a dirty write** and **not a lost update**, because they update **different rows** (Alice's row vs Bob's row).
- It's still a **race condition**: it only happens because the transactions ran concurrently.
- Write skew is a **generalization of the lost update**. When both transactions happen to write the **same** row, you get a lost update or dirty write instead.

| Anomaly | Same row written? | Example |
|---|---|---|
| Lost update | ✅ Same row | Two counters both write 1000 − x |
| **Write skew** | ❌ Different rows | Alice and Bob each update their own row |

## 3. Why it's hard to prevent

| Usual fix | Works for write skew? | Why |
|---|---|---|
| Atomic operations (`SET x = x + 1`) | ❌ | Several objects are involved |
| Automatic lost-update detection | ❌ | Not detected by PostgreSQL / MySQL repeatable read, Oracle serializable or SQL Server snapshot isolation |
| Database constraints | ⚠️ Rarely | You'd need a **multi-row constraint** ("at least 1 doctor on call"). Most databases don't support this, though triggers or materialized views can sometimes help |
| **Serializable isolation** | ✅ **Best fix** | Prevents all race conditions |
| **Explicit locks** (`SELECT ... FOR UPDATE`) | ✅ Second-best | Locks the rows the decision depends on |

```sql
BEGIN TRANSACTION;
SELECT * FROM doctors
  WHERE on_call = true AND shift_id = 1234
  FOR UPDATE;              -- lock every on-call doctor row
UPDATE doctors SET on_call = false
  WHERE name = 'Alice' AND shift_id = 1234;
COMMIT;
```

Bob's transaction now **waits** for Alice's lock, then sees only 1 doctor on call and is refused.

## 4. More real-world examples

| Example | Check (read) | Action (write) | What goes wrong |
|---|---|---|---|
| **Meeting room booking** | No overlapping booking for room 123, noon–1pm? | `INSERT` booking | Two people book the same room at the same time |
| **Multiplayer game** | Is the board position free? | Move a figure there | Two different figures land on the same square |
| **Claiming a username** | Is "ram" taken? | `INSERT` user "ram" | Two accounts with the same name. ✅ Easy fix: a **UNIQUE constraint** |
| **Double-spending** | Is the balance enough? | `INSERT` spending item | Two purchases together push the balance negative |

## 5. The pattern behind all write skew

1. **SELECT** checks a condition (≥ 2 doctors on call, no booking exists, username is free, money is left).
2. **The application decides** whether to go ahead, based on that result.
3. **WRITE** (INSERT / UPDATE / DELETE) and COMMIT.

That write **changes the result of step 1**. If you ran the SELECT again after committing, the answer would be different. The steps can also come in a different order (write first, then check).

## 6. Phantoms

- In the doctors example, the check **returned rows** (the on-call doctors), so `FOR UPDATE` had something to lock. ✅
- In the booking, username and double-spending examples, the check looks for the **absence** of rows ("no booking exists"). The query returns **nothing**, so **there's nothing to lock**. ❌
- The other transaction then **inserts a new row** that matches the search condition.

> **Phantom:** a write in one transaction **changes the result of a search query** in another transaction, usually by inserting a row that didn't exist before.

```
Txn A: SELECT bookings for room 123, 12–1pm → 0 rows (nothing to lock)
Txn B: SELECT bookings for room 123, 12–1pm → 0 rows
Txn A: INSERT booking 12–1pm → COMMIT
Txn B: INSERT booking 12–1pm → COMMIT   ❌ double booked
```

- Snapshot isolation **prevents phantoms for read-only queries**.
- In **read-write** transactions, phantoms can still cause write skew.

## 7. Materializing conflicts (last resort)

If the problem is "**nothing to lock**", **create rows to lock**:

- Make a table of **room × time slot** rows (e.g. every 15-minute slot for the next 6 months), created in advance.
- A booking transaction first locks the matching slot rows with `SELECT ... FOR UPDATE`, then checks for overlaps and inserts.
- The slot table **stores no booking data**. It exists only as a set of **locks**.

This turns a phantom into a normal **lock conflict on rows that exist**.

**Downsides:**
- It's hard and error-prone to get right.
- Concurrency logic **leaks into your data model**.

➡️ Use it **only as a last resort**. **Serializable isolation is preferred.**

## 8. Why this leads to serializability

Weak isolation levels are a mess:
- They're **hard to understand** and **inconsistent** across databases ("repeatable read" means different things in different places).
- It's **hard to tell** whether your application code is safe at a given level.
- **Race conditions are hard to test**, because they only appear with unlucky timing.

The researchers' answer since the 1970s: **use serializable isolation**. The result is the same as running transactions **one at a time**, which prevents **all** race conditions.

Three ways to implement it:
1. **Actual serial execution** → [Actual-Serial-Execution.md](Actual-Serial-Execution.md)
2. **Two-phase locking (2PL)** → [Two-Phase-Locking.md](Two-Phase-Locking.md)
3. **Serializable snapshot isolation (SSI)** → [Serializable-Snapshot-Isolation.md](Serializable-Snapshot-Isolation.md)

## 9. Conclusion

**Write skew** happens when two transactions read the same data, make a decision, and then write to **different** rows, so together they break a rule that each one checked on its own (two doctors both going off call). It's a generalization of the lost update, and it is **not** prevented by snapshot isolation, atomic operations or automatic lost-update detection. If the check finds **existing rows**, you can lock them with `SELECT ... FOR UPDATE`. If the check looks for rows that **don't exist yet** (bookings, usernames), there's nothing to lock. That is a **phantom**: another transaction's insert changes your search result. Fixes are a **unique constraint** where possible, **materializing conflicts** as a last resort, or best of all **serializable isolation**.

### Interview one-liners
- **Write skew:** "Two transactions read the same data, then update different rows based on it, and together they violate an invariant. Snapshot isolation doesn't stop it."
- **Phantom:** "One transaction's write changes the result of another transaction's search query. You can't lock rows that don't exist yet."

### Self-check questions
1. Walk through the doctors on-call example. Why does snapshot isolation fail here?
2. How is write skew different from a lost update?
3. Why don't atomic operations or lost-update detection help with write skew?
4. Why does `SELECT ... FOR UPDATE` fix the doctors case but not the meeting room case?
5. What is a phantom?
6. What is materializing conflicts, and why is it a last resort?
7. Name the three ways to implement serializable isolation.
