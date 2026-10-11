# 03 · Multi-Leader Replication — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 5, "Multi-Leader Replication" (pp. 168–177).
> Folder guide: [README.md](README.md) · Previous: [02-Replication-Lag-and-Stale-Reads.md](02-Replication-Lag-and-Stale-Reads.md) · Next: [04-Handling-Write-Conflicts.md](04-Handling-Write-Conflicts.md)

## 1. What it is

**The problem with a single leader:** **every write must go through one node**. If you can't reach the leader, for example because of a network interruption, **you can't write**.

> **Multi-leader replication** (also called **master–master** or **active/active**): **several nodes accept writes**. Each leader forwards its changes to every other node, so **each leader is also a follower of the other leaders**.

(If the database is partitioned, each **partition** has its own single leader. That's still single-leader replication.)

## 2. When to use it (use cases)

> Inside **one datacenter**, multi-leader is **rarely worth** the extra complexity.

### Use case 1: Multi-datacenter operation

```
        Datacenter 1 (e.g. Europe)                Datacenter 2 (e.g. US)
   ┌────────────────────────────────┐        ┌────────────────────────────────┐
   │ clients ─▶ LEADER 1            │ ◀────▶ │           LEADER 2 ◀─ clients  │
   │              │   (followers)   │ async  │  (followers)  │                │
   │              ▼                 │ +      │               ▼                │
   │        follower, follower      │conflict│       follower, follower       │
   └────────────────────────────────┘ resol. └────────────────────────────────┘
```

Inside each datacenter it's normal leader–follower replication. **Between datacenters**, the leaders replicate to each other **asynchronously**.

| | Single-leader across datacenters | Multi-leader (one leader per datacenter) |
|---|---|---|
| **Performance** | Every write crosses the internet to the leader's datacenter, adding latency | Writes are handled in the **local** datacenter, and the cross-datacenter delay is **hidden from users** |
| **Datacenter outage** | Failover promotes a follower in another datacenter | Each datacenter **keeps running on its own**, and replication **catches up** when the failed one returns |
| **Network problems** | **Very sensitive**: writes go synchronously over the unreliable inter-datacenter link | **More tolerant**: a temporary link problem doesn't stop writes |

- **Tools:** Tungsten Replicator (MySQL), BDR (PostgreSQL), GoldenGate (Oracle). Some databases support it natively.
- ❌ **The big downside:** the same data can be changed **concurrently in two datacenters**, causing **write conflicts** (see [04](04-Handling-Write-Conflicts.md)).
- ❌ Multi-leader support is often **bolted on**, so **auto-increment keys, triggers and integrity constraints** can break. It's considered **"dangerous territory"** to avoid if possible.

### Use case 2: Clients with offline operation
- Example: a **calendar app** on your phone, laptop and tablet. You must be able to **read and add meetings while offline**.
- **Each device has a local database that acts as a leader**, and devices **sync** asynchronously when they're online.
- **Replication lag can be hours or days.**
- It's multi-datacenter **taken to the extreme**: each device is a "datacenter" with a very unreliable network.
- Hard to get right (there's a long history of broken calendar sync). **CouchDB** is designed for this.

### Use case 3: Collaborative editing
- **Etherpad** and **Google Docs**: several people edit the same document at once.
- Each user's edits apply **instantly to their local copy**, then replicate asynchronously to the server and other users.
- **Locking the document** while one person edits = **single-leader** with transactions (no conflicts, but slow collaboration).
- **Tiny units of change (one keystroke) and no locking** = **multi-leader**: fast collaboration, but it **needs conflict resolution**.

## 3. Replication topologies

A **topology** is the set of paths along which writes travel between leaders. With two leaders there's only one option (each sends to the other). With more, there are choices:

```
 (a) CIRCULAR                (b) STAR / TREE               (c) ALL-TO-ALL
                                     root
   L1 ──▶ L2                          L1                     L1 ◀──▶ L2
   ▲       │                        ↙  ↓  ↘                   ▲ ╲   ╱ ▲
   │       ▼                      L2   L3   L4                │   ╳   │
   L4 ◀── L3                                                  ▼ ╱   ╲ ▼
                                                             L4 ◀──▶ L3
```

| Topology | How it works | Pros | Cons |
|---|---|---|---|
| **Circular** (MySQL's default) | Each node receives from one node and forwards (plus its own writes) to the next | Simple | **One failed node breaks the ring**. Writes pass through many nodes |
| **Star / tree** | A root node forwards writes to all the others | Simple | **The root is a single point of failure** |
| **All-to-all** | Every leader sends to every other leader | **Best fault tolerance**: messages can take other paths | Messages can **arrive out of order** |

### Avoiding infinite loops (circular and star)
- Writes must be **forwarded** through several nodes.
- Each node has a **unique ID**, and each write is **tagged with the IDs of every node it has passed through**.
- A node **ignores** a change that already carries its own ID.
- When a node fails, the topology usually has to be **reconfigured by hand**.

### Problem with all-to-all: messages overtaking each other

```
Client A: INSERT row x  ──▶ Leader 1 ──(slow link)─────────────────▶ Leader 2  (arrives 2nd)
Client B: UPDATE row x  ──▶ Leader 3 ──(fast link)──▶ Leader 2  (arrives 1st)
Leader 2 sees "update a row that doesn't exist" ❌
```

- This is a **causality** problem, like [consistent prefix reads](02-Replication-Lag-and-Stale-Reads.md#4-problem-3-consistent-prefix-reads-cause-before-effect): the update **depends on** the insert.
- **Timestamps aren't enough**, because clocks can't be trusted to be in sync (Chapter 8).
- **Version vectors** are the right tool (see [08](08-Detecting-Concurrent-Writes.md#6-version-vectors-multiple-replicas)).
- ⚠️ Many systems handle this badly. At the time of writing, **PostgreSQL BDR** had **no causal ordering**, and **Tungsten Replicator** **didn't even detect conflicts**. **Read the docs and test thoroughly.**

## 4. Single-leader vs multi-leader

| | Single-leader | Multi-leader |
|---|---|---|
| Who accepts writes | One leader | Several leaders |
| Write conflicts | ❌ None, because writes are ordered by the leader | ✅ **Yes, and they must be resolved** |
| Write latency across regions | High (go to the leader's region) | Low (local leader) |
| Datacenter outage | Needs failover | Other datacenters keep working |
| Works offline | No | ✅ Yes (each device is a leader) |
| Complexity | Low | **High** (conflicts, topology, causality) |
| Examples | PostgreSQL, MySQL, MongoDB | CouchDB, BDR, Tungsten, GoldenGate |

## 5. Conclusion

**Multi-leader replication** lets **several nodes accept writes**, and each leader also replicates from the others. It's rarely worth it inside one datacenter, but it shines in three cases. **Multi-datacenter** setups get local writes, survive datacenter outages and tolerate network problems. **Offline clients** like calendar apps treat each device as a leader that syncs later. **Collaborative editing** like Google Docs treats every keystroke as a small change with no locks. The price is **write conflicts** when the same data changes in two places, plus fragile interactions with auto-increment keys, triggers and constraints. Leaders are connected in a **topology**: circular and star are simple but fragile (they need node-ID tags to avoid loops), while all-to-all is robust but lets **writes arrive out of causal order**, which calls for version vectors rather than timestamps. Avoid it unless you need it, and test the conflict handling carefully.

### Interview one-liners
- **Multi-leader:** "Several nodes accept writes and replicate to each other asynchronously. It's great for multiple regions and offline clients, but you must handle write conflicts."
- **Topologies:** "Circular and star are simple but have single points of failure. All-to-all is fault-tolerant but writes can arrive out of order, so you need causal ordering."
- **When to avoid it:** "Inside a single datacenter, where single-leader is simpler and conflict-free."

### Self-check questions
1. What is the main limitation of single-leader replication that multi-leader solves?
2. Compare single-leader and multi-leader across datacenters on performance, outages and network problems.
3. Why is a calendar app an example of multi-leader replication?
4. How does collaborative editing relate to multi-leader replication? When is it single-leader?
5. Draw the circular, star and all-to-all topologies and give the pros and cons of each.
6. How do circular topologies avoid infinite replication loops?
7. How can writes arrive out of order in all-to-all, and why don't timestamps fix it?
8. Why is multi-leader replication called "dangerous territory"?
