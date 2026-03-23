
# Load Balancing: The Art of Distributing Work at Scale


***

## The Bridge: Think Like a Grocery Store Manager

Picture a busy Saturday afternoon at a large grocery store. One checkout lane is open. The line stretches past the frozen food aisle. Customers are frustrated, items are melting, and the single cashier is visibly overwhelmed — scanning items as fast as humanly possible, but still losing the battle against the crowd.

Now the store manager walks in. She doesn't hire a faster cashier. She opens ten more lanes.

Within minutes, the crowd disperses. Each lane processes customers at a comfortable pace. No single cashier is sprinting. The total throughput of the store increases dramatically, not because any individual worker got faster, but because the **work was distributed**.

This, at its core, is load balancing.

In software systems, your servers are the cashiers. Your incoming network requests are the customers. And the load balancer is the store manager — intelligently routing traffic so that no single machine drowns while others sit idle.

What makes this deceptively tricky is that the store manager must also handle:

- A cashier calling in sick (server going down)
- VIP customers who need to return to the same lane (session persistence)
- Lanes that process different item types (routing by request type)
- Rush hours that demand spinning up new lanes dynamically (auto-scaling)

By the end of this article, we'll have built a complete mental model for all of these challenges.

***

## Chapter 1: The Single-Server Problem

Let's start at the very beginning — with a single server.

In the early days of the web, a single machine hosted your entire application. Users sent requests, the server responded, everyone was happy. This works fine when you have a handful of users, but as traffic grows, that single machine hits a hard ceiling imposed by physics:

- **CPU saturation**: The processor can only handle so many threads concurrently.
- **Memory exhaustion**: RAM fills up with active sessions and cached data.
- **Network bandwidth limits**: The NIC (Network Interface Card) can only push so many bytes per second.
- **Disk I/O bottlenecks**: Reads and writes compete for the same spindle or SSD bus.

We call this hitting the **vertical scaling wall**. You can upgrade your server (buy more RAM, faster CPU), but eventually you hit diminishing returns — and hardware upgrades get exponentially more expensive.

```svgbob
     Users
       |
       | (thousands of requests)
       v
  +----------+
  |  Server  |  <-- CPU: 100%, RAM: maxed, starting to drop requests
  +----------+
       |
       v
   Database
```

*In Figure 1, notice that all traffic funnels into one machine. There is no redundancy — if this server crashes, the entire application goes down. This is called a Single Point of Failure (SPOF).*

The solution is to stop trying to make one machine do everything, and instead distribute the work across many machines — **horizontal scaling**. But the moment you have multiple servers, you immediately face a new problem: *how does a client know which server to talk to?*

This is exactly the problem that load balancing solves.

***

## Chapter 2: Enter the Load Balancer

A load balancer sits between your clients and your pool of backend servers. It receives every incoming request and decides which server should handle it.

```svgbob
                     +------------------+
   Client A -------> |                  | -------> Server 1
   Client B -------> |  Load Balancer   | -------> Server 2
   Client C -------> |                  | -------> Server 3
   Client D -------> |                  | -------> Server 4
                     +------------------+
```

*In Figure 2, the load balancer acts as the single entry point for all clients. From the client's perspective, they are talking to one address. Behind the scenes, four servers are sharing the work. This abstraction is central to how modern distributed systems achieve both scale and resilience.*

The load balancer is doing several things simultaneously:

1. **Accepting connections** from clients at a public IP/port
2. **Selecting a backend server** using a routing algorithm
3. **Forwarding the request** to that server
4. **Returning the response** back to the client

From the client's perspective, the load balancer *is* the server. The client never knows (or cares) about the backend pool. This property is called **transparency**, and it's a foundational design principle.

***

## Chapter 3: Where Does the Load Balancer Live?

Before we get into routing algorithms, we need to understand that "load balancer" is not one thing. It can operate at different layers of the networking stack, and the layer it operates at determines what information it can use to make routing decisions.

### Layer 4 (Transport Layer) Load Balancing

A Layer 4 load balancer works at the TCP/UDP level. It sees IP addresses and ports but has no idea what's inside the packets. It's fast and lightweight — it makes decisions purely based on network metadata.

Think of it like a postal sorting facility that routes packages by zip code without ever opening them.

**What it knows:** Source IP, destination IP, TCP/UDP port
**What it doesn't know:** HTTP headers, cookies, URL paths, request body

**Use case:** High-throughput, low-latency scenarios where you don't need application-level awareness.

### Layer 7 (Application Layer) Load Balancing

A Layer 7 load balancer operates at the HTTP/HTTPS level. It can read headers, cookies, URL paths, and even request bodies. This makes it far more intelligent — it can route `/api/images` traffic to an image processing server cluster and `/api/payments` to a payments-specific cluster.

Think of it like a concierge at a hotel who reads your reservation details and sends you to the right floor.

**What it knows:** Everything in the HTTP request — headers, cookies, URL, body
**What it doesn't know:** Nothing — it has full visibility

**Use case:** Microservices routing, A/B testing, canary deployments, content-based routing.

```svgbob
  OSI Model                Load Balancer Type
  ---------                ------------------

  Layer 7 (Application) -- L7 LB: reads HTTP headers, URLs, cookies
      |
  Layer 4 (Transport)   -- L4 LB: reads IP addresses and TCP ports
      |
  Layer 3 (Network)
      |
  Layer 2 (Data Link)
```

*In Figure 3, we can see that higher-layer load balancers have more context. The trade-off is processing overhead — reading and parsing HTTP headers takes more CPU cycles than simply inspecting an IP address.*

***

## Chapter 4: The Routing Algorithms — How Decisions Are Made

This is where the real intelligence lives. The algorithm a load balancer uses to pick a server is called its **load balancing algorithm** or **scheduling policy**. We'll group them by behavior, because that framing is far more useful than a flat list.

### Group 1: Stateless Algorithms (No Memory of Past Decisions)

These algorithms treat every request independently. They require no bookkeeping, making them extremely fast and easy to implement.

#### Round Robin

The simplest algorithm in existence. Requests are distributed in a circular queue — Server 1 gets the first request, Server 2 gets the second, Server 3 gets the third, and then we cycle back to Server 1.

```python
class RoundRobinBalancer:
    def __init__(self, servers: list[str]):
        self.servers = servers
        self.index = 0

    def get_next_server(self) -> str:
        server = self.servers[self.index]
        self.index = (self.index + 1) % len(self.servers)
        return server

# Usage
balancer = RoundRobinBalancer(["server1", "server2", "server3"])
for _ in range(6):
    print(balancer.get_next_server())
# Output: server1, server2, server3, server1, server2, server3
```

**When it works well:** Homogeneous servers handling similar-sized requests (e.g., serving static files).

**When it breaks:** If some requests take much longer than others (e.g., a video transcode job vs. a health check), you can accidentally overload one server with all the heavy jobs by pure bad luck.

#### Weighted Round Robin

The natural evolution of Round Robin. We assign each server a **weight** proportional to its capacity. A server with weight 3 receives three requests for every one request sent to a server with weight 1.

```python
class WeightedRoundRobinBalancer:
    def __init__(self, servers: dict[str, int]):
        # Expand servers list based on weight
        self.pool = []
        for server, weight in servers.items():
            self.pool.extend([server] * weight)
        self.index = 0

    def get_next_server(self) -> str:
        server = self.pool[self.index]
        self.index = (self.index + 1) % len(self.pool)
        return server

# Server A has 3x the capacity of Server B
balancer = WeightedRoundRobinBalancer({"server_a": 3, "server_b": 1})
for _ in range(4):
    print(balancer.get_next_server())
# Output: server_a, server_a, server_a, server_b
```

**Use case:** Heterogeneous server fleets — common during incremental hardware upgrades when new machines coexist with older ones.

#### Random

Randomly picks a server from the pool. Surprisingly, this performs comparably to Round Robin for large request volumes due to the law of large numbers. It's trivially simple to implement and scales well in distributed environments where multiple load balancer nodes don't need to synchronize a shared counter.

***

### Group 2: Stateful Algorithms (Memory-Aware Decisions)

These algorithms track some runtime state (active connections, response times) to make smarter routing decisions. They require bookkeeping but produce better distribution under uneven workloads.

#### Least Connections

Instead of blindly cycling through servers, we route each new request to whichever server currently has the **fewest active connections**.

```python
import threading

class LeastConnectionsBalancer:
    def __init__(self, servers: list[str]):
        self.connections = {server: 0 for server in servers}
        self.lock = threading.Lock()

    def get_next_server(self) -> str:
        with self.lock:
            server = min(self.connections, key=self.connections.get)
            self.connections[server] += 1
            return server

    def release_connection(self, server: str):
        with self.lock:
            if self.connections[server] > 0:
                self.connections[server] -= 1
```

Notice the `threading.Lock()` — this is important. In a concurrent environment, two requests arriving simultaneously might both read the same "minimum connections" value before either increments the counter. Without a lock, we'd have a race condition.

**When it shines:** APIs with variable request durations. Long-running WebSocket connections, video streaming, file uploads — all scenarios where some connections hold open much longer than others.

#### Weighted Least Connections

Combines Least Connections with server weights. The effective load of a server is calculated as `active_connections / weight`. The server with the lowest score gets the next request.

```python
class WeightedLeastConnectionsBalancer:
    def __init__(self, servers: dict[str, int]):
        self.weights = servers
        self.connections = {server: 0 for server in servers}

    def get_next_server(self) -> str:
        # Score = active connections / weight (lower is better)
        scores = {
            server: self.connections[server] / self.weights[server]
            for server in self.weights
        }
        return min(scores, key=scores.get)
```


#### Resource-Based (Adaptive)

The most sophisticated stateful approach. The load balancer actively polls each server for real-time resource metrics — CPU percentage, memory usage, request queue depth — and routes to the server with the most available headroom.

This was historically expensive to implement (requires agents on each server), but in modern containerized environments (Kubernetes, ECS), these metrics are surfaced natively through APIs. Services like AWS Application Load Balancer with target group health metrics operate on similar principles.

***

### Group 3: Affinity Algorithms (Routing Consistency)

Sometimes we need the same client to always land on the same server. These algorithms trade some distribution efficiency for routing consistency.

#### IP Hash (Source IP Affinity)

Computes a hash of the client's IP address and maps it consistently to a server. As long as the server pool doesn't change, the same client always hits the same server.

```python
import hashlib

class IPHashBalancer:
    def __init__(self, servers: list[str]):
        self.servers = servers

    def get_server_for_ip(self, client_ip: str) -> str:
        hash_value = int(hashlib.md5(client_ip.encode()).hexdigest(), 16)
        index = hash_value % len(self.servers)
        return self.servers[index]

balancer = IPHashBalancer(["server1", "server2", "server3"])
print(balancer.get_server_for_ip("192.168.1.10"))  # Always maps to same server
print(balancer.get_server_for_ip("192.168.1.10"))  # Same result
```

**The fatal flaw:** If you add or remove a server, `len(self.servers)` changes, and almost every client remaps to a different server. For a caching layer, this is catastrophic — you'd lose cache locality for most of your users simultaneously.

This is exactly the pain point that motivated one of the most elegant algorithms in distributed systems.

***

## Chapter 5: Consistent Hashing — Solving the Redistribution Problem

Let's talk about the "broken state" first. Imagine you're using IP Hash to route users to one of three cache servers. Each server holds cached session data for roughly one-third of your user base. You add a fourth server to handle growing traffic.

With simple modulo hashing, `hash(ip) % 4` now maps almost everyone to a different server. You've effectively invalidated ~75% of your cache. Your database sees a thundering herd of cache misses and likely falls over.

**Consistent Hashing** was designed precisely for this problem. Instead of `hash(ip) % n`, we arrange both servers and keys on a circular ring of hash values (conceptually, a clock face spanning 0 to 2^32).

```svgbob
                     0
              .------+------.
           .´         S1     `.
         .´                    `.
    270  +         Ring          + 90
         `.                    .´
           `.    S3      S2   .´
              `------+------´
                    180
```

*In Figure 4, three servers (S1, S2, S3) are placed on the ring by hashing their identifiers. Each client request is also hashed, placed on the ring, and routed to the first server encountered when walking clockwise. Notice that the ring is circular — a request placed just before position 0 wraps around to the server just past 0.*

When we add a new server S4, it lands at some position on the ring. Only the clients whose hashes fall between S3 and S4 (the previous server going counterclockwise) need to remap. All other clients are unaffected. On average, only `1/n` of keys move when a node is added or removed.

```python
import hashlib
import bisect

class ConsistentHashBalancer:
    def __init__(self, servers: list[str], replicas: int = 100):
        self.replicas = replicas  # Virtual nodes per server
        self.ring = {}
        self.sorted_keys = []

        for server in servers:
            self.add_server(server)

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_server(self, server: str):
        for i in range(self.replicas):
            virtual_node_key = f"{server}#{i}"
            hash_val = self._hash(virtual_node_key)
            self.ring[hash_val] = server
            bisect.insort(self.sorted_keys, hash_val)

    def remove_server(self, server: str):
        for i in range(self.replicas):
            virtual_node_key = f"{server}#{i}"
            hash_val = self._hash(virtual_node_key)
            del self.ring[hash_val]
            self.sorted_keys.remove(hash_val)

    def get_server(self, client_ip: str) -> str:
        if not self.ring:
            return None
        hash_val = self._hash(client_ip)
        # Find the first server clockwise on the ring
        idx = bisect.bisect(self.sorted_keys, hash_val)
        if idx == len(self.sorted_keys):
            idx = 0  # Wrap around
        return self.ring[self.sorted_keys[idx]]
```

Notice the `replicas` parameter — this introduces **virtual nodes**. Without them, servers might cluster unevenly on the ring, causing one server to handle a disproportionate slice. Virtual nodes spread each physical server across multiple positions on the ring, producing much more even distribution. Typically, 100–200 virtual nodes per server is a good default.

**Where you'll see this in the wild:** Amazon DynamoDB, Apache Cassandra, Redis Cluster, and Memcached all use consistent hashing (or variants of it) for key distribution. Understanding this algorithm is one of the highest-signal answers you can give in a system design interview.

***

## Chapter 6: Health Checks — The Immune System of Your Fleet

We've talked about routing, but we've been assuming all servers are healthy. What happens when Server 2 crashes? Without health checking, the load balancer would keep sending 33% of traffic into a black hole.

Health checks are the load balancer's immune system. They continuously probe backend servers and automatically remove unhealthy ones from the rotation.

### Passive Health Checks

The load balancer monitors actual traffic responses. If a server returns HTTP 500 errors or times out more than X times in Y seconds, it's marked as unhealthy.

**Advantage:** Zero overhead — no extra network calls.
**Disadvantage:** Real user requests fail before the problem is detected.

### Active Health Checks

The load balancer proactively sends synthetic "probe" requests on a configurable interval — typically a lightweight `GET /health` endpoint. If the server doesn't respond with a 200 OK within a timeout window, it's pulled from the pool.

```python
import asyncio
import aiohttp
from enum import Enum

class ServerStatus(Enum):
    HEALTHY = "healthy"
    UNHEALTHY = "unhealthy"

class HealthChecker:
    def __init__(
        self,
        servers: list[str],
        health_path: str = "/health",
        interval_seconds: int = 10,
        timeout_seconds: int = 2,
        failure_threshold: int = 3,
    ):
        self.servers = servers
        self.health_path = health_path
        self.interval = interval_seconds
        self.timeout = timeout_seconds
        self.threshold = failure_threshold
        self.failure_counts = {s: 0 for s in servers}
        self.status = {s: ServerStatus.HEALTHY for s in servers}

    async def check_server(self, session: aiohttp.ClientSession, server: str):
        url = f"{server}{self.health_path}"
        try:
            async with session.get(url, timeout=aiohttp.ClientTimeout(total=self.timeout)) as resp:
                if resp.status == 200:
                    self.failure_counts[server] = 0
                    self.status[server] = ServerStatus.HEALTHY
                else:
                    self._record_failure(server)
        except Exception:
            self._record_failure(server)

    def _record_failure(self, server: str):
        self.failure_counts[server] += 1
        if self.failure_counts[server] >= self.threshold:
            self.status[server] = ServerStatus.UNHEALTHY
            print(f"[ALERT] Server {server} marked UNHEALTHY")

    def get_healthy_servers(self) -> list[str]:
        return [s for s, status in self.status.items() if status == ServerStatus.HEALTHY]

    async def run(self):
        async with aiohttp.ClientSession() as session:
            while True:
                tasks = [self.check_server(session, s) for s in self.servers]
                await asyncio.gather(*tasks)
                await asyncio.sleep(self.interval)
```

*A production health checker like this runs as an independent async loop. Notice the `failure_threshold` — we don't pull a server on a single failure. Network blips happen. We wait for three consecutive failures before marking a server unhealthy. This avoids flapping.*

The typical lifecycle looks like this:

```svgbob
  Server is added
       |
       v
  [STARTING] -- grace period, no traffic
       |
       v
  [HEALTHY] <----+
       |         |
  Health check   | Recovery: N consecutive successes
  fails N times  |
       |         |
       v         |
  [UNHEALTHY] ---+
       |
       v
  (Removed from rotation, no new requests routed here)
```

*In Figure 5, notice the grace period for a STARTING server. This prevents the load balancer from routing traffic to a server that hasn't finished initializing — a common source of startup errors in containerized environments.*

***

## Chapter 7: Session Persistence (Sticky Sessions)

Here's a subtle problem with stateless load balancing. Imagine a user adds items to a shopping cart. That cart state is stored in memory on Server 1. The user clicks "Checkout" — the load balancer, doing its job, sends this request to Server 2. Server 2 has no knowledge of the cart. The cart appears empty. The user is confused and frustrated.

This is the **session affinity problem**.

### Solution A: Sticky Sessions

The load balancer injects a special cookie (e.g., `SERVERID=server1`) into the user's browser. On every subsequent request, the load balancer reads this cookie and routes the user back to the same server.

**The trade-off:** You've partially given up your horizontal scaling benefits. If Server 1 gets a lot of "heavy" users who each trigger long sessions, it can become a hotspot while Server 2 sits underutilized. Server failures also destroy session affinity — when Server 1 crashes, all its sticky users must be re-assigned.

### Solution B: Externalize State (The Better Approach)

The more scalable solution is to move session state out of the application servers entirely and into a shared, fast external store like **Redis** or **Memcached**.

```svgbob
   Client
     |
     v
  Load Balancer (any algorithm, stateless)
     |
     +----------+----------+
     v          v          v
  Server 1   Server 2   Server 3
     |          |          |
     +----------+----------+
                |
                v
           Redis / Memcached
           (Shared Session Store)
```

*In Figure 6, all application servers read and write session data to a central Redis cluster. Now, any server can handle any request for any user — true stateless routing is restored. The load balancer is free to use the most efficient algorithm without worrying about affinity.*

This architectural shift (externalizing state) is one of the most important principles behind building horizontally scalable systems. It's a design decision that will come up in virtually every system design interview.

***

## Chapter 8: High Availability of the Load Balancer Itself

A sharp interviewer will often ask: "Isn't the load balancer itself a single point of failure?"

They're absolutely right to ask. If your entire traffic enters through one load balancer and that machine fails, your service is down regardless of how many healthy backend servers you have.

The solution is to run load balancers in **active-passive** or **active-active** pairs, using a protocol called **VRRP (Virtual Router Redundancy Protocol)** or its equivalents.

```svgbob
                       Virtual IP: 10.0.0.1
                              |
               +--------------+--------------+
               |                             |
      +--------+--------+         +----------+------+
      |   LB Primary    |         |   LB Secondary  |
      |  (Active)       |         |  (Standby)      |
      +--------+--------+         +--------+--------+
               |    heartbeat <------>    |
               |                         |
               +----------+--------------+
                          |
               +----------+----------+
               v          v          v
            Server 1   Server 2   Server 3
```

*In Figure 7, both load balancers share a Virtual IP (VIP). The primary LB owns the VIP and handles all traffic. The secondary monitors the primary via heartbeat. If the primary fails to send a heartbeat within the timeout window, the secondary immediately claims the VIP. To clients, the IP address never changed — the failover is invisible.*

In cloud environments, managed load balancers (AWS ALB/NLB, GCP Load Balancer, Azure Load Balancer) handle all of this redundancy for you automatically. Behind the scenes, cloud providers run fleets of load balancer nodes with built-in failover. This is one of the core value propositions of managed cloud infrastructure.

***

## Chapter 9: Global Load Balancing (GeoDNS and GSLB)

So far, we've been discussing load balancing within a single data center. But what if you're operating globally — with data centers in North America, Europe, and Asia-Pacific?

**The problem:** A user in Tokyo connecting to a server in Virginia experiences 150+ ms of round-trip latency just due to the physical distance light travels through fiber. That's before any application processing.

**Global Server Load Balancing (GSLB)** routes users to the geographically closest (or most appropriate) data center.

### DNS-Based Global Load Balancing

The most common mechanism uses DNS. When a user resolves `api.yourapp.com`, a DNS server with geographic awareness returns the IP address of the nearest regional load balancer.

```svgbob
    User in Tokyo                      User in New York
         |                                     |
         v                                     v
  DNS Query: api.yourapp.com          DNS Query: api.yourapp.com
         |                                     |
         v                                     v
   GeoDNS Server                       GeoDNS Server
   "You're in APAC"                    "You're in US-EAST"
         |                                     |
         v                                     v
  Tokyo Data Center                  Virginia Data Center
  Load Balancer (10.ap.0.1)         Load Balancer (10.us.0.1)
         |                                     |
     Servers                              Servers
```

*In Figure 8, the same DNS query returns different IP addresses depending on the geographic origin of the request. This is called **GeoDNS**. Notice that each regional cluster has its own load balancer — GSLB works at the DNS level, while regional load balancing handles traffic within the data center.*

**Limitations of DNS-based GSLB:**

- DNS responses are cached (TTL). If a data center goes down, it can take minutes to hours for all clients to stop routing there, depending on their DNS cache TTL.
- Anycast routing (used by Cloudflare and similar CDNs) is more sophisticated — the same IP is announced from multiple locations, and BGP routes clients to the nearest announcement point at the network layer.

***

## Chapter 10: Load Balancing in Modern Architectures

### Microservices and Service Mesh

In a microservices architecture, load balancing isn't just at the edge — it happens between every pair of services that communicate with each other. Service A calling Service B needs a load balancer to distribute across B's instances.

Two patterns dominate here:

**Server-Side Load Balancing:** A dedicated load balancer (e.g., an internal Nginx or HAProxy instance, or a Kubernetes Service) sits between the caller and the callee. The caller just calls one address.

**Client-Side Load Balancing:** The calling service itself maintains a list of available instances (often from a service registry like Consul or Eureka) and applies a load balancing algorithm locally. Netflix's **Ribbon** library was an early example. Today, **Envoy Proxy** (the core of Istio service mesh) implements client-side load balancing as a sidecar.

```svgbob
  Service Mesh Pattern (Sidecar Proxy)

  +------------------------+      +------------------------+
  |  Service A             |      |  Service B             |
  |  +---------+           |      |  +---------+           |
  |  |  App    +--+        |      |  |  App    |           |
  |  +---------+  |        |      |  +---------+           |
  |               v        |      +------------------------+
  |  +---------+  |        |
  |  | Envoy   +----------->  Service B Instance 1
  |  | Sidecar |  |        |
  |  +---------+  |        |  ->  Service B Instance 2
  +---------------+--------+
                            -->  Service B Instance 3
```

*In Figure 9, each service has an Envoy sidecar proxy injected alongside it. The application code talks to its local sidecar, which handles load balancing, retries, circuit breaking, and observability. The application developer doesn't write any load balancing logic — it's infrastructure-level concern, handled transparently.*

### Kubernetes Load Balancing

Kubernetes has load balancing baked into its primitives:

- **ClusterIP Service:** Internal DNS name that load-balances across pod replicas using iptables/ipvs rules. This is L4 round-robin by default.
- **NodePort / LoadBalancer Service:** Exposes services externally. The `LoadBalancer` type provisions a cloud load balancer (e.g., AWS NLB) automatically.
- **Ingress:** An L7 load balancer that routes HTTP/HTTPS traffic based on hostname and path rules to different backend services.

```python
# Conceptually, a Kubernetes Ingress rule works like this routing logic:

def route_request(host: str, path: str) -> str:
    rules = [
        {"host": "api.myapp.com",    "path": "/v1/users",    "service": "user-service"},
        {"host": "api.myapp.com",    "path": "/v1/products", "service": "product-service"},
        {"host": "admin.myapp.com",  "path": "/",            "service": "admin-service"},
    ]

    for rule in rules:
        if host == rule["host"] and path.startswith(rule["path"]):
            return rule["service"]

    return "default-backend"

# Route traffic based on host + path combination
print(route_request("api.myapp.com", "/v1/users/123"))    # --> user-service
print(route_request("api.myapp.com", "/v1/products/456")) # --> product-service
```


***

## Chapter 11: The Algorithms Compared — A Decision Framework

Rather than memorizing a list, think about algorithm selection as answering three questions:

**1. Are my servers homogeneous?**

- Yes → Round Robin
- No (different capacities) → Weighted Round Robin

**2. Do my requests have highly variable durations?**

- No (similar request sizes) → Round Robin is fine
- Yes (mixed short/long tasks) → Least Connections or Weighted Least Connections

**3. Do I need routing consistency for the same client?**

- Need consistency, static server pool → IP Hash
- Need consistency, dynamic server pool → Consistent Hashing
- Need consistency, can externalize state → Don't use affinity; use shared session store

```svgbob
  Start
    |
    v
  Homogeneous servers?
    |         |
   Yes        No
    |         |
    v         v
  Round    Weighted
  Robin    Round Robin
    |
    v
  Variable request durations?
    |              |
   Yes             No
    |              |
    v              v
  Least          (stay with
  Connections     Round Robin)
    |
    v
  Need client-to-server affinity?
    |               |
   Yes              No
    |               |
    v               v
  Dynamic pool?   All good,
    |              use above
  Yes    No
    |      |
    v      v
  Consistent  IP Hash
  Hashing
```

*In Figure 10, this decision tree gives you a practical framework for algorithm selection in an interview or real-world scenario. Most questions have one clearly correct answer given the constraints.*

***

## Chapter 12: Common Interview Questions and What They're Really Testing

These questions come up frequently in senior and staff-level interviews at top-tier companies. Understanding what the interviewer is testing beneath the surface is as important as knowing the answer.

**Q: "How would you design a load balancer?"**

*What they're testing:* Do you know the difference between L4 and L7? Can you discuss connection tracking, health checking, and the consistency/availability trade-off in the control plane?

*Key points to hit:* VIP + ECMP or VRRP for HA, health check polling, algorithm selection based on workload, and stateless vs. stateful backends.

**Q: "What's the difference between a load balancer and an API gateway?"**

*What they're testing:* Depth of knowledge. An API gateway does everything an L7 load balancer does, plus: authentication, rate limiting, request transformation, and API key management. They're often implemented on top of the same technology (Nginx, Envoy) but serve different architectural roles.

**Q: "How does Cassandra/Redis distribute data?"**

*What they're testing:* Whether you understand consistent hashing in a storage context. The answer involves virtual nodes and consistent hashing, demonstrating that load balancing principles apply beyond just HTTP traffic.

**Q: "What happens to sticky sessions when a server dies?"**

*What they're testing:* Can you identify the operational risk of session affinity and propose the correct solution — externalizing state to a shared store like Redis?

***

## Chapter 13: Putting It All Together — A System Design Sketch

Let's close by sketching the load balancing architecture for a production-grade global web application, synthesizing everything we've covered.

```svgbob
  Internet
     |
     v
  GeoDNS (routes to nearest region)
     |
  +--------------------------------------+
  |   Region: US-EAST                    |
  |                                      |
  |  +-------+    +-------+              |
  |  |  LB   |    |  LB   | (HA pair,    |
  |  | (A)   +----+ (B)   |  VRRP)       |
  |  +---+---+    +---+---+              |
  |      |            |                  |
  |      +-----+------+                  |
  |            |  (L7, path-based)        |
  |     +------+-------+                 |
  |     v              v                 |
  |  /api/*         /static/*            |
  |  App Servers    CDN / Object Store   |
  |  (fleet)                             |
  |     |                                |
  |  +------+                            |
  |  | Redis|  (shared session store)    |
  |  +------+                            |
  |     |                                |
  |  +------+                            |
  |  |  DB  |  (primary + replicas)      |
  |  +------+                            |
  +--------------------------------------+
```

*In Figure 11, this architecture layers multiple load balancing mechanisms: GeoDNS for global routing, an HA pair at the regional edge, L7 routing for traffic splitting between application servers and static content, and Redis for session state — enabling fully stateless application servers. Each layer independently solves a different failure mode.*

***

## Key Takeaways

Let's crystallize the mental model we've built:

- **Load balancing** is the practice of distributing work across multiple servers to achieve higher throughput, lower latency, and fault tolerance. The store manager analogy holds — it's not about making any single worker faster, but about using the full capacity of your fleet.
- **L4 vs. L7** is a trade-off between speed and intelligence. L4 is faster; L7 knows more. Use L7 when you need content-based routing (microservices, A/B testing, path-based routing).
- **Algorithm selection** should be driven by workload characteristics: Round Robin for homogeneous loads, Least Connections for variable-duration requests, Consistent Hashing for dynamic pools requiring affinity.
- **Health checking** is non-negotiable in production. Passive checks catch failures from real traffic; active checks catch failures before users see them.
- **Session persistence** through sticky sessions is a scaling antipattern. The correct solution is to externalize session state.
- **The load balancer itself** must be highly available — use active-passive pairs with a Virtual IP.
- **Modern architectures** push load balancing deeper — into service meshes, Kubernetes primitives, and client-side libraries — not just at the network edge.

Understanding load balancing at this depth — from the grocery store analogy all the way to consistent hashing and service meshes — gives you the vocabulary to design resilient systems and the depth to satisfy even the most probing interview questions.

---
