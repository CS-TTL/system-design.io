
# Key-Value Stores and TTL/Eviction Strategies

> *"The fastest data is the data you never have to fetch twice."*
> — Distributed Systems folklore

***

## The Pantry Analogy: Before We Write a Single Line of Code

Imagine you are a chef in a busy restaurant kitchen. Your full refrigerator is in the back — it holds everything: every ingredient, every sauce, every garnish. But when service begins, you don't run to the back fridge every time you need salt. You keep a small **countertop tray** right next to your station: the most-used spices, pre-prepped garnishes, and tonight's sauces. This tray is small, fast to reach, and always within arm's length.

Now here's the tension: that countertop tray has limited space. If you keep adding things to it without removing old ones, it overflows and becomes useless. So you have to make decisions:

- **Which items should I put on the tray?** (What goes in the cache)
- **When should I remove something?** (Eviction strategies)
- **How long is this sauce still good?** (Time-To-Live / TTL)

That countertop tray? That is your **key-value store**. The refrigerator is your database. The rules you use to manage your tray are **eviction strategies**. The "sell-by" label on the sauce is your **TTL**.

***

## Part 1: What Is a Key-Value Store?

At its atomic core, a key-value store maps a unique **key** to a **value**. Think of it as the most primitive form of a database — a lookup table where you hand it a key and it hands back the corresponding value.

```python
# The simplest possible key-value store: a Python dictionary
store = {}

# Write (PUT)
store["user:1001:name"] = "Alice"
store["user:1001:session"] = "abc123"

# Read (GET)
name = store["user:1001:name"]   # → "Alice"

# Delete (DEL)
del store["user:1001:session"]
```

Three operations. That's it — `PUT`, `GET`, `DELETE`. The beauty is in the simplicity. But the moment you ask a key-value store to handle **millions of keys**, **concurrent reads/writes**, **persistence**, and **limited memory**, the engineering problem explodes in complexity.

### The Real-World Cast of Characters

| System | Primary Use Case | Storage Model |
| :-- | :-- | :-- |
| **Redis** | Caching, sessions, pub/sub, leaderboards | In-memory (with optional disk persistence) |
| **Memcached** | Pure distributed caching | In-memory only |
| **DynamoDB** | Persistent NoSQL at scale | Disk-backed (SSD) |
| **etcd** | Distributed config / service discovery | Disk-backed (Raft consensus) |
| **RocksDB** | Embedded storage engine | Disk-backed (LSM-tree) |


***

## Part 2: Under the Hood — How a Key-Value Store Actually Works

### The Hash Table: The Engine Room

If you open the hood of any in-memory key-value store, you'll find a **hash table** at the center of it. A hash table gives average O(1) reads and writes — that's the superpower that makes key-value stores fast.

```
svgbob
        Key: "user:1001"
              |
              v
    +--------------------+
    |   Hash Function    |  hash("user:1001") → 42
    +--------------------+
              |
              v
    +---+---+---+---+---+---+---+---+
    | 0 | 1 | . | . |42 | . | . |63 |    ← Bucket Array
    +---+---+---+---+---+---+---+---+
                        |
                        v
              +------------------+
              | key: "user:1001" |
              | val: "Alice"     |
              | ttl: 3600s       |   ← The stored entry
              | metadata: {...}  |
              +------------------+
```

*In this diagram, the hash function transforms the string key into a bucket index (42). The actual key-value pair — along with TTL metadata — lives at that bucket. This is exactly how Redis stores entries internally.*

Redis maintains **two hash tables simultaneously** during a resize (called **incremental rehashing**), so the server never has to block on a massive resize operation.  Modern systems like Valkey embed the key/value directly into a unified structure, reducing memory overhead and improving CPU cache locality.

***

## Part 3: Time-To-Live (TTL) — Putting an Expiration Date on Your Data

### The Problem: Stale Data Is Worse Than No Data

Imagine caching a user's authentication token. Great — it's fast to look up. But what happens when that token expires on the auth server? Without a removal mechanism, your application will happily serve an expired token — a security vulnerability or confusing UX.

This is the pain point: **some data has a natural lifespan**. Session tokens expire. Rate-limit counters reset every minute. DNS records should be flushed after their TTL elapses. We need to say: *"This key is valid for X seconds — after that, it disappears automatically."*

### Two Eviction Clocks

Redis enforces TTL using both strategies simultaneously:

- **Passive Expiry (Lazy Deletion)**: The system checks if a key has expired *only when you try to access it*. Cheap, but expired keys can linger if never accessed again.
- **Active Expiry (Proactive Deletion)**: A background process periodically samples random keys and deletes expired ones.

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Set a key with a TTL of 60 seconds
r.set("session:abc123", "user_data_here", ex=60)

# Check remaining TTL
remaining = r.ttl("session:abc123")
print(f"Key expires in {remaining} seconds")  # e.g., 58

# After 60 seconds, the key is gone
time.sleep(61)
result = r.get("session:abc123")
print(result)  # → None
```

```
svgbob
  T=0s         T=30s        T=60s       T=61s
   |             |             |           |
   +------+      +------+      +-----+     +------+
   | SET  |      | GET  |      | GET |     | GET  |
   | key  |      | key  |      | key |     | key  |
   | EX60 |      |      |      |     |     |      |
   +------+      +------+      +-----+     +------+
   key written   → "value"     → "value"   → nil (expired)
                 TTL: 30s      TTL: 1s     gone
```

*Notice that the key is fully readable right up until TTL elapses. The moment T=61s arrives, the passive expiry check on the next GET detects expiration and returns nil.*

***

## Part 4: The Real Challenge — When Memory Fills Up

### The Pain Point: The Tray Is Full

Back to our chef analogy. The countertop tray is completely full. A new ingredient arrives. What do you remove to make space? This is the **eviction problem** — and your answer depends entirely on your workload.

Redis supports **eight eviction policies**, organized into three conceptual families.

***

### Family 1: "Do Nothing"

**`noeviction`**: When memory is full, return an error on any write. Reads still work. Use this when data loss is unacceptable — like a job queue or a primary datastore.

***

### Family 2: Recency-Based (LRU)

**Least Recently Used (LRU)** evicts the item that was accessed furthest in the past.

```
svgbob
  Access Timeline (oldest → newest):

  [key:D] → [key:A] → [key:C] → [key:B]   ← most recent

  Memory full. Need to evict one key.

  LRU evicts: [key:D]  (accessed longest ago)
```

*key:D sits at the far left — accessed longest ago. LRU always targets the "cold" end of the access timeline.*

**Redis's Approximate LRU**: True LRU requires a global sorted list — too expensive for millions of keys. Redis instead **samples a small pool of keys** (default: 5), compares last-access timestamps, and evicts the oldest among the sample. This gives ~95% accuracy at a fraction of the cost.

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key: str):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)   # mark as recently used
        return self.cache[key]

    def put(self, key: str, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # evict LRU (front)
```

Redis offers two LRU variants:

- **`allkeys-lru`** — Apply LRU to the entire keyspace
- **`volatile-lru`** — Apply LRU only to keys that *have a TTL set* (protecting permanent keys)

***

### Family 3: Frequency-Based (LFU)

**Least Frequently Used (LFU)** asks a different question: *how often has this key been accessed overall?* It evicts keys with the lowest access count.

```
svgbob
  Access Frequency:

  key:A  ████████████████  (count: 16)
  key:B  ████              (count:  4)
  key:C  ██████████████    (count: 14)
  key:D  █                 (count:  1)

  Memory full. LFU evicts: [key:D]
```

**The Cache Pollution Problem**: A naive LFU would protect a key accessed 10,000 times during a viral event months ago — even though it's completely irrelevant today. Redis solves this with a **decay model**: counts decrease logarithmically over time, so old popularity doesn't provide eternal protection.

```python
from collections import defaultdict, OrderedDict

class LFUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.min_freq = 0
        self.key_to_val_freq = {}
        self.freq_to_keys = defaultdict(OrderedDict)

    def _update_freq(self, key: str):
        val, freq = self.key_to_val_freq[key]
        del self.freq_to_keys[freq][key]
        if not self.freq_to_keys[freq]:
            del self.freq_to_keys[freq]
            if self.min_freq == freq:
                self.min_freq += 1
        new_freq = freq + 1
        self.key_to_val_freq[key] = [val, new_freq]
        self.freq_to_keys[new_freq][key] = None

    def get(self, key: str):
        if key not in self.key_to_val_freq:
            return -1
        self._update_freq(key)
        return self.key_to_val_freq[key][^0]

    def put(self, key: str, value):
        if self.capacity <= 0:
            return
        if key in self.key_to_val_freq:
            self.key_to_val_freq[key][^0] = value
            self._update_freq(key)
            return
        if len(self.key_to_val_freq) >= self.capacity:
            evict_key, _ = self.freq_to_keys[self.min_freq].popitem(last=False)
            if not self.freq_to_keys[self.min_freq]:
                del self.freq_to_keys[self.min_freq]
            del self.key_to_val_freq[evict_key]
        self.key_to_val_freq[key] = [value, 1]
        self.freq_to_keys[key] = None
        self.min_freq = 1
```

*The key insight here is the `min_freq` pointer. It always points to the frequency bucket the next eviction will come from. We reset it to `1` on every new insert (new keys always start at frequency 1).*

***

### Family 4: TTL-Guided Eviction

**`volatile-ttl`**: When memory is full, evict the key whose TTL expires soonest. The logic: *"This key is about to disappear anyway — let's give it a little push."*

```
svgbob
  Key TTL Remaining:

  key:session:A  ←────────────────────────────► 3600s
  key:session:B  ←────────────────────────────► 300s
  key:cache:C    ←────────────────────────────► 250s
  key:rate:D     ←►                              5s

  volatile-ttl evicts: [key:rate:D]
```

*key:rate:D has the shortest remaining TTL — it was about to vanish anyway. `volatile-ttl` identifies it as the most disposable candidate.*

This policy shines when application code **deliberately assigns shorter TTLs to lower-priority data**. If you can encode expiration intent in your TTL values, `volatile-ttl` lets the store respect that signal precisely.

***

## Part 5: The Decision Framework

```
svgbob
  START: What kind of data are you storing?
         |
         +---> [ALL data is cache data, no permanent keys]
         |           |
         |           +---> Recency-biased access?   → allkeys-lru
         |           +---> Frequency-biased access? → allkeys-lfu
         |           +---> No clear pattern?        → allkeys-random
         |
         +---> [Mixed: some permanent, some cache]
         |           |
         |           +---> Protect certain keys from eviction?
         |                 Use volatile-* policies (only evict TTL keys)
         |                 |
         |                 +---> Keys have meaningful TTL? → volatile-ttl
         |                 +---> Recency matters?          → volatile-lru
         |                 +---> Frequency matters?        → volatile-lfu
         |
         +---> [Eviction is unacceptable]
                     |
                     +---> noeviction
```

*Walk this tree based on what your data represents. The left branch handles pure caching; the right handles hybrid stores serving both ephemeral and permanent data side-by-side.*

### Scenario Mapping

| Scenario | Recommended Policy | Why |
| :-- | :-- | :-- |
| Session store + config data together | `volatile-lru` | Protect config (no TTL), evict sessions  |
| Pure CDN edge cache | `allkeys-lru` | All data equally cache-worthy |
| Leaderboard / trending data | `allkeys-lfu` | Popular content stays; cold trends evict |
| Rate limiter (1-minute windows) | `volatile-ttl` | Short-TTL counters go first  |
| Job queue / task store | `noeviction` | Jobs must never silently disappear |
| Feature flag cache | `volatile-lru` | Infrequent updates, LRU protection sufficient |


***

## Part 6: Building a TTL-Aware Cache from Scratch

For an interview, you may be asked to build a cache that combines eviction AND TTL. The efficient approach uses a **min-heap** sorted by expiration timestamp:

```python
import heapq
import time

class TTLCache:
    def __init__(self):
        self.store = {}           # key → (value, expiry_timestamp)
        self.expiry_heap = []     # min-heap: (expiry_timestamp, key)

    def set(self, key: str, value, ttl_seconds: int):
        expiry = time.time() + ttl_seconds
        self.store[key] = (value, expiry)
        heapq.heappush(self.expiry_heap, (expiry, key))

    def get(self, key: str):
        self._purge_expired()
        if key not in self.store:
            return None
        value, expiry = self.store[key]
        if time.time() > expiry:
            del self.store[key]
            return None
        return value

    def _purge_expired(self):
        """Active expiry: sweep expired keys from the heap."""
        now = time.time()
        while self.expiry_heap and self.expiry_heap[^0][^0] <= now:
            expiry, key = heapq.heappop(self.expiry_heap)
            if key in self.store and self.store[key] == expiry:
                del self.store[key]

# Demo
cache = TTLCache()
cache.set("token:alice", "Bearer abc123", ttl_seconds=2)
print(cache.get("token:alice"))   # → "Bearer abc123"
time.sleep(3)
print(cache.get("token:alice"))   # → None (expired)
```


***

## Part 7: TTL + Eviction — The Full Picture

In a production Redis deployment, TTL and eviction work as **complementary layers** — they are not the same thing, and they solve different problems:

```
svgbob
  Memory Lifecycle of a Key in Redis:

  SET key "val" EX 3600
         |
         v
  +---------+-----------+-----------+---------+
  | Key     | Writable  | Memory    | TTL     |
  | Written | Readable  | Healthy   | Running |
  +---------+-----------+-----------+---------+
                              |
              Memory pressure detected
                              |
              +---------------------+
              | Eviction Policy     |
              | (LRU / LFU / TTL)  |
              +---------------------+
                    /           \
          Evicted               Survived
          (gone now)            (stays until
                                 TTL fires naturally)
```

*Two independent forces act on every key: the eviction policy (triggered by memory pressure) and the TTL clock (always running). A key is removed by whichever fires first.*

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Session: has TTL → candidate for volatile-lru eviction
r.set("session:xyz789", '{"user_id": 42}', ex=3600)

# App config: no TTL → protected from volatile-* policies
r.set("config:feature_flags", '{"dark_mode": true}')

# Rate limiter: short TTL → first candidate for volatile-ttl
r.set("rate:user:42:/api/posts", "5", ex=60)
```


***

## Part 8: The FIFO Baseline — A Brief Historical Note

Before LRU became dominant, **FIFO (First In, First Out)** was common. The first key you added is the first to go. FIFO is simple and fast, but critically **oblivious to usage**: it might evict your most-accessed key simply because it was inserted first. This was a well-known problem in 1990s web proxy caches, leading to poor hit rates on popular content. The shift to LRU was a major practical win — and today FIFO is primarily a teaching baseline.

***

## Part 9: Interview War Room

### Coding Round Cheat Sheet

| Problem | Core Pattern | Complexity Target |
| :-- | :-- | :-- |
| Implement LRU Cache | HashMap + Doubly Linked List | O(1) get/put |
| Implement LFU Cache | HashMap + freq-bucketed OrderedDicts | O(1) get/put |
| Design TTL-aware cache | HashMap + Min-Heap | O(log n) per op |
| Thread-safe cache | Locks / Segment locking | O(1) amortized |

### System Design Round — The 5 Pillars

When asked to *design a key-value store* (a common Microsoft staff-level prompt ), structure your answer around these pillars:

1. **Core API**: State `PUT`, `GET`, `DELETE` with expected time complexity upfront
2. **Storage Layer**: In-memory (hash table) vs. disk-backed (LSM-tree). Discuss write/read amplification trade-offs
3. **Eviction Policy**: Name the policy, justify against the stated workload, mention approximate vs. exact LRU trade-offs
4. **Partitioning**: Consistent hashing minimizes key remapping during node adds/removals[^10]
5. **Consistency Model**: Can reads see stale data? Quorum reads/writes for strong consistency

A strong answer opener: *"Given a read-heavy workload with hot-key patterns, I'd use an in-memory hash table with `allkeys-lfu` eviction, approximate LFU with decay, and consistent hashing with virtual nodes for elastic scaling..."*

***

## Part 10: Real-World Nuances

### The Hot Key Problem

A celebrity posts on a social platform — millions of users suddenly request the same Redis key. One node gets crushed. Solutions:

- **Local caching**: Each app server caches the hot key in a small Python `dict` with a short TTL, reducing Redis traffic
- **Key sharding**: Write the value to multiple keys (`post:celeb:id:shard:1`, `:shard:2`, etc.) and randomly read from any shard


### Write Strategies

```
svgbob
  WRITE-THROUGH:        WRITE-AROUND:         WRITE-BACK:

  App → Cache → DB      App → DB only          App → Cache
                         (cache fills on           |
                          next read)            async → DB
                                                (delayed write)

  ✓ Always consistent   ✓ Good for one-off     ✓ Fastest writes
  ✗ Slower writes         bulk writes          ✗ Risk of data loss
```

*Each strategy represents a different trade-off between write latency, consistency, and durability. Write-through is safest; write-back is fastest but risks losing data if the cache crashes before flushing.*

### Memory Fragmentation

When keys of different sizes are written and deleted repeatedly, memory develops small unusable gaps. Redis 4.0 introduced **Active Defragmentation** that compacts memory in real time. Memcached uses a **slab allocator** — pre-allocated fixed-size memory chunks that prevent fragmentation but can waste space if item sizes don't align with slab sizes.

***

## The Mental Model to Carry Forward

Let's return to our chef's countertop tray one final time. By now, you see it's not a simple shelf — it's a precision instrument:

- Every item carries a **TTL** — a freshness label running down continuously
- The tray has a **capacity limit** — the Redis `maxmemory` constraint
- When it fills up, a **policy** decides who leaves — LRU, LFU, volatile-ttl, or others
- Some items are **protected** by having no TTL, guarded by `volatile-*` policies
- Two eviction clocks run simultaneously — proactive background sweeps and passive lazy checks on access

A key-value store is, at its heart, a **managed forgetting machine**. Its job is not to remember everything forever — it's to remember the *right things* long enough, as cheaply as possible, and forget the rest gracefully. Mastering TTL and eviction strategy is mastering the art of selective memory.
