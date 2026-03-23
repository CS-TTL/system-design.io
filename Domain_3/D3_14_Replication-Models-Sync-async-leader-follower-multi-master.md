
# Replication Models: Sync/Async, Leader-Follower, and Multi-Master

## A Curriculum for Software Engineering Interviews
---

## The Newspaper Office Analogy

Before we write a single line of code or draw a single diagram, let us ground
ourselves in something entirely human: a busy newspaper editorial room circa 1985.

Picture the *Chicago Tribune* on election night. In the center of the room sits
the **Editor-in-Chief** — the single authority. Every journalist funnels their
stories through her. She edits, approves, and stamps the final copy. Then she
passes it to a team of **copy runners** stationed around the room who race to
get identical copies of each page to the printing presses across the building.

The Editor-in-Chief is your **leader node**. The copy runners are your
**follower replicas**. The act of distributing approved pages is **replication**.

Now ask: what happens when a runner is slow? The editor might wait — that is
**synchronous** behavior. Or she might say "I'll send it, and trust you'll get
it eventually" — that is **asynchronous** behavior. And what if the *Tribune*
opens a second editorial office in New York that *also* accepts final-copy
decisions? Now you have **multi-master replication** — with all the conflict
drama that implies.

This analogy is the skeleton for everything we are about to build. By the end
of this article, you will not just memorize definitions — you will feel *why*
each model was invented, understand the pain it solves, and recognize the
new pain it introduces.

---

## Why Replication Exists at All

### The Single-Node Problem

Let us start with the simplest possible system: one database server, one
application, one copy of the data. It works beautifully — until it does not.

```

┌────────────┐        ┌────────────────────┐
│   Client   │───────>│   Database Server  │
└────────────┘        │  (Single Node)     │
└────────────────────┘

```

In Figure 1, all reads and writes go to a single node. The entire world
depends on this one machine. Notice there is no backup, no redundancy.

Three brutal realities will eventually hit this architecture:

1. **Hardware failure.** Disks fail. Power supplies blow. When that single
   node dies, your entire service goes with it.

2. **Geographic latency.** A user in Tokyo connecting to a server in
   New York will experience ~150ms of round-trip latency — a painful user
   experience for interactive applications.

3. **Read throughput ceiling.** A single node can only serve so many
   concurrent read queries before CPU, memory, and I/O become bottlenecks.
   You cannot read-scale a single machine indefinitely.

These three pain points — **availability**, **latency**, and **read
throughput** — are exactly the problems that replication was designed to
solve. Every model we discuss is a different strategy for addressing some
subset of these three.

> **Replication** is the act of maintaining multiple copies of the same data
> on different nodes, so that failures are survivable, reads can be
> distributed, and geographic proximity can be achieved.

---

## Part 1: The Atomic Unit — Leader-Follower Replication

### The Basic Concept

Leader-follower replication (also called primary-replica, or historically,
master-slave) is the foundational model. It is the "Team with One Captain"
pattern. One node — the **leader** — accepts all write operations. Every
other node — the **followers** — receive a stream of changes from the leader
and apply them to their own local copy.

```

                         WRITE
                           │
                           ▼
                    ┌────────────┐
                    │   LEADER   │ ◄── Single source of truth for writes
                    └─────┬──────┘
                          │  Replication Stream
              ┌───────────┼───────────┐
              ▼           ▼           ▼
       ┌──────────┐ ┌──────────┐ ┌──────────┐
       │Follower 1│ │Follower 2│ │Follower 3│
       └──────────┘ └──────────┘ └──────────┘
            │            │            │
           READ          READ         READ
    ```

*Figure 2: Leader-Follower topology. Writes flow exclusively to the Leader.
The replication stream carries change events downstream. Followers absorb
reads, distributing the query load. Notice that each follower is a read
endpoint — clients are typically routed to followers for SELECT queries.*

The elegance here is the **simplicity of writes**. Because only one node
accepts writes, there can never be a write-write conflict. All changes are
serialized through a single gate. This is exactly what happens in PostgreSQL
streaming replication and MySQL binary log replication — two of the most
widely deployed database replication systems in the world.

### How the Replication Stream Works

When the leader processes a write, it records the change in a special log
before (or as) it is applied. This log goes by different names in different
systems:

| System       | Log Name                  |
|--------------|---------------------------|
| PostgreSQL   | Write-Ahead Log (WAL)     |
| MySQL        | Binary Log (binlog)       |
| MongoDB      | Oplog (operations log)    |

Followers connect to the leader and continuously stream this log, replaying
every operation in the exact same order. Because the *order* of operations is
preserved, followers converge to the same state as the leader — eventually.

That word "eventually" is the seed of our first problem.

---

## Part 2: The First Fork — Synchronous vs. Asynchronous Replication

### The Core Tension

Here is where things get interesting. The leader has committed a write to its
own disk. Now it needs to send that change to followers. The fundamental
question is: **does the leader wait for followers to confirm receipt before
telling the client "success"?**

This single question splits the world into two replication strategies.

### Synchronous Replication

In **synchronous replication**, the leader does not confirm a write to the
client until at least one follower has acknowledged that it has durably
persisted the change.

```

Client         Leader          Follower
│               │                │
│──── WRITE ───>│                │
│               │──── replicate >│
│               │                │ (disk write)
│               │<── ACK ────────│
│<──── OK ──────│                │
│               │                │
│         [commit complete]      │

```

*Figure 3: Synchronous replication sequence. Notice that the client's "OK"
response is blocked until the Follower's acknowledgment (ACK) arrives back at
the Leader. The vertical time axis flows downward — the client waits during
the full round trip.*

**The promise:** If the leader crashes immediately after confirming the write,
the data is safe — it already lives on at least one follower. You have
**zero data loss** (RPO = 0, in recovery terminology).

**The price:** Every single write now pays the cost of a network round trip
to the follower. If the follower is slow, laggy, or experiencing network
issues, your writes stall. If the follower crashes entirely, the leader
**blocks all writes** until the follower recovers or is demoted.

```python
import time
import threading

class SynchronousLeader:
    def __init__(self, followers):
        self.data = {}
        self.followers = followers  # list of follower objects

    def write(self, key, value):
        # Step 1: Write locally
        self.data[key] = value

        # Step 2: Block until ALL sync followers confirm
        acks = []
        for follower in self.followers:
            ack = follower.replicate(key, value)  # blocking call
            acks.append(ack)

        # Step 3: Only return success after all acks received
        if all(acks):
            return {"status": "OK", "durable": True}
        else:
            # Rollback or flag inconsistency
            return {"status": "ERROR", "durable": False}
```

In the code above, notice that `follower.replicate()` is a **blocking call**
inside the loop. The leader waits. This models the network round-trip
faithfully. Any follower that hangs will freeze the entire `write()` method.

### Asynchronous Replication

In **asynchronous replication**, the leader writes locally and immediately
confirms "success" to the client. Replication to followers happens in the
background, without the client waiting.

```
  Client         Leader          Follower
    │               │                │
    │──── WRITE ───>│                │
    │               │ (local write)  │
    │<──── OK ──────│                │
    │               │                │
    │               │~~~ replicate ~~>│  (background, later)
    │               │                │
```

*Figure 4: Asynchronous replication. Observe that the client receives "OK"
before the follower has any knowledge of the write. The dashed arrow (~~~)
indicates a background, non-blocking replication event that happens after
the client transaction is complete.*

**The promise:** Writes are fast. The client experiences minimal latency.
Followers can be geographically distant without penalizing write performance.
The leader is never blocked by a slow or failing follower.

**The price:** If the leader crashes in the window *after* confirming to the
client but *before* the follower has replicated, that write **is permanently
lost**. The client believes their data is safe. It is not.

```python
import asyncio

class AsyncLeader:
    def __init__(self, followers):
        self.data = {}
        self.followers = followers
        self.replication_queue = asyncio.Queue()

    async def write(self, key, value):
        # Step 1: Write locally
        self.data[key] = value

        # Step 2: Immediately return OK — do NOT wait
        asyncio.create_task(self._replicate_background(key, value))
        return {"status": "OK", "durable": False}  # note: not yet durable!

    async def _replicate_background(self, key, value):
        # Runs concurrently, leader doesn't wait
        for follower in self.followers:
            await follower.replicate(key, value)
```

Notice the critical comment: `"durable": False`. The client gets "OK", but the
data is only safe on the leader's disk. The followers are playing catch-up.

### The Best of Both Worlds: Semi-Synchronous Replication

A hybrid approach called **semi-synchronous replication** emerged to thread
the needle between these two extremes. MySQL introduced this as a
first-class feature. The rule is: wait for *at least one* follower to
acknowledge, but do not wait for all of them.

> In MySQL, `rpl_semi_sync_master_enabled = 1` enables this behavior.
> The source waits until at least one replica has received and logged the
> events, then commits the transaction — guaranteeing no data loss even
> if the leader crashes, without the full latency cost of waiting for
> all followers.

PostgreSQL offers even finer control through its `synchronous_commit`
parameter, which supports multiple levels:


| PostgreSQL `synchronous_commit` Setting | Behavior |
| :-- | :-- |
| `off` | No wait, highest performance, data loss risk |
| `local` | Wait for local WAL write only |
| `remote_write` | Wait for follower to receive WAL (semi-sync) |
| `on` (default) | Wait for follower to flush WAL to disk |
| `remote_apply` | Wait for follower to apply and query-serve data |

*Table 1: PostgreSQL's replication sync ladder. Each level trades a bit of
performance for a stronger consistency guarantee. In practice, `on` is the
most common synchronous production setting.*

### Choosing Your Model

We can frame the decision as a straightforward tradeoff matrix:


| Dimension | Synchronous | Asynchronous | Semi-Sync |
| :-- | :-- | :-- | :-- |
| Write Latency | Higher | Lower | Moderate |
| Data Loss Risk | None (RPO = 0) | Yes (RPO > 0) | Minimal (1 node) |
| Leader Blocking | Yes | No | Minimal |
| Best For | Finance, banking | Analytics, CDN | Most OLTP systems |

*Table 2: Replication strategy comparison. "RPO" means Recovery Point
Objective — the maximum acceptable data loss window measured in time.*

---

## Part 3: Growing Pains — The Limitations of Leader-Follower

The leader-follower model is elegant, but it has three well-known stress
fractures that appear at scale.

### Problem 1: Replication Lag

Because followers are applying changes asynchronously, they may fall behind
the leader — sometimes by milliseconds, sometimes by seconds. This gap is
called **replication lag**.

```
  Timeline ───────────────────────────────────────────>

  Leader:   [W1]──[W2]──[W3]──[W4]──[W5]
                                  │
  Follower: [W1]──[W2]──[W3]──...│ (lagging here)
                                  │
                             Replication Lag
```

*Figure 5: Replication lag visualization. The Leader has processed writes
W1 through W5. The Follower is still applying W3. If a client reads from
the follower right now, they will get stale data — missing W4 and W5.*

This creates a phenomenon called **read-your-own-writes inconsistency**. A
user updates their profile picture, then immediately refreshes the page. If
the read hits a lagging follower, they see their old photo. The user thinks
the write failed and tries again — confusion and frustration ensue.

Mitigation strategies include:

- **Sticky sessions:** Route a user's reads to the leader (or a specific
follower) for a brief window after a write.
- **Read-after-write consistency:** After a write, track the write's log
sequence number (LSN) and only allow reads from followers that have
caught up to at least that LSN.
- **Monotonic reads:** Ensure a single user's subsequent reads always come
from the same replica, preventing "time travel" where they see newer
data and then older data.


### Problem 2: The Leader is a Write Bottleneck

All writes funnel through one node. As write volume grows, that single leader
becomes a bottleneck. You cannot scale writes simply by adding more followers
— followers only help with read throughput.

### Problem 3: Failover Complexity

When the leader crashes, a new one must be elected. This process — called
**leader election** or **failover** — is non-trivial:

- Which follower is most caught up? (The one with the smallest replication lag.)
- What if two followers both believe they are the new leader? (**Split brain** — a catastrophic scenario where two nodes both accept writes independently.)
- What happens to writes that the old leader accepted but followers never
received? (They must be discarded, causing data loss.)

Systems like **Raft** and **Paxos** are consensus algorithms designed
specifically to handle leader election safely. Tools like **etcd** and
**ZooKeeper** implement these algorithms so that database systems can rely
on them for failover coordination.

---

## Part 4: Multi-Master Replication — When One Captain is Not Enough

### The Motivation

Leader-follower works brilliantly for most applications. But consider two
scenarios where it breaks down:

**Scenario A — Multi-Datacenter deployments:**
Your application runs in three datacenters: US-East, EU-West, and AP-Southeast.
With a single leader in US-East, every write from Tokyo must travel to
Virginia and back — roughly 200ms round-trip. For an interactive application,
that is unacceptable.

**Scenario B — Offline-capable clients:**
Consider a mobile notes app. You need to write notes while offline, then
sync when connectivity returns. Who is the leader when the phone is offline?
There is none — the device itself must be able to accept writes locally.

Both scenarios share the same need: **multiple, geographically distributed
nodes that can each accept writes independently**. This is **multi-master
replication**, also called leader-leader or active-active replication.

### The Architecture

```
         US-EAST                    EU-WEST
    ┌─────────────┐            ┌─────────────┐
    │  MASTER  1  │◄──────────>│  MASTER  2  │
    │ (accepts W) │  Bidirect. │ (accepts W) │
    └──────┬──────┘  Replication└──────┬──────┘
           │                           │
     ┌─────┴─────┐               ┌─────┴─────┐
     │ Follower  │               │ Follower  │
     └───────────┘               └───────────┘

               AP-SOUTHEAST
          ┌─────────────┐
          │  MASTER  3  │
          │ (accepts W) │
          └──────┬──────┘
                 │
           ┌─────┴─────┐
           │ Follower  │
           └───────────┘
```

*Figure 6: Multi-master topology across three datacenters. Each datacenter
has its own Master that can accept writes from local clients with low latency.
Masters replicate to each other bidirectionally (the double-headed arrows).
Each Master also replicates down to local Followers for read scaling within
the datacenter. Notice that writes no longer need to cross continental
network boundaries.*

In this setup, a Tokyo user writes to Master 3. That write is applied locally
in milliseconds. Then Master 3 asynchronously replicates the change to
Masters 1 and 2 in the background. The user gets a fast response. The system
eventually converges to consistency.

The word "eventually" just became our biggest enemy.

### The Fundamental Problem: Conflicts

In a single-master system, write conflicts are impossible by definition —
all writes are serialized through one node. But in a multi-master system,
two different clients in two different datacenters can simultaneously modify
the same piece of data. When the replication streams arrive at each master,
they discover they have **diverged**.

Consider this scenario:

```
  t=0ms  User A (EU) reads  record: { title: "Draft Article" }
  t=0ms  User B (US) reads  record: { title: "Draft Article" }
  
  t=10ms  User A writes: { title: "Final Article" }   --> Master 2 (EU)
  t=10ms  User B writes: { title: "Published Story" } --> Master 1 (US)
  
  t=500ms  Masters sync:
           Master 1 receives: title = "Final Article"
           Master 2 receives: title = "Published Story"
           
           *** CONFLICT: Which title is correct? ***
```

*Figure 7 (text diagram): A classic write-write conflict. Both users read the
same initial state. Both write different values concurrently to different
masters. When replication delivers both changes to each master, the system
faces an irreconcilable difference. Neither master knows which write should
"win."*

This is the fundamental challenge of multi-master replication, and it has no
single perfect solution — only strategies with different tradeoffs.

---

## Part 5: Conflict Resolution Strategies

We can group conflict resolution strategies into three behavioral categories:
**Avoidance**, **Automatic Resolution**, and **Application-Level Resolution**.

### Category 1: Avoidance (Don't Let Conflicts Happen)

The cleanest solution to conflicts is preventing them in the first place.

**Partition writes by key space.** Assign each record to a "home" master.
User A's records always write to Master 1. User B's records always write to
Master 2. No record is ever updated from two different masters simultaneously.
This degrades gracefully into a form of sharding — but you lose the
"write anywhere" benefit.

**CRDTs (Conflict-Free Replicated Data Types).** This is the mathematically
elegant approach. CRDTs are data structures designed so that any two versions
can always be merged without conflict, regardless of order. A grow-only
counter, for instance, only allows increments — any two divergent copies
can be merged by taking the maximum value.

```python
class GrowOnlyCounter:
    """
    A CRDT that only allows increments.
    Any two replicas can merge without conflict.
    """
    def __init__(self, node_id, num_nodes):
        self.node_id = node_id
        self.counters =  * num_nodes  # per-node counter vector

    def increment(self):
        self.counters[self.node_id] += 1

    def value(self):
        return sum(self.counters)

    def merge(self, other_counter):
        """Merge is simply element-wise maximum. Always safe, always correct."""
        self.counters = [
            max(self.counters[i], other_counter.counters[i])
            for i in range(len(self.counters))
        ]

# Usage: Two replicas diverge, then merge
replica_1 = GrowOnlyCounter(node_id=0, num_nodes=2)
replica_2 = GrowOnlyCounter(node_id=1, num_nodes=2)

replica_1.increment()  # node 0 increments
replica_1.increment()  # node 0 increments again
replica_2.increment()  # node 1 increments independently

# When they sync, merge is deterministic:
replica_1.merge(replica_2)
print(replica_1.value())  # Always 3, regardless of sync order
```

The beauty of the `merge()` method is its commutativity — it does not matter
which replica merges into which, or in what order. The result is always the
same. This is the mathematical foundation behind systems like **Riak** and
**Redis CRDT** extensions.

### Category 2: Automatic Resolution (Let the System Decide)

When avoidance is not possible, the system can apply automatic heuristics.

**Last Writer Wins (LWW):** Assign each write a timestamp. When a conflict is
detected, the write with the *higher* timestamp wins. The other write is
silently discarded.

```python
import time

class LWWRegister:
    """Last Writer Wins Register — used in Cassandra, DynamoDB."""
    def __init__(self):
        self.value = None
        self.timestamp = 0

    def write(self, value, timestamp=None):
        ts = timestamp or time.time()
        if ts > self.timestamp:
            self.value = value
            self.timestamp = ts

    def merge(self, other):
        """Merge by keeping whichever write has the higher timestamp."""
        if other.timestamp > self.timestamp:
            self.value = other.value
            self.timestamp = other.timestamp

# Conflict scenario:
reg_us = LWWRegister()
reg_eu = LWWRegister()

reg_us.write("Published Story", timestamp=1000.01)
reg_eu.write("Final Article",   timestamp=1000.00)  # slightly earlier

# After merge, US write wins (higher timestamp)
reg_us.merge(reg_eu)
print(reg_us.value)  # "Published Story"
```

LWW is simple and widely used (Cassandra uses it as a default conflict
resolution strategy). Its dark side: **it silently drops data**. The write
from EU is permanently gone — and no one was notified.

**Version Vectors (Vector Clocks):** Rather than relying on wall-clock time
(which can skew across nodes), version vectors track a logical sequence
number per node. This allows the system to detect whether two writes
are in a causal relationship ("B happened after A") or truly concurrent
("A and B happened independently and we have a genuine conflict").

```python
class VersionVector:
    def __init__(self, node_id):
        self.node_id = node_id
        self.clock = {}  # { node_id: sequence_number }

    def increment(self):
        self.clock[self.node_id] = self.clock.get(self.node_id, 0) + 1

    def is_concurrent_with(self, other):
        """
        Two version vectors are concurrent if neither dominates the other.
        This means a genuine conflict exists.
        """
        self_gt = any(
            self.clock.get(n, 0) > other.clock.get(n, 0)
            for n in set(self.clock) | set(other.clock)
        )
        other_gt = any(
            other.clock.get(n, 0) > self.clock.get(n, 0)
            for n in set(self.clock) | set(other.clock)
        )
        return self_gt and other_gt  # both advanced = concurrent = conflict
```

Amazon's **Dynamo** paper (2007) popularized version vectors for distributed
databases. The system uses them to detect conflicts and surface them to the
application layer for resolution — rather than silently dropping data.

### Category 3: Application-Level Resolution (Let the Developer Decide)

For some conflict types, neither the system nor a simple heuristic can
determine the correct answer. Consider two users booking the same airline seat
from two different regional masters simultaneously. Both writes succeed locally.
When the replication stream arrives, the system detects a double-booking.
No timestamp trick can make one booking "more correct" than the other —
the business logic must handle this.

Application-level conflict resolution hooks allow developers to register
a callback function that is invoked when a conflict is detected:

```python
from enum import Enum

class ConflictResolutionStrategy(Enum):
    LAST_WRITE_WINS = "lww"
    FIRST_WRITE_WINS = "fww"
    CUSTOM = "custom"

class MultiMasterNode:
    def __init__(self, node_id, strategy=ConflictResolutionStrategy.LAST_WRITE_WINS):
        self.node_id = node_id
        self.data = {}
        self.strategy = strategy
        self.custom_resolver = None

    def register_conflict_resolver(self, resolver_fn):
        """
        resolver_fn(local_record, remote_record) -> resolved_record
        Developer supplies domain-specific logic here.
        """
        self.custom_resolver = resolver_fn
        self.strategy = ConflictResolutionStrategy.CUSTOM

    def apply_remote_write(self, key, remote_value, remote_ts):
        if key not in self.data:
            self.data[key] = {"value": remote_value, "ts": remote_ts}
            return

        local = self.data[key]
        if self.strategy == ConflictResolutionStrategy.LAST_WRITE_WINS:
            if remote_ts > local["ts"]:
                self.data[key] = {"value": remote_value, "ts": remote_ts}
        elif self.strategy == ConflictResolutionStrategy.CUSTOM:
            resolved = self.custom_resolver(local["value"], remote_value)
            self.data[key] = {"value": resolved, "ts": max(local["ts"], remote_ts)}

# Example: Custom resolver for a shopping cart (merge both carts)
def cart_conflict_resolver(local_cart, remote_cart):
    """Union of items — never lose an item from either cart."""
    return list(set(local_cart + remote_cart))

node = MultiMasterNode("node_us")
node.register_conflict_resolver(cart_conflict_resolver)
```

This pattern appears in CouchDB's conflict model, where the application
developer is explicitly given control over conflict resolution — the database
surfaces all conflicting revisions and the application merges them.

---

## Part 6: When to Use What — A Decision Framework

We have covered three major topologies. Now let us build a practical
decision framework for system design interviews.

### The Three Questions

When you encounter a replication question in an interview, answer these
three in order:

1. **What is your write pattern?** Single geographic region or multi-region?
2. **What is your tolerance for data loss?** Is "eventual consistency" acceptable?
3. **What is your consistency model?** Do users need to see their own writes
immediately? Do all users need the same view simultaneously?

### The Decision Map

```
  START
    │
    ▼
  Multi-region writes needed?
    │
    ├── NO ──────────────────────────────────>  LEADER-FOLLOWER
    │                                            │
    │                                            ▼
    │                                         Data loss tolerable?
    │                                            │
    │                                            ├── NO  --> Synchronous
    │                                            └── YES --> Asynchronous
    │
    └── YES ─────────────────────────────────>  MULTI-MASTER
                                                 │
                                                 ▼
                                              Conflict avoidance possible?
                                                 │
                                                 ├── YES --> Partition by key
                                                 │          or use CRDTs
                                                 └── NO  --> LWW / Vectors /
                                                            App-level resolver
```

*Figure 8: A decision flowchart for selecting a replication model. Start with
the fundamental write-geography question. Notice that multi-master is only
chosen when geographic write independence is required — it is not a free
upgrade. Its complexity cost is significant.*

### Real-World System Mapping

| System | Model | Why |
| :-- | :-- | :-- |
| PostgreSQL | Leader-Follower (async default) | Simplicity; sync available per-session |
| MySQL | Leader-Follower + Semi-Sync | Semi-sync balances durability/performance |
| MongoDB | Replica Set (Leader-Follower) | Automatic failover via Raft election |
| CouchDB | Multi-Master | Offline-first mobile use cases |
| Cassandra | Leaderless (multi-master variant) | Peer-to-peer, tunable consistency |
| DynamoDB | Multi-Master (within region) | Global Tables for cross-region writes |
| Google Spanner | Synchronous multi-region | Globally consistent via TrueTime API |

*Table 3: Production system replication choices. Notice that Spanner is the
only system that achieves both global write distribution AND strong
consistency — it does so using GPS-synchronized atomic clocks, a feat most
systems cannot replicate without Google's infrastructure.*

---

## Part 7: Replication and the CAP Theorem

No discussion of replication models is complete without connecting to the
CAP theorem — the theoretical backbone of distributed systems design.

The CAP theorem states that a distributed system can only guarantee **two** of
the following three properties simultaneously:

- **C**onsistency — every read receives the most recent write (or an error)
- **A**vailability — every request receives a response (not necessarily the most recent)
- **P**artition Tolerance — the system continues to operate despite network splits

When a network partition occurs (and in distributed systems, it **will** occur),
you must choose: do you return potentially stale data (choose A), or do you
return an error until consistency is restored (choose C)?

```
                    CAP Triangle

                    Consistency
                        /\
                       /  \
                      /    \
                     /  CP  \
                    /        \
                   /----------\
                  /     |      \
                 / CA   |  AP   \
                /       |       \
               ─────────────────
         Availability       Partition
                             Tolerance

   CP Examples: HBase, Zookeeper, PostgreSQL (sync)
   AP Examples: CouchDB, Cassandra, DynamoDB (eventual)
   CA: Only possible in single-node systems (no partition tolerance needed)
```

*Figure 9: The CAP triangle. Each vertex represents one guarantee. Real
systems live on the edges or in the interior — never at all three vertices.
Note: "CA" (consistency + availability without partition tolerance) is only
achievable in single-datacenter, single-node deployments.*

Here is how replication choices map onto CAP:

- **Synchronous leader-follower** → CP system. Writes are blocked during
partition, preserving consistency.
- **Asynchronous leader-follower** → AP system. Reads return (potentially stale)
data during partition, preserving availability.
- **Multi-master with eventual consistency** → AP system. All nodes stay
available, but different nodes may return different values temporarily.

> An important nuance: The PACELC model (proposed by Daniel Abadi) extends
> CAP by noting that even *without* a partition, there is always a tradeoff
> between **latency** and **consistency**. Synchronous replication has higher
> latency but stronger consistency — even in normal, partition-free operation.
> This is why the PACELC model is considered more practically useful than
> CAP alone in real production decisions.

---

## Part 8: Interview Patterns and What Interviewers Actually Want

In a system design interview, replication questions rarely come in isolation.
They are usually embedded in broader architecture problems. Here is how to
recognize and answer them confidently.

### Pattern 1: "Design a system for X with high availability"

Any time you hear "high availability," immediately mention replication. Walk
through the leader-follower model first. Explain that follower nodes ensure
reads continue even during leader failure. Then discuss your failover strategy.

**Talking points:**

- Automatic leader election via a consensus protocol (Raft/Paxos)
- Health checks and heartbeats to detect leader failure
- Minimum replication lag threshold before a follower can be promoted


### Pattern 2: "Your service needs to operate across multiple geographic regions"

This is the signal for multi-master. Explain the write-latency motivation first
(always show the problem before the solution). Then discuss the conflict
challenge candidly — interviewers value the honesty that multi-master is not
free.

**Talking points:**

- Each region has its own master for low-latency local writes
- Asynchronous cross-region replication
- Conflict resolution strategy (LWW for most cases, CRDT for counters/sets,
application-level for business-critical scenarios)


### Pattern 3: "How do you handle replication lag?"

This tests operational depth. The answer should progress through several
mitigation strategies:

1. **Monitoring:** Track replication lag as a first-class metric. Alert when
lag exceeds a threshold (e.g., > 30 seconds is typically dangerous).
2. **Read routing:** Route reads that need freshness directly to the leader.
3. **Read-after-write:** Track LSN per user, route reads to replicas that
have caught up to that LSN.
4. **Synchronous commit for critical operations:** Use `synchronous_commit = on`
for financial transactions; use async for analytics queries.
```python
class ReplicationAwareRouter:
    """
    Routes read queries to an appropriate replica
    based on acceptable lag and required freshness.
    """

    def __init__(self, leader, followers):
        self.leader = leader
        self.followers = followers

    def route_read(self, query, require_fresh=False, min_lsn=None):
        if require_fresh:
            # Critical read — only the leader is safe
            return self.leader.execute(query)

        if min_lsn is not None:
            # Find a follower that has caught up to the required LSN
            for follower in self.followers:
                if follower.current_lsn() >= min_lsn:
                    return follower.execute(query)
            # No follower caught up yet — fall back to leader
            return self.leader.execute(query)

        # Default: load-balance across all followers
        import random
        target = random.choice(self.followers)
        return target.execute(query)
```


---

## Part 9: A Complete Mental Model

Let us close by stepping back and unifying everything into a single, coherent
mental model.

**Replication is fundamentally about copies.** More copies mean more
resilience and more read capacity. But keeping copies identical costs time and
coordination.

The three models we covered represent three points on a spectrum of
**centralization**:

```
                MORE CENTRALIZED                         LESS CENTRALIZED
                      │                                         │
                      ▼                                         ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │  SINGLE LEADER        SEMI-SYNC          MULTI-MASTER           │
   │  (full sync)      (one ack minimum)    (all nodes write)        │
   │                                                                  │
   │  ← stronger consistency                  higher availability →  │
   │  ← simpler conflict model                harder to reason   →  │
   │  ← lower write throughput                higher write scale →  │
   └─────────────────────────────────────────────────────────────────┘
```

*Figure 10: The replication centralization spectrum. Moving left gains
consistency guarantees and operational simplicity. Moving right gains
availability and geographic flexibility. The ideal position depends
entirely on your application's requirements.*

There is no universally correct model. There is only the right model for
your specific combination of **write geography**, **consistency requirements**,
and **failure tolerance**. Great engineers do not memorize answers — they
internalize tradeoffs.

When you walk into your next system design interview and the word "database"
comes up, ask three questions internally:

1. Where are writes coming from? (Geography determines topology.)
2. What happens if we lose a write? (Tolerance determines sync mode.)
3. What happens if two nodes disagree? (Conflict strategy determines
multi-master feasibility.)

Answer those three questions, and you will always have a defensible,
well-reasoned replication architecture — one that reflects not just knowledge,
but genuine engineering judgment.

---

## Appendix: Quick Reference Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────────┐
│                  REPLICATION MODELS — QUICK REFERENCE                │
├────────────────────┬───────────────────┬─────────────────────────────┤
│ MODEL              │ KEY GUARANTEE     │ KEY RISK                    │
├────────────────────┼───────────────────┼─────────────────────────────┤
│ Leader-Follower    │ No write conflict │ Leader = single write        │
│ (Async)            │ High read scale   │ point + replication lag     │
├────────────────────┼───────────────────┼─────────────────────────────┤
│ Leader-Follower    │ Zero data loss    │ Write latency + leader       │
│ (Sync)             │ Strong consistency│ blocks on follower failure   │
├────────────────────┼───────────────────┼─────────────────────────────┤
│ Semi-Synchronous   │ 1-node durability │ Slightly higher latency      │
│                    │ Fast writes       │ than pure async              │
├────────────────────┼───────────────────┼─────────────────────────────┤
│ Multi-Master       │ Multi-region write│ Write conflicts, complex     │
│ (Active-Active)    │ High availability │ resolution logic required    │
└────────────────────┴───────────────────┴─────────────────────────────┘
```

*This cheat sheet summarizes the one key guarantee and one key risk per
model. In an interview, pair each model with a real-world system that uses it:
PostgreSQL (leader-follower), MySQL semi-sync (semi-synchronous), and
CouchDB or DynamoDB Global Tables (multi-master).*

```

***

The article draws on established distributed systems theory around leader-follower topology, synchronous and asynchronous replication tradeoffs, PostgreSQL's `synchronous_commit` levels, MySQL's semi-synchronous replication plugin, multi-master conflict patterns, CRDT-based conflict avoidance, Last Writer Wins strategy used in Cassandra, and the CAP theorem's role in replication design decisions.