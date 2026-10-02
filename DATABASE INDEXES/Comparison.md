# Index Comparison: Hash Index vs SSTable/LSM-Tree vs B-Tree

> Detailed notes: [Hash-Indexes.md](Hash-Indexes.md) · [SSTables-and-LSM-Trees.md](SSTables-and-LSM-Trees.md) · [B-Tree.md](B-Tree.md)

| Aspect | Hash Index (Bitcask) | SSTable / LSM-Tree | B-Tree |
|---|---|---|---|
| **Core idea** | In-memory hash map: key → byte offset in an append-only log | Memtable flushed to **sorted** segment files (SSTables), merged in the background | Balanced tree of fixed-size **pages** (~4 KB) |
| **Data on disk** | Unsorted, append-only log segments | Sorted, immutable segments | Sorted pages, **overwritten in place** |
| **Write style** | Append (sequential) | Append and flush (sequential) | Random page overwrites |
| **Write speed** | Very fast | **Very high throughput** | Slower (random I/O, page splits) |
| **Point lookup** | Very fast: one seek | Memtable, then newest to oldest segment (Bloom filters help) | Root to leaf, O(log n), usually 3–4 levels. Predictable |
| **Range queries** | ❌ Not supported efficiently | ✅ Efficient (sorted) | ✅ Efficient (sorted, sibling pointers) |
| **Memory needed** | **All keys must fit in RAM** | Only a **sparse index** plus the memtable | Little. The tree lives on disk, hot pages are cached |
| **Dataset larger than RAM?** | Values yes, keys no | ✅ Yes | ✅ Yes |
| **Space management** | Segments + compaction + merging | Compaction (size-tiered or leveled) | Page splits. Free space inside pages |
| **Deletes** | Tombstone | Tombstone, removed during compaction | Remove the key from its page |
| **Crash recovery** | Hash-map snapshots on disk + checksums | WAL to rebuild the memtable | WAL (redo log) or copy-on-write |
| **Concurrency** | One writer, many readers (immutable segments) | Simple: background merge + atomic segment swap | Needs **latches** (lightweight locks) |
| **Compression** | Not covered | ✅ Compressed blocks | Limited (abbreviated keys) |
| **Main weakness** | Keys must fit in RAM. No range queries | Missing-key lookups are slow. Compaction uses background I/O | Lower write throughput. Fragmentation |
| **Best used when** | Few distinct keys, very frequent updates (e.g. video play counters) | **Write-heavy** workloads and large datasets | **Read-heavy** and transactional (OLTP) workloads needing predictable reads |
| **Examples** | Bitcask (Riak) | LevelDB, RocksDB, Cassandra, HBase, Lucene | PostgreSQL, MySQL (InnoDB), Oracle, SQL Server, LMDB |
| **Why it exists** | Simplest fast index for a key-value log | Removes the hash index limits (RAM, range queries) while keeping fast sequential writes | Standard index since 1970. Stable, balanced, great for reads |
