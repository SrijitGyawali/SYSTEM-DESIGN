# Actual Serial Execution — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 7, "Actual Serial Execution" (pp. 252–256).
> Related: [ACID.md](ACID.md) · [Snapshot-Isolation.md](Snapshot-Isolation.md) · [Write-Skew-and-Phantoms.md](Write-Skew-and-Phantoms.md)

## 1. The idea

> **Remove concurrency entirely: run one transaction at a time, in order, on a single thread.**

- No two transactions ever overlap, so there are **no race conditions** to detect or prevent.
- The isolation is **serializable by definition**.
- It's one of the three ways to get serializability (the others are **two-phase locking** and **serializable snapshot isolation**).

```
Queue of transactions ──▶ [ single thread ] ──▶ T1 → T2 → T3 → T4 ...
                           (one at a time, no locks needed)
```

## 2. Why this only became practical around 2007

For about 30 years, multi-threading was considered essential for speed. Two things changed:

1. **RAM became cheap.** The whole **active dataset can fit in memory**, so transactions don't wait for the disk and run very fast.
2. **OLTP transactions are short.** They do only a few reads and writes. Long **analytics queries are read-only**, so they can run on a **snapshot** (snapshot isolation) **outside** the serial loop.

**Used by:** **VoltDB / H-Store**, **Redis**, **Datomic**.

**Trade-off:**
- ✅ It can be **faster** than a concurrent system, because it has **no locking or coordination overhead**.
- ❌ Throughput is **limited to one CPU core**.

## 3. Stored procedures: the key requirement

### The problem with interactive transactions

Normally the application talks to the database **one statement at a time**, over the network:

```
App ──SELECT──▶ DB
App ◀──result── DB      (network round trip)
App  (thinks...)
App ──UPDATE──▶ DB
App ◀──ok────── DB      (another round trip)
App ──COMMIT──▶ DB
```

- Most of the time is spent **waiting on the network** and on application code.
- With a single thread, the database would **sit idle** waiting, and throughput would be **dreadful**.
- (Transactions also never wait for a **human**. On the web, one transaction lives within **one HTTP request**.)

### The solution: stored procedures

The application sends the **entire transaction code to the database ahead of time** as a **stored procedure**. The database runs it **start to finish in one go**:

```
App ──"run go_off_call('Alice', 1234)"──▶ DB  [check + update + commit, all in memory]
App ◀──────────────── result ──────────── DB   (one round trip)
```

If all the data is in memory, it runs with **no network or disk waits**, so it's very fast.

➡️ **Serial execution systems don't allow interactive multi-statement transactions. You must use stored procedures.**

## 4. Pros and cons of stored procedures

### Why they had a bad reputation
| Problem | Detail |
|---|---|
| **Ugly, outdated languages** | Each vendor has its own: PL/SQL (Oracle), T-SQL (SQL Server), PL/pgSQL (PostgreSQL). Few libraries |
| **Hard to manage** | Harder to debug, version-control, deploy, test and monitor than application code |
| **Risky** | One database serves many application servers. A slow or memory-hungry procedure can hurt everyone |

### How modern systems fix this
- Use **general-purpose languages**: **VoltDB** → Java / Groovy, **Datomic** → Java / Clojure, **Redis** → **Lua**.
- With stored procedures + in-memory data, a single thread achieves **good throughput**.

### Bonus: replication with stored procedures (VoltDB)
- Instead of copying data changes, VoltDB **runs the same stored procedure on every replica**.
- Procedures must be **deterministic**: they must produce the same result on every node. For example, the current time must come from special deterministic APIs.

## 5. Scaling up with partitioning

**Problem:** one thread = one CPU core = a **write bottleneck**.

**Solution:** **partition the data** (supported in VoltDB):
- If each transaction touches **only one partition**, each partition gets its **own thread**.
- Give each CPU core its own partition, and throughput **scales linearly** with the number of cores.

```
Partition 1 ──▶ [thread on core 1]
Partition 2 ──▶ [thread on core 2]      each runs serially, all run in parallel
Partition 3 ──▶ [thread on core 3]
```

**But cross-partition transactions are slow:**
- They must run in **lock-step across every partition** they touch.
- VoltDB reports about **1,000 cross-partition writes/sec**, orders of magnitude slower than single-partition writes, and **adding machines doesn't help**.
- **Simple key-value data** partitions easily. Data with **many secondary indexes** needs lots of cross-partition coordination.

## 6. When serial execution works (summary of constraints)

| Constraint | Why |
|---|---|
| ✅ **Every transaction must be small and fast** | One slow transaction **stalls everything** |
| ✅ **The active dataset must fit in memory** | A disk read inside the single thread makes the whole system very slow |
| ✅ **Write throughput must fit on one CPU core**, or partition cleanly | The single thread is the bottleneck |
| ⚠️ **Cross-partition transactions are limited** | There's a hard ceiling on how many you can do |

**Anti-caching** (footnote): if a transaction needs data that's not in memory, **abort it**, load the data in the background while other transactions run, then **restart** it.

## 7. Conclusion

**Actual serial execution** achieves serializable isolation in the simplest way possible: **run one transaction at a time on a single thread**, so concurrency problems like write skew and phantoms can't happen. It became practical because **RAM is cheap** (the data fits in memory) and **OLTP transactions are short**. To avoid waiting on the network, transactions are submitted as **stored procedures**, now written in normal languages like Java or Lua (VoltDB, Datomic, Redis). Throughput is limited to **one CPU core**, which **partitioning** can lift if transactions stay within one partition. Cross-partition transactions are very slow. It's a great fit for **small, fast transactions on in-memory data with moderate write load**.

### Interview one-liner
> "Actual serial execution runs transactions one at a time on a single thread, so it is serializable by definition. It works because data fits in RAM and OLTP transactions are short, and it needs stored procedures to avoid network round trips. Examples are Redis and VoltDB. It's limited to one CPU core unless you partition, and cross-partition transactions are slow."

### Self-check questions
1. Why is serial execution automatically serializable?
2. What two changes made single-threaded execution practical around 2007?
3. Why are interactive transactions a problem for serial execution? How do stored procedures fix it?
4. Why did stored procedures have a bad reputation, and how do modern systems fix that?
5. Why must VoltDB's stored procedures be deterministic?
6. How does partitioning help, and why are cross-partition transactions slow?
7. List the four constraints for serial execution to work well.
8. What is anti-caching?
