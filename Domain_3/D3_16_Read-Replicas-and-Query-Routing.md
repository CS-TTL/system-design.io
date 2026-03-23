
# Read Replicas and Query Routing

*Scaling reads without sacrificing write consistency*

***

## Bridge: The Library Analogy

Imagine a bustling public library where patrons constantly **borrow** (read) books while librarians **return** and **re‑shelve** (write) them. If every patron had to wait for a librarian to finish shelving before they could browse, the line would stretch out the door. Instead, the library creates **exact copies** of the most popular sections and places them in reading rooms. Patrons grab a copy from the nearest room, while librarians continue updating the master collection in the back office.

In this analogy:

- **Primary database** = the master collection (single source of truth)
- **Read replicas** = the reading‑room copies (read‑only, eventually consistent)
- **Query routing** = the rule that sends patrons to a reading room for borrows and to the back office for returns

Just as the library scales read traffic by adding more copies, a database scales read‑heavy workloads by adding read replicas and intelligently routing queries.

***

## 1. Core Concept: What Is a Read Replica?

A **read replica** is a read‑only copy of a primary database instance that is kept up‑to‑date via asynchronous replication. Writes go to the primary; reads can be served by any replica, reducing the load on the primary and increasing overall read throughput.

### 1.1 Atomic Unit

Start with a single primary‑replica pair:

```
Primary (writes)  -->  Replica (reads)
        ^                 |
        |  async replication (lag)
        +-----------------+
```

* The replica applies changes from the primary’s write‑ahead log (WAL) with a small, measurable lag (typically milliseconds to seconds).
* Applications issue **write** statements (INSERT, UPDATE, DELETE) to the primary and **read** statements (SELECT) to the replica.


### 1.2 Why We Need It

Most web‑scale applications are **read‑heavy** (often 80‑90% reads). A single primary can become a bottleneck because:

- CPU, memory, and I/O are saturated by read queries.
- Lock contention on hot rows increases latency for writes.
- Scaling up (bigger instance) hits cost and hardware limits quickly.

Adding a replica linearly adds read capacity: *N* replicas give roughly *N×* read throughput, while write capacity stays tied to the primary.

***

## 2. Identifying the Limitation: Replication Lag

The simple primary‑replica model works until **replication lag** becomes visible to users. If a replica is behind the primary, a read may return stale data.

### 2.1 The Pain Point

Consider a social‑media feed: a user posts a photo (write) and immediately tries to view it (read). If the read hits a replica that hasn’t yet applied the upload, the photo appears missing—a confusing experience.

### 2.2 The Fix: Consistency‑Aware Routing

We add a **routing policy** that decides, per query, whether to hit the primary or a replica based on the required consistency:


| Consistency Need | Routing Decision |
| :-- | :-- |
| **Strong** (must see own writes) | Primary |
| **Eventual** (can tolerate lag) | Any replica |
| **Bounded staleness** (lag < X ms) | Replica only if its estimated lag ≤ X |

This turns the broken state (unaware reads) into a fix: the router inspects each query (or session) and picks the right endpoint.

***

## 3. Adding Features: From Basic to Modern Routing

### 3.1 Application‑Level Splitting

The simplest fix lives inside the service code. The application maintains two connection pools:

```python
# Pseudo‑code
write_pool  = create_pool(primary_url)
read_pool   = create_pool(replica_urls)

def handle_request():
    if request.is_write():
        conn = write_pool.acquire()
    else:
        conn = read_pool.acquire()
    # execute query …
```

*Pros*: maximum flexibility—custom lag‑aware selection, session pinning, per‑tenant policies.
*Cons*: every service team must re‑implement and maintain the logic, handle failover, and track topology changes.

### 3.2 Proxy‑Based Routing

A **database middleware** (proxy) sits between app and DB, transparently routing writes to the primary and reads to replicas. The app sees a single endpoint; the proxy does the work.

```
+--------+     +-----------+     +----------+
| App    | --->|   Proxy   | --->| Primary  |
|        |     | (router)  |     | (writes) |
+--------+     +-----------+     +----------+
                         |
                         |   +----------+   +----------+
                         +-->| Replica1 |   | Replica2 |
                             +----------+   +----------+
```

Popular implementations:

- **Heimdall Proxy** (AWS/Azure/GCP Marketplace) – provides read/write split with strong consistency options.
- **Pgpool‑II** – load balances SELECT statements across PostgreSQL servers, optionally with connection pooling.
- **AWS RDS Proxy** – can be configured to favor read replicas for SELECT traffic.

*Pros*: centralizes routing logic, handles failover, can add observability (metrics on lag, query mix).
*Cons*: extra hop adds latency; requires careful tuning of health checks and lag thresholds.

### 3.3 DNS‑Based Load Balancing

Instead of a smart proxy, use weighted or latency‑based DNS records (e.g., Amazon Route 53) to distribute read traffic across replicas.

- **Weighted routing**: assign each replica a weight; DNS returns IPs proportionally.
- **Latency routing**: clients are directed to the replica with lowest network latency.

This approach works well when replicas are geographically distributed (cross‑region read replicas).
*Pros*: no extra software layer, leverages existing CDN/DNS infrastructure.
*Cons*: less granular control; difficult to enforce per‑query consistency; DNS TTL changes propagate slowly.

### 3.4 Hybrid \& Advanced Policies

Modern systems combine the above techniques:


| Technique | When to Use |
| :-- | :-- |
| **Session pinning** | After a write, pin reads to primary for a short window (e.g., 200 ms) to guarantee read‑your‑writes. |
| **Lag‑aware selection** | Replica health checks report replication lag; router avoids replicas exceeding a threshold. |
| **Query classification** | INSERTS/UPDATES/DELETES → primary; SELECT → replica; SELECT … FOR UPDATE → primary (needs lock). |
| **Multi‑tenant routing** | Different tenants may have different consistency requirements; router respects per‑tenant flags. |

These patterns appear in frameworks like **DatabaseRouter** shown in the Oneuptime blog and the Azure read‑replica documentation.

***

## 4. Categorical Chunking: Types of Query Routing Strategies

We group strategies by **behavior** rather than by implementation detail.

### 4.1 Consistency‑Driven

- **Strong‑consistency routing**: writes and any read that must see the latest data go to primary.
- **Eventual‑consistency routing**: all reads go to replicas; accept stale data.
- **Bounded‑staleness routing**: reads go to replicas only if estimated lag ≤ threshold.


### 4.2 Traffic‑Splitting

- **Round‑robin**: cycles through replica list for each read.
- **Weighted**: assigns more traffic to higher‑capacity replicas.
- **Least‑connections**: picks replica with fewest active connections.
- **Response‑time based**: directs to replica with lowest recent latency.


### 4.3 Failover‑Aware

- **Health‑check routing**: removes unresponsive replicas from the pool.
- **Cascading‑replica routing**: routes to a replica that itself replicates from another replica (useful for multi‑tier scaling).
- **Geographic routing**: chooses replica in the same region as the user to lower latency.


### 4.4 Operation‑Based

- **Read‑only vs. read‑write splitting**: any transaction containing a write forces primary.
- **Statement‑level vs. transaction‑level**: some routers examine each statement; others look at the whole transaction scope.

***

## 5. Visual Explanation: ASCII \& SVG‑Bobs

Below is an SVG‑Bobb diagram showing a typical three‑replica setup with a proxy router.

```
+--------+     +-------------+     +--------+
|  App   | --->|  Proxy      | --->| Primary|
| (reads/writes) | (router)  | (writes) |
+--------+     +-------------+     +--------+
                         |
            +------------+------------+------------+
            |            |            |            |
    +-------v----+ +------v------+ +------v------+
    | Replica 1  | | Replica 2   | | Replica 3    |
    | (reads)    | | (reads)     | | (reads)      |
    +------------+ +-------------+ +--------------+
```

**How to read this diagram**:

- The **App** talks to a single logical endpoint (the Proxy).
- The Proxy inspects each incoming query: if it’s a write (INSERT/UPDATE/DELETE) or a read that demands strong consistency, it forwards to **Primary**; otherwise it picks a replica using its load‑balancing algorithm (e.g., round‑robin).
- Arrows from Primary to each replica represent the asynchronous replication stream; the slight delay is the replication lag.

***

## 6. Code Walk‑Through: Python Implementation

Let’s look at a practical, production‑ready router that supports lag‑aware, round‑robin replica selection and session pinning.

```python
import time
import random
from typing import List, Tuple
import psycopg2
from psycopg2.extras import RealDictCursor

class ReadReplicaRouter:
    def __init__(
        self,
        primary_dsn: str,
        replica_dsns: List[str],
        max_lag_ms: int = 200,
        pin_after_write_ms: int = 200,
    ):
        self.primary_dsn = primary_dsn
        self.replica_dsns = replica_dsns
        self.max_lag_ms = max_lag_ms
        self.pin_after_write_ms = pin_after_write_ms
        self._last_write_ts = 0.0
        self._replica_index = 0

    # -----------------------------------------------------------------
    # Helper: get current replication lag in seconds from a replica
    # -----------------------------------------------------------------
    def _get_replica_lag(self, dsn: str) -> float:
        try:
            with psycopg2.connect(dsn) as conn:
                with conn.cursor() as cur:
                    # PostgreSQL: pg_last_xact_replay_timestamp() gives replay time
                    cur.execute(
                        "SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))"
                    )
                    lag_sec = cur.fetchone()[^0]
                    return lag_sec * 1000  # to ms
        except Exception:
            return float("inf")  # treat as unhealthy

    # -----------------------------------------------------------------
    # Choose a read connection: respect pinning and lag thresholds
    # -----------------------------------------------------------------
    def _pick_replica(self) -> str:
        now = time.time()
        # If we are within the pin window after a write, force primary
        if now - self._last_write_ts < self.pin_after_write_ms / 1000:
            return self.primary_dsn

        # Filter replicas by lag
        viable = []
        for dsn in self.replica_dsns:
            lag = self._get_replica_lag(dsn)
            if lag <= self.max_lag_ms:
                viable.append((dsn, lag))

        if not viable:
            # Fallback to primary if no replica meets lag requirement
            return self.primary_dsn

        # Round‑robin among viable replicas
        self._replica_index = (self._replica_index + 1) % len(viable)
        return viable[self._replica_index][^0]

    # -----------------------------------------------------------------
    # Public API
    # -----------------------------------------------------------------
    def write(self, sql: str, params: Tuple = ()) -> List[dict]:
        self._last_write_ts = time.time()
        with psycopg2.connect(self.primary_dsn) as conn:
            with conn.cursor(cursor_factory=RealDictCursor) as cur:
                cur.execute(sql, params)
                conn.commit()
                if cur.description:
                    return cur.fetchall()
                return []

    def read(self, sql: str, params: Tuple = ()) -> List[dict]:
        dsn = self._pick_replica()
        with psycopg2.connect(dsn) as conn:
            with conn.cursor(cursor_factory=RealDictCursor) as cur:
                cur.execute(sql, params)
                if cur.description:
                    return cur.fetchall()
                return []

# Usage example
router = ReadReplicaRouter(
    primary_dsn="host=primary.db.internal port=5432 dbname=mydb user=app password=secret",
    replica_dsns=[
        "host=replica1.db.internal port=5432 dbname=mydb user=app password=secret",
        "host=replica2.db.internal port=5432 dbname=mydb user=app password=secret",
        "host=replica3.db.internal port=5432 dbname=mydb user=app password=secret",
    ],
    max_lag_ms=150,
    pin_after_write_ms=200,
)

# Write – goes to primary
router.write(
    "INSERT INTO posts (user_id, content) VALUES (%s, %s)",
    (42, "Hello replicas!")
)

# Read – goes to a lag‑healthy replica (or primary if pinned)
posts = router.read("SELECT * FROM posts WHERE user_id = %s", (42,))
```

**Explanation of the code**

1. **Lag detection** – each replica reports its replay lag via `pg_last_xact_replay_timestamp()`.
2. **Pinning** – after a write, reads are forced to the primary for a configurable window (default 200 ms).
3. **Round‑robin** – among replicas that satisfy the lag threshold, we cycle to spread load evenly.
4. **Fallback** – if no replica meets the lag requirement, we safely route to the primary to avoid serving stale data.

This example illustrates how the “problem‑solution narrative” plays out in code: we start with a naïve router, notice the lag problem, then add pinning and lag‑aware selection.

***

## 7. Trade‑Offs \& Decision Matrix

| Strategy | Complexity | Consistency Control | Operational Overhead | Best Fit |
| :-- | :-- | :-- | :-- | :-- |
| Application‑level | High (per‑service) | Fine‑grained (per‑query) | High (each team maintains) | Custom policies, multi‑tenant SaaS |
| Proxy‑based | Medium (single component) | Medium (router‑level) | Medium (proxy ops, monitoring) | General‑purpose apps wanting transparency |
| DNS‑based | Low (DNS changes) | Low (traffic‑only) | Low (no extra software) | Geo‑distributed read‑only workloads, CDN‑edge |
| Hybrid (proxy + DNS) | Medium‑High | High | Medium‑High | Large scale, strict SLOs, multi‑region |

**Key take‑aways**

- **Never sacrifice write consistency** for read scalability unless the business can tolerate staleness.
- **Monitor replication lag continuously** and expose it via metrics (Prometheus, CloudWatch).
- **Test failover**: promote a replica to primary and verify the router adapts without manual DNS changes.
- **Use connection pooling** at both the router and replica sides to avoid connection storms under load spikes.

***

## 8. Historical Context

The read‑replica pattern emerged in the early 2000s as MySQL replication gained popularity. Early websites (e.g., Slashdot, LiveJournal) used **master‑slave** setups to scale reads. The term “read replica” became mainstream with Amazon RDS (2009) and Google Cloud SQL (2011), which offered managed replication and built‑in read‑endpoint routing.

Over the last decade, the ecosystem matured:

- **2013‑2015**: Introduction of database proxies (MaxScale, ProxySQL) that added query‑routing intelligence.
- **2016‑2018**: Cloud providers added **auto‑scaling read replica pools** (Aurora Read Replicas, Azure Hyperscale).
- **2019‑2022**: Focus on **bounded‑staleness** and **session consistency** APIs (AWS RDS Reader Endpoint with `read_after_write` flag).
- **2023‑Present**: **Machine‑learning‑driven lag prediction** and **adaptive routing** (e.g., Google Cloud Spanner’s read‑selector).

Understanding this history helps us appreciate why modern routers combine lag‑awareness, pinning, and multi‑tier replication.

***

## 9. Best‑Practice Checklist

When designing a read‑replica + query‑routing system, run through this checklist:

- [ ] **Measure your read/write ratio** – if reads < 60%, the added complexity may not pay off.
- [ ] **Quantify acceptable staleness** – define lag SLAs per business transaction (e.g., “feed may be 2 s stale”).
- [ ] **Implement lag monitoring** – expose replica lag as a metric and alert on spikes.
- [ ] **Choose routing layer** – start with application‑level if you need custom policies; otherwise adopt a managed proxy.
- [ ] **Test failure scenarios** – kill a replica, promote a replica, network partition; verify router behaves correctly.
- [ ] **Document consistency guarantees** – make it clear to API consumers which calls are strong vs. eventual.
- [ ] **Plan for cascading replicas** – if you need > 5‑10 replicas, consider a multi‑tier tree to reduce primary‑to‑replica traffic.
- [ ] **Review cost** – each replica incurs storage and compute; use auto‑scaling to shrink during off‑peak hours.

***

## 10. Conclusion

Read replicas turn a single‑node write bottleneck into a horizontally scalable read layer, while query routing decides *where* each operation lands to balance performance and consistency.

We began with a simple library analogy, identified the core problem (replication lag causing stale reads), and iteratively added features: application‑level splitting, proxy‑based routing, DNS load‑balancing, and advanced policies like session pinning and lag‑aware selection.

By chunking strategies by behavior—consistency‑driven, traffic‑splitting, failover‑aware, operation‑based—we can pick the right tool for the job. Visual diagrams and a concrete Python router illustrate how the ideas become code.

Finally, a historical perspective and a best‑practice checklist equip you to evaluate, implement, and operate a read‑replica system that meets the scalability demands of modern applications without surprising your users with out‑of‑date data.

*Remember*: the goal isn’t merely to add more copies; it’s to **route wisely**, so every user gets the data they need, when they need it, while the system stays responsive under load.

---
