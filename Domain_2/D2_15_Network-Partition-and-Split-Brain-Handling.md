
# Network Partition and Split-Brain Handling

---

## Part 1 — The Bridge: A Tale of Two City Halls

Imagine a city that spans two islands — East Isle and West Isle — connected
by a single bridge. Both islands have their own City Hall, but they operate as
**one unified government**. Every morning, a courier crosses the bridge with
a synchronized ledger: births, deaths, tax records, property filings. Both
City Halls stay in perfect agreement.

Then, one winter, the bridge floods. For three days, neither island can
communicate with the other. Life continues on both sides. People are born.
Properties are sold. Disputes are resolved. Both City Halls keep making
decisions, independently, **as if they were the sole authority**.

When the bridge reopens and the couriers finally exchange ledgers, they
discover a nightmare: the same property has been sold to two different
families. A citizen declared dead on West Isle applied for a permit on East
Isle. The records are contradictory, and both sides believe they are correct.

This is not a story about city planning. **This is exactly what happens in a
distributed database during a network partition.**

This scenario — where two (or more) segments of a system independently
believe they are the single source of truth — has a name in distributed
computing: **Split-Brain**. Understanding how it happens, why it is
catastrophic, and how modern systems defend against it is one of the most
important concepts you will face in a system design interview.

We are going to build this understanding from the ground up.

---

## Part 2 — The Atomic Unit: What Is a Network Partition?

Before we tackle split-brain, we need a precise definition of the event
that causes it.

### Nodes and the Illusion of a Cluster

In a distributed system, we have a set of **nodes** — independent servers
that collectively act as one logical system. These nodes constantly
communicate by exchanging **heartbeat messages**: small, frequent signals
that say, *"I'm alive, I'm here, and here is my current state."*

```

svgbob
+---------+      heartbeat      +---------+
|  Node A |<------------------->|  Node B |
+---------+                     +---------+
^                               ^
|          heartbeat            |
+--------->  Node C  <----------+
+---------+

```

*In the diagram above, all three nodes are exchanging heartbeats freely.
This is the healthy, "normal operations" state.*

A **network partition** occurs when the network link between some nodes
fails — not the nodes themselves, but the communication channel between
them. The nodes are still running. They are still processing requests. But
they can no longer talk to each other.

```

svgbob
+---------+    X  PARTITION  X   +---------+
|  Node A |   /  /  /  /  /  /  |  Node B |
+---------+                      +---------+
^                                ^
|                                |
+----> Node C? (which side?) <---+
+---------+

```

*In the diagram above, a partition has severed the link between Node A and
Node B. Node C now faces a cruel choice: which side does it belong to?*

The critical insight is this: **from Node A's perspective, Node B appears to
be dead**. Node A stops receiving heartbeats. It has no way to distinguish
between "Node B crashed" and "the network link failed." This is the root
of all the trouble that follows.

### Why the Confusion Matters

If Node A concludes "Node B is dead," it might promote itself (or another
node) to become the new leader. Meanwhile, Node B has drawn the same
conclusion about Node A and has **also** promoted itself.

Suddenly, you have two leaders. Two sources of truth. Two nodes accepting
writes. This is split-brain.

---

## Part 3 — The CAP Theorem: The Unavoidable Trade-Off

To understand why split-brain is so hard to solve, we need to understand
a foundational constraint of distributed systems: **the CAP Theorem**,
formalized by Eric Brewer in 2000.

The theorem states that a distributed system can only guarantee **two** of
the following three properties simultaneously:

| Property | What It Means |
|---|---|
| **Consistency (C)** | Every read receives the most recent write, or an error |
| **Availability (A)** | Every request receives a (non-error) response |
| **Partition Tolerance (P)** | The system continues operating despite network partitions |

The critical realization — and the one many engineers miss — is that **P is
not really optional**. Networks fail. In any real-world distributed system
running across multiple machines, data centers, or cloud regions, you
*will* experience partitions. You must tolerate them.

This means the real choice, during a partition, is always:
**C vs. A**. Do you choose Consistency or Availability?

```

svgbob
CAP THEOREM

            +-------------+
            |      C      |
            |  Consistency|
            +------+------+
           /        \
          /          \
    +------+          +------+
|  CP  |          |  AP  |
+------+          +------+
\          /
\        /
+------+------+
|      P      |
| Partition   |
| Tolerance   |
+-------------+

CA: Theoretical only (no real partition tolerance)
CP: Choose consistency; reject writes when uncertain
AP: Choose availability; allow writes; risk divergence

```

*The diagram above maps the three corners of the CAP triangle. In practice,
all distributed systems fall into either CP or AP territory. Notice that
'CA' systems exist only in theory — or in single-node setups where
partitions are not possible.*

**CP Systems (e.g., HBase, Zookeeper, etcd):** When a partition occurs,
the minority partition refuses writes and returns an error. Data stays
consistent but the system becomes partially unavailable.

**AP Systems (e.g., Cassandra, DynamoDB, CouchDB):** When a partition
occurs, all nodes continue accepting writes. When the partition heals,
the system has to reconcile conflicting data — a process called
**conflict resolution**.

Neither choice is "wrong." It depends entirely on your use case. A banking
transaction ledger *must* be CP. A social media "likes" counter can
afford to be AP.

---

## Part 4 — The Broken State: Anatomy of a Split-Brain Scenario

Now let us watch split-brain unfold in slow motion. We will use a classic
three-node cluster with a **primary/replica** (leader/follower) setup.

### The Setup

We have three nodes: `Leader`, `Replica-1`, and `Replica-2`. The Leader
accepts all writes and replicates them to the replicas. Clients always
write to the Leader.

```

svgbob
CLIENT WRITES
|
v
+---------+
|  LEADER |  <--  Accepts all writes
+---------+
/     \
/       \
+----------+ +----------+
| REPLICA  | | REPLICA  |
|    1     | |    2     |
+----------+ +----------+
Replication of writes

```

### The Partition Event

Now a network switch fails. `Replica-1` can no longer communicate with
`Leader` or `Replica-2`. From `Replica-1`'s viewpoint, the other two
nodes have vanished.

```

svgbob
CLIENT-A                  CLIENT-B
|                          |
v                          v
+---------+    PARTITION   +----------+
|  LEADER |  X  X  X  X   | REPLICA-1|
+---------+                +----------+
|                     (Thinks leader is dead.
|                      Promotes itself to leader!)
+----------+
| REPLICA-2|
+----------+

PARTITION ISLAND 1        PARTITION ISLAND 2
[Leader + Replica-2]      [Replica-1 (now self-promoted)]

```

*In the diagram above, the X's represent the broken network link.
Replica-1, isolated on its own island, has not received a heartbeat from
the Leader in 300ms. It initiates a leader election and, since no one
can contest it, crowns itself the new leader. It then starts accepting
writes from Client-B.*

*Meanwhile, the original Leader is perfectly healthy and continues
accepting writes from Client-A. We now have two nodes writing to two
separate state machines.*

### The Aftermath: Data Divergence

Suppose the following writes happen concurrently:

```python
# Island 1 (Original Leader)
client_a.write(key="account_balance", value=5000)  # User deposits $5000
client_a.write(key="account_balance", value=4500)  # User withdraws $500

# Island 2 (Self-Promoted Replica-1)
client_b.write(key="account_balance", value=4800)  # User withdraws $200
client_b.write(key="account_balance", value=3800)  # User buys plane ticket
```

When the partition heals, both islands synchronize. **What is the correct
balance?** There is no deterministic answer without a conflict resolution
strategy. This is split-brain in its most dangerous form: silent data
corruption.

---

## Part 5 — Layer 1 Fix: Quorum-Based Consensus

The first and most foundational defense against split-brain is **quorum**.
The idea is elegantly simple: **a node may only take authoritative action
if it has the agreement of a majority (quorum) of all nodes.**

### The Quorum Formula

For a cluster of `N` nodes, a quorum requires:

$\text{Quorum} = \lfloor N/2 \rfloor + 1$

For a 3-node cluster: quorum = 2.
For a 5-node cluster: quorum = 3.
For a 7-node cluster: quorum = 4.

The genius of this formula is that **two separate partitions can never
*both* achieve quorum simultaneously**, because that would require more
than `N` nodes in total.

```
svgbob
  5-NODE CLUSTER PARTITIONED INTO 3 + 2

  PARTITION A                PARTITION B
  [Node1, Node2, Node3]      [Node4, Node5]
  Count = 3                  Count = 2
  Quorum? YES (3 >= 3)       Quorum? NO  (2 < 3)

  Partition A = ACTIVE        Partition B = DEGRADED
  (Can elect leader,          (Cannot elect leader,
   accept writes)              must refuse writes)
```

*Notice in the diagram that only one partition can ever reach the majority
threshold. This is the mathematical guarantee that prevents two "leaders"
from emerging simultaneously.*

### Python Simulation: Quorum Check

```python
import math
from typing import List

class QuorumChecker:
    def __init__(self, total_nodes: int):
        self.total_nodes = total_nodes
        self.quorum_size = math.floor(total_nodes / 2) + 1

    def has_quorum(self, reachable_nodes: List[str]) -> bool:
        """
        Returns True if this partition has quorum and may act as leader.
        """
        count = len(reachable_nodes)
        print(f"[QuorumCheck] Reachable: {count}/{self.total_nodes} "
              f"(need {self.quorum_size})")
        return count >= self.quorum_size

    def should_accept_writes(self, reachable_nodes: List[str]) -> bool:
        if self.has_quorum(reachable_nodes):
            print("[Decision] Quorum achieved. Accepting writes.")
            return True
        else:
            print("[Decision] No quorum. REFUSING writes to prevent split-brain.")
            return False


# Simulate a 5-node cluster partition
checker = QuorumChecker(total_nodes=5)

print("--- Partition A (3 nodes) ---")
checker.should_accept_writes(["node1", "node2", "node3"])

print("\n--- Partition B (2 nodes) ---")
checker.should_accept_writes(["node4", "node5"])
```

**Output:**

```
--- Partition A (3 nodes) ---
[QuorumCheck] Reachable: 3/5 (need 3)
[Decision] Quorum achieved. Accepting writes.

--- Partition B (2 nodes) ---
[QuorumCheck] Reachable: 2/5 (need 3)
[Decision] No quorum. REFUSING writes to prevent split-brain.
```

Quorum is the backbone of production consensus algorithms like **Paxos**
and **Raft**, and it is used in virtually every modern distributed
datastore: etcd, Zookeeper, Consul, MongoDB, and CockroachDB.

---

## Part 6 — Layer 2 Fix: The Raft Consensus Algorithm

Quorum tells us *when* to act. But we need a protocol that describes
*how* nodes elect a leader and replicate state. The most widely understood
algorithm in modern distributed systems is **Raft**, introduced by Diego
Ongaro and John Ousterhout in 2014.

Raft was explicitly designed to be more understandable than its predecessor
Paxos, which was notoriously difficult to implement correctly.

### The Three Roles

Every node in a Raft cluster occupies exactly one role at any given time:

- **Leader:** The single authority. Accepts writes, replicates log entries
to followers, sends periodic heartbeats.
- **Follower:** Passive. Accepts log entries from the leader. Votes in
elections.
- **Candidate:** Temporary role. A follower that has timed out waiting for
a heartbeat and is attempting to become leader.

```
svgbob
  STATE MACHINE FOR A RAFT NODE

  +----------+  timeout, no heartbeat   +-----------+
  | FOLLOWER |------------------------->| CANDIDATE |
  +----------+                          +-----------+
       ^                                      |
       |                          wins majority votes
       |  discovers                           |
       |  current leader                      v
       |                               +----------+
       +-------------------------------+  LEADER  |
                                       +----------+
                                            |
                                   sends heartbeats
                                   to all followers
```


### Leader Election: The Heartbeat Timer

The leader election mechanism is the key to Raft's split-brain prevention.
Here is how it works:

1. Every follower maintains an **election timeout** — a random timer
between (typically) 150ms and 300ms.
2. When the follower receives a heartbeat from the leader, it **resets**
this timer.
3. If the timer expires before a heartbeat arrives, the follower
transitions to **Candidate**, increments its **term** (a monotonically
increasing epoch counter), and requests votes from all other nodes.
4. A node grants its vote to the first candidate it hears from in a
given term, if the candidate's log is at least as up-to-date as its own.
5. The first candidate to collect votes from a **quorum** of nodes becomes
the new leader.

The random timeout is critical — it ensures that not all nodes time out
simultaneously, preventing a chaotic multi-way election tie.

### Raft's Split-Brain Guarantee

Consider a partition scenario. The old leader ends up in the **minority
partition** (fewer than quorum nodes). It continues sending heartbeats —
but receives no acknowledgment from followers. It cannot commit new log
entries because it cannot achieve quorum replication.

Meanwhile, the **majority partition** elects a new leader (correctly, with
quorum votes). When the old leader's partition rejoins the cluster, it
detects a **higher term** number in messages from the new leader and
**immediately steps down**, transitioning back to follower state. Its
uncommitted log entries are discarded and overwritten.

```
svgbob
  RAFT PARTITION SCENARIO

  BEFORE PARTITION (Term 1):
  Leader(A) ----replicates----> Follower(B), Follower(C), Follower(D), Follower(E)

  PARTITION OCCURS:
  Island 1: [A (old leader), B]     Island 2: [C, D, E]
  (minority, 2 of 5)                (majority, 3 of 5)

  Island 2 elects new leader:
  C becomes Leader in Term 2
  (C, D, E have quorum)

  Island 1 cannot commit (no quorum)
  A is stuck — writes fail silently

  PARTITION HEALS:
  A receives message with Term=2
  A: "My term is 1, theirs is 2. I am no longer leader."
  A transitions to FOLLOWER.
  A's uncommitted entries are overwritten by C's log.
```


### Python Sketch: Term-Based Leadership Check

```python
from dataclasses import dataclass, field
from enum import Enum

class NodeRole(Enum):
    FOLLOWER = "follower"
    CANDIDATE = "candidate"
    LEADER = "leader"

@dataclass
class RaftNode:
    node_id: str
    current_term: int = 0
    role: NodeRole = NodeRole.FOLLOWER
    voted_for: str = None
    log: list = field(default_factory=list)

    def receive_heartbeat(self, leader_id: str, leader_term: int):
        """
        Process an incoming heartbeat from a leader.
        If the leader's term is higher, step down immediately.
        """
        if leader_term > self.current_term:
            print(f"[{self.node_id}] Discovered higher term {leader_term}. "
                  f"Stepping down from {self.role.value} to follower.")
            self.current_term = leader_term
            self.role = NodeRole.FOLLOWER
            self.voted_for = None
        elif self.role == NodeRole.LEADER and leader_term == self.current_term:
            # Two leaders in same term — should never happen in correct Raft
            raise RuntimeError("SPLIT-BRAIN DETECTED: two leaders in same term!")

    def start_election(self, cluster_size: int):
        """Initiate a leader election."""
        self.current_term += 1
        self.role = NodeRole.CANDIDATE
        self.voted_for = self.node_id
        print(f"[{self.node_id}] Starting election for Term {self.current_term}")

    def receive_vote(self, votes_received: int, cluster_size: int):
        """Check if we have enough votes to become leader."""
        quorum = (cluster_size // 2) + 1
        if votes_received >= quorum:
            self.role = NodeRole.LEADER
            print(f"[{self.node_id}] Won election! Became leader for "
                  f"Term {self.current_term} with {votes_received} votes.")
        else:
            print(f"[{self.node_id}] Not enough votes "
                  f"({votes_received}/{quorum} needed). Reverting to follower.")
            self.role = NodeRole.FOLLOWER


# Simulate a stale leader detecting a newer term
old_leader = RaftNode(node_id="node-A", current_term=1,
                      role=NodeRole.LEADER)
print(f"Initial state: {old_leader.node_id} is {old_leader.role.value} "
      f"in Term {old_leader.current_term}")

# Old leader receives a heartbeat from the new leader (Term 2)
old_leader.receive_heartbeat(leader_id="node-C", leader_term=2)
print(f"Updated state: {old_leader.node_id} is now {old_leader.role.value} "
      f"in Term {old_leader.current_term}")
```


---

## Part 7 — Layer 3 Fix: Fencing Tokens

Quorum and Raft are remarkably effective, but they operate on the
assumption that nodes behave correctly — that a node, upon learning it
is no longer leader, immediately stops accepting writes.

In reality, a **slow** or **garbage-collected** node might pause for
seconds or minutes, then resume activity while still believing it is
the leader. This is the **"zombie leader"** problem.

Fencing tokens provide a bulletproof second line of defense.

### The Pain Point

Imagine the following sequence of events:

1. Node A is the leader. It acquires a lock on a shared resource.
2. Node A pauses for 40 seconds due to a JVM garbage collection stop.
3. The rest of the cluster times out, declares Node A dead, and elects
Node B as the new leader.
4. Node B acquires the lock and begins writing.
5. Node A's GC pause ends. It resumes, still thinking it is the leader,
and begins writing to the shared resource.

We now have two nodes writing concurrently. A race condition. Potential
data corruption.

### The Fix: Monotonically Increasing Tokens

Every time a new leader is elected, the lock service issues a
**fencing token** — a monotonically increasing integer (1, 2, 3...).
Every write request to a shared resource **must include** the current
fencing token.

The shared resource (database, file system, etc.) maintains the **highest
token it has ever seen**. If it receives a write with a **lower** token, it
rejects it — no questions asked.

```
svgbob
  FENCING TOKEN FLOW

  Lock Service
  +----------+
  | Token: 33|   <--  Node A becomes leader, gets token 33
  +----------+

  [40s GC pause on Node A]

  +----------+
  | Token: 34|   <--  Node B becomes leader, gets token 34
  +----------+
        |
        |  Write(token=34, data="...")
        v
  +----------+
  | Storage  |  "Token 34 >= max_seen 33. ACCEPTED."
  | max=34   |
  +----------+

  [Node A resumes from GC pause]
        |
        |  Write(token=33, data="STALE DATA")
        v
  +----------+
  | Storage  |  "Token 33 < max_seen 34. REJECTED."
  | max=34   |
  +----------+
```

*In the diagram above, notice that the storage layer itself enforces the
token rule. Even if Node A is completely confused about its role,
the storage system acts as the final gatekeeper. This is the key insight:
we move the enforcement to the resource being protected, not just the
node claiming to be the leader.*

### Python Implementation: Fencing Token Guard

```python
import threading

class FencedStorage:
    """
    A storage system that enforces fencing tokens to prevent
    stale writes from zombie leaders.
    """

    def __init__(self):
        self._data = {}
        self._max_seen_token = 0
        self._lock = threading.Lock()

    def write(self, token: int, key: str, value: str) -> bool:
        with self._lock:
            if token < self._max_seen_token:
                print(f"[Storage] REJECTED write (token={token}, "
                      f"max_seen={self._max_seen_token}). "
                      f"Stale leader detected!")
                return False

            self._max_seen_token = max(self._max_seen_token, token)
            self._data[key] = value
            print(f"[Storage] ACCEPTED write (token={token}): "
                  f"{key} = {value}")
            return True

    def read(self, key: str):
        return self._data.get(key)


class LeaderNode:
    def __init__(self, node_id: str, fencing_token: int):
        self.node_id = node_id
        self.fencing_token = fencing_token

    def write(self, storage: FencedStorage, key: str, value: str):
        print(f"[{self.node_id}] Attempting write with token "
              f"{self.fencing_token}")
        return storage.write(self.fencing_token, key, value)


# Simulate stale leader scenario
storage = FencedStorage()

# New legitimate leader (token 34)
new_leader = LeaderNode("node-B", fencing_token=34)
new_leader.write(storage, "account:user_1", "balance=5000")

# Zombie old leader (token 33) resumes from GC pause
zombie_leader = LeaderNode("node-A", fencing_token=33)
zombie_leader.write(storage, "account:user_1", "balance=3000")  # REJECTED

print(f"\nFinal value: {storage.read('account:user_1')}")
```

**Output:**

```
[node-B] Attempting write with token 34
[Storage] ACCEPTED write (token=34): account:user_1 = balance=5000

[node-A] Attempting write with token 33
[Storage] REJECTED write (token=33, max_seen=34). Stale leader detected!

Final value: balance=5000
```


---

## Part 8 — Layer 4 Fix: STONITH (Shoot The Other Node In The Head)

Sometimes, software-level fencing is not enough. In high-availability
clusters managing shared physical storage — think SAN arrays, NFS mounts,
or raw block devices — a confused node writing to a shared disk can cause
**filesystem corruption** that no amount of software logic can undo.

For these scenarios, the industry developed **STONITH**: Shoot The Other
Node In The Head. Yes, that is its actual acronym, and it is exactly as
aggressive as it sounds.

STONITH is the practice of **physically isolating a suspect node** by
cutting off its power, disabling its network port, or issuing a hardware
reset through out-of-band management interfaces (IPMI, iDRAC, iLO).

The philosophy is brutal but sound: **it is better to forcibly shut down a
confused node than to let it corrupt shared state.** A node that is
powered off cannot write corrupted data. Full stop.

```
svgbob
  STONITH ARCHITECTURE

  Cluster Manager (Pacemaker/Corosync)
  +-----------------------------+
  | Detects split-brain risk    |
  | Decides which node to fence |
  +----------+------------------+
             |
             | IPMI/iDRAC command
             v
  +-----------+      +-----------+
  |  Node A   |      |  Node B   |
  |  (FENCED) |      | (ACTIVE)  |
  |  POWER OFF|      |           |
  +-----------+      +-----------+
  "You are dead.     "You are the
   We moved on."      sole leader."
```

STONITH is widely used in Red Hat High Availability clusters, SUSE Linux
Enterprise clusters, and any setup where shared storage is involved.
The key components are:

- **Power Fencing:** Cut power via IPMI. Most reliable. Node cannot run
at all.
- **Storage Fencing:** Use SCSI-3 Persistent Reservations to deny a
node's access to a shared disk.
- **Network Fencing:** Disable the network port on the switch via SNMP
or API call, isolating the node from all communication.

---

## Part 9 — Layer 5 Fix: Conflict Resolution for AP Systems

Everything we have discussed so far is aimed at **preventing** split-brain
by maintaining a single authoritative leader. But for **AP systems** that
deliberately allow divergent writes during a partition, we need a different
philosophy: **detect and resolve conflicts after the fact.**

### The Timestamp Approach (Last Write Wins)

The simplest strategy is **Last Write Wins (LWW)**: when merging two
conflicting values, keep the one with the later timestamp and discard the
older one.

This is what Cassandra does by default.

```python
from dataclasses import dataclass
from typing import Optional
import time

@dataclass
class VersionedValue:
    value: str
    timestamp: float  # Unix epoch

class LastWriteWinsStore:
    """
    A simple AP-style store that resolves conflicts using timestamps.
    Used in systems like Cassandra and DynamoDB.
    """

    def __init__(self):
        self._store: dict[str, VersionedValue] = {}

    def write(self, key: str, value: str, timestamp: Optional[float] = None):
        ts = timestamp or time.time()
        existing = self._store.get(key)

        if existing is None or ts > existing.timestamp:
            self._store[key] = VersionedValue(value=value, timestamp=ts)
            print(f"[LWW] Stored: {key} = '{value}' (ts={ts:.3f})")
        else:
            print(f"[LWW] Discarded stale write: '{value}' "
                  f"(ts={ts:.3f} <= existing ts={existing.timestamp:.3f})")

    def read(self, key: str) -> Optional[str]:
        entry = self._store.get(key)
        return entry.value if entry else None


# Simulate two islands writing concurrently during a partition
store = LastWriteWinsStore()

# Island 1 writes at T=1000.0
store.write("balance", "5000", timestamp=1000.0)

# Island 2 writes at T=1001.5 (later timestamp wins)
store.write("balance", "3800", timestamp=1001.5)

# Stale write from Island 1 arrives late
store.write("balance", "4500", timestamp=999.0)

print(f"\nFinal resolved value: {store.read('balance')}")
```

**Output:**

```
[LWW] Stored: balance = '5000' (ts=1000.000)
[LWW] Stored: balance = '3800' (ts=1001.500)
[LWW] Discarded stale write: '4500' (ts=999.000 <= existing ts=1001.500)

Final resolved value: 3800
```

The weakness of LWW is that it **silently discards data**. In the example
above, the Island 1 deposit of \$5000 is lost. For financial systems,
this is unacceptable. For a social media post "likes" counter, it is
probably fine.

### Vector Clocks: Causal Conflict Detection

For systems that need to detect *and present* conflicts to the
application layer — rather than silently dropping data — **vector clocks**
provide a more nuanced solution, famously used in Amazon's Dynamo.

A vector clock is a dictionary mapping each node to a logical counter.
Every write increments the writing node's counter. By comparing two
vector clocks, we can determine:

- `A happened before B` (A's clock is dominated by B's)
- `B happened before A` (B's clock is dominated by A's)
- `A and B are concurrent` (neither dominates — a true conflict!)

```python
from typing import Dict

VectorClock = Dict[str, int]

def compare_clocks(vc1: VectorClock, vc2: VectorClock) -> str:
    """
    Compare two vector clocks to determine their causal relationship.
    """
    all_nodes = set(vc1.keys()) | set(vc2.keys())

    vc1_ahead = any(vc1.get(n, 0) > vc2.get(n, 0) for n in all_nodes)
    vc2_ahead = any(vc2.get(n, 0) > vc1.get(n, 0) for n in all_nodes)

    if vc1_ahead and not vc2_ahead:
        return "vc1 dominates (vc1 happened after vc2)"
    elif vc2_ahead and not vc1_ahead:
        return "vc2 dominates (vc2 happened after vc1)"
    elif vc1_ahead and vc2_ahead:
        return "CONCURRENT CONFLICT: manual merge required"
    else:
        return "identical"


# Version from Island 1: Node A wrote twice, Node B once
island1_clock = {"node_a": 3, "node_b": 1, "node_c": 0}

# Version from Island 2: Node C wrote three times independently
island2_clock = {"node_a": 1, "node_b": 0, "node_c": 3}

result = compare_clocks(island1_clock, island2_clock)
print(f"Conflict detection result: {result}")

# Scenario where one clearly happened after the other
before_clock = {"node_a": 2, "node_b": 1}
after_clock  = {"node_a": 3, "node_b": 2}
print(f"Sequential writes: {compare_clocks(before_clock, after_clock)}")
```

**Output:**

```
Conflict detection result: CONCURRENT CONFLICT: manual merge required
Sequential writes: vc2 dominates (vc2 happened after vc1)
```

When a concurrent conflict is detected, the system surfaces **both
versions** to the application layer (or a human operator) for a semantic
merge. Amazon's shopping cart, for example, would merge two divergent
carts by taking the **union** of items — erring on the side of including
more items rather than losing a customer's selection.

---

## Part 10 — Real-World Systems: How They Handle It

Let us map all the concepts above to specific production systems you are
likely to be asked about in interviews.

### System-by-System Analysis

| System | Type | Split-Brain Strategy | Notes |
| :-- | :-- | :-- | :-- |
| **etcd** | CP | Raft consensus + quorum | Used by Kubernetes control plane |
| **ZooKeeper** | CP | ZAB protocol + quorum | Leader election, distributed locks |
| **Cassandra** | AP | Tunable consistency + LWW | `QUORUM` reads/writes configurable |
| **DynamoDB** | AP (default) | Vector clocks + LWW | Eventual consistency by default |
| **MongoDB (PSS)** | CP | Raft + majority write concern | `w: majority` prevents split-brain |
| **CockroachDB** | CP | Raft per range | Multi-region strong consistency |
| **PostgreSQL** | CA (single-node) | Patroni + etcd for HA | Uses external consensus for failover |

### Deep Dive: Kubernetes and etcd

Kubernetes stores all cluster state — pod definitions, service configs,
secrets — in **etcd**. Since etcd uses Raft, a Kubernetes control plane
**requires a majority of etcd nodes to be healthy** to make any changes.

This is why a standard production Kubernetes cluster runs **3 or 5 etcd
nodes** (never 2 or 4 — even numbers do not improve partition tolerance
because the majority threshold does not improve).

```
svgbob
  KUBERNETES ETCD CLUSTER (3 NODES)

  +-------+    +-------+    +-------+
  | etcd-1|    | etcd-2|    | etcd-3|
  | LEADER|<-->| FOLLOW|<-->| FOLLOW|
  +-------+    +-------+    +-------+
      |
   All K8s API writes go here
   (kubectl apply, pod scheduling, etc.)

  If etcd-1 partitions from etcd-2 and etcd-3:
  etcd-2 and etcd-3 elect a new leader (quorum = 2).
  etcd-1 becomes read-only until it rejoins.
  K8s cluster remains operational.
```


---

## Part 11 — The Partition Healing Problem

We have spent most of our time discussing what happens *during* a
partition. But what happens *when the partition heals*?

The process of two rejoining partitions reconciling their divergent state
is called **merge resolution**, and it is one of the trickiest problems in
distributed systems engineering.

### For CP Systems (Raft/Paxos)

The minority partition's uncommitted log entries are simply overwritten by
the majority partition's canonical log. This is safe because, by design,
the minority partition could never have committed any entries (it lacked
quorum). So there is nothing to "merge" — only data that was *attempted*
but never acknowledged to clients.

### For AP Systems (Cassandra, Dynamo)

The process is more complex. During a partition, both sides accepted writes.
The system now has two divergent histories. The merge process involves:

1. **Read Repair:** When a read is issued, the system queries multiple
replicas and detects divergence. It reconciles in the background.
2. **Anti-Entropy Repair:** Background processes (like Cassandra's
`nodetool repair`) compare Merkle trees across nodes to identify
divergent data ranges and synchronize them efficiently.
3. **Conflict Resolution Policy:** Apply LWW, vector clock comparison, or
application-level merge logic to resolve individual key conflicts.
```
svgbob
  ANTI-ENTROPY REPAIR WITH MERKLE TREES

  Node A's Merkle Tree       Node B's Merkle Tree
  (after partition)          (after partition)

       [ROOT: abc123]             [ROOT: def456]  <-- roots differ!
       /            \             /            \
  [hash1a]       [hash2a]   [hash1a]       [hash2b]  <-- only right subtree differs
  /     \        /     \    /     \        /     \
 [k1] [k2]    [k3] [k4]  [k1] [k2]    [k3'] [k4']

  By comparing Merkle tree hashes top-down,
  nodes can identify EXACTLY which key ranges diverged
  without transferring the entire dataset.
  Only [k3] and [k4] need to be synchronized.
```

*In the diagram above, the Merkle tree allows the system to identify
divergent data in O(log N) comparisons rather than scanning every key.
This is the efficiency trick that makes anti-entropy repair practical at
petabyte scale.*

---

## Part 12 — Interview Cheat Sheet: What Interviewers Actually Ask

After all the deep-dive content, let us synthesize the key patterns you
are likely to encounter in a system design or distributed systems
interview.

### Common Interview Questions and Directional Answers

**Q: "How does your distributed database prevent two leaders?"**

> We use a consensus algorithm like Raft. A node can only become a leader
> by collecting votes from a quorum (majority) of nodes. Since two
> partitions cannot simultaneously hold a majority of `N` total nodes,
> split-brain leadership is mathematically impossible. We also use
> fencing tokens as a second line of defense against zombie leaders.

**Q: "What happens to availability during a network partition?"**

> It depends on our CP vs. AP choice. In a CP system (like etcd), the
> minority partition becomes unavailable — it refuses writes to prevent
> inconsistency. In an AP system (like Cassandra), all nodes remain
> available, but we accept eventual consistency and resolve conflicts post-
> partition using strategies like LWW or vector clocks.

**Q: "How do you handle an odd vs. even number of nodes in a cluster?"**

> We always prefer odd numbers (3, 5, 7). Even numbers do not improve
> fault tolerance. A 4-node cluster has a quorum of 3, meaning it can only
> tolerate 1 failure — the same as a 3-node cluster. But the 4-node
> cluster costs 33% more. An even split (2+2) leaves both partitions
> below quorum, taking down the entire cluster. Odd numbers guarantee
> that at least one partition will always have majority.

**Q: "What is a fencing token and why do we need it?"**

> A fencing token is a monotonically increasing integer issued by a
> lock service every time a new leader is elected. All writes to shared
> resources must include the current token. The resource rejects any
> write with a token lower than the highest it has seen. This prevents
> zombie leaders — nodes that paused (e.g., GC pause) and resumed
> after a new leader was elected — from corrupting shared state.

### The Decision Framework

When designing a distributed system, always ask these three questions
in order:

```
svgbob
  PARTITION HANDLING DECISION TREE

  Does a network partition occur?
              |
              v
  Can your application tolerate stale reads?
       /                      \
      YES                      NO
       |                        |
       v                        v
  Choose AP System          Choose CP System
  (Cassandra, Dynamo)       (etcd, ZooKeeper,
  Use LWW or vector clocks   CockroachDB)
  for conflict resolution    Minority partition
                             becomes unavailable
                                    |
                                    v
                        Do you have shared physical storage?
                             /              \
                            YES              NO
                             |               |
                             v               v
                          Add STONITH     Fencing tokens
                          for hardware    are sufficient
                          isolation
```


---

## Conclusion: The Bridge Rebuilt

We started with two city halls cut off by a flood. We have now seen the
technical architecture that prevents their ledger catastrophe.

The modern distributed system's answer to network partitions is
**defense in depth**:

- **Quorum** ensures only one partition can hold authority at a time
- **Raft/Paxos** provides the precise protocol for leader election and
log replication within that quorum constraint
- **Fencing tokens** guard against zombie leaders at the resource level
- **STONITH** provides the nuclear option — hardware-level isolation for
shared-storage environments
- **Conflict resolution** (LWW, vector clocks, Merkle trees) gives AP
systems a principled way to reconcile divergence after healing

No single mechanism is sufficient. Production systems combine all of these
layers. When you are asked about split-brain in an interview, the
sophisticated answer is not just "use Raft" — it is to demonstrate that
you understand *why* each layer is necessary and *what gap* it fills that
the previous layer leaves open.

The bridge between East Isle and West Isle is now guarded not just by a
courier, but by a mathematician, a hardware engineer, a protocol designer,
and a clock keeper — each watching for the moment the flood returns.

---

*Key terms for review: Network Partition · CAP Theorem · Split-Brain ·
Quorum · Raft · Paxos · Fencing Token · STONITH · Vector Clock ·
Last Write Wins (LWW) · Anti-Entropy Repair · Merkle Tree · Leader
Election · Term Number · Zombie Leader*

```
