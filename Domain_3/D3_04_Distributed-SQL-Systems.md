
# Distributed SQL Systems
### *From a Single Cash Register to a Global Bank — How SQL Learned to Scale*

---

## The Bridge: A Bank with One Teller

Imagine a bank. Not a modern one — an old-fashioned one from the 1950s, with a
single teller window. Every customer who walks in must wait in a single line. The
teller handles one transaction at a time: deposits, withdrawals, transfers. The
system is simple, consistent, and easy to reason about. There is only one ledger,
one authority, one source of truth.

This is your traditional relational database — a single-node SQL system. It is
elegant. It works beautifully for small towns.

Now imagine that bank needs to serve an entire continent. Customers in São Paulo,
Tokyo, Frankfurt, and San Francisco all need to make transactions simultaneously —
and they all need to see each other's balances in real time, with zero tolerance
for an account being double-spent. One teller window is not just slow anymore.
It is a structural impossibility.

The naive solution is to open more bank branches. But now you have a new problem:
**how do the branches stay in sync?** If a customer deposits money in Tokyo, can
someone in Frankfurt immediately withdraw it? What happens if the phone line
between branches goes down mid-transaction?

This is the fundamental problem that **Distributed SQL systems** were invented to
solve. We want the *familiarity and correctness* of SQL (one ledger, ACID
guarantees) with the *scale and resilience* of distributed systems (many branches,
no single point of failure). Getting both at the same time is hard — and
understanding *why* it is hard is where our journey begins.

---

## Part 1: The Foundation — What Made Traditional SQL Great (and Limited)

### The Single-Node Paradise

Before we break things apart, we need to appreciate what we had. A traditional
RDBMS like PostgreSQL or MySQL running on a single powerful server gives us a
beautiful set of guarantees, known by the acronym **ACID**:

- **Atomicity** — A transaction either fully happens or it does not. No partial
  states.
- **Consistency** — Every transaction leaves the database in a valid state.
- **Isolation** — Concurrent transactions behave as if they ran sequentially.
- **Durability** — Once committed, data survives crashes.

These guarantees are what made SQL the backbone of banking, e-commerce,
healthcare, and virtually every serious business application since the 1970s.
They are not just nice-to-haves — they are the mathematical contract your
application depends on.

```

Traditional Single-Node Architecture
─────────────────────────────────────

     [ Application Servers ]
            │
            ▼
    ┌───────────────┐
    │  SQL Database │  ← One machine holds everything
    │               │
    │  ┌─────────┐  │
    │  │ Storage │  │  ← All data lives here
    │  └─────────┘  │
    └───────────────┘
    Simple. Consistent. But physically limited
to the capacity of ONE machine.

```

### The Breaking Point: Vertical Scaling's Ceiling

The first instinct when a database slows down is to buy a bigger machine — more
RAM, faster CPUs, better SSDs. This is called **vertical scaling** (scaling *up*).
For decades, it worked. Moore's Law kept delivering more powerful hardware every
18 months.

But vertical scaling has a hard ceiling. At some point, there is no bigger machine
to buy. And even if there were, a single machine means a single point of failure.
When that machine goes down — hardware failure, OS crash, datacenter outage —
your entire application goes offline.

The internet era brought two forces that shattered the single-node model:
1. **Data volume** grew beyond what any single machine could hold or process.
2. **Global users** demanded low-latency access regardless of geography.

The first response from the industry was not pretty.

---

## Part 2: The Dark Ages — Manual Sharding and Its Scars

### The Desperate Solution: Application-Level Sharding

When companies like early Facebook, Twitter, and e-commerce giants hit the
database ceiling, they did what engineers do: they hacked around the problem.
The approach became known as **sharding** — manually partitioning data across
multiple database instances.

The idea is simple: if you have 100 million users, put users 0–33M on Server A,
33M–66M on Server B, and 66M–100M on Server C. Each server holds a "shard."

```

Application-Level Sharding
───────────────────────────

     [ Application Logic ]
     (must know shard routing)
         │    │    │
         ▼    ▼    ▼
    ┌──────┐ ┌──────┐ ┌──────┐
    │ DB-A │ │ DB-B │ │ DB-C │
    │Users │ │Users │ │Users │
    │ 0-33M│ │33-66M│ │66-100M│
    └──────┘ └──────┘ └──────┘
    The application must know which shard to query.
Databases know nothing about each other.
Cross-shard JOINs happen in application code.

```

This works — until it does not. The problems are severe:

- **Cross-shard queries are nightmares.** Want to JOIN users with orders when they
  live on different shards? You write application code to stitch results manually.
- **Re-sharding is catastrophic.** When Server A fills up, moving data requires
  orchestrated migrations that take hours or days and risk downtime.
- **No cross-shard transactions.** ACID guarantees evaporate the moment a
  transaction touches two shards.
- **Schema changes require coordination.** Alter a table and you must run it
  against every shard independently.

> *This was the broken state of the world circa 2005–2012. Something had to give.*

---

## Part 3: The NoSQL Interlude — Horizontal Scale Without SQL

### Trading Guarantees for Scale

Around 2007–2012, a new class of databases emerged under the banner of **NoSQL**:
Cassandra, MongoDB, DynamoDB, HBase. The philosophy was radical — abandon ACID,
abandon SQL, gain infinite horizontal scale.

These systems were genuinely groundbreaking. Cassandra could write data across
dozens of nodes with remarkable throughput. DynamoDB could handle millions of
requests per second. The tradeoff was **eventual consistency** — your data would
*eventually* be correct, but replicas could diverge in the meantime.

For certain workloads — user activity feeds, shopping carts, sensor telemetry —
this was acceptable. But for financial transactions, inventory management, or
anything where a wrong answer is worse than a slow answer, eventual consistency
was a non-starter.

```

The Spectrum of Database Tradeoffs
────────────────────────────────────

Strong Consistency ◄──────────────────────────► Availability \& Scale
│                                                  │
┌────┴────┐        ┌──────────────┐       ┌────────────┴──────┐
│ Single  │        │ Distributed  │       │   NoSQL           │
│ Node SQL│        │     SQL      │       │ (Cassandra, Mongo)│
└─────────┘        └──────────────┘       └───────────────────┘
One server      Best of both worlds      Eventual consistency
ACID perfect    ACID + scale             High throughput

Distributed SQL occupies the hardest middle position.

```

The industry eventually realized: we traded too much. Business logic written for
strong consistency is nearly impossible to retrofit for eventual consistency.

A new question emerged: *Can we build a database that has the scalability of NoSQL
and the guarantees of SQL?*

---

## Part 4: The Birth of Distributed SQL — Google Spanner Lights the Way

### The 2012 Paper That Changed Everything

In 2012, Google published a landmark paper: **"Spanner: Google's Globally
Distributed Database."** It described a system that:

- Ran SQL queries with full ACID transactions
- Spanned hundreds of datacenters globally
- Guaranteed external consistency (stronger than serializability)
- Handled petabytes of data across millions of servers

The trick Google used was audacious: they built **TrueTime** — a globally
synchronized clock API using GPS receivers and atomic clocks in every datacenter.
This allowed Spanner to assign globally consistent timestamps to transactions,
enabling distributed ACID without traditional coordination bottlenecks.

Most companies cannot deploy atomic clocks. But Google's paper proved the
*concept* was achievable and inspired a generation of open-source systems:

- **CockroachDB** (2015)
- **TiDB** (2015, by PingCAP)
- **YugabyteDB** (2016)
- **Amazon Aurora DSQL** (2024)

These systems replaced TrueTime with probabilistic clock uncertainty bounds and
consensus protocols — making the technology accessible without GPS hardware.

---

## Part 5: The Architecture of a Distributed SQL System

### The Three-Layer Mental Model

Modern Distributed SQL systems share a common architectural pattern. Think of it
like a well-organized company with three distinct departments, each with a single
responsibility:

```

Distributed SQL — Layered Architecture
────────────────────────────────────────

┌─────────────────────────────────────────┐
│         SQL / Query Layer               │  ← "The Interpreter"
│  Parse SQL → Plan → Optimize → Execute  │
│  Speaks standard SQL to the outside     │
└──────────────────┬──────────────────────┘
│
┌──────────────────▼──────────────────────┐
│       Transaction Layer                 │  ← "The Referee"
│  MVCC, Distributed Txn Coordinator,     │
│  Timestamp Management, Deadlock Detect  │
└──────────────────┬──────────────────────┘
│
┌──────────────────▼──────────────────────┐
│    Distributed Storage Layer            │  ← "The Warehouse"
│  Key-Value Store, Range Sharding,       │
│  Raft Consensus Groups, RocksDB         │
└─────────────────────────────────────────┘

Each layer has clean interfaces to the one above.
The SQL layer does not care HOW data is stored.
The storage layer does not care about SQL syntax.

```

---

### Layer 1: The Storage Layer — Sharding Without the Pain

The foundation of a Distributed SQL system is a distributed key-value store. All
data — rows, indexes, metadata — is encoded as key-value pairs and distributed
across nodes using **range-based sharding**.

```python
def encode_row_as_kv(table_id: str, primary_key: tuple, columns: dict) -> dict:
    """
    In Distributed SQL, every row is stored as a key-value pair.
    Key encodes the table + primary key; value encodes all columns.
    """
    key = f"/{table_id}/" + "/".join(str(k) for k in primary_key)
    value = {col: val for col, val in columns.items()}
    return {"key": key, "value": value}


row = encode_row_as_kv(
    table_id="users",
    primary_key=(1001,),
    columns={"name": "Alice", "email": "alice@example.com", "balance": 5000}
)
# key:   "/users/1001"
# value: {"name": "Alice", "email": "alice@example.com", "balance": 5000}
```

The system automatically splits and merges ranges as data grows or shrinks.
When a range exceeds ~64MB, it splits into two. The application never sees
this happen.

```
Range-Based Auto-Sharding Over Time
──────────────────────────────────────

Initial state:
  [ /users/1 ──────────────────────── /users/1,000,000 ]
                    Node 1

After growth — automatic split:
  [ /users/1 ──── /users/500,000 ]  [ /users/500,001 ── /users/1,000,000 ]
          Node 1                              Node 2

After more growth:
  [ 1─250k ]   [ 250k─500k ]   [ 500k─750k ]   [ 750k─1M ]
   Node 1        Node 2          Node 3           Node 4

  No application code changes needed.
  The cluster reorganizes itself transparently.
```


#### Replication via the Raft Consensus Protocol

Each range has a **Raft group** — typically 3 or 5 replicas spread across
different availability zones. One replica is the **Leader**; the others are
**Followers**.

```
Raft Group for a Single Range
───────────────────────────────

        ┌─────────────┐
        │   LEADER    │  ← Accepts all writes for this range
        │   (Node 1)  │
        └──────┬──────┘
               │  Replicates log entries
       ┌───────┴────────┐
       ▼                ▼
 ┌──────────┐     ┌──────────┐
 │ FOLLOWER │     │ FOLLOWER │
 │ (Node 2) │     │ (Node 3) │
 └──────────┘     └──────────┘

 Write commits only after MAJORITY (2 of 3) acknowledge it.
 If Node 1 crashes → 2 and 3 elect a new leader automatically.
 No data is lost. Recovery happens in seconds.
```

```python
from dataclasses import dataclass, field
from typing import List
from enum import Enum


class NodeRole(Enum):
    LEADER = "leader"
    FOLLOWER = "follower"


@dataclass
class LogEntry:
    term: int
    index: int
    command: str


@dataclass
class RaftNode:
    node_id: int
    role: NodeRole = NodeRole.FOLLOWER
    current_term: int = 0
    log: List[LogEntry] = field(default_factory=list)
    commit_index: int = -1

    def append_entry(self, entry: LogEntry) -> bool:
        if self.role != NodeRole.LEADER:
            raise PermissionError("Only the leader can append entries")
        self.log.append(entry)
        return True

    def replicate_to_follower(self, follower: "RaftNode", entry: LogEntry) -> bool:
        follower.log.append(entry)
        return True  # simulated acknowledgment


def simulate_raft_write(nodes: List[RaftNode], command: str) -> str:
    leader = next(n for n in nodes if n.role == NodeRole.LEADER)
    followers = [n for n in nodes if n.role == NodeRole.FOLLOWER]

    new_entry = LogEntry(
        term=leader.current_term,
        index=len(leader.log),
        command=command
    )
    leader.append_entry(new_entry)

    acks = 1  # leader counts itself
    for follower in followers:
        if leader.replicate_to_follower(follower, new_entry):
            acks += 1

    quorum = (len(nodes) // 2) + 1
    if acks >= quorum:
        leader.commit_index = new_entry.index
        return f"COMMITTED at index {new_entry.index} ({acks}/{len(nodes)} acks)"
    else:
        return f"FAILED — only {acks}/{len(nodes)} acks, needed {quorum}"


nodes = [
    RaftNode(node_id=1, role=NodeRole.LEADER, current_term=1),
    RaftNode(node_id=2, role=NodeRole.FOLLOWER, current_term=1),
    RaftNode(node_id=3, role=NodeRole.FOLLOWER, current_term=1),
]

result = simulate_raft_write(nodes, "SET /users/1001 = {name: Alice, balance: 5000}")
print(result)
# Output: COMMITTED at index 0 (3/3 acks)
```


---

### Layer 2: The Transaction Layer — ACID in a Distributed World

#### Two-Phase Commit (2PC)

```
Two-Phase Commit (2PC)
───────────────────────

PHASE 1 — PREPARE:
  Coordinator ──► Node A: "Can you commit txn-42?"
  Coordinator ──► Node B: "Can you commit txn-42?"

  Node A ──► Coordinator: "YES, prepared and locked"
  Node B ──► Coordinator: "YES, prepared and locked"

PHASE 2 — COMMIT (only if ALL said YES):
  Coordinator ──► Node A: "COMMIT txn-42"
  Coordinator ──► Node B: "COMMIT txn-42"
  ✓ Transaction complete. Locks released.

CRITICAL FLAW:
  If the Coordinator crashes between Phase 1 and 2,
  nodes are STUCK — they voted YES but never got a
  final COMMIT or ABORT. This is the "blocking problem."

FIX: Make the Coordinator itself a Raft group.
  Its decisions are replicated and durable.
  A new leader recovers in-flight transactions automatically.
```


#### MVCC: Non-Blocking Reads

```python
from typing import Dict, List, Tuple, Any


class MVCCStore:
    """
    Simplified MVCC key-value store.
    Each key maps to a list of (timestamp, value) tuples.
    Readers never block writers. Writers never block readers.
    """

    def __init__(self):
        self._store: Dict[str, List[Tuple[int, Any]]] = {}
        self._current_ts = 0

    def _next_ts(self) -> int:
        self._current_ts += 1
        return self._current_ts

    def write(self, key: str, value: Any) -> int:
        ts = self._next_ts()
        self._store.setdefault(key, []).append((ts, value))
        return ts

    def read(self, key: str, snapshot_ts: int) -> Any:
        """Return the most recent version at or before snapshot_ts."""
        versions = [
            (ts, val)
            for ts, val in self._store.get(key, [])
            if ts <= snapshot_ts
        ]
        if not versions:
            return None
        return max(versions, key=lambda x: x)


store = MVCCStore()
ts1 = store.write("/users/1001", {"balance": 5000})
ts2 = store.write("/users/1001", {"balance": 4800})  # withdrawal
ts3 = store.write("/users/1001", {"balance": 5300})  # deposit

print(store.read("/users/1001", snapshot_ts=ts1))  # {"balance": 5000}
print(store.read("/users/1001", snapshot_ts=ts2))  # {"balance": 4800}
print(store.read("/users/1001", snapshot_ts=ts3))  # {"balance": 5300}
```

```
MVCC — Multiple Versions of the Same Row
──────────────────────────────────────────

 Key: /users/1001

 Version @ ts=100  →  { balance: 5000 }   ← initial
 Version @ ts=105  →  { balance: 4800 }   ← after withdrawal
 Version @ ts=112  →  { balance: 5300 }   ← after deposit

 Txn A (started at ts=102) reads:  { balance: 5000 }
 Txn B (started at ts=108) reads:  { balance: 4800 }
 Txn C (started at ts=115) reads:  { balance: 5300 }

 Old versions are garbage-collected when no active
 transaction needs them anymore.
```


---

### Layer 3: The SQL Layer — Distributed Query Planning

```
Distributed Query Execution
─────────────────────────────

  Client: SELECT * FROM orders WHERE user_id = 1001

  ┌─────────────────────────────────────┐
  │         SQL Gateway Node            │
  │  1. Parse SQL                       │
  │  2. Consult shard map               │
  │  3. Identify: user 1001 → Shard C   │
  │  4. Send subquery to Shard C only   │
  │  5. Stream results back to client   │
  └──────────────┬──────────────────────┘
                 │  targeted — NOT broadcast
                 ▼
          ┌──────────────┐
          │   SHARD C    │  ← Only this shard queried
          │ (Nodes 7,8,9)│
          └──────────────┘

  Shards A, B, D–Z are not contacted at all.
  This is the power of predicate pushdown + co-location.
```

```python
def plan_distributed_join(
    left_table: str,
    right_table: str,
    join_key: str,
    left_size_mb: int,
    right_size_mb: int,
) -> dict:
    """
    Broadcast Join: replicate the smaller table to all nodes of the larger.
    Hash Redistribute Join: reshuffle both tables by the join key.
    """
    if left_size_mb < 100:
        return {
            "strategy": "BROADCAST_JOIN",
            "broadcast_table": left_table,
            "probe_table": right_table,
            "reason": f"{left_table} is small ({left_size_mb}MB); broadcast it"
        }
    else:
        return {
            "strategy": "HASH_REDISTRIBUTE_JOIN",
            "redistribute_by": join_key,
            "reason": "Both tables large; repartition by join key for co-location"
        }


print(plan_distributed_join("users", "orders", "user_id", 50, 10_000))
# {"strategy": "BROADCAST_JOIN", ...}

print(plan_distributed_join("users", "orders", "user_id", 500, 10_000))
# {"strategy": "HASH_REDISTRIBUTE_JOIN", ...}
```


---

## Part 6: The CAP Theorem — The Fundamental Constraint

### You Cannot Have Everything

In 2000, Eric Brewer formulated the **CAP Theorem**: in a distributed system,
you can guarantee at most **two** of three properties simultaneously:


| Property | What It Means |
| :-- | :-- |
| **C**onsistency | Every read returns the most recent write, or an error |
| **A**vailability | Every request receives a non-error response |
| **P**artition Tolerance | System operates despite network partitions |

Since network partitions are a physical reality — cables get cut, packets drop —
**P is not optional**. The real choice is between **C** and **A** during a
partition event.

- **CP Systems** (Distributed SQL): Choose consistency. The minority partition
refuses requests rather than risk stale data.
- **AP Systems** (Cassandra, DynamoDB): Choose availability. All nodes respond,
but may return stale or conflicting data.

Distributed SQL systems are firmly **CP**. For financial systems, a
"service unavailable" error is infinitely preferable to transferring money twice.

---

## Part 7: The Major Players — Three Paths to the Same Goal

```
Distributed SQL System Comparison
───────────────────────────────────

  ┌──────────────────┬──────────────────┬─────────────────┬──────────────────┐
  │  Feature         │  CockroachDB     │  TiDB           │  YugabyteDB      │
  ├──────────────────┼──────────────────┼─────────────────┼──────────────────┤
  │ Wire Protocol    │ PostgreSQL       │ MySQL           │ PostgreSQL       │
  │ Consensus        │ Raft             │ Raft            │ Raft             │
  │ Storage Engine   │ Pebble/RocksDB   │ TiKV/RocksDB    │ DocDB/RocksDB    │
  │ Sharding Style   │ Range-based      │ Region-based    │ Tablet-based     │
  │ Analytics        │ Good             │ Excellent(TiFlash)│ Good           │
  │ Geo-Partitioning │ First-class      │ Placement Rules │ Tablespace-based │
  │ Default Isolation│ Serializable     │ Snapshot        │ Snapshot         │
  └──────────────────┴──────────────────┴─────────────────┴──────────────────┘
```


#### CockroachDB — Built to Survive Anything

CockroachDB was explicitly designed to survive catastrophic failures — hence the
name. Its standout feature is **geo-partitioning**: pin specific rows (e.g., EU
user records) to specific geographic regions for GDPR compliance. It uses
**Serializable isolation** by default — the strongest possible — preventing all
concurrency anomalies.

#### TiDB — The HTAP Powerhouse

TiDB (by PingCAP) pairs its row-oriented storage engine (TiKV) with a columnar
engine called **TiFlash** on the same cluster. This enables **Hybrid
Transactional/Analytical Processing (HTAP)**: run analytics directly on fresh
transactional data, with no ETL pipeline needed. Its MySQL wire protocol
compatibility makes migration from MySQL stacks nearly seamless.

#### YugabyteDB — The PostgreSQL Native

YugabyteDB provides the most complete PostgreSQL compatibility of the three,
supporting stored procedures, triggers, and advanced PostgreSQL-specific features
out of the box. Its two-layer architecture — YQL over DocDB — maps cleanly onto
Kubernetes-native deployments.

---

## Part 8: The Consistency Models — A Spectrum, Not a Switch

```
Isolation Level Spectrum
──────────────────────────

Weakest ◄─────────────────────────────────────────────► Strongest

READ          READ         REPEATABLE    SNAPSHOT      SERIALIZABLE
UNCOMMITTED   COMMITTED    READ          ISOLATION
    │              │            │              │               │
Dirty reads   No dirty    No non-rep.   Frozen DB       No anomalies
  allowed      reads        reads        snapshot         possible
                                          per txn
```

| Isolation Level | Prevents | Still Possible |
| :-- | :-- | :-- |
| Read Uncommitted | Nothing | Dirty reads, phantoms |
| Read Committed | Dirty reads | Non-repeatable reads |
| Repeatable Read | Dirty + non-repeatable | Phantom reads |
| Snapshot Isolation | Most anomalies | Write skew (rare) |
| Serializable | All anomalies | Nothing |

Most Distributed SQL systems default to **Snapshot Isolation** for performance,
with **Serializable** as an opt-in. Google Spanner offers **External Consistency**
— stronger than serializable — guaranteeing that if transaction B starts after
transaction A commits in wall-clock time, B will always see A's writes.

---

## Part 9: Clock Synchronization — The Hidden Complexity

### Why Time Is Hard in Distributed Systems

We need to assign timestamps to transactions to implement MVCC and determine
ordering. But clocks on different machines drift by milliseconds, and milliseconds
matter at scale. Google solved this with **TrueTime** (GPS + atomic clocks).
Open-source systems use **Hybrid Logical Clocks (HLC)**:

```python
import time
from dataclasses import dataclass


@dataclass
class HybridLogicalClock:
    """
    HLC combines physical time with a logical counter.
    Guarantees: if event A causally precedes B,
    then timestamp(A) < timestamp(B) — always.
    """
    physical: int = 0   # milliseconds since epoch
    logical: int = 0    # tie-breaker counter

    def now(self) -> "HybridLogicalClock":
        wall = int(time.time() * 1000)
        new_physical = max(self.physical, wall)
        new_logical = (self.logical + 1) if new_physical == self.physical else 0
        return HybridLogicalClock(new_physical, new_logical)

    def receive(self, remote: "HybridLogicalClock") -> "HybridLogicalClock":
        """Advance clock upon receiving a message. Ensures causality."""
        wall = int(time.time() * 1000)
        new_physical = max(self.physical, remote.physical, wall)

        if new_physical == self.physical == remote.physical:
            new_logical = max(self.logical, remote.logical) + 1
        elif new_physical == self.physical:
            new_logical = self.logical + 1
        elif new_physical == remote.physical:
            new_logical = remote.logical + 1
        else:
            new_logical = 0

        return HybridLogicalClock(new_physical, new_logical)

    def __lt__(self, other: "HybridLogicalClock") -> bool:
        return (self.physical, self.logical) < (other.physical, other.logical)


node_a = HybridLogicalClock()
node_b = HybridLogicalClock()

ts_a = node_a.now()
ts_b = node_b.receive(ts_a)

print(f"Node A: physical={ts_a.physical}, logical={ts_a.logical}")
print(f"Node B: physical={ts_b.physical}, logical={ts_b.logical}")
print(f"Causality preserved (A < B): {ts_a < ts_b}")
# Output: Causality preserved (A < B): True
```


---

## Part 10: Interview Patterns — What Companies Actually Ask

### The Three Canonical Problems

#### Pattern 1: "Design a globally available financial system"

- Shard accounts by `user_id` with range-based partitioning
- Use Raft replication with 3–5 replicas across availability zones
- Choose **Serializable isolation** for financial correctness
- Apply geo-partitioning to keep EU data in EU regions (GDPR)
- Use 2PC with a Raft-backed coordinator for cross-shard transactions
- Use HLC for timestamp ordering without hardware clock dependency


#### Pattern 2: "How would you scale this SQL database beyond one machine?"

Walk through the full evolution: vertical scaling → manual sharding →
Distributed SQL. Demonstrate you understand the *why* at each stage, not just
the *what*. Show the scars of manual sharding to justify the architectural leap.

#### Pattern 3: "What consistency model would you choose for X?"

```
Business Requirement → Isolation Level Mapping
───────────────────────────────────────────────

 Bank transfer (money movement)
   → SERIALIZABLE (prevent double-spend at any cost)

 Flight seat reservation
   → SERIALIZABLE (overbooking is catastrophic)

 E-commerce inventory check
   → SNAPSHOT ISOLATION (slight staleness acceptable)

 Leaderboard / analytics dashboard
   → READ COMMITTED (fresh enough, high throughput)

 Social media like count
   → Eventual consistency (approximate counts are fine)
```


### Vocabulary That Signals Mastery

| Term | What It Demonstrates |
| :-- | :-- |
| **Raft / Paxos** | How replicas stay consistent |
| **MVCC** | Non-blocking concurrency control |
| **Range / Tablet / Shard** | Data distribution model |
| **2PC + Raft coordinator** | Distributed transaction safety |
| **Serializable isolation** | Strongest SQL consistency guarantee |
| **Hybrid Logical Clocks** | Distributed time ordering |
| **CAP Theorem** | Fundamental distributed system tradeoffs |
| **Predicate pushdown** | Distributed query optimization |
| **HTAP** | Mixed OLTP + OLAP architecture |
| **TrueTime** | Google Spanner's clock model |


---

## Part 11: When NOT to Use Distributed SQL

```
Decision Tree: Should You Use Distributed SQL?
────────────────────────────────────────────────

       Does your data fit comfortably
       on a single server?
               │
         YES   │   NO
          ▼    │    ▼
    Use plain  │   Do you need SQL
    PostgreSQL │   and ACID guarantees?
    (simpler,  │        │
     faster)   │  YES   │  NO
               │   ▼    │   ▼
               │ Dist.  │  NoSQL
               │  SQL ✓ │  (Cassandra, DynamoDB)
               │
        Is the primary workload
        pure analytics (no writes)?
               │
         YES   │   NO
          ▼    │    ▼
    ClickHouse │  Distributed SQL ✓
    BigQuery   │
```

Situations where other databases win:

- **Small-scale apps**: Plain PostgreSQL is simpler to operate and faster for low
traffic workloads.
- **Pure analytics**: Column-oriented databases (BigQuery, Snowflake, ClickHouse)
have fundamentally superior formats for scan-heavy queries.
- **Sub-millisecond latency**: Consensus round-trips add overhead. In-memory
single-node systems win on raw speed.
- **Eventual consistency is genuinely fine**: For metrics, logging, or social
counters, ACID complexity provides no benefit. Use Cassandra or DynamoDB.

---

## Epilogue: The Promise Fulfilled

We started with a bank that had one teller. By the time we close this chapter,
we have built the intellectual model for a system that can serve millions of
customers simultaneously, across dozens of datacenters, on five continents —
with the same guarantees as that original single teller. No double-spending.
No lost deposits. No inconsistent balances.

Distributed SQL is the result of decades of hard-won engineering insight: the
realization that we do not have to choose between *correctness* and *scale*. With
Raft consensus, MVCC, distributed transaction coordination, and careful clock
management, we can have both.

The systems that implement this vision — CockroachDB, TiDB, YugabyteDB, Google
Spanner, Amazon Aurora DSQL — represent the current frontier of database
engineering. Understanding them deeply — not just how to *use* them, but how they
*work* — is what separates a senior engineer from the rest.

The next time an interviewer asks you to design a globally consistent, highly
available SQL system, you will not panic. You will draw the three-layer
architecture, explain the Raft groups, describe MVCC, and walk through the
distributed transaction protocol with the calm authority of someone who has traced
every wire in the building.

---