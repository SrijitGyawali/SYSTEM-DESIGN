# SSTables & LSM-Trees — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 3, "SSTables and LSM-Trees".
> Related: [B-Tree.md](B-Tree.md)

## 1. From a plain log to an SSTable

- A basic log segment stores key-value pairs **in the order they were written**. A later value for a key wins over an earlier one.
- **One change:** require each segment to be **sorted by key**.
- That format is a **Sorted String Table (SSTable)**.
- Each key appears **only once** per merged segment (compaction already ensures this).

## 2. Three big advantages of SSTables (over hash-indexed logs)

### ① Merging is simple and efficient (like mergesort)
- Read all input segments **side by side**, look at the first key of each, copy the **lowest** key to the output, and repeat.
- This works even when the files are **bigger than memory**, because it only reads sequentially.
- **Same key in several segments?** Keep the value from the **most recent** segment and drop the older ones.

```
Segment 1 (older): handbag:8786  handful:40308  handicap:65995  handkerchief:16324
Segment 2:         handcuffs:2729  handful:42307  handlebars:...
Segment 3 (newest): handful:44662  handprinted:33632 ...
                         │ merge (keep newest value per key)
                         ▼
Merged: handbag:8786  handcuffs:2729  handful:44662  handicap:65995 ...
```

### ② A sparse in-memory index is enough
- You don't need **every** key in memory, only **one key every few KB**.
- Looking for `handiwork`? You know the offsets of `handbag` and `handsome`. Because the file is sorted, jump to `handbag` and **scan forward** until you find it (or pass where it would be).

```
Sparse index (in memory)        SSTable on disk (sorted)
handbag   → 102134   ─────────▶ handbag, handcuffs, handful, handicap, handiwork, ...
handsome  → 106195   ─────────▶ handsome, handstand, ...
hangout   → 110237
```

- Why not just binary search the file? Keys and values are **variable-length**, so you can't tell where one record ends without an index.

### ③ Compression
- Reads scan a range anyway, so **group records into blocks and compress** them.
- Each sparse-index entry points to the **start of a compressed block**.
- This saves **disk space** and **I/O bandwidth**.

## 3. How to build and maintain SSTables

Writes arrive in **random order**, so how does the data get sorted?
Keeping a sorted structure **in memory** is easy: use a balanced tree such as a **red-black or AVL tree**.

### The LSM storage engine workflow

```
 WRITE ──▶ [WAL on disk] (append-only, unsorted, for crash recovery)
   │
   ▼
 MEMTABLE (in-memory balanced tree, sorted)
   │  grows past a threshold (a few MB)
   ▼
 FLUSH as a new SSTable (newest segment)   ← writes continue to a fresh memtable
   │
   ▼
 SSTable_n, SSTable_n-1, ..., SSTable_1 (on disk)
   ▲
   └── background MERGE + COMPACTION (drops overwritten/deleted values)

 READ: memtable → newest SSTable → next older → ... → oldest
```

1. **Write**: insert into the **memtable** (an in-memory balanced tree).
2. **Flush**: when the memtable is larger than **a few MB**, write it to disk as an **SSTable**. It's already sorted, so this is fast. New writes go to a fresh memtable meanwhile.
3. **Read**: check the **memtable**, then the **newest segment**, then older ones in turn.
4. **Compaction**: periodically **merge** segments in the background and drop overwritten or deleted values.

### Crash problem and fix
- **Problem:** if the database crashes, writes still in the memtable (not yet flushed) are **lost**.
- **Fix:** also **append every write to a log on disk** (a WAL).
  - It's **unsorted**, which is fine because it is only used to **rebuild the memtable** after a crash.
  - Once the memtable is flushed to an SSTable, its log can be **discarded**.

## 4. LSM-Tree: name and real-world use

- **LSM-Tree = Log-Structured Merge-Tree** (Patrick O'Neil et al.), built on earlier work on log-structured filesystems.
- **LSM storage engines** are engines that merge and compact sorted files.
- **Memtable** and **SSTable** were named in **Google's Bigtable** paper.

| System | Notes |
|---|---|
| **LevelDB, RocksDB** | Embeddable key-value libraries. LevelDB can be used in Riak instead of Bitcask |
| **Cassandra, HBase** | Inspired by Bigtable |
| **Lucene** (Elasticsearch, Solr) | Full-text search: **term → postings list** (IDs of documents containing the word), stored in SSTable-like sorted files that are merged in the background |

## 5. Performance optimizations

### Bloom filters
- **Problem:** looking up a key that **doesn't exist** is slow. You check the memtable and then **every segment back to the oldest**, possibly reading disk each time.
- **Fix:** a **Bloom filter**, a memory-efficient structure that approximates a set. It can say "**this key is definitely not here**", which skips unnecessary disk reads.

### Compaction strategies

| Strategy | How it works | Used by |
|---|---|---|
| **Size-tiered** | Newer, smaller SSTables are merged into older, larger ones | HBase, Cassandra |
| **Leveled** | The key range is split into smaller SSTables, and older data moves into separate **levels**. Compaction is more incremental and uses **less disk space** | LevelDB (hence the name), RocksDB, Cassandra |

## 6. Conclusion: why LSM-trees work

- **Core idea:** keep a **cascade of SSTables that are merged in the background**. It's simple and effective.
- Works well even when the **dataset is much bigger than memory**.
- **Sorted data** → efficient **range queries** (all keys between a min and max).
- **Sequential disk writes** → **very high write throughput**.

## 7. LSM-Tree vs B-Tree (interview favorite)

| | LSM-Tree | B-Tree |
|---|---|---|
| Write style | Append-only, sequential | Overwrites pages in place |
| Write throughput | **Higher** | Lower (random writes) |
| Reads | May check several segments (Bloom filters help) | One path from root to leaf, more predictable |
| Crash safety | WAL to rebuild the memtable | WAL (redo log) for page writes |
| Concurrency | Simpler: background merge plus atomic segment swap | Needs latches |
| Background work | Compaction can compete with live traffic | None needed |
| Examples | LevelDB, RocksDB, Cassandra, HBase | Most relational DBs (PostgreSQL, MySQL InnoDB) |

## 8. Quick recap (memorize this)

- **SSTable** = segment file **sorted by key**, each key once.
- **Advantages**: mergesort-style merging, **sparse index**, **block compression**.
- **Write path**: WAL → **memtable** (red-black/AVL tree) → flush to an **SSTable** at a few MB.
- **Read path**: memtable → newest SSTable → oldest.
- **Compaction**: merge segments and drop old or deleted values. **Size-tiered** vs **leveled**.
- **Bloom filter**: quickly rules out missing keys.
- **Strengths**: high write throughput, range queries, works with data much larger than RAM.

### Self-check questions
1. What is an SSTable, and how does it differ from a plain log segment?
2. How do you merge segments that are bigger than memory? Which value wins for a duplicate key?
3. Why can the in-memory index be sparse? Walk through finding `handiwork`.
4. How do writes arriving in random order end up sorted on disk?
5. What happens to memtable data during a crash, and how do you prevent losing it?
6. Why are lookups for missing keys slow, and how does a Bloom filter help?
7. Size-tiered vs leveled compaction: what's the difference, and who uses which?
8. When would you choose an LSM-tree over a B-tree?

## 9. In simple words: how it all fits together

An **LSM-tree** is the whole storage design, not just the tree in memory. **Writes** are first appended to a **WAL** on disk for crash safety, then inserted into the **memtable**, a balanced tree (red-black or AVL) in RAM that keeps keys sorted. When the memtable grows past a few MB, it is **flushed** to disk as a new **SSTable**, a sorted, immutable file, and a fresh memtable takes new writes. Each flush creates a new SSTable; SSTables never "fill up". **Reads** check the memtable, then SSTables from **newest to oldest**, and the newest value wins. **Compaction** merges SSTables in the background and drops old values and tombstones. A **sparse index** keeps a few keys per SSTable with their file offsets, so you jump close to the key and scan a little. A **Bloom filter** tells you when a key is **definitely not** in an SSTable ("maybe here" otherwise), which skips useless disk reads. Memtable = RAM, SSTables = disk, LSM-tree = the whole thing.
