# 02 · Replication Lag and Stale Reads — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 5, "Problems with Replication Lag" (pp. 161–167).
> Folder guide: [README.md](README.md) · Previous: [01-Single-Leader-Replication.md](01-Single-Leader-Replication.md) · Next: [03-Multi-Leader-Replication.md](03-Multi-Leader-Replication.md)

## 1. Where stale reads come from

**Read-scaling architecture:** most web traffic is **reads**, so add **many followers** and spread reads across them.

- This only works with **asynchronous** replication. With synchronous replication, one dead node would stop every write, and the more nodes you have, the more likely one is down.
- **Asynchronous followers can be behind the leader**, so a read from a follower can return **stale (outdated)** data.
- The same query on the leader and on a follower can return **different answers**.

### Eventual consistency
- If you stop writing and wait, the followers **eventually catch up**. This is **eventual consistency**.
- The term was coined by Douglas Terry and popularized by Werner Vogels. It became the **battle cry of NoSQL**, but async followers in **relational** databases behave the same way.
- "Eventually" is **deliberately vague**: there's **no limit** on how far behind a replica can fall.
- **Replication lag** (the delay between a write on the leader and on a follower) is usually **a fraction of a second**, but can grow to **seconds or minutes** under load or network problems. Then the inconsistencies become **real bugs**.

The book describes three problems and the guarantee that fixes each one.

## 2. Problem 1: Reading your own writes

```
User 1234:  ──"insert comment"──▶ LEADER ─ ─ ─ (replicating slowly) ─ ─ ─▶ FOLLOWER
User 1234:  ──"show my comments"──────────────────────────────────────────▶ FOLLOWER
                                          returns the OLD data, so the comment looks lost 😟
```

> **Read-after-write consistency** (also called **read-your-writes**): if a user reloads the page, they **always see the updates they submitted themselves**. It promises nothing about **other** users' updates.

### How to implement it (with leader-based replication)

| Technique | How it works | Catch |
|---|---|---|
| **Read the user's own data from the leader** | e.g. **always read your own profile from the leader** and other people's profiles from followers | You must know what the user *might* have changed without querying it |
| **Time window after an update** | For **1 minute after the user's last update**, read from the leader. Or avoid followers that are more than 1 minute behind | If most things are editable, most reads hit the leader and read scaling is lost |
| **Remember the timestamp of the last write** | The client sends its last-write timestamp. The replica serving the read must be at least that up to date, otherwise **use another replica or wait** | The timestamp can be **logical** (log sequence number) or the **system clock**, in which case clock sync becomes critical |
| **Multiple datacenters** | Any request that must go to the leader has to be **routed to the datacenter with the leader** | Extra complexity |

### Cross-device read-after-write
The same user on a **laptop and a phone** should see their own update on both:
- The "last update timestamp" must be **stored centrally**, because one device doesn't know about the other's writes.
- The two devices may be **routed to different datacenters** (home broadband vs mobile network). You may need to **route all of a user's devices to the same datacenter**.

## 3. Problem 2: Monotonic reads (time going backward)

```
User 1234 adds a comment.
User 2345 refreshes:  read #1 ──▶ follower 1 (small lag) → sees the comment ✅
User 2345 refreshes:  read #2 ──▶ follower 2 (big lag)   → comment is GONE ❌
                      "time moved backward"
```

This is likely when **each request goes to a random server**.

> **Monotonic reads:** if one user makes several reads in a row, they **never see older data after seeing newer data**. They might still see old data, but **time never goes backward**.

- It's **weaker than strong consistency** but **stronger than eventual consistency**.
- **Solution:** each user **always reads from the same replica**, for example chosen by a **hash of the user ID** instead of at random.
- If that replica fails, the user's reads must be **rerouted** to another replica.

## 4. Problem 3: Consistent prefix reads (cause before effect)

Two writes with a **causal dependency** (the answer depends on the question):

```
Mr. Poons:  "How far into the future can you see, Mrs. Cake?"   ← written first
Mrs. Cake:  "About ten seconds usually, Mr. Poons."              ← written second

An observer reading through followers, where Poons's partition lags more:
Mrs. Cake:  "About ten seconds usually, Mr. Poons."     ← the answer arrives first 🔮
Mr. Poons:  "How far into the future can you see, Mrs. Cake?"
```

> **Consistent prefix reads:** if a sequence of writes happens in a certain order, **anyone reading them sees them in that same order**.

- This is mainly a problem in **partitioned (sharded) databases**. Partitions work **independently**, so there's **no global order of writes**, and a reader can see some parts new and others old.
- If a database **applies all writes in one order**, the problem can't happen.
- **Solutions:**
  - Write **causally related data to the same partition**, though that isn't always efficient.
  - Use algorithms that **track causal dependencies** (happens-before, see [08-Detecting-Concurrent-Writes.md](08-Detecting-Concurrent-Writes.md)).

## 5. The three guarantees side by side

| Guarantee | Prevents | Simple fix |
|---|---|---|
| **Read-after-write (read-your-writes)** | Your own update seems to vanish | Read your own data from the leader, or from a replica at least as new as your last write |
| **Monotonic reads** | Seeing newer data, then older data | Pin each user to one replica (hash of user ID) |
| **Consistent prefix reads** | Seeing an effect before its cause | Put causally related writes in the same partition, or track causality |

Ordered from weakest to strongest overall:
- **Eventual consistency**
- these **session guarantees**
- **strong consistency** (linearizability, Chapter 9)

## 6. Solutions for replication lag: the bigger picture

- Ask: **"What if replication lag becomes minutes or hours?"** If that would hurt users, design for a **stronger guarantee** such as read-after-write.
- ❌ **Pretending async replication is synchronous** leads to problems later.
- Applications **can** work around lag (e.g. read some data from the leader), but that is **complex and easy to get wrong** in application code.
- It's better if the **database does the right thing**. That's why **transactions** exist: stronger guarantees make the application simpler.
- Many distributed databases **dropped transactions**, claiming they're too expensive and that **eventual consistency is inevitable** at scale. The book calls this **overly simplistic** (see Chapters 7 and 9).

### Monitoring replication lag
- In **leader-based** systems it's easy: **lag = the leader's log position − the follower's log position**. Databases expose this as a metric, so **alert** on it.
- In **leaderless** systems it's much harder (see [07-Quorums.md](07-Quorums.md#7-monitoring-staleness)).

## 7. Conclusion

Spreading reads over **asynchronous followers** scales reads but makes the system **eventually consistent**: followers lag behind, usually by milliseconds but sometimes by minutes. That lag causes three visible problems. **Your own write seems lost** (fixed by **read-after-write** consistency: read your own data from the leader or from a sufficiently fresh replica). **Time goes backward** across refreshes (fixed by **monotonic reads**: pin each user to one replica). **An answer appears before its question** across partitions (fixed by **consistent prefix reads**: keep causally related writes together or track causality). Don't pretend async replication is sync. Design for large lag, monitor it, and prefer databases or transactions that give stronger guarantees over fragile workarounds in application code.

### Interview one-liners
- **Eventual consistency:** "If writes stop, replicas eventually converge, but there's no bound on how long that takes."
- **Read-your-writes:** "A user always sees their own updates. Implement it by reading the user's own data from the leader, or tracking their last-write position."
- **Monotonic reads:** "A user never sees time go backward. Route each user to the same replica."
- **Consistent prefix:** "Causally ordered writes are seen in order. It mainly matters for sharded databases."

### Self-check questions
1. Why does read scaling only work with asynchronous replication?
2. What is eventual consistency, and why is "eventually" vague?
3. Describe the read-your-writes problem and four ways to fix it.
4. What extra issues does cross-device read-after-write bring?
5. Explain monotonic reads with an example. How do you guarantee it?
6. Explain the Mr. Poons and Mrs. Cake example. Why is it mostly a sharding problem?
7. Rank eventual consistency, monotonic reads and strong consistency by strength.
8. How do you measure replication lag in a leader-based system?
