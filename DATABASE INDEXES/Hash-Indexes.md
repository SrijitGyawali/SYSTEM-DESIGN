# Hash Indexes — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 3, "Hash Indexes".
> Next: [SSTables-and-LSM-Trees.md](SSTables-and-LSM-Trees.md) → [B-Tree.md](B-Tree.md)

## 1. The idea

- A key-value store works like a **dictionary / hash map**.
- The data is stored as an **append-only log file**.
- **Index:** an **in-memory hash map** from each key to its **byte offset** in the data file.

```
In-memory hash map            Log file on disk (append-only)
 key   → byte offset          offset 0:  123456,{"name":"London","attractions":[...]}
 123456 → 0      ───────────▶
 42     → 64     ───────────▶ offset 64: 42,{"name":"San Francisco","attractions":[...]}
```

- **Write:** append the key-value pair to the file, then update the hash map with the new offset. This works for both inserts and updates.
- **Read:** look up the offset in the hash map, **seek** there, and read the value. That's **one disk seek**, or **zero** if the data is already in the filesystem cache.

## 2. Real-world example: Bitcask (Riak's default storage engine)

- **High-performance reads and writes.**
- **Constraint:** all **keys must fit in RAM**. Values can be larger than memory because they live on disk.
- **Best for:** values that are **updated very often** but with a **manageable number of distinct keys**.
  - Example: key = cat video URL, value = play count, incremented on every play. Lots of writes per key, few keys.

## 3. Running out of disk space: segments and compaction

An append-only log grows forever, so:

1. **Split the log into segments.** Close a segment file once it reaches a certain size and write to a new one.
2. **Compaction:** throw away **duplicate keys** and keep only the **most recent value** for each key.
3. **Merge** several segments while compacting, since compaction usually makes them much smaller.

```
Segment (before):  mew:1078 purr:2103 purr:2104 mew:1079 mew:1080 mew:1081 purr:2105 ...
Compacted:         mew:1082  purr:2108          (only the latest value per key)
```

- Segments are **never modified** once written, so merged output goes to a **new file**.
- Merging runs in a **background thread**. Reads and writes keep using the old segments meanwhile.
- When it finishes, reads **switch to the new segment** and the old files are **deleted**.
- **Each segment has its own in-memory hash map.** A lookup checks the **newest segment's map first**, then the next oldest, and so on. Merging keeps the number of segments small.

## 4. Real implementation issues

| Issue | Solution |
|---|---|
| **File format** | Use a **binary format** (length in bytes, then the raw string) instead of CSV, so no escaping is needed |
| **Deleting records** | Append a special deletion record called a **tombstone**. During merging it tells the process to discard all earlier values for that key |
| **Crash recovery** | In-memory hash maps are lost on restart. Rebuilding them by reading every segment is slow, so **Bitcask stores a snapshot of each segment's hash map on disk** and loads it quickly |
| **Partially written records** | A crash in the middle of an append leaves a corrupted record. **Checksums** detect it so it can be ignored |
| **Concurrency control** | **One writer thread** (writes are strictly sequential). Segments are append-only and immutable, so **many threads can read** concurrently |

## 5. Why append-only instead of updating in place?

It looks wasteful, but it's a good design:

- **Sequential writes are much faster** than random writes, especially on spinning hard disks, and to some extent on SSDs too.
- **Simpler concurrency and crash recovery.** A crash can't leave a value half old and half new.
- **No fragmentation.** Merging old segments keeps data files compact.

## 6. Limitations of hash indexes

1. **The hash table must fit in memory.** With too many keys you're stuck. An on-disk hash map performs badly: lots of random I/O, expensive to grow, and fiddly collision handling.
2. **Range queries are inefficient.** You can't scan `kitty00000` to `kitty99999`. Each key must be looked up separately.

These two limitations lead to **SSTables and LSM-trees**, which keep keys **sorted**.

## 7. Quick recap (memorize this)

- **Hash index** = in-memory hash map from **key → byte offset** in an append-only log.
- **Bitcask**: fast reads and writes, but **all keys must fit in RAM**. Good for many updates to few keys.
- **Segments + compaction + merging** (background thread, new file, atomic switch, delete old) keep disk use under control.
- **Tombstones** for deletes, **hash-map snapshots** for fast recovery, **checksums** for partial writes, **single writer / many readers**.
- **Append-only wins**: sequential writes, simple crash recovery, no fragmentation.
- **Limits**: keys must fit in memory, **no efficient range queries**.

### Self-check questions
1. How does a lookup work with a hash index? How many disk seeks does it need?
2. What workload is Bitcask ideal for, and what is its main constraint?
3. What is compaction, and how is merging done without blocking reads and writes?
4. How do you delete a key from an append-only log?
5. How does Bitcask recover quickly after a crash? How does it handle half-written records?
6. Why is append-only better than overwriting in place?
7. What are the two big limitations of hash indexes, and which structure fixes them?

## 8. In simple words: append-only logs, hash tables and hash indexes

**Append-only log** means you never edit old data; you only add new records at the end of a file, like a diary written in pen. To update a key you write a new line and the **newest one wins**, and compaction later removes the old lines. Appending is fast (sequential writes) and safe (a crash can't half-overwrite old data). A **hash table** is an array of slots: a hash function turns a key into a slot number, collisions are handled by probing or chaining, and when the **load factor** gets too high the table grows and rehashes. Hashing scatters keys randomly, so it has **no order and no ranges**. In a **hash index** like Bitcask, the data file on disk is **not arranged by the hash**. Records are simply appended in arrival order. The **hash map lives in RAM** and stores, for each key, the **byte offset** (a position inside the disk file, not a memory address) of its latest value. The hash function, probing and load factor are used only inside that RAM map to find the key quickly. Then the database seeks to that offset and reads the value in **one disk read**. Think of it as a sticky note in your pocket ("video_1 → page 40") pointing into a notebook you only ever write at the end of.
