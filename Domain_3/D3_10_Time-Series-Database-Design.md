
# Time-Series Database Design

> *"We don't just store data. We store the story of how things change."*

***

## The Weather Station Analogy: A Bridge Into Time

Imagine you are responsible for a network of weather stations scattered across a continent. Every single second, each station sends you a reading: temperature, humidity, wind speed, barometric pressure. By the end of a single day, one station alone has generated 86,400 data points per sensor. Multiply that by 10,000 stations, and you have nearly **a billion records per day** — before you've even looked at the data.

Now imagine trying to answer questions like:

- *"What was the average temperature in Chicago between 2 PM and 4 PM last Tuesday?"*
- *"Show me the wind speed trend for the last 30 days, aggregated by hour."*
- *"Alert me when any station's reading drops more than 5 degrees in under 60 seconds."*

A traditional relational database — the kind we use for storing user accounts or financial transactions — would collapse under this workload. Not because it's bad software, but because it was designed for a fundamentally different problem. This is the core intuition behind **Time-Series Databases (TSDBs)**. They are not databases with timestamps bolted on. They are architectural rethinks, built from the ground up around the assumption that **time is the primary axis of every query**.

***

## Chapter 1: The Atomic Unit — What Is Time-Series Data?

### The Data Point

The most primitive building block is the **data point**: a value observed at a specific moment in time.

```python
# The simplest possible time-series data point
data_point = {
    "timestamp": 1711051200,   # Unix epoch (seconds since Jan 1, 1970)
    "metric": "cpu.usage",
    "value": 72.4              # percentage
}
```

Now let's add reality. A data point doesn't arrive alone — it arrives as part of a **series**: an ordered stream of measurements from a single source, for a single metric, over time.

```python
cpu_series = [
    {"timestamp": 1711051200, "host": "web-01", "value": 72.4},
    {"timestamp": 1711051210, "host": "web-01", "value": 73.1},
    {"timestamp": 1711051220, "host": "web-01", "value": 74.8},
    {"timestamp": 1711051230, "host": "web-01", "value": 74.9},
]
```

Notice something critical: `host` never changes between rows. Only `timestamp` and `value` change. This observation is the **secret weapon** we'll use to compress data by 10–40x.

### The Structural Vocabulary

The distinction between **tags** and **fields** is one of the most common schema mistakes candidates make in interviews:

- **Tags** → metadata that *identify* the series; low-cardinality, indexed, used in `WHERE` clauses. Examples: `host`, `region`, `service`.
- **Fields** → the *measured values*; they change with every data point and are **not** indexed. Examples: `cpu_usage`, `temperature`, `latency_ms`.

| Concept | InfluxDB Term | TimescaleDB Term | Prometheus Term |
| :-- | :-- | :-- | :-- |
| Named stream | Measurement | Hypertable | Metric |
| Static label | Tag | Label column | Label |
| Changing val | Field | Value column | Sample value |
| Sample time | Timestamp | `time` column | Timestamp |


***

## Chapter 2: Why Traditional Databases Fail Here

### The Pain Point: Random Writes Kill Performance

A B-tree — the index structure used by virtually every RDBMS — keeps data **sorted and balanced at all times**. Every insert must find the correct leaf node and place the record there. If the page is full, it splits the node, cascading updates upward. This is called **write amplification**, and it is devastating for time-series workloads writing 100,000+ data points per second.

The benchmark numbers tell the story: a well-tuned PostgreSQL handles ~100,000 inserts/second under optimal conditions. InfluxDB routinely sustains **1–2 million writes per second** on equivalent hardware.

The fix? We need a structure that turns **random writes into sequential writes**.

***

## Chapter 3: The Write Engine — LSM Trees and the TSM

### Log-Structured Merge Trees (LSM Trees)

The LSM tree (O'Neil et al., 1996) is the foundational data structure powering most modern TSDBs. The core insight is elegant: **don't organize data as you write it. Write fast, organize later**.

**Step 1 — The MemTable (In-Memory Buffer)**

All incoming writes first land in an in-memory sorted structure. This is extremely fast — it's all RAM.

```
Incoming writes (as they arrive):
  t=1711051240, cpu=73.3
  t=1711051200, cpu=72.4   ← arrived out of order!

MemTable (auto-sorted in memory):
  t=1711051200, cpu=72.4
  t=1711051240, cpu=73.3
```

**Step 2 — Flush to SSTables**

When the MemTable hits a size threshold, it's flushed to disk as an immutable **SSTable** (Sorted String Table). This flush is a single sequential write — the fastest possible disk operation.

**Step 3 — Background Compaction**

Over time, multiple SSTables accumulate. A background process merges them, removes duplicates, and produces larger, more efficient files.

```svgbob
                      Time-Series Write Path
 ─────────────────────────────────────────────────────────────

  Client
    │
    │  write(t=T, v=V)
    ▼
 ┌──────────┐    append    ┌────────────────┐
 │  WAL Log │─────────────►│   MemTable     │  (In-Memory)
 │(durability)│            │  sorted buffer │
 └──────────┘             └────────┬───────┘
                                   │  flush (threshold reached)
                                   ▼
                          ┌────────────────┐
                          │  SSTable L0    │  (Immutable on disk)
                          └───────┬────────┘
                                  │
                    ┌─────────────┼────────────┐
                    │  Background Compaction    │
                    │  (merge + sort + compress)│
                    └─────────────┬────────────┘
                                  │
                          ┌───────▼────────┐
                          │  SSTable L1    │
                          └───────┬────────┘
                                  │
                          ┌───────▼────────┐
                          │  SSTable L2    │  (Longest retention tier)
                          └────────────────┘
```

*Figure 1: The LSM write path. Writes are always sequential. Compaction is asynchronous. This separation is what enables the extreme write throughput.*

### InfluxDB's TSM: LSM Trees Tailored for Time

InfluxDB adapted the LSM concept into its **Time-Structured Merge Tree (TSM)**. The key modification: TSM files are organized first by **series key**, then by **time**. A TSM file contains an Index Block (series keys → byte offsets), Data Blocks (timestamps and values stored separately in columnar form), and a Footer.

```
TSM File Layout:
┌──────────────────────────────────────────────────┐
│  Data Block: cpu,host=web-01 | t=100..200        │
│  Data Block: cpu,host=web-01 | t=200..300        │
│  Data Block: mem,host=web-01 | t=100..200        │
│  ...                                              │
│  Index: [series_key → block offsets]             │
│  Footer                                           │
└──────────────────────────────────────────────────┘
```

This layout means a time-range query touches exactly the relevant data blocks — no unrelated series are scanned.

***

## Chapter 4: The Compression Engine — Making Data Tiny

### The Pain Point: Raw Time-Series Data Is Deeply Redundant

If a temperature sensor reads 22.1°C, 22.2°C, 22.1°C, 22.3°C — we're storing nearly identical numbers over and over. We need compression that **exploits temporal structure**.

### Delta-of-Delta Encoding for Timestamps

Instead of storing raw 64-bit Unix timestamps, we store differences between consecutive timestamps:

```python
def delta_of_delta_encode(timestamps):
    result = [timestamps[^0]]          # First timestamp stored raw
    first_delta = timestamps - timestamps[^0]
    result.append(first_delta)
    for i in range(2, len(timestamps)):
        delta = timestamps[i] - timestamps[i-1]
        dod = delta - first_delta     # Delta of delta — often 0!
        result.append(dod)
    return result

ts = [1711051200, 1711051210, 1711051220, 1711051230, 1711051240]
print(delta_of_delta_encode(ts))
# [1711051200, 10, 0, 0, 0]
# Four zeros instead of four 10-digit numbers!
```


### XOR Encoding for Floats (Gorilla Compression)

Facebook's **Gorilla paper (2015)** introduced a brilliant approach: XOR two consecutive floats. Since they're usually close in value, they share most of their bits — and the XOR reveals only the *changed* bits.

```python
import struct

def float_to_bits(f):
    return struct.unpack('Q', struct.pack('d', f))[^0]

bits1 = float_to_bits(72.4)
bits2 = float_to_bits(72.5)
xor   = bits1 ^ bits2

# Most bits are identical (0 in XOR) — only the
# changing window of bits needs to be stored.
print(f"Meaningful bits in XOR: {xor.bit_length()}")  # Much smaller than 64
```

This is why Gorilla compression achieves **10:1 to 40:1 compression ratios** on real-world metrics data.

```
  Data Type       Technique                   Typical Ratio
  ─────────────────────────────────────────────────────────
  Timestamps  ──► Delta-of-Delta + Zigzag       ~20:1
  Integers    ──► Simple8b / RLE                ~10:1
  Floats      ──► XOR / Gorilla                 ~12:1
  Strings     ──► Dictionary encoding            ~5:1
  Booleans    ──► Run-Length Encoding            ~32:1
```


***

## Chapter 5: Partitioning — Divide and Conquer Time

### The Pain Point: A Single Table Does Not Scale

Even with perfect compression, storing billions of data points in one logical table means every query — even one for yesterday's data — could scan the entire dataset. We need to **physically isolate data by time**.

### Time-Based Partitioning (Sharding)

The solution is **time-based partitioning**: each chunk covers a fixed time window. InfluxDB calls these *shards*; TimescaleDB calls them *chunks*.

```svgbob
  Data Timeline:
  ──────────────────────────────────────────────────────
    Jan 1       Jan 8       Jan 15      Jan 22
      │           │           │           │
      ├──Shard 1──┤──Shard 2──┤──Shard 3──┤──Shard 4──►
      │  (7 days) │  (7 days) │  (7 days) │  (current)
```

The benefits are substantial:

1. **Write locality** — all current writes land in the newest shard, no contention with historical shards.
2. **Trivial TTL** — to delete data older than 30 days, drop Shard 1. No expensive row-by-row `DELETE`.
3. **Query pruning** — a "last 24 hours" query doesn't open older shards at all.
4. **Independent compression** — older shards can be compressed more aggressively since they're immutable.
```python
# TimescaleDB: Creating a partitioned hypertable
cur.execute("""
    CREATE TABLE system_metrics (
        time        TIMESTAMPTZ      NOT NULL,
        host        TEXT             NOT NULL,
        region      TEXT             NOT NULL,
        cpu_usage   DOUBLE PRECISION,
        memory_mb   DOUBLE PRECISION
    );
""")

cur.execute("""
    SELECT create_hypertable(
        'system_metrics', 'time',
        chunk_time_interval => INTERVAL '1 day'
    );
""")

# Automatically compress chunks older than 7 days
cur.execute("""
    ALTER TABLE system_metrics SET (
        timescaledb.compress,
        timescaledb.compress_segmentby = 'host, region'
    );
    SELECT add_compression_policy('system_metrics', INTERVAL '7 days');
""")
```

```svgbob
  Query: SELECT avg(cpu_usage) FROM metrics
         WHERE time > NOW() - INTERVAL '2 days'
         AND host = 'web-01';

  Partition Map:
  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
  │  Chunk-1    │   │  Chunk-2    │   │  Chunk-3    │
  │ Jan 19-20   │   │ Jan 21-22   │   │ Jan 23-24   │
  │  (old)      │   │  (recent)   │   │  (current)  │
  └─────────────┘   └─────────────┘   └─────────────┘
         ✗                 ✓                 ✓
     Pruned by         Scanned           Scanned
     query planner    (in range)        (in range)
```

*Figure 2: Partition pruning in action. Chunk-1 is eliminated entirely by the query planner — zero disk I/O is wasted on out-of-range data.*

***

## Chapter 6: Indexing — Finding Needles Without Scanning Haystacks

### Bloom Filters: The Probabilistic Bouncer

Even with time partitioning, a chunk might contain hundreds of SSTables. Reading each to check if it contains a given series would be expensive. **Bloom filters** solve this:

```python
class BloomFilter:
    """
    Answers: 'Is series_key in this SSTable?'
    False negatives:  NEVER  (if it says no → definitely absent)
    False positives:  possible but tunable (e.g., 1% rate)
    """
    def might_contain(self, series_key: str) -> bool:
        # Uses k hash functions + bit array
        # If ANY bit position is 0 → definitely not present
        # If ALL positions are 1 → probably present
        ...

def query_sstables(series_key, sstables):
    # Only SSTables that pass the Bloom filter get disk reads
    return [sst for sst in sstables
            if sst.bloom_filter.might_contain(series_key)]
```

In practice, Bloom filters eliminate **90%+ of unnecessary SSTable reads**, dramatically improving query latency for sparse series.

### Inverted Index for Tag Queries

`WHERE region = 'us-east-1' AND environment = 'production'` needs to quickly find all matching series. An **inverted index** maps each tag value to the set of series keys carrying that tag:

```
"region=us-east-1"  → {cpu,host=web-01, cpu,host=web-02, ...}
"env=production"    → {cpu,host=web-01, cpu,host=web-05, ...}

Query: region=us-east-1 AND env=production
→ Set intersection → {web-01}
→ Only read data for web-01
```

This is why **high-cardinality tags are catastrophic** — with 10 million unique user IDs as tags, the inverted index consumes enormous memory and the set intersections become prohibitively slow.

***

## Chapter 7: Data Lifecycle Management

### The Aging Pipeline

Your dashboard needs 1-second granularity for the last hour. For last year's trend, hourly averages are perfectly sufficient. Storing a full year of 1-second data is wasteful and expensive — the solution is a **tiered data lifecycle**:

```svgbob
  Data Lifecycle Pipeline:

  Hot Tier (SSD)           Warm Tier (HDD)       Cold Tier (Object Store)
  ─────────────────────────────────────────────────────────────────────────

  Raw: 1s resolution  ──►  5min rollups      ──►  1hr rollups
  Retention: 7 days        Retention: 90 days     Retention: 2 years
  Latency: <10ms           Latency: <100ms         Latency: seconds

       │                        │                        │
       ▼ auto-expire            ▼ auto-expire            ▼ auto-expire
    Deleted                  Deleted                  Deleted
```


### Continuous Aggregates: Pre-Computing the Future

Rather than computing rollups on-demand, modern TSDBs maintain **continuously refreshed materialized views**:

```python
# TimescaleDB: An always-fresh hourly average view
cur.execute("""
    CREATE MATERIALIZED VIEW cpu_hourly_avg
    WITH (timescaledb.continuous) AS
    SELECT
        time_bucket('1 hour', time) AS bucket,
        host,
        region,
        avg(cpu_usage)   AS avg_cpu,
        max(cpu_usage)   AS max_cpu,
        min(cpu_usage)   AS min_cpu,
        count(*)         AS sample_count
    FROM system_metrics
    GROUP BY bucket, host, region
    WITH NO DATA;
""")

cur.execute("""
    SELECT add_continuous_aggregate_policy(
        'cpu_hourly_avg',
        start_offset      => INTERVAL '3 hours',
        end_offset        => INTERVAL '1 hour',
        schedule_interval => INTERVAL '1 hour'
    );
""")
```


***

## Chapter 8: Schema Design Patterns

### The High-Cardinality Anti-Pattern

This is the single most common TSDB schema mistake, and it can bring a production system to its knees:

```python
# ❌ WRONG: Using user_id as a tag
bad_point = {
    "measurement": "api_requests",
    "tags": {"user_id": "usr_4f8b2c1d9e3a"},  # Millions of unique users!
    "fields": {"response_time_ms": 145},
    "timestamp": 1711051200000000000
}
# With 10M users → 10M unique series keys
# → Inverted index explodes → Compaction stalls → System fails

# ✅ CORRECT: High-cardinality values go in fields
good_point = {
    "measurement": "api_requests",
    "tags": {
        "endpoint":    "/api/v2/checkout",    # Low cardinality ✓
        "region":      "us-east-1",           # Low cardinality ✓
        "status":      "200"                  # Low cardinality ✓
    },
    "fields": {
        "response_time_ms": 145,
        "user_id": "usr_4f8b2c1d9e3a"         # High cardinality → field ✓
    },
    "timestamp": 1711051200000000000
}
```

The rule of thumb: if a tag value has more than ~100,000 unique values across your entire dataset, make it a field instead.

***

## Chapter 9: Query Patterns and Optimization

### Filter by Time First — Always

Effective TSDB queries follow a consistent pattern: **time filter → tag filter → aggregate**. This ordering maps to how data is physically organized on disk:

```python
from influxdb_client import InfluxDBClient

client = InfluxDBClient(url="http://localhost:8086", token="my-token", org="myorg")
query_api = client.query_api()

# ✅ GOOD: Time first → tag filter → aggregation
flux = """
from(bucket: "system_metrics")
    |> range(start: -1h)                              // 1. Time filter FIRST
    |> filter(fn: (r) =>
        r._measurement == "system_metrics" and
        r.host == "web-01" and
        r.region == "us-east-1"                       // 2. Tag filter SECOND
    )
    |> filter(fn: (r) => r._field == "cpu_usage")
    |> aggregateWindow(every: 5m, fn: mean)           // 3. Aggregate LAST
    |> yield(name: "mean_cpu")
"""

# ❌ BAD: No time filter = full table scan across ALL time
bad_flux = """
from(bucket: "system_metrics")
    |> filter(fn: (r) => r.host == "web-01")   // No range()! Never do this.
"""
```


### Window Functions: Trend Analysis in SQL

```python
# Detecting rapid CPU spikes with LAG() window function
cur.execute("""
    SELECT
        time, host, cpu_usage,
        cpu_usage - LAG(cpu_usage) OVER (
            PARTITION BY host ORDER BY time
        ) AS cpu_delta,
        CASE
            WHEN cpu_usage - LAG(cpu_usage) OVER (
                PARTITION BY host ORDER BY time
            ) > 20 THEN 'SPIKE_ALERT'
            ELSE 'NORMAL'
        END AS status
    FROM system_metrics
    WHERE time > NOW() - INTERVAL '1 hour'
      AND host = 'web-01'
    ORDER BY time DESC;
""")
```


***

## Chapter 10: Distributed Architecture

### Sharding + Replication Together

In a distributed TSDB, the sharding key is always **time + series key**. This keeps all data for a given series in a given time window on the same node, minimizing cross-node reads for range queries.

```svgbob
  Distributed TSDB Cluster:

                    ┌─────────────┐
                    │  Ingestion  │
                    │   Gateway   │
                    └──────┬──────┘
                           │ hash(series_key) % N_shards
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
      ┌──────────┐  ┌──────────┐  ┌──────────┐
      │  Node 1  │  │  Node 2  │  │  Node 3  │
      │ Shards   │  │ Shards   │  │ Shards   │
      │ A, D, G  │  │ B, E, H  │  │ C, F, I  │
      └────┬─────┘  └────┬─────┘  └────┬─────┘
           └──────── Async Replication ─┘
                  (Replication Factor = 2)
```

*Figure 3: A 3-node cluster with RF=2. The gateway routes writes via series-key hash. Each shard has a primary and one async replica. Query nodes fan out to all shards and merge results.*

TSDBs typically favor **AP (Available + Partition Tolerant)** over strong consistency — losing a few metrics during a network partition is far preferable to a monitoring system going completely dark.

***

## Chapter 11: System Design Interview Walkthrough

### Prompt: "Design a Real-Time Metrics Platform for 1M Servers"

**Step 1: Capacity Estimation**

```python
servers         = 1_000_000
metrics_per_srv = 10          # cpu, mem, disk, net_in, net_out...
sample_rate_s   = 10          # one sample every 10 seconds
bytes_per_point = 16          # compressed (timestamp + value)

writes_per_sec     = servers * metrics_per_srv / sample_rate_s
storage_per_day_gb = writes_per_sec * 86400 * bytes_per_point / 1e9
storage_90_days_tb = storage_per_day_gb * 90 / 1024

# Writes/second:      1,000,000
# Storage/day:        ~1,382 GB
# Storage (90 days):  ~121 TB
```

**Step 2: Full Architecture**

```svgbob
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Server 1 │  │ Server 2 │  │  ...     │  (1M servers)
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       └──────────────▼─────────────┘
                 Metrics Agents (Telegraf)
                        │
                        ▼
             ┌──────────────────────┐
             │    Kafka Cluster      │
             │  (buffering + replay) │
             └──────────┬───────────┘
                        │
                ┌───────▼────────┐
                │  Ingest Workers │  (batch 10K points/flush)
                └───────┬────────┘
                        ▼
             ┌──────────────────────┐
             │    TSDB Cluster       │
             │  3-node, RF=2         │
             │  + Continuous Aggr.   │
             │  + Retention Tiers    │
             └──────────┬───────────┘
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
     ┌──────────┐  ┌──────────┐ ┌──────────┐
     │ Grafana  │  │ Query API│ │  Alert   │
     │Dashboard │  │(history) │ │  Engine  │
     └──────────┘  └──────────┘ └──────────┘
```

**Step 3: Key Design Decisions**

1. **Write path**: Agents → Kafka (backpressure buffer) → Ingest Workers (batching) → TSDB. Never write directly from 1M agents.
2. **Batching**: Workers accumulate 1,000–10,000 points before flushing, converting millions of tiny writes into thousands of larger sequential ones.
3. **Retention tiers**: Raw 10s data for 7 days → 1-min rollups for 30 days → 1-hr rollups for 90 days → 1-day rollups indefinitely.
4. **Tag design**: `host`, `region`, `environment`, `service` as tags. `user_id`, `request_id` as fields.
5. **Query routing**: Dashboards query continuous aggregates by default. Raw data tables only for drill-downs.

***

## Chapter 12: Interview Quick Reference

### Concepts Every Candidate Must Know

| Concept | One-Line Explanation |
| :-- | :-- |
| LSM Tree | Write to memory, flush sequentially, compact in background |
| TSM (InfluxDB) | LSM variant: files organized by series key then time |
| Delta-of-delta encoding | Compress timestamps by storing differences of differences |
| Gorilla compression | Compress floats via XOR of consecutive values |
| Time partitioning | Physically divide data by time windows for write isolation \& TTL |
| Cardinality explosion | High-cardinality tags blow up the inverted index |
| Continuous aggregates | Pre-computed, auto-refreshing rollup views |
| Bloom filter | Skip SSTables that definitely don't contain queried series |
| Downsampling | Replace raw points with summaries for older time ranges |
| Write amplification | The B-tree penalty — one logical write causes many physical writes |

### The 3-Question Framework

When faced with any TSDB interview problem, always ask:

1. **What is the write rate?** → Determines Kafka buffering needs and cluster sizing.
2. **What is the query access pattern?** → Real-time vs. historical drives retention tier design.
3. **What are your cardinality constraints?** → Tag design choices are irreversible at scale.

***

## Closing Thought

Time-series databases are one of the most elegant examples of how understanding your data's **natural structure** unlocks orders-of-magnitude performance improvements. The timestamps are nearly regular → exploit that for delta encoding. Values change slowly → exploit that for XOR compression. Writes always move forward in time → exploit that with append-only LSM trees. Old data is queried less → exploit that with tiered storage.

Every design decision in a TSDB flows from a single observation: **time is not just a column — it is the organizing principle of the entire system.** When you internalize that, the architecture stops looking like a collection of arbitrary tricks and starts looking like the only logical conclusion.