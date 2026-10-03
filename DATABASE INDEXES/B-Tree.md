# B-Trees — Revision Notes

> Source: *Designing Data-Intensive Applications* (Martin Kleppmann), Chapter 3, "B-Trees".

## 1. What is a B-tree?

- The **most widely used index structure**. Introduced in 1970, still the default index in almost every relational database (and many NoSQL ones).
- Like SSTables, it keeps **key-value pairs sorted by key**, so you get fast lookups **and** range queries.
- That's where the similarity with log-structured indexes (LSM-trees) ends.

| | Log-structured (LSM / SSTables) | B-tree |
|---|---|---|
| Unit of storage | Variable-size **segments** (MBs) | Fixed-size **pages** (usually 4 KB) |
| How it writes | Appends sequentially, never modifies a file | **Overwrites pages in place** |
| Matches hardware? | Less directly | Yes, disks are also made of fixed-size blocks |

## 2. Structure: a tree of pages

- Each page has an **address** on disk, so one page can **refer** to another (a pointer, but on disk).
- One page is the **root**. Every lookup starts there.
- A page holds **keys + references to child pages**. Each child covers a **continuous range of keys**, and the keys between the refs mark the boundaries.
- The bottom pages are **leaf pages**. They hold the actual keys with their **values inline**, or references to where the values live.

### Lookup example (Figure 3-6): find `user_id = 251`

```
Root:   [ref | 100 | ref | 200 | ref | 300 | ref | 400 | ref | 500 | ref]
                                  │
                       200 ≤ key < 300
                                  ▼
Page:   [ref | 210 | ref | 230 | ref | 250 | ref | 270 | ref | 290 | ref]
                                          │
                               250 ≤ key < 270
                                          ▼
Leaf:   [250 val | 251 val | 252 val | 253 val | 254 val]   ← found 251
```

At each level, pick the ref whose range contains the key, follow it, and repeat until you reach a leaf.

## 3. Branching factor

- **Branching factor** = number of child references in one page.
- Figure 3-6 has a branching factor of **6**. Real databases have **several hundred**.
- Higher branching factor → fewer levels → fewer disk reads.

## 4. Updating and inserting

- **Update an existing key**: find its leaf page, change the value, write the page back. References to the page stay valid.
- **Insert a new key**: find the page whose range covers the key and add it there.
  - If the page is **full**, **split** it into two half-full pages and update the **parent** with the new boundary.

### Page split example (Figure 3-7): insert `334`

```
Before:
Parent: [ref | 310 | ref | 333 | ref | 345 | ref | (spare)]
                                  │ 333 ≤ key < 345
Leaf:   [333 | 335 | 337 | 340 | 342]   ← FULL, no room for 334

After adding 334:
Parent: [ref | 310 | ref | 333 | ref | 337 | ref | 345 | ref]   ← new boundary 337
                                  │            │
                       333 ≤ key < 337    337 ≤ key < 345
                                  ▼            ▼
Leaf A: [333 | 334 | 335 | spare]   Leaf B: [337 | 340 | 342 | spare]
```

(Deleting keys while keeping the tree balanced is more involved than inserting.)

## 5. Why B-trees stay fast

- Splitting keeps the tree **balanced**: with *n* keys, depth is always **O(log n)**.
- Most databases fit in a tree **3–4 levels deep**, so only a few page reads per lookup.
- A 4-level tree with 4 KB pages and a branching factor of 500 can store up to **256 TB**.

## 6. Making B-trees reliable

**The basic write is overwriting a page in place**, keeping its location so all references stay valid. LSM-trees never do this; they only append.

- On an **HDD** this means moving the disk head to the right spot and rewriting the sector.
- On an **SSD** it's more complicated: the SSD must erase and rewrite large blocks at a time.

### Problem 1: Crashes during multi-page writes
A page split writes **3 pages**: the two halves plus the parent. If the database crashes partway through, the index is **corrupted**, for example an **orphan page** with no parent.

**Fix: Write-Ahead Log (WAL)**, also called a **redo log**
- An **append-only file**. Every B-tree change is written here **before** it's applied to the tree pages.
- After a crash, the WAL is replayed to bring the B-tree back to a **consistent state**.

### Problem 2: Concurrency
Several threads updating pages in place could see the tree in a half-updated state.

**Fix: latches** (lightweight locks) protecting the tree's data structures.
- LSM-trees have it easier here: they merge in the background and **atomically swap** old segments for new ones.

## 7. B-tree optimizations

1. **Copy-on-write instead of WAL** (e.g. LMDB): write a modified page to a **new location**, then create new versions of the parent pages pointing to it. This also helps with concurrency control (snapshot isolation).
2. **Abbreviated keys**: interior pages only need enough of a key to act as a boundary. Shorter keys mean more keys per page, a higher branching factor and fewer levels. (This variant is sometimes called a **B+ tree**.)
3. **Sequential leaf layout**: try to put leaf pages in key order on disk so range scans don't need a disk seek per page. This is hard to maintain as the tree grows; LSM-trees get it more easily because they rewrite big segments during merging.
4. **Sibling pointers**: each leaf links to its left and right neighbours, so you can scan keys in order without going back up to the parent.
5. **Fractal trees**: B-tree variants that borrow log-structured ideas to reduce disk seeks (nothing to do with fractals).

## 8. Quick recap (memorize this)

- **B-tree = sorted tree of fixed-size pages (≈4 KB)**, root → interior pages → leaves.
- **Lookup**: follow the ref whose key range contains your key, level by level.
- **Branching factor**: refs per page, typically several hundred.
- **Insert into a full page → split into two + update the parent** → tree stays balanced, depth O(log n), usually 3–4 levels.
- **Writes overwrite pages in place** (unlike append-only LSM-trees).
- **WAL** for crash safety, **latches** for concurrency.
- **Optimizations**: copy-on-write, abbreviated keys (B+ tree), sequential leaves, sibling pointers, fractal trees.

### Self-check questions
1. How is a B-tree page different from an LSM segment?
2. Walk through looking up key 251 in Figure 3-6.
3. What happens when you insert into a full page?
4. Why do you need a WAL? What can go wrong without one?
5. Why do LSM-trees need less concurrency control than B-trees?
6. How much data can a 4-level tree with branching factor 500 hold?

## 9. In simple words: how it all fits together

A **B-tree** splits the database into fixed-size **pages** (about 4 KB) on disk, arranged as a tree. Each interior page holds **sorted keys and references to child pages**, and each child covers one range of keys. **Leaf pages** hold the actual values. A lookup starts at the root and follows the right reference at each level until it reaches a leaf, which takes **O(log n)** page reads, usually 3–4. The number of child references per page is the **branching factor** (not "balancing factor"), typically several hundred. Updates **overwrite the page in place**. If an insert finds its page full, the page **splits** in two and the parent gets a new boundary key. Splits can travel up to the root, and the tree grows from the top, so **all leaves always stay at the same depth** (stricter than the AVL rule that heights differ by at most 1). The 4 KB page size doesn't make the tree balanced; the splits do. A **WAL** protects against crashes during multi-page writes, and **latches** protect against threads seeing a half-updated tree. Unlike an LSM-tree, which never edits files and only appends, a B-tree edits pages in place. That makes reads predictable but writes slower.
