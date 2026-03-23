
# Wide-Column Stores: Partitioning and Compaction

> *A deep-dive into how distributed databases like Cassandra and HBase organize, route, and reclaim data at massive scale.*

***

## The Filing Cabinet That Got Too Big

Imagine you are the office manager of a law firm with ten thousand clients. On day one, you have a single filing cabinet. Every new client folder goes in alphabetically. Life is simple. Reads are fast. Writes are instant.

Then the firm explodes in size. You have a hundred thousand clients and a hundred thousand folders. One cabinet is now a warehouse. Worse, lawyers are constantly updating folders, adding new pages, and occasionally shredding old ones. The single filing cabinet becomes a bottleneck — every paralegal is fighting to access it simultaneously.

So you make two design decisions. First, you **partition** the warehouse: floor A houses clients A–M, floor B houses clients N–Z. Each floor has its own staff. Now paralegal teams work in parallel without blocking each other. Second, every time a new page arrives, instead of hunting down the original folder to insert it, your staff drops it in a fast-intake tray at the front. At night, a maintenance crew merges all the intake tray papers into the right folders, discarding duplicates and shredded content. This night-time merging is **compaction**.

This is the intuition behind wide-column stores. Systems like **Apache Cassandra**, **Google Bigtable**, and **Apache HBase** face this exact problem — millions of rows, billions of columns, continuous writes — and they solve it with the same two-pronged strategy. We will explore each strategy in depth.

***

## What Makes a Store "Wide-Column"?

Before we talk about partitioning and compaction, we need to establish the data model we are working with, because it shapes every design decision.

### The Data Model at a Glance

In a relational database, a table has a fixed schema. Every row has the same columns, and empty values waste space or require nulls. Wide-column stores take a different approach: rows share a **row key** as their identifier, but each row can carry a completely different set of **columns**. Columns are grouped into **column families** — logical buckets that are stored together on disk.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  TABLE: user_activity                                                     │
│                                                                           │
│  Row Key          │  CF: profile          │  CF: events                   │
│───────────────────┼───────────────────────┼──────────────────────────────│
│  user:1001        │  name: "Alice"         │  login:2024-01-01: true       │
│                   │  age: 29              │  purchase:2024-01-03: "book"  │
│───────────────────┼───────────────────────┼──────────────────────────────│
│  user:1002        │  name: "Bob"           │  login:2024-01-02: true       │
│                   │  (no age stored)      │  (no purchase recorded)       │
└──────────────────────────────────────────────────────────────────────────┘
```

*In this figure, notice that `user:1002` has no `age` and no `purchase` column. Nothing is stored for absent data — there are no NULLs taking up space. This sparsity is a first-class feature.*

The row key is the **only** built-in index. This is not a limitation — it is a deliberate design choice. Because all data for a row key lives together on disk, a single-key lookup is blazing fast. The trade-off is that arbitrary cross-column queries require table scans. Wide-column stores are **query-first** systems: you design your schema around the queries, not the entities.

***

## Part I: Partitioning

### The Core Problem: One Node Is Not Enough

A single filing cabinet, no matter how well organized, has a ceiling. Wide-column stores are built to serve **hundreds of terabytes** across clusters of dozens or hundreds of machines. We need a strategy to split data across nodes, route writes to the correct node, and find data on reads — all without a central coordinator that becomes a bottleneck.

### The Naive Approach: Range-Based Partitioning

The simplest idea is to split data by key ranges: node 1 handles rows A–F, node 2 handles G–L, and so on. This is called **range partitioning**.

```
  ┌────────────┐    ┌────────────┐    ┌────────────┐
  │  Node 1    │    │  Node 2    │    │  Node 3    │
  │  A - F     │    │  G - L     │    │  M - Z     │
  └────────────┘    └────────────┘    └────────────┘
```

Range partitioning has one great property: **range scans are fast**. All rows starting with `user:2024-01` live on the same node. But it has a brutal failure mode: **hotspots**. If most of your users have names starting with "S" — or if your row keys are timestamps — all writes pile onto one node. That node sweats while its neighbours sip coffee.

### Enter Consistent Hashing: The Hash Ring

To solve the hotspot problem, we **hash the partition key** and use the hash value to assign data to nodes. The elegant structure for this is the **consistent hash ring**. We map the entire hash space — all 64-bit integers — as a ring. Each node is assigned a position on this ring. A partition key is hashed (Cassandra uses the **Murmur3** algorithm), and the data is owned by the **first node clockwise** from that hash value.

```
                    ┌──────────────────────────────────┐
                    │          HASH RING               │
                    │                                  │
                    │           Node A                 │
                    │          (token: 0)              │
                    │        ┌──────┐                  │
                    │    ┌───┤      ├───┐              │
                    │    │   └──────┘   │              │
                    │  Node D        Node B            │
                    │ (token:        (token:           │
                    │  -4.6B)         2.3B)            │
                    │    │              │              │
                    │    └───┐      ┌───┘              │
                    │        └──────┘                  │
                    │         Node C                   │
                    │        (token: -2.3B)            │
                    └──────────────────────────────────┘

  Key "alice" ──► hash = 1,680,000,000 ──► falls between Node B and Node C
                  ──► assigned to Node C (first node clockwise)
```

*In the diagram above, we see four nodes placed at token positions on the ring. A partition key is hashed to a numeric value, placed on the ring, and ownership goes to the next clockwise node. No central lookup table is needed — any node can independently compute who owns any key.*

The critical property of consistent hashing is that when a node joins or leaves, **only the keys in the affected arc need to be remapped**. Add Node E between Node B and Node C, and only Node C's data in that arc migrates. In a naive modulo-based approach, adding a node reshuffles nearly all data.

### The Problem: Uneven Single-Token Assignment

Early Cassandra assigned each physical node a **single position** on the ring, causing two real-world headaches:

1. **Uneven load**: Nodes ended up with different-sized arcs by chance.
2. **Slow rebalancing**: Adding a node required manual token calculation and large data transfers from exactly one arc.
| Partition Key | Murmur3 Hash | Owner |
| :-- | :-- | :-- |
| `jim` | -2,245,462,676 | Node D |
| `carol` | 7,723,358,927 | Node A |
| `johnny` | -6,723,372,854 | Node D |
| `suzy` | 1,168,604,627 | Node B |

Notice `jim` and `johnny` both fall on Node D — with a single token per node, skew is common.

### The Fix: Virtual Nodes (VNodes)

Cassandra's answer, introduced in version 1.2, is **virtual nodes (VNodes)**. Instead of placing each physical node at one position on the ring, Cassandra assigns each node **many small, non-contiguous token ranges** — 256 by default.

```
Physical Ring with VNodes:

  ── Node A ──┬── Node C ──┬── Node B ──┬── Node A ──┬── Node C ──► ...
              │            │            │            │
           (VNode 1)    (VNode 1)    (VNode 1)    (VNode 2)    (VNode 2)
```

*In this figure, each physical node (A, B, C) owns multiple scattered arc segments. The total arc size per node converges to roughly equal shares — like a pie divided into many thin slices distributed around the ring.*

VNodes deliver three concrete benefits:

- **Even load distribution**: With 256 tokens per node, statistical averaging ensures near-uniform data distribution.
- **Faster bootstrap**: A new node steals small arcs from many neighbours in parallel, rather than a large arc from one.
- **Graceful degradation**: When a node fails, its arc tokens are served by many different neighbours, spreading the load instead of overwhelming one successor.


### Partition Key Design: The Developer's Responsibility

The hash ring solves cluster-level distribution. But the developer still controls **what goes into the partition key**. Poor partition key design is the most common cause of production performance problems.

**Rule 1: Maximize cardinality.** A partition key with only 3 possible values can only spread data across 3 effective shards. `user_id` (millions of values) is far better than `country_code` (200 values).

**Rule 2: Avoid monotonically increasing keys.** Raw timestamps as partition keys create write hotspots even in hash-partitioned systems when used as the sole dimension.

**Rule 3: Avoid unbounded partitions.** A single partition lives on one node. If one partition key maps to 50 GB of data, that node becomes a bottleneck. Use **composite partition keys** to bucket data:

```python
# BAD: sensor_id alone leads to unbounded partition growth
# All readings for one sensor end up on one node forever
#
# CREATE TABLE sensor_readings (
#     sensor_id   UUID,
#     ts          TIMESTAMP,
#     value       FLOAT,
#     PRIMARY KEY (sensor_id, ts)
# );

# GOOD: bucket by month to cap partition size
# Partition key = (sensor_id, year_month)
#
# CREATE TABLE sensor_readings_bucketed (
#     sensor_id   UUID,
#     year_month  TEXT,         -- e.g., "2024-01"
#     ts          TIMESTAMP,
#     value       FLOAT,
#     PRIMARY KEY ((sensor_id, year_month), ts)
# );

# Python simulation of partition key design
def make_partition_key(sensor_id: str, timestamp: str) -> str:
    """
    Create a bucketed partition key from sensor_id + year_month.
    Caps partition size to one month of readings per sensor.
    """
    year_month = timestamp[:7]  # e.g., "2024-01" from "2024-01-15T12:00:00"
    return f"{sensor_id}:{year_month}"


# Example writes
readings = [
    ("sensor_42", "2024-01-01T10:00:00", 23.5),
    ("sensor_42", "2024-01-15T14:00:00", 24.1),
    ("sensor_42", "2024-02-01T09:00:00", 22.8),
]

for sensor_id, ts, value in readings:
    pk = make_partition_key(sensor_id, ts)
    print(f"  Row → partition_key={pk!r}, clustering_key={ts!r}, value={value}")

# Output:
#   Row → partition_key='sensor_42:2024-01', clustering_key='2024-01-01T10:00:00', value=23.5
#   Row → partition_key='sensor_42:2024-01', clustering_key='2024-01-15T14:00:00', value=24.1
#   Row → partition_key='sensor_42:2024-02', clustering_key='2024-02-01T09:00:00', value=22.8
```

The `(sensor_id, year_month)` composite partition key caps partition size to one month of readings per sensor. Queries within a month remain a single-partition read. Queries spanning months need multiple partition reads — a deliberate and predictable trade-off.

***

## Part II: The Write Path and the LSM Tree

Now that we know **where** data goes, we need to understand **how** it gets written and stored. This is where the **Log-Structured Merge Tree (LSM Tree)** comes in — the storage engine at the heart of Cassandra, HBase, RocksDB, and most modern wide-column systems.

### Why Not B-Trees?

Traditional relational databases use **B-Trees** as their on-disk structure. B-Trees are brilliant for read-heavy workloads: they keep data sorted, support range queries, and are cache-friendly. But they have one weakness at write-intensive scale: **random writes**. When you update a value in a B-Tree, the database must find the existing page on disk, load it into memory, modify it, and write it back. At millions of writes per second, this random I/O creates a storm of disk head movements.

### The Atomic Unit: MemTable

The LSM Tree's key insight is: **never modify data on disk**. Instead, buffer all writes in memory first.

When a write arrives, it goes to two places simultaneously:

1. A **Write-Ahead Log (WAL)** — an append-only log on disk for durability (crash recovery only).
2. The **MemTable** — an in-memory sorted data structure (usually a Red-Black Tree or Skip List).
```
  WRITE REQUEST ──► WAL (disk append, sequential, fast)
                └─► MemTable (in-memory sorted structure, instant)

  No random I/O. No page lookup. No read-modify-write.
```

*In the above flow, the WAL write is sequential (fast on both HDDs and SSDs), and the MemTable write is a pure memory operation. We have turned random I/O into sequential I/O — the core performance win of the LSM design*.

The MemTable absorbs writes until it reaches a size threshold (e.g., 128 MB). At that point, it is **frozen** (made immutable) and a new MemTable is opened for fresh writes. The frozen MemTable is **flushed to disk** as a new SSTable file.

### SSTables: Immutable Sorted Files

A **Sorted String Table (SSTable)** is an immutable, sorted, compressed file on disk. Because the MemTable was sorted in memory, the resulting SSTable is already sorted by key — no re-sorting step needed during flush.

```
  MemTable (in memory)          SSTable (on disk)
  ┌────────────────────┐        ┌──────────────────────────────┐
  │ alice   → 29       │        │ Block 1: alice:29, bob:35    │
  │ bob     → 35       │ flush  │ Block 2: carol:22, dave:41   │
  │ carol   → 22       │ ─────► │ Index: {alice:0, carol:512}  │
  │ dave    → 41       │        │ Bloom Filter: [0,1,0,1,1...] │
  └────────────────────┘        └──────────────────────────────┘
```

*In this diagram, the MemTable (left) is a sorted in-memory structure. On flush, it becomes an SSTable (right) with three components: the data blocks, an index for fast lookup, and a Bloom filter for probabilistic membership testing. We will revisit these during the read path.*

Each SSTable has an **index block** mapping keys to byte offsets, enabling binary search within the file. It also carries a **Bloom filter** — a compact probabilistic structure that can answer "is key X definitely NOT in this file?" with zero false negatives, dramatically reducing unnecessary disk reads.

### The Growing Problem: Too Many SSTables

Here is where the system starts to strain. Each MemTable flush creates a new SSTable. After a few hours of heavy writes, you might have hundreds of SSTables on disk, creating three problems:

**Problem 1 — Read Amplification.** When we read key `alice`, we must check every SSTable that *might* contain her. The Bloom filter helps, but if she has been updated 10 times, her key exists in 10 SSTables. At N SSTables, that is up to N disk reads per query in the worst case.

**Problem 2 — Space Amplification.** Each update to `alice` leaves the old version in an older SSTable. Neither is deleted until someone merges them. Disk space bloats with obsolete versions.

**Problem 3 — Tombstone Accumulation.** Deletes in LSM systems do not erase data. Instead, they write a special marker called a **tombstone**. The tombstone hides the old value during reads, but the old value still occupies disk space. If tombstones pile up, they slow both reads and compaction.

The fix for all three problems is **compaction**.

***

## Part III: Compaction

### The Night-Time Janitor

Compaction is the background process that periodically merges SSTables together. More precisely, compaction:

1. Selects a set of SSTables as candidates.
2. Merges them using a **k-way merge sort** (like merging K sorted arrays).
3. Drops obsolete versions (keeps the newest for each key).
4. Drops tombstones (along with their associated deleted data, once old enough).
5. Writes a new, larger SSTable.
6. Deletes the old SSTables, reclaiming disk space.
```
BEFORE COMPACTION:

  SSTable 1: [alice:v1, bob:v2, dave:v1]
  SSTable 2: [alice:v2, carol:v1]
  SSTable 3: [bob:v3, carol:TOMBSTONE]

AFTER COMPACTION (k-way merge, keep latest, drop tombstoned):

  SSTable merged: [alice:v2, bob:v3, dave:v1]
  (carol wiped: carol:v1 deleted by tombstone, tombstone itself dropped)
```

*In the example above, compaction resolves three SSTables into one. `alice:v1` is superseded by `alice:v2`. `bob:v2` is superseded by `bob:v3`. `carol:v1` is deleted by the tombstone. The output is clean and compact*.[^10]

### The Fundamental Trade-off Triangle

Compaction is not free. Every byte it merges is a byte written to disk *again* — called **write amplification**. We are always balancing three competing metrics:

```
           Write Amplification
                   ▲
                   │
                   │  ┌──── Leveled ─────┐
                   │  │  low space amp,   │
                   │  │  high write amp   │
                   │  └──────────────────┘
                   │
                   │  ┌── Size-Tiered ───┐
                   │  │  low write amp,   │
                   │  │  high space amp   │
                   │  └──────────────────┘
                   │
   ────────────────┼──────────────────────►  Space Amplification
                   │
                Read Amplification (the third axis of the trade-off)
```

*This triangle illustrates that no strategy eliminates all three problems simultaneously. All compaction strategies are moving along these axes, not off them*.

***

### Strategy 1: Size-Tiered Compaction (STCS)

**The default in Cassandra and ScyllaDB**.

**The Idea:** Group SSTables by size. When you have enough SSTables of similar size (typically 4), merge them into one larger SSTable. The result grows into the next size tier, and the process repeats. Data cascades downward through size tiers like a waterfall.

```
  TIER 1 (small ~10MB):    [A] [B] [C] [D]  ──► merge ──► [ABCD ~40MB]
                                                                │
  TIER 2 (medium ~40MB):   [W] [X] [Y] [ABCD] ─► merge ──► [WXYZ-ABCD ~160MB]
                                                                │
  TIER 3 (large ~160MB):   [P] [Q] [WXYZ...]  ── ... ──► [huge sealed file]
```

*In this cascade, small SSTables are born from MemTable flushes. Four similar-sized files trigger a merge, yielding a file 4× larger. This larger file eventually meets others its size, triggering another merge. The intuition is that data "falls" down tiers as it ages.*

```python
from collections import defaultdict

def select_stcs_candidates(
    sstables: list[dict],
    min_threshold: int = 4,
    bucket_low: float = 0.5,
    bucket_high: float = 1.5
) -> list[dict]:
    """
    Group SSTables by size into buckets.
    Select the first bucket that meets the min_threshold for compaction.
    Each SSTable dict: {'name': str, 'size_mb': float}
    """
    if not sstables:
        return []

    sorted_tables = sorted(sstables, key=lambda s: s["size_mb"])
    buckets = defaultdict(list)

    for table in sorted_tables:
        placed = False
        for bucket_key in list(buckets.keys()):
            avg = sum(t["size_mb"] for t in buckets[bucket_key]) / len(buckets[bucket_key])
            if bucket_low * avg <= table["size_mb"] <= bucket_high * avg:
                buckets[bucket_key].append(table)
                placed = True
                break
        if not placed:
            buckets[table["size_mb"]].append(table)  # start a new bucket

    # Return the first bucket with enough members
    for bucket in buckets.values():
        if len(bucket) >= min_threshold:
            return bucket[:min_threshold]

    return []


# Example
sstables = [
    {"name": "ss1", "size_mb": 10},
    {"name": "ss2", "size_mb": 11},
    {"name": "ss3", "size_mb": 12},
    {"name": "ss4", "size_mb": 10},
    {"name": "ss5", "size_mb": 85},
    {"name": "ss6", "size_mb": 90},
]

candidates = select_stcs_candidates(sstables)
print("Compaction candidates:", [c["name"] for c in candidates])
# Output: Compaction candidates: ['ss1', 'ss4', 'ss2', 'ss3']
# (The four ~10MB files are selected; the 85-90MB files wait for more peers)
```

**Pros of STCS**:

- Very low write amplification — each byte is rewritten roughly $\log_4 N$ times.
- Excellent for write-heavy workloads: IoT ingestion, event streams, log data.
- Fast MemTable flushes translate directly to fast write throughput.

**Cons of STCS**:

- High **space amplification**: during a compaction, both old and new SSTables coexist (up to 2× temporary disk space).
- High **read amplification**: SSTables within a tier have overlapping key ranges, so a read may check all files in a tier.
- **Tombstone danger**: deletes buried in large SSTables accumulate and can cause `TombstoneOverwhelmingException` in Cassandra under delete-heavy workloads.

***

### Strategy 2: Leveled Compaction (LCS)

**The default in RocksDB, LevelDB, and optionally in Cassandra**.

LCS was designed to fix the read amplification and space amplification problems of STCS, at the cost of higher write amplification.

**The Core Invariant:** At every level except L0, **no two SSTables have overlapping key ranges**. A single read can be answered by checking *at most one SSTable per level*. This is the guarantee that makes reads predictable.

```
  L0:  [A-Z] [A-Z] [A-Z] [A-Z]    ← newly flushed, may overlap freely
  L1:  [A-E] [F-J] [K-O] [P-Z]    ← non-overlapping, ≤10MB each
  L2:  [A-B] [C-D] [E-F] ... [Y-Z]← non-overlapping, ≤100MB each
  L3:  [A]   [B]   [C]   ... [Z]  ← non-overlapping, ≤1000MB each
```

*In this diagram, L0 has four SSTables that all span the full key range A–Z (freshly flushed from MemTables). L1 onwards are partitioned: each SSTable owns a disjoint key range. A read for key "G" checks at most one file per level — the one whose range includes G.*

When L0 accumulates too many files (usually 4), a compaction is triggered: all L0 files are merged with all overlapping L1 files, producing new non-overlapping L1 files. If L1 then exceeds its size budget, the cascade continues to L2, and so on.

```python
def find_overlapping_sstables(
    sstables_level: list[dict],
    input_ranges: list[tuple]
) -> list[dict]:
    """
    Given SSTables in a level (each with min_key, max_key),
    return those whose key range overlaps any of the input_ranges.

    Two ranges [a,b] and [c,d] overlap iff: a <= d AND c <= b
    """
    overlapping = []
    for sstable in sstables_level:
        for (range_min, range_max) in input_ranges:
            if sstable["min_key"] <= range_max and range_min <= sstable["max_key"]:
                overlapping.append(sstable)
                break
    return overlapping


# L0 SSTables to compact (all span full range)
l0_ranges = [("A", "Z"), ("A", "M"), ("B", "F")]

# Current L1 SSTables with non-overlapping ranges
l1_tables = [
    {"name": "l1_1", "min_key": "A", "max_key": "E"},
    {"name": "l1_2", "min_key": "F", "max_key": "J"},
    {"name": "l1_3", "min_key": "K", "max_key": "O"},
    {"name": "l1_4", "min_key": "P", "max_key": "Z"},
]

overlaps = find_overlapping_sstables(l1_tables, l0_ranges)
print("L1 files to merge with L0:", [s["name"] for s in overlaps])
# Output: L1 files to merge with L0: ['l1_1', 'l1_2', 'l1_3', 'l1_4']
# (All L1 files participate because L0 spans A-Z)
```

**Pros of LCS**:

- Low **read amplification**: at most one file checked per level — predictable and fast.
- Low **space amplification**: only ~10% extra space needed at any point.
- Consistent **read latency** — critical for SLA-bound workloads like user-facing APIs.

**Cons of LCS**:

- **High write amplification**: data is rewritten as it cascades through levels. In the worst case, a byte written to L0 is rewritten at each level all the way down — multiplying I/O by the level count.
- More **CPU and I/O pressure** from background compaction, which competes with foreground reads and writes.
- Not ideal for pure write-throughput workloads like log collection or bulk ingestion.

***

### Strategy 3: Time-Window Compaction (TWCS)

**Designed specifically for time-series workloads in Cassandra.**

Time-series data has a beautiful property: **old data is never updated**. Yesterday's sensor readings will not be modified today. TWCS exploits this.

**The Idea:** Partition SSTables by the time window of the data they contain. Compact *within* a window using STCS, but **never merge across windows**. A window's SSTables are eventually compacted into a single sealed SSTable and left untouched forever.

```
  WINDOW: Jan 1-7   → compact all Jan 1-7 SSTables → one sealed SSTable ✓
  WINDOW: Jan 8-14  → compact all Jan 8-14 SSTables → one sealed SSTable ✓
  WINDOW: Jan 15-21 → [actively compacting]
  WINDOW: Jan 22+   → [current, receiving writes]
```

When data ages beyond its TTL (time-to-live), entire SSTable files are **dropped** — no key-by-key compaction needed. Deletion is essentially free. TWCS achieves remarkably low write and space amplification for time-series, but breaks entirely if you write out-of-order data or update old records. Never use TWCS unless your workload is truly append-only and time-ordered.

***

### Compaction Strategy Quick-Reference

| Strategy | Write Amp | Read Amp | Space Amp | Best For |
| :-- | :-- | :-- | :-- | :-- |
| STCS | Low | High | High | Write-heavy: IoT, logs, events |
| LCS | High | Low | Low | Read-heavy: user profiles, config |
| TWCS | Very Low | Low* | Very Low | Time-series, append-only data |

*\*TWCS read amplification is low because data is naturally partitioned by time, reducing the SSTable search space per query*.

***

## Part IV: The Full Read Path

Now we can trace the full lifecycle of a read request and see how every component works together:

```
  CLIENT REQUEST: GET alice
        │
        ▼
  ┌──────────────────────────────────────────────────────┐
  │ Step 1: Route via Consistent Hashing                  │
  │   hash("alice") = 1,234,567,890                       │
  │   Token → Node B                                      │
  └────────────────────────┬─────────────────────────────┘
                            │
                            ▼
  ┌──────────────────────────────────────────────────────┐
  │ Step 2: Check MemTable (in-memory)                    │
  │   Is alice in the current MemTable? → No              │
  └────────────────────────┬─────────────────────────────┘
                            │
                            ▼
  ┌──────────────────────────────────────────────────────┐
  │ Step 3: Consult Bloom Filters for each SSTable        │
  │   SSTable 1 Bloom: alice? → Definitely NOT  ✗ skip   │
  │   SSTable 2 Bloom: alice? → Possibly YES    ✓ check  │
  │   SSTable 3 Bloom: alice? → Definitely NOT  ✗ skip   │
  └────────────────────────┬─────────────────────────────┘
                            │
                            ▼
  ┌──────────────────────────────────────────────────────┐
  │ Step 4: Binary search SSTable 2 index                 │
  │   Index: {alice: byte_offset=2048}                    │
  │   Read data block at offset 2048                      │
  │   Found: alice → v2                                   │
  └────────────────────────┬─────────────────────────────┘
                            │
                            ▼
  ┌──────────────────────────────────────────────────────┐
  │ Step 5: Return latest version                         │
  │   Only SSTable 2 matched → return alice:v2            │
  └──────────────────────────────────────────────────────┘
```

*In this five-step trace, partitioning (step 1) routes us to the right node, the MemTable (step 2) catches the freshest data, Bloom filters (step 3) eliminate most SSTables without disk reads, the index (step 4) pinpoints the exact byte offset, and version reconciliation (step 5) returns the correct answer. Compaction makes step 3 faster by reducing the total number of SSTables — fewer files means fewer filter checks.*

***

## Part V: Interview Patterns and Practical Intuition

These questions appear regularly in senior and staff-level system design interviews at companies using wide-column stores at scale.

### "Why are monotonically increasing keys dangerous?"

Because they create **write hotspots** in range-partitioned systems (HBase) and unbalanced access patterns in hash-partitioned systems when combined with sequential clustering keys. Use a high-cardinality prefix — `userID:timestamp` is far safer than a raw `timestamp` as a row key.

### "How do you handle compaction falling behind writes?"

This is called **compaction debt**. Symptoms: increasing SSTable count, degrading read latency, and `TombstoneOverwhelmingException` in Cassandra. Solutions:

- Reduce write throughput temporarily (back-pressure).
- Increase compaction thread pool size.
- Switch from STCS to LCS for read-critical column families.
- Increase the compaction throughput rate limit.

At Netflix scale, compaction lag is monitored as a **key SLA metric**, and separate compaction pools are used for different column families to prevent resource contention.

### "What is the role of the Bloom filter and what happens if it's wrong?"

Bloom filters have **no false negatives** — if it says "definitely not here," trust it completely. They can have **false positives** — "possibly here" — meaning we might check an SSTable that does not contain the key. The false positive rate is tunable: lower FPR means a larger filter (more memory). A 1% FPR is common; pushing to 0.1% roughly doubles the filter's memory footprint.

### "When would you choose HBase over Cassandra?"

HBase favors **strong consistency** (master-based architecture) and **excellent sequential range scans** (range-partitioned). Cassandra favors **availability and partition tolerance** (leaderless) and **even write distribution** (hash-partitioned). Use HBase when you need ACID-like row-level operations or scan-heavy analytics; use Cassandra when you need multi-region availability and raw write throughput.

***

## Closing: The System Is a Negotiation

Wide-column stores are not magic — they are a carefully negotiated set of trade-offs, each made to serve a specific problem. Partitioning by consistent hashing distributes write load at the cost of natural ordering. The LSM Tree absorbs write bursts at the cost of background compaction work. Compaction strategies each trade one form of amplification for another.

As engineers, our job is to understand which trade-offs are acceptable for our workload:

- **Write-dominant? Use STCS.** Accept the read penalty; your writes will fly.
- **Read-dominant? Use LCS.** Accept the write penalty; your reads will be predictable.
- **Time-series? Use TWCS.** Accept the constraint of time-ordered, non-updated data; your storage efficiency will be remarkable.

The filing cabinet analogy holds all the way down: the goal is never a *perfect* system. It is a system whose imperfections land exactly where you can afford them.

***