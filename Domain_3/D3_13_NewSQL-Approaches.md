
# NewSQL: Scaling the Unscalable — A Complete Guide

> *"The problem was never SQL itself. The problem was that the architecture
> underneath SQL was designed for a world with one server, one city, and one
> timezone."*

---

## Part I: The Bridge — A Story of a Bank

Imagine a bank in the 1980s. It has a single branch, a single vault, and a
single teller window. When you walk in, the teller checks the ledger, verifies
your balance, and processes your transaction. The system is *perfectly
consistent*. If you withdraw $100, the ledger updates immediately. No one else
can see your old balance. This is ACID compliance in its purest, most intuitive
form.

Now imagine the bank explodes in popularity. Within years, millions of customers
want to transact simultaneously, from New York to Tokyo. The bank has two
choices:

**Option A:** Build a bigger, faster branch. Install a stronger vault, hire more
tellers, upgrade the pneumatic tube system. This is **vertical scaling** — just
make the one machine more powerful.

**Option B:** Open hundreds of smaller branches spread across the globe. Each
branch handles local customers. This is **horizontal scaling** — spread the load
across many machines.

Option A has a hard ceiling. There's only so big a single vault can be. Option
B sounds great, but now we have a new problem: *What if a customer withdraws
from the New York branch at the exact moment someone else is wiring money into
their account from Tokyo?* Both branches have their own ledgers. How do they
stay synchronized? Who arbitrates the truth?

This, in one parable, is the central problem of distributed databases — and the
reason NewSQL was born.

---

## Part II: The Legacy — Why SQL Struggled to Scale

### The Golden Age of Relational Databases

In the 1970s, Edgar Codd introduced the relational model, and for the next three
decades, relational databases like Oracle, DB2, and MySQL were the undisputed
kings of data management. They brought us something precious: **ACID guarantees**.

- **Atomicity**: A transaction is all-or-nothing
- **Consistency**: Data moves from one valid state to another
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed data survives crashes

```python
# A classic ACID transaction in Python using psycopg2 (PostgreSQL)
import psycopg2

conn = psycopg2.connect("dbname=bank user=admin")
conn.autocommit = False  # We control the transaction boundary
cursor = conn.cursor()

try:
    cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
    cursor.execute("UPDATE accounts SET balance = balance + 100 WHERE id = 2")
    conn.commit()      # <-- Atomicity: Both succeed or neither does
    print("Transfer committed.")
except Exception as e:
    conn.rollback()    # <-- All-or-nothing guarantee
    print(f"Transaction failed, rolled back: {e}")
finally:
    cursor.close()
    conn.close()
```

This code works beautifully — on a *single machine*. But in the 2000s, the
internet happened. Companies like Google, Amazon, and Facebook were suddenly
managing billions of rows, petabytes of data, and millions of concurrent users.
The single server hit its physical ceiling.

### The Four Overheads Killing Traditional SQL at Scale

Researchers at MIT and Carnegie Mellon (including Michael Stonebraker, who later
built VoltDB) studied where traditional databases spent their time. The results
were startling. On a typical OLTP workload:

```
Where Traditional SQL Spends Its Time (OLTP Workload)
────────────────────────────────────────────────────
  Buffer Pool Management   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓  34%
  Locking & Latching       ▓▓▓▓▓▓▓▓▓▓▓     25%
  Write-Ahead Logging      ▓▓▓▓▓▓▓▓         19%
  Actual Useful Work       ▓▓▓▓▓▓▓▓         12%
  Other Overhead           ▓▓▓▓▓             10%
────────────────────────────────────────────────────
```

Only 12% of the database's effort was on *actual work*. The rest was managing
the machinery of traditional architecture. This was the "broken state" that made
a revolution inevitable.

---

## Part III: The NoSQL Escape Hatch (And Its Hidden Price)

### The BASE Alternative

When the scaling crisis hit, engineers at Google (BigTable, 2006) and Amazon
(Dynamo, 2007) published landmark papers describing a new approach. The community
called it **NoSQL**. The philosophy was bold: *relax the consistency guarantees,
and scaling becomes easy.*

NoSQL systems adopted the **BASE** model:

- **B**asically **A**vailable: The system responds, even if some data is stale
- **S**oft-State: State may change over time without input (eventual consistency)
- **E**ventual Consistency: All replicas will *eventually* agree, but not
necessarily right now

This is like our bank branches operating independently. The New York branch
processes withdrawals without calling Tokyo. Fast? Absolutely. But now a clever
customer could withdraw their full balance from both branches simultaneously
before the ledger syncs. The bank would call that **fraud**. The database calls
it a **consistency violation**.

### The CAP Theorem: You Can't Have Everything

Eric Brewer's CAP Theorem (formalized in 2002) stated that in a distributed
system, you can only guarantee **two** of three properties:

```
           Consistency
               /\
              /  \
             /    \
            /      \
           /  C + A  \  ← Google Spanner aims here
          /____________\
         /              \
        /                \
Availability ─────────── Partition Tolerance
      ↑                        ↑
   Cassandra               HBase, Zookeeper
   DynamoDB
   (AP systems)           (CP systems)
```

*In the diagram above, we are deliberately placing systems at the vertices to
show their primary tradeoff. In practice, all distributed systems must tolerate
partitions — so the real choice is between consistency (CP) and availability
(AP) when a network partition occurs.*

NoSQL systems largely chose **AP** (Availability + Partition Tolerance) — they
stay up and respond, but data can temporarily be inconsistent. This was
acceptable for product recommendations or social media "likes." It was
catastrophic for financial transactions, healthcare records, and inventory
management.

The industry had traded one problem for another. Millions of lines of application
code now had to compensate for missing ACID guarantees, adding complex retry
logic, conflict resolution handlers, and "read-your-writes" hacks. There had to
be a better way.

---

## Part IV: The Third Way — Enter NewSQL

The term **NewSQL** was coined by analyst Matthew Aslett in 2011. The insight
was deceptively simple: *the problem wasn't SQL or the relational model — the
problem was the single-node architecture underneath it.* What if we redesigned
the engine from scratch, keeping SQL and ACID, but building it natively for a
distributed, multi-node world?

The NewSQL promise:

> **SQL expressiveness + ACID guarantees + NoSQL-style horizontal scalability**

It wasn't about compromising. It was about rebuilding the bank — not with a
single giant branch, but with a network of branches that share a single, globally
synchronized ledger.

### The Three Families of NewSQL Systems

NewSQL systems fall into three broad architectural families, grouped by *how*
they achieve distributed ACID:


| Family | Core Mechanism | Examples | Best For |
| :-- | :-- | :-- | :-- |
| **New Architecture** | Built from scratch with distributed-first design | Google Spanner, CockroachDB | Global, geo-distributed OLTP |
| **SQL Engines over NoSQL** | SQL layer on top of distributed KV stores | TiDB (over TiKV) | MySQL migration + horizontal scale |
| **In-Memory NewSQL** | All data in RAM, partitioned, serial execution | VoltDB, MemSQL | Ultra-low-latency OLTP |

We'll explore each family in depth after we understand the foundational
architecture that makes them all work.

---

## Part V: The Blueprint — How NewSQL Works Inside

### Layer 1: The Shared-Nothing Foundation

The first design principle of all NewSQL systems is **shared-nothing
architecture**. In a shared-nothing cluster, every node owns its own CPU, memory,
and disk. There are no shared resources. This is radically different from
traditional SQL, where all nodes had to share the same storage.

```
Traditional SQL (Shared-Everything)     NewSQL (Shared-Nothing)
════════════════════════════════        ════════════════════════════════
 ┌────────┐ ┌────────┐                   ┌─────────────────────────────┐
 │ Node 1 │ │ Node 2 │                   │     CockroachDB Cluster      │
 └────┬───┘ └───┬────┘                   │                              │
      │         │                        │  ┌───────┐   ┌───────┐      │
      └────┬────┘                        │  │ Node 1│   │ Node 2│      │
      ┌────▼────┐                        │  │CPU+RAM│   │CPU+RAM│      │
      │ SHARED  │                        │  │+Disk  │   │+Disk  │      │
      │  DISK   │  ← bottleneck          │  └───────┘   └───────┘      │
      └─────────┘                        │       ┌───────────┐         │
                                         │       │   Node 3  │         │
                                         │       │ CPU+RAM   │         │
                                         │       │ +Disk     │         │
                                         │       └───────────┘         │
                                         │  No shared storage!         │
                                         └─────────────────────────────┘
```

*In the left diagram, both nodes compete for the same disk. A bottleneck at any
layer is a bottleneck for the whole system. In the right diagram, each node is
fully self-sufficient. Adding a new node to the cluster is as easy as plugging
in a new machine.*

### Layer 2: Ranges, Shards, and the Distributed Key-Value Store

NewSQL systems don't store data as relational tables internally. Under the hood,
they convert all data into a **sorted, distributed key-value store**. Each
row in a SQL table gets mapped to a key in the KV store, and the global keyspace
is divided into contiguous **ranges** (also called shards or tablets).

```python
# Conceptual illustration: How a SQL table maps to a KV store
# (This is a simplification of CockroachDB's internal encoding)

# SQL Table: accounts
# +----+-------+---------+
# | id | name  | balance |
# +----+-------+---------+
# |  1 | Alice |    1000 |
# |  2 | Bob   |     500 |
# |  3 | Carol |    2500 |

# Internal KV Representation
kv_store = {
    "/table/accounts/1/id":      1,
    "/table/accounts/1/name":    "Alice",
    "/table/accounts/1/balance": 1000,

    "/table/accounts/2/id":      2,
    "/table/accounts/2/name":    "Bob",
    "/table/accounts/2/balance": 500,

    "/table/accounts/3/id":      3,
    "/table/accounts/3/name":    "Carol",
    "/table/accounts/3/balance": 2500,
}

# These key-value pairs are sorted and split into RANGES
range_1 = {k: v for k, v in kv_store.items() if k <= "/table/accounts/2/"}
range_2 = {k: v for k, v in kv_store.items() if k >  "/table/accounts/2/"}

# Each range is then replicated across 3+ nodes for fault tolerance
print(f"Range 1 contains {len(range_1)} entries -> replicated to nodes 1,2,3")
print(f"Range 2 contains {len(range_2)} entries -> replicated to nodes 2,3,4")
```

This mapping is powerful. The SQL layer at the top translates your `SELECT`
query into KV operations. The storage layer handles distribution and replication.
The two concerns are completely decoupled.

### Layer 3: The Five-Layer Architecture (CockroachDB Model)

One of the clearest illustrations of NewSQL's layered design comes from
CockroachDB's published architecture. Each layer has a single, focused
responsibility:

```
CockroachDB Architecture: Five Clean Layers
══════════════════════════════════════════════
┌──────────────────────────────────────────┐
│  Layer 1: SQL                            │
│  -  Parses and plans SQL queries          │
│  -  Translates SQL → KV operations        │
│  -  Supports standard PostgreSQL dialect  │
├──────────────────────────────────────────┤
│  Layer 2: Transactional                  │
│  -  Ensures ACID for multi-key changes    │
│  -  Manages write intents & timestamps    │
│  -  Handles conflict detection            │
├──────────────────────────────────────────┤
│  Layer 3: Distribution                   │
│  -  Routes requests to correct range      │
│  -  Manages range splits and merges       │
│  -  Presents cluster as single entity     │
├──────────────────────────────────────────┤
│  Layer 4: Replication (Raft)             │
│  -  Replicates ranges across nodes        │
│  -  Leader election per range             │
│  -  Ensures consensus before commit       │
├──────────────────────────────────────────┤
│  Layer 5: Storage (Pebble / RocksDB)     │
│  -  Reads and writes KV data to disk      │
│  -  Uses LSM Trees for efficient writes   │
│  -  Supports MVCC timestamps              │
└──────────────────────────────────────────┘
```

*Each layer communicates only with its immediate neighbor. This separation of
concerns is why NewSQL systems can evolve their storage engine independently from
their SQL parser — a design philosophy borrowed from modern operating systems.*

---

## Part VI: The Cornerstone — Consensus Protocols

This is where the real magic happens. To maintain consistency across distributed
nodes, NewSQL systems rely on **consensus protocols** — algorithms that allow a
cluster of machines to agree on a single value, even in the presence of node
failures.

### The Problem Without Consensus (Show the Broken State)

Imagine three replicas of the same range:

```
     Node A (Leader)     Node B (Follower)     Node C (Follower)
      balance=1000          balance=1000          balance=1000
          │
          ▼
  Client writes: SET balance=900
          │
    Node A updates ✓
          │
    [Network partition — Node B and C don't receive the update]
          │
    Another client reads from Node B
          ▼
      Returns 1000  ← STALE DATA!
```

Without consensus, we have a split-brain scenario. Two clients see two different
truths. This is exactly what NewSQL must prevent.

### Raft: Consensus Made Understandable

Raft (developed by Diego Ongaro at Stanford, 2014) was designed explicitly to
be *easier to understand* than its predecessor, Paxos. It is used by CockroachDB,
TiDB (via TiKV), and YugabyteDB. Every range in a NewSQL cluster has exactly
one Raft group.

Every node in a Raft group is in one of three states:

```
              ┌─────────────┐
              │   Follower  │◄───── All nodes start here
              └──────┬──────┘
                     │ Election timeout expires
                     │ (no heartbeat from leader)
                     ▼
              ┌─────────────┐
              │  Candidate  │ ──── Votes for itself, requests votes
              └──────┬──────┘
                     │ Receives majority votes
                     ▼
              ┌─────────────┐
              │   Leader    │ ──── Handles all writes for this range
              └─────────────┘
                     │ Sends periodic heartbeats to followers
                     ▼
              ┌─────────────┐
              │   Follower  │ ◄─── Steps down if it sees a higher term
              └─────────────┘
```

*Notice in this diagram that the state machine is intentionally simple: three
states, clear transitions. This is Raft's core design philosophy — reduce the
state space so failures are easier to reason about.*

The write flow in Raft is equally clean:

```python
# Conceptual simulation of a Raft write operation
class RaftCluster:
    def __init__(self, nodes: list, quorum_size: int):
        self.nodes = nodes
        self.leader = nodes
        self.quorum_size = quorum_size  # typically (n/2) + 1
        self.committed_log = []

    def write(self, entry: dict) -> bool:
        # Step 1: Leader appends to its own log
        self.leader["log"].append(entry)

        # Step 2: Leader sends AppendEntries RPC to all followers
        acknowledgements = 1  # Leader counts itself
        for follower in self.nodes[1:]:
            if self._send_append_entries(follower, entry):
                acknowledgements += 1

        # Step 3: Commit only when a majority (quorum) acknowledges
        if acknowledgements >= self.quorum_size:
            self.committed_log.append(entry)
            print(f"[COMMITTED] Entry '{entry}' — {acknowledgements}/{len(self.nodes)} nodes agree")
            return True
        else:
            print(f"[FAILED] Only {acknowledgements} nodes acknowledged — quorum not reached")
            return False

    def _send_append_entries(self, follower: dict, entry: dict) -> bool:
        # In real Raft, this is an RPC call. We simulate latency/failure here.
        import random
        if follower["alive"] and random.random() > 0.1:  # 90% success rate
            follower["log"].append(entry)
            return True
        return False

# Example: 5-node cluster, quorum = 3
nodes = [
    {"id": i, "log": [], "alive": True} for i in range(5)
]
# Simulate one node being offline
nodes["alive"] = False
nodes["alive"] = False[^2]

cluster = RaftCluster(nodes, quorum_size=3)
cluster.write({"table": "accounts", "key": "user:1:balance", "value": 900})
```

The key insight: **a write is only committed when a majority (quorum) of nodes
acknowledge it**. With 5 nodes, we need 3 confirmations. This means the system
can tolerate 2 simultaneous node failures and still make progress.

### Multi-Paxos vs. Raft: A Quick Comparison

```
┌─────────────────┬──────────────────────┬──────────────────────┐
│   Property      │    Multi-Paxos       │       Raft           │
├─────────────────┼──────────────────────┼──────────────────────┤
│ Used by         │ Google Spanner       │ CockroachDB, TiDB    │
│ Leader election │ Complex, multi-phase │ Simple timer-based   │
│ Log gaps        │ Allowed (complex)    │ Not allowed (simpler)│
│ Understandability│ Very hard           │ Designed to be clear │
│ Performance     │ Comparable           │ Comparable           │
└─────────────────┴──────────────────────┴──────────────────────┘
```

*Google chose Multi-Paxos for Spanner partly for historical reasons and partly
because their internal expertise predated Raft. Most modern NewSQL systems
default to Raft for its clarity and simpler implementation.*

---

## Part VII: MVCC — Reading Without Blocking

### The Pain Point: Readers Blocking Writers

In traditional SQL, when a transaction reads data, it acquires a **shared lock**.
When another transaction writes, it acquires an **exclusive lock**. These locks
conflict — readers block writers, and writers block readers. At scale, under
thousands of concurrent transactions, the system grinds to a halt.

### Multi-Version Concurrency Control (MVCC)

NewSQL systems solve this with **MVCC**. Instead of locking a row during a read,
the system keeps *multiple versions* of the row, each tagged with a timestamp.
Readers always read from the version that was committed before their transaction
started. They never block writers, and writers never block readers.

```
           MVCC: A Timeline of Row Versions
════════════════════════════════════════════════════════
  Row: accounts/user:1/balance

  Time  ──────────────────────────────────────────▶

  T=10  │  version@T10: {balance: 1000}  ← Write by Tx1
        │
  T=20  │  version@T20: {balance: 900}   ← Write by Tx2
        │
  T=30  │  version@T30: {balance: 850}   ← Write by Tx3
        │
  T=35  │  Tx4 begins: "READ AS OF T25"
        │  → Returns version@T20 (900) ✓
        │  → Does NOT block ongoing writes!
════════════════════════════════════════════════════════
```

*Notice that in this diagram, Tx4 reads a consistent snapshot of the data as it
existed at T=25, without interfering with any concurrent write operations. This
is the "snapshot isolation" guarantee that makes NewSQL reads non-blocking.*

```python
# Simplified MVCC version store simulation in Python
from dataclasses import dataclass, field
from typing import Optional, List

@dataclass
class Version:
    timestamp: int
    value: any
    is_committed: bool = False

class MVCCStore:
    def __init__(self):
        self.versions: dict[str, List[Version]] = {}

    def write(self, key: str, value: any, timestamp: int):
        """Write a new version of the key with a given timestamp."""
        if key not in self.versions:
            self.versions[key] = []
        v = Version(timestamp=timestamp, value=value, is_committed=False)
        self.versions[key].append(v)
        self.versions[key].sort(key=lambda x: x.timestamp)
        return v

    def commit(self, key: str, timestamp: int):
        """Commit the version at the given timestamp."""
        for v in self.versions.get(key, []):
            if v.timestamp == timestamp:
                v.is_committed = True
                return True
        return False

    def read(self, key: str, read_timestamp: int) -> Optional[any]:
        """
        Return the most recent COMMITTED version
        that was written at or before read_timestamp.
        """
        best = None
        for v in self.versions.get(key, []):
            if v.is_committed and v.timestamp <= read_timestamp:
                best = v  # Keep the latest one that qualifies
        return best.value if best else None


# Demo
store = MVCCStore()

# Two concurrent writes at different timestamps
v1 = store.write("user:1:balance", 1000, timestamp=10)
v2 = store.write("user:1:balance", 900,  timestamp=20)
v3 = store.write("user:1:balance", 850,  timestamp=30)

# Commit T=10 and T=20, but NOT T=30 (in-flight transaction)
store.commit("user:1:balance", 10)
store.commit("user:1:balance", 20)
# T=30 is intentionally NOT committed yet

# A reader starting at T=25 sees the world as of T=20
balance = store.read("user:1:balance", read_timestamp=25)
print(f"Balance as of T=25: {balance}")  # Output: 900

# A reader starting at T=35 still sees T=20 (T=30 is uncommitted)
balance = store.read("user:1:balance", read_timestamp=35)
print(f"Balance as of T=35: {balance}")  # Output: 900, NOT 850
```


### The Clock Problem: HLC (Hybrid Logical Clocks)

MVCC requires that every node can assign *globally meaningful timestamps*. But
in a distributed system, no two clocks are perfectly synchronized. A naive
implementation where Node A assigns T=100 and Node B assigns T=99 could
silently reverse causal order.

CockroachDB uses **Hybrid Logical Clocks (HLC)** — a combination of physical
wall clock time and a logical counter. The HLC ensures:

1. If event A *causally happens before* B, then HLC(A) < HLC(B)
2. Timestamps always move forward, even when clocks drift

---

## Part VIII: TrueTime — Google's Atomic Clock Advantage

Google took timestamp management a step further with **TrueTime**, the secret
weapon embedded inside Google Spanner. Rather than a single point in time,
TrueTime returns an *interval*: `[earliest, latest]` — a bounded range of
uncertainty about the current moment.

```
TrueTime API
════════════════════════════════════════
  Call: TT.now()
  Returns: [T_earliest, T_latest]

  Example:  [16:04:23.001, 16:04:23.007]
            ───────────────────────────
            └──────┬──────────────┘
                   ε = 6ms uncertainty window
                   (ensured by GPS + atomic clocks
                    in Google data centers)
════════════════════════════════════════

  If   TT.after(t)   → True means t has definitely passed
  If   TT.before(t)  → True means t has definitely not passed

  Commit Wait:
  Before committing Tx at timestamp Ts,
  Spanner WAITS until TT.after(Ts) is True.
  This guarantees no future transaction
  can get a timestamp ≤ Ts.
```

*This is the key insight: by using atomic clocks and GPS receivers in data
centers, Google bounds clock uncertainty to ~7ms. When Spanner wants to commit
a transaction at timestamp T, it simply waits out the uncertainty window before
releasing the commit. This "commit wait" is the price of external consistency —
and it's only milliseconds.*

This gives Spanner something extraordinary: **external consistency** (the
strongest form of consistency in distributed systems — stronger even than
linearizability). If transaction T1 commits before T2 starts in real-world time,
Spanner guarantees that T1's timestamp is strictly less than T2's.

---

## Part IX: The Major Players

### Google Spanner — The Pioneer

Spanner (2012 paper, publicly available via Google Cloud as Cloud Spanner) was
the first system to offer globally-distributed, externally-consistent SQL
transactions. It is the system NewSQL is measured against.

```
Google Spanner Architecture

┌─────────────────────────────────────────────────────┐
│                  Cloud Spanner                       │
│                                                      │
│  ┌──────────────┐   ┌──────────────┐                │
│  │  Zone: US-E  │   │  Zone: EU-W  │                │
│  │ ┌──────────┐ │   │ ┌──────────┐ │                │
│  │ │ Spanserver│ │   │ │ Spanserver│ │               │
│  │ │  Paxos   │ │   │ │  Paxos   │ │                │
│  │ │  Group 1 │ │   │ │  Group 1 │ │                │
│  │ └──────────┘ │   │ └──────────┘ │                │
│  │              │   │              │                 │
│  │  TrueTime   │   │  TrueTime   │                  │
│  │ (GPS+Atomic)│   │ (GPS+Atomic)│                  │
│  └──────────────┘   └──────────────┘                │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │         Colossus Distributed File System       │ │
│  └────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

**Key Characteristics of Spanner:**

- Multi-Paxos consensus, one Paxos group per shard
- TrueTime for external consistency
- Hierarchical directory-based data organization
- Two-Phase Commit (2PC) for cross-shard transactions, with each participant
being a Paxos group (so even 2PC is fault-tolerant)
- Proprietary to Google Cloud (cannot self-host)


### CockroachDB — Spanner for the Rest of Us

CockroachDB (open-source, named for its survival instincts) is the closest
public analog to Spanner. It takes Spanner's ideas and makes them deployable on
any cloud, or even bare metal.

**Key Characteristics of CockroachDB:**

- **Multi-Raft**: Each range has its own Raft group; thousands of Raft groups
co-exist efficiently via the "MultiRaft" optimization
- **HLC** instead of TrueTime (no atomic clocks, uses Hybrid Logical Clocks)
- PostgreSQL-compatible SQL dialect
- Serializable isolation by default (strongest ANSI isolation level)
- Self-healing: automatically rebalances ranges when nodes join or leave
- Runs on any cloud (AWS, GCP, Azure) or on-prem

```python
# Connecting to CockroachDB is identical to PostgreSQL
import psycopg2

# CockroachDB speaks the PostgreSQL wire protocol
conn = psycopg2.connect(
    host="localhost",
    port=26257,
    database="bank",
    user="root",
    sslmode="disable"
)

cursor = conn.cursor()

# Create a table — pure standard SQL
cursor.execute("""
    CREATE TABLE IF NOT EXISTS accounts (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        name STRING NOT NULL,
        balance DECIMAL NOT NULL CHECK (balance >= 0)
    )
""")

# CockroachDB handles distribution and replication transparently
cursor.execute("INSERT INTO accounts (name, balance) VALUES (%s, %s)", ("Alice", 1000))
cursor.execute("INSERT INTO accounts (name, balance) VALUES (%s, %s)", ("Bob", 500))
conn.commit()

# This transfer is ACID-compliant across the entire distributed cluster
def transfer_funds(conn, from_user, to_user, amount):
    with conn:
        cursor = conn.cursor()
        cursor.execute(
            "UPDATE accounts SET balance = balance - %s WHERE name = %s",
            (amount, from_user)
        )
        cursor.execute(
            "UPDATE accounts SET balance = balance + %s WHERE name = %s",
            (amount, to_user)
        )
    print(f"Transferred ${amount} from {from_user} to {to_user}")

transfer_funds(conn, "Alice", "Bob", 100)
```


### TiDB — The HTAP Innovator

TiDB (developed by PingCAP) takes a different architectural approach. Rather
than a monolithic system, it separates concerns into distinct components:

```
TiDB Architecture: Separation of Compute and Storage

┌─────────────────────────────────────────────────────────┐
│                      TiDB Layer                          │
│           (Stateless SQL Compute Nodes)                  │
│  ┌────────┐   ┌────────┐   ┌────────┐                  │
│  │ TiDB 1 │   │ TiDB 2 │   │ TiDB 3 │  ← Scale compute │
│  └────┬───┘   └───┬────┘   └───┬────┘    independently  │
└───────┼───────────┼────────────┼───────────────────────┘
        │           │            │
        └───────────┴────────────┘
                    │
           ┌────────▼────────────────────────┐
           │        PD (Placement Driver)    │
           │    Metadata + Scheduling + TSO  │
           │   (Timestamp Oracle — central   │
           │    logical clock generator)     │
           └────────┬────────────────────────┘
                    │
     ┌──────────────┴──────────────┐
     │                             │
┌────▼──────────────┐   ┌──────────▼──────────────┐
│      TiKV          │   │       TiFlash            │
│  (Row Storage)     │   │   (Column Storage)       │
│  Raft consensus    │   │   MPP for analytics      │
│  OLTP workloads    │   │   OLAP workloads         │
└────────────────────┘   └─────────────────────────┘
```

*The key innovation in TiDB is the presence of both TiKV (row-oriented) and
TiFlash (column-oriented) storage layers. Notice that TiDB can route a query
to either layer depending on whether it's transactional (TiKV) or analytical
(TiFlash). This is the HTAP capability — Hybrid Transactional and Analytical
Processing.*

**What makes TiDB unique:**

- MySQL-compatible dialect (easiest migration from legacy MySQL stacks)
- HTAP: TiKV for OLTP + TiFlash for OLAP in the same cluster
- Percolator-based distributed transaction model (inspired by Google's Percolator
paper)
- Centralized Timestamp Oracle (TSO) for global logical time — a deliberate
trade-off for simplicity over TrueTime's complexity
- Open-source and cloud-native


### VoltDB — In-Memory Radical

VoltDB takes a completely different approach to scaling. Rather than distributing
a disk-based system, it keeps *everything in RAM* and uses aggressive
partitioning to eliminate the need for locks entirely.

```
VoltDB: Single-Threaded Partition Model

  Partition 1         Partition 2         Partition 3
  (Node 1 / RAM)      (Node 2 / RAM)      (Node 3 / RAM)
  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │ Single       │     │ Single       │     │ Single       │
  │ Execution    │     │ Execution    │     │ Execution    │
  │ Thread       │     │ Thread       │     │ Thread       │
  │             │     │             │     │             │
  │ Tx1 → done  │     │ Tx3 → done  │     │ Tx5 → done  │
  │ Tx2 → done  │     │ Tx4 → done  │     │ Tx6 → done  │
  │             │     │             │     │             │
  │ NO LOCKS!   │     │ NO LOCKS!   │     │ NO LOCKS!   │
  └─────────────┘     └─────────────┘     └─────────────┘
         │                   │                   │
         └───────────────────┴───────────────────┘
                 Coordinator for cross-partition
                   transactions (rare case)
```

*Because each partition has only ONE thread that runs transactions to completion
serially, VoltDB eliminates the need for locking entirely. There's no
contention. This sounds limiting, but in practice, if your transactions are
partitioned correctly, cross-partition coordination is rare.*

**VoltDB's overhead eliminations:**

- No buffer pool (all data in RAM)
- No locking or latching (single-threaded per partition)
- No traditional write-ahead log (uses replication for durability)
- Result: **~45x throughput** improvement over traditional RDBMS on OLTP

The trade-off: your entire dataset must fit in memory across the cluster. It's
ideal for financial trading platforms, digital advertising bid systems, and
real-time gaming — all use cases requiring sub-millisecond latency.

---

## Part X: Putting It All Together — The Interview Perspective

### Where NewSQL Sits in the CAP Theorem

NewSQL systems are predominantly **CP** systems (Consistent + Partition
Tolerant). When a network partition occurs, they favor **consistency over
availability** — they will return an error rather than serve stale data.

```
                    C (Consistency)
                        /\
                       /  \
                      / CP \
    Spanner ●        /──────\ ← NewSQL territory
    CockroachDB ●   /        \
    TiDB ●         /          \
                  /            \
────────────────────────────────────────
P (Partition Tolerance)                 A (Availability)
                  \            /
                   \    AP    /
    Cassandra ●     \────────/ ← NoSQL territory
    DynamoDB ●       \      /
    Couchbase ●       \    /
                       \  /
                        \/
```

The nuance: modern systems like Google Spanner and CockroachDB do everything
possible to maximize availability *within* the CP camp. They use replication
factors of 3 or 5 to tolerate node failures while still serving consistent reads.

### When to Choose What: The Decision Framework

```python
# A decision framework for choosing your database architecture
def choose_database(
    needs_acid: bool,
    scale_beyond_single_node: bool,
    read_heavy: bool,
    write_throughput: str,   # "low", "medium", "high", "extreme"
    latency_requirement: str # "relaxed", "strict", "sub-millisecond"
) -> str:

    if not needs_acid and not scale_beyond_single_node:
        return "Traditional SQL (PostgreSQL / MySQL)"

    if not needs_acid and scale_beyond_single_node:
        return "NoSQL (Cassandra / DynamoDB / MongoDB)"

    if needs_acid and scale_beyond_single_node:

        if latency_requirement == "sub-millisecond":
            return "In-Memory NewSQL (VoltDB)"

        if write_throughput == "extreme" and read_heavy:
            return "HTAP NewSQL (TiDB) — separate OLTP/OLAP engines"

        if latency_requirement == "strict":
            # Need true external consistency
            return "Google Cloud Spanner (TrueTime guarantee)"

        # General distributed OLTP
        return "CockroachDB (open-source, multi-cloud, PostgreSQL-compatible)"

    return "Evaluate your specific constraints further"

# Test the framework
print(choose_database(
    needs_acid=True,
    scale_beyond_single_node=True,
    read_heavy=False,
    write_throughput="high",
    latency_requirement="strict"
))  # → Google Cloud Spanner

print(choose_database(
    needs_acid=True,
    scale_beyond_single_node=True,
    read_heavy=True,
    write_throughput="high",
    latency_requirement="relaxed"
))  # → TiDB (HTAP for mixed workloads)
```


### Key Interview Concepts at a Glance

The following concepts are the most frequently tested in system design and
database interviews. We group them by *theme* rather than just listing them:

**Theme 1: Consistency Guarantees**

- **Serializability**: Transactions appear to execute one at a time (strongest
isolation; default in CockroachDB)
- **External Consistency**: If T1 commits before T2 starts in real time, T1's
timestamp < T2's timestamp (Google Spanner via TrueTime; stronger than
serializability)
- **Snapshot Isolation**: Each transaction sees a consistent snapshot of the
DB; implemented via MVCC in most NewSQL systems

**Theme 2: Replication and Fault Tolerance**

- **Quorum writes**: A write requires `(n/2) + 1` acknowledgments before commit
- **Leader election**: Raft/Paxos automatically elect a new leader if the
current one fails; typically completes in ~150–300ms
- **Replication factor**: Most NewSQL systems default to 3 replicas per range
(tolerates 1 failure); production setups use 5 (tolerates 2 failures)

**Theme 3: Scalability Mechanisms**

- **Range splits**: When a range grows too large, it automatically splits into
two smaller ranges and rebalances across nodes
- **Range merges**: When ranges become too small, they're merged to reduce
overhead
- **Leaseholder**: In CockroachDB, the Raft leader for a range is called the
"leaseholder" — it handles all reads and coordinates writes for that range

**Theme 4: The Trade-offs You Must Articulate**


| Trade-off | NewSQL position | Rationale |
| :-- | :-- | :-- |
| Consistency vs. Availability | Prefers consistency | ACID is non-negotiable |
| Latency vs. Durability | Adds ~2-7ms per commit | Quorum write cost |
| Flexibility vs. Schema | Fixed schema | SQL requires structure |
| Operational Simplicity | More complex than single-node | Distributed coordination |
| Cost vs. Scalability | More expensive than NoSQL | But cheaper than outages |


---

## Part XI: Advanced Concepts — HTAP and the Unified Database Vision

### The Traditional Separation: OLTP vs. OLAP

For decades, we ran two separate database ecosystems side-by-side:

- **OLTP** (Online Transaction Processing): Row-oriented, optimized for fast
writes and point reads (e.g., "get this user's balance")
- **OLAP** (Online Analytical Processing): Column-oriented, optimized for
aggregations and scans (e.g., "compute the total revenue by region last month")

Data was periodically extracted from OLTP systems and loaded into OLAP data
warehouses via ETL (Extract, Transform, Load) pipelines — introducing hours of
latency between a transaction and its availability in analytics.

### HTAP: The Convergence

TiDB's HTAP architecture eliminates this separation. The same data is available
in both row format (TiKV) for transactions and column format (TiFlash) for
analytics, with *real-time replication* between the two:

```
HTAP Data Flow in TiDB
════════════════════════════════════════════════════════════════
                          ┌─────────────┐
  Application             │  TiDB SQL   │  Client Query Layer
  Writes ────────────────►│   Server    │
                          └──────┬──────┘
                                 │
                    ┌────────────▼─────────────┐
                    │   Raft Replication Log    │
                    └────────────┬─────────────┘
               ┌─────────────────┴──────────────────┐
               │                                    │
               ▼                                    ▼
     ┌──────────────────┐               ┌───────────────────┐
     │      TiKV         │               │     TiFlash        │
     │  (Row-oriented)   │               │  (Column-oriented) │
     │                  │               │                   │
     │  Users   Orders  │               │  Orders  Revenue  │
     │  ───────────────  │               │  ─────────────── │
     │  Alice   #1001   │◄─ Async ──────►│  #1001   $99.99  │
     │  Bob     #1002   │  replication  │  #1002   $149.99 │
     │                  │               │                   │
     │  Fast INSERTS    │               │  Fast column scan │
     │  Point reads     │               │  Aggregations     │
     └──────────────────┘               └───────────────────┘
          OLTP                                OLAP
     (Transactions)                       (Analytics)

  Both stores are consistent — no ETL pipeline needed!
════════════════════════════════════════════════════════════════
```

*In Figure above, notice that both TiKV and TiFlash receive data from the same
Raft replication log — no ETL pipeline, no batch jobs, no hours of delay. An
analytical query issued moments after a transaction can see the latest
transactional data. This is the HTAP breakthrough.*

---

## Part XII: Putting It in Context — A Realistic Architecture

Let's ground everything with a concrete interview scenario: a global fintech
application processing payments.

**Requirements:**

- 1M transactions/second at peak
- Users in US, EU, and Asia
- Strict ACID guarantees (regulatory requirement)
- Real-time fraud detection (needs live analytics)
- 99.999% availability (five-nines)

**The NewSQL Solution:**

```
Global Fintech Architecture with NewSQL
══════════════════════════════════════════════════════════════
  US-East                EU-West               APAC-Singapore
  ┌─────────────┐        ┌─────────────┐       ┌─────────────┐
  │  App Tier   │        │  App Tier   │       │  App Tier   │
  │ (Stateless) │        │ (Stateless) │       │ (Stateless) │
  └──────┬──────┘        └──────┬──────┘       └──────┬──────┘
         │                      │                     │
  ┌──────▼──────────────────────▼─────────────────────▼──────┐
  │                  CockroachDB / Cloud Spanner              │
  │                    (Global NewSQL Cluster)                │
  │                                                          │
  │  Range: /accounts/US/    Range: /accounts/EU/   ...     │
  │   Leader: US-East           Leader: EU-West             │
  │   Replicas: EU, APAC        Replicas: US, APAC          │
  │                                                          │
  │  ┌──────────────────────────────────────────────────┐   │
  │  │              Raft / Paxos Consensus               │   │
  │  │         (per-range, across all regions)           │   │
  │  └──────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────┘
         │
  ┌──────▼────────────────────────┐
  │  Real-time Fraud Analytics    │
  │  (TiFlash / BigQuery Sink)    │
  │  Live ML scoring, no ETL lag  │
  └───────────────────────────────┘
══════════════════════════════════════════════════════════════
```

**Why NewSQL here instead of NoSQL?**

- Regulatory requirement mandates ACID — NoSQL is off the table
- Global user base mandates horizontal scale — traditional SQL hits its ceiling
- Fraud detection needs live data — HTAP eliminates the ETL bottleneck
- Five-nines availability requires automatic failover — Raft provides it

**What you're trading:**

- ~5-7ms additional latency per cross-region write (Raft quorum roundtrip)
- Increased operational complexity vs. a single PostgreSQL instance
- Higher infrastructure cost vs. NoSQL (but far lower than consistency-bug
remediation costs in financial systems)

---

## Closing Thoughts: The Architecture Mindset

NewSQL is not a silver bullet. It is a carefully engineered response to a
specific category of problems: applications that need *both* the relational
expressiveness of SQL *and* the elastic scale of distributed systems, without
sacrificing the ACID guarantees that make data trustworthy.

Understanding NewSQL deeply means understanding:

- Why traditional SQL couldn't scale (architectural overhead, not the language)
- Why NoSQL's trade-offs are unacceptable for transactional workloads
- How Raft and Paxos provide consensus in the presence of failure
- How MVCC gives us non-blocking reads with consistent snapshots
- How TrueTime solves the timestamp problem with physics instead of software
- How HTAP collapses the OLTP/OLAP boundary for real-time intelligence

In a senior engineering interview, when the interviewer asks "what database would
you choose for a globally distributed payment system?", the answer is never
"just use Postgres" and never "just use Cassandra." The answer is a demonstrated
understanding of trade-offs — and NewSQL's elegant reframing of the
scalability-consistency tension is exactly the kind of depth that separates
strong candidates from exceptional ones.

---
