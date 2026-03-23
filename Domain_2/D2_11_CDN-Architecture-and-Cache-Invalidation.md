
# CDN Architecture and Cache Invalidation

---

## Part 1: The Analogy — Think Like a Supermarket Chain

Before we write a single line of code or draw a single architecture diagram, let's step away from computers entirely.

Imagine you run a national supermarket chain. Your **headquarters warehouse** is located in Chicago. It holds every product you sell — every flavor of chips, every brand of milk, every size of cereal box. The warehouse is the definitive source of truth. But here's the problem: you have customers in Miami, Seattle, Denver, and Boston. If every customer had to drive to Chicago to buy milk, your business would collapse. Nobody wants a 4-hour round trip for groceries.

So what do you do? You **open local stores** in each city. Each local store stocks the *most popular* products — the items customers buy most. When someone in Miami needs milk, they go to the Miami store. Fast, convenient, local. The Chicago warehouse still exists and still holds everything, but it only gets involved when a local store runs out of stock and needs a resupply.

This, in its entirety, is the mental model for a **Content Delivery Network (CDN)**.

- The **Chicago warehouse** = your **Origin Server** (your backend, your data center)
- The **local stores in each city** = **Edge Servers / Points of Presence (PoPs)**
- The **products on shelves** = **cached content** (HTML, CSS, JS, images, videos, API responses)
- The **customer buying milk** = the **end user** requesting a webpage or asset
- The **resupply run from Chicago** = a **cache miss** that triggers a fresh fetch from the origin

Keep this analogy in your mind throughout this article. Every technical detail we introduce maps back to it. When we talk about cache invalidation, we're really asking: *"How does the Miami store know when the Chicago warehouse has updated its product, so it stops selling stale inventory?"*

---

## Part 2: The Problem CDNs Solve — Why Distance is the Enemy

### Latency is Physically Unavoidable

Here's a fundamental, humbling truth about networking: **data cannot travel faster than the speed of light**. Light covers roughly 200,000 km/second through fiber-optic cable (slightly slower than in a vacuum due to refraction). That sounds fast — and it is — but the Earth is 40,000 km in circumference. A request from a user in Tokyo to a server in New York City involves approximately 10,800 km of distance. Even at the speed of light, that's ~54 milliseconds **one-way**. Round-trip? ~108ms — and that's *before* we account for packet processing, TCP handshakes, and TLS negotiation.

Studies by Google found that **53% of mobile users abandon a site if it takes longer than 3 seconds to load**. Amazon famously quantified that every 100ms of latency costs them 1% in sales. Latency is not a soft problem — it's a hard business metric.

Let's model this numerically to make it concrete:

```python

# Speed of light through fiber optic cable (approx)

SPEED_OF_LIGHT_FIBER_KM_PER_MS = 200  # km/ms

def calculate_round_trip_latency_ms(distance_km: float) -> float:
    """
    Calculate theoretical minimum round-trip latency.
    Multiply by 2 for round trip.
    """
    one_way_ms = distance_km / SPEED_OF_LIGHT_FIBER_KM_PER_MS
    return one_way_ms * 2

# Example: User in Tokyo, Server in New York

distance_tokyo_to_nyc = 10_838  # km
latency = calculate_round_trip_latency_ms(distance_tokyo_to_nyc)

print(f"Theoretical min RTT (Tokyo → NYC): {latency:.1f} ms")

# Output: Theoretical min RTT (Tokyo → NYC): 108.4 ms

# With a CDN edge in Tokyo (e.g., ~50km away)

distance_local_edge = 50
latency_cdn = calculate_round_trip_latency_ms(distance_local_edge)

print(f"Theoretical min RTT with local CDN edge: {latency_cdn:.2f} ms")

# Output: Theoretical min RTT with local CDN edge: 0.50 ms

```

The difference is staggering. A CDN doesn't break the laws of physics — it **works around them** by moving the data closer before the user even asks for it.

---

## Part 3: The Architecture — Building the CDN Layer by Layer

We'll build this architecture iteratively, just like a real system scales. We start minimal and add complexity only when we can clearly articulate the *need* for it.

### Layer 1: The Naive Architecture (No CDN)

At its most primitive, a web application has a single origin server. Every user, regardless of location, sends requests directly to that server.

```svgbob
  [User in Tokyo] ---------> [Origin Server in NYC]
  [User in London] --------> [Origin Server in NYC]
  [User in São Paulo] -----> [Origin Server in NYC]
                              ^
                              |
                    Single point of failure
                    High latency for all non-US users
                    Bandwidth bottleneck
```

*In Figure 1, notice all arrows converge on a single box. Every request — whether it's a tiny icon or a 4K video — must travel all the way to New York. The origin server bears 100% of the load.*

**The pain points are clear:**

1. High latency for geographically distant users
2. Single point of failure — one server going down means global downtime
3. Bandwidth saturation — the origin serves *everything*, including static assets that never change
4. Poor scalability for traffic spikes (e.g., a viral news article)

### Layer 2: Adding a CDN — The Edge Layer

We introduce the first CDN component: **edge servers** grouped into **Points of Presence (PoPs)**. A PoP is a regional data center — typically a co-location facility — strategically placed in major population hubs worldwide. [web:3]

```svgbob
                     +------------------+
                     |  Origin Server   |
                     |  (NYC)           |
                     +--------+---------+
                              |
           +------------------+------------------+
           |                  |                  |
  +--------+------+  +--------+------+  +--------+------+
  |  PoP: Tokyo   |  |  PoP: London  |  |  PoP: São Paulo|
  |  Edge Servers |  |  Edge Servers |  |  Edge Servers  |
  +-------+-------+  +-------+-------+  +-------+--------+
          |                  |                   |
   [Tokyo Users]      [London Users]     [Brazilian Users]
```

*In Figure 2, notice that the origin server is now "upstream" — it feeds the PoPs, but it is no longer the first responder for most requests. Each PoP is the local store from our supermarket analogy.*

Each PoP typically contains [web:13]:

- **Cache servers** — store copies of popular content (the "shelves")
- **Load balancers** — distribute requests across cache servers within the PoP
- **Routers/BGP infrastructure** — handle traffic routing decisions
- **Security appliances** — WAF, DDoS scrubbers (for security-enabled CDNs)


### Layer 3: The Request Lifecycle — A Cache Hit vs. Cache Miss

Understanding the two possible paths a request can take is foundational for any interview.

**Path A: Cache HIT (The Happy Path)**

```svgbob
  User in Tokyo
       |
       | 1. GET /logo.png
       v
  PoP in Tokyo
       |
       | 2. "I have /logo.png in cache!"
       |
       | 3. Serve directly from edge cache
       v
  User in Tokyo  ✓ (Low latency, ~0.5ms)
       
  Origin Server: 
       (never contacted)
```

**Path B: Cache MISS (The Fetch Path)**

```svgbob
  User in Tokyo
       |
       | 1. GET /new-article.html
       v
  PoP in Tokyo
       |
       | 2. "Cache miss — I don't have this"
       v
  Origin Server (NYC)
       |
       | 3. Return /new-article.html
       | 4. Edge stores a copy with TTL
       v
  PoP in Tokyo
       |
       | 5. Serve to user + cache for future requests
       v
  User in Tokyo  ✓ (Higher latency first time, fast after)
```

*In Figure 3 and 4, the critical distinction is whether the PoP must consult the origin. The cache hit path is entirely "local" — no cross-continental round trip. The cache miss is slower but pays a "warming" cost that future requests amortize away.*

### Layer 4: The Origin Shield — A Middle Tier

Here's a scenario that reveals the next limitation. Suppose your CDN has 200 PoPs globally and you publish a major breaking news story. All 200 PoPs get a simultaneous cache miss for the same content. All 200 PoPs send a request to your origin at the same moment — a phenomenon called a **thundering herd** or **cache stampede**. Your origin gets hammered with 200 simultaneous fetch requests.

The fix? We add an **Origin Shield** — a single, designated "super PoP" that sits between all edge PoPs and the origin server. [web:12]

```svgbob
  +------------+   +------------+   +------------+
  | PoP Tokyo  |   | PoP London |   | PoP Sydney |
  +-----+------+   +-----+------+   +-----+------+
        |                |                |
        |                v                |
        +-------> +--------------+ <------+
                  | Origin Shield|
                  | (e.g., US-East) |
                  +------+-------+
                         |
                         | Only 1 request to origin
                         v
                  +------+-------+
                  | Origin Server|
                  +--------------+
```

*In Figure 5, all PoPs funnel their cache misses to the Origin Shield first. The Shield checks its own cache — if it has the content, it satisfies all PoPs without touching the origin. If it also has a miss, only a single request propagates to the origin. This collapses 200 origin requests into exactly 1.*

This architectural tier is used extensively by Cloudflare, Fastly, and AWS CloudFront ("Origin Shield" is literally the CloudFront feature name).

---

## Part 4: Routing — How Does My Request Know Which PoP to Use?

This is a question interviewers love to ask as a follow-up. When a user types `www.netflix.com`, how does their browser end up talking to the *nearest* edge server? There are two dominant approaches.

### DNS-Based Routing (Geo DNS)

The DNS resolver itself becomes the traffic director. When you query `cdn.example.com`, the CDN's **authoritative DNS server** inspects the source IP of the DNS query (or EDNS Client Subnet information), geo-locates it, and returns the IP address of the nearest PoP. [web:3]

```svgbob
  User in Tokyo
       |
       | 1. DNS Query: "What is cdn.example.com?"
       v
  CDN Authoritative DNS
       |
       | 2. Source IP detected as Japanese ISP
       | 3. Returns IP of Tokyo PoP
       v
  User in Tokyo
       |
       | 4. HTTP Request → Tokyo PoP IP
       v
  Tokyo PoP ✓
```

**Advantages of DNS-based routing:**

- Simple, widely supported
- Works with all clients, no protocol changes needed

**Disadvantages:**

- DNS TTL means routing updates are slow (stale DNS can send users to wrong PoP)
- DNS queries don't carry real-time load information


### Anycast Routing

Anycast is a networking technique where **multiple servers share the same IP address**. The BGP routing protocol naturally routes traffic to the topologically closest server with that IP. [web:3]

```svgbob
  IP Address: 1.1.1.1 (shared by ALL PoPs)
  
  User in Tokyo
       |
       | GET to 1.1.1.1
       v
  Internet Routers: "Nearest 1.1.1.1 is Tokyo PoP"
       v
  Tokyo PoP ✓

  User in London
       |
       | GET to 1.1.1.1
       v
  Internet Routers: "Nearest 1.1.1.1 is London PoP"
       v
  London PoP ✓
```

*In Figure 7, both users use the exact same IP address. The routing magic happens invisibly inside the BGP routing fabric of the internet itself. This is how Cloudflare's 1.1.1.1 DNS resolver works — it's deployed on hundreds of nodes globally under one IP.*

**Advantages of Anycast:**

- Sub-millisecond routing decisions (no DNS lookup delay)
- Automatic failover if a PoP goes down (BGP reconverges)
- Excellent DDoS mitigation (attack traffic absorbed across all PoPs)

**Disadvantages:**

- TCP session stickiness is tricky (mid-session routing changes break connections)
- More complex to operate and debug

Most modern CDNs (Cloudflare, Fastly) use **Anycast for performance-critical delivery** and supplement with DNS-based geo-routing for specific use cases. [web:3]

---

## Part 5: Cache Mechanics — The Engine Inside the Edge Server

Now we go inside a single edge server and understand *how* it decides what to store, for how long, and when to evict content.

### The Cache Key

Every cached object needs a unique identifier. The **cache key** is the fingerprint that tells the edge server, "this request maps to this cached response." The simplest cache key is just the URL:

```python

# Simplest cache key

cache_key = "https://example.com/logo.png"
```

But in practice, cache keys can be more sophisticated:

```python
def build_cache_key(
    url: str,
    vary_headers: dict,
    include_query_params: bool = True
) -> str:
    """
    Build a composite cache key from URL + relevant request headers.
    
    'Vary' headers tell the CDN that the same URL may return different
    content depending on (e.g.) Accept-Encoding or Accept-Language.
    """
    import hashlib
    import json

    key_parts = {"url": url}

    if include_query_params:
        # Query params are already in URL
        pass

    if vary_headers:
        # Normalize header values for consistent keying
        normalized = {k.lower(): v.lower() for k, v in vary_headers.items()}
        key_parts["vary"] = normalized

    key_string = json.dumps(key_parts, sort_keys=True)
    return hashlib.sha256(key_string.encode()).hexdigest()

# Example usage

key_1 = build_cache_key(
    "https://cdn.example.com/app.js",
    vary_headers={"accept-encoding": "gzip"}
)

key_2 = build_cache_key(
    "https://cdn.example.com/app.js",
    vary_headers={"accept-encoding": "br"}  # Brotli-encoded version
)

# key_1 != key_2 → stored as separate cache entries

print(key_1 == key_2)  # False — correctly cached separately
```

A critical interview insight: **poorly designed cache keys cause cache pollution**. If you accidentally include a session token or a timestamp in your cache key, every user gets their own cache entry and your hit rate collapses to near zero.

### Cache Headers — The Contract Between Origin and Edge

The origin server communicates its caching intent to edge servers through HTTP response headers. These are not optional niceties — they are the **official caching contract**. [web:11]


| Header | Example | Meaning |
| :-- | :-- | :-- |
| `Cache-Control: max-age=3600` | `Cache-Control: max-age=86400` | Cache this for 86,400 seconds (1 day) |
| `Cache-Control: no-cache` | `Cache-Control: no-cache` | Must revalidate with origin before each use |
| `Cache-Control: no-store` | `Cache-Control: no-store` | Never cache this (sensitive data) |
| `Cache-Control: s-maxage=3600` | `Cache-Control: s-maxage=7200` | Shared (CDN) cache TTL, overrides max-age |
| `Surrogate-Control` | `Surrogate-Control: max-age=604800` | CDN-specific TTL, stripped before sending to browser |
| `ETag` | `ETag: "abc123"` | Content fingerprint for conditional requests |
| `Last-Modified` | `Last-Modified: Tue, 18 Mar 2026` | Last modification timestamp |

```python

# Simulating how a CDN edge server interprets Cache-Control headers

import time
from dataclasses import dataclass
from typing import Optional

@dataclass
class CacheEntry:
    content: bytes
    etag: str
    stored_at: float
    ttl_seconds: int
    
    @property
    def is_fresh(self) -> bool:
        age = time.time() - self.stored_at
        return age < self.ttl_seconds
    
    @property
    def age_seconds(self) -> float:
        return time.time() - self.stored_at


def parse_max_age(cache_control: str) -> Optional[int]:
    """
    Extract TTL from Cache-Control header.
    's-maxage' takes precedence for shared (CDN) caches.
    """
    directives = {
        part.strip().split("="): part.strip().split("=") if "=" in part else None
        for part in cache_control.split(",")
    }
    
    # CDN shared cache directive takes priority
    if "s-maxage" in directives:
        return int(directives["s-maxage"])
    if "max-age" in directives:
        return int(directives["max-age"])
    return None  # No caching instruction found

# Test

header = "public, max-age=3600, s-maxage=86400"
ttl = parse_max_age(header)
print(f"CDN will cache for: {ttl} seconds ({ttl/3600:.0f} hours)")

# Output: CDN will cache for: 86400 seconds (24 hours)

```


### Eviction Policies — What Happens When the Cache is Full?

An edge server has finite storage. When it fills up, it must decide which content to **evict** (remove) to make room for new content. This is where eviction policies come in. [web:7]

**Group 1: Usage-Based Policies (Keep what's popular)**

- **LRU (Least Recently Used):** Evicts the object that was *last accessed* the longest time ago. Works well when access patterns have temporal locality (recently-used items are likely to be used again).
- **LFU (Least Frequently Used):** Evicts the object with the *lowest access count*. Excellent for stable, long-running workloads with consistent popularity patterns. Weakness: a "burst" of traffic for a new item doesn't help its counter enough before it gets evicted.

**Group 2: Time-Based Policies (Keep what's fresh)**

- **TTL (Time To Live):** Each cache entry has an expiration timestamp. When the TTL expires, the entry is removed regardless of how popular it is. [web:7] This is the most common CDN eviction mechanism and it dovetails directly with `Cache-Control: max-age`.

**Group 3: Hybrid Policies (Modern production approach)**

- **ARC (Adaptive Replacement Cache):** Maintains two LRU queues — one for recently used and one for frequently used — and dynamically adjusts the boundary between them based on current workload patterns.

```python
from collections import OrderedDict

class LRUCache:
    """
    Classic LRU Cache using an OrderedDict.
    O(1) get and put operations.
    """
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key: str) -> bytes | None:
        if key not in self.cache:
            return None
        # Move to end (most recently used)
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key: str, value: bytes) -> None:
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        
        if len(self.cache) > self.capacity:
            # Evict LRU item (leftmost = oldest)
            evicted_key, _ = self.cache.popitem(last=False)
            print(f"Evicted: {evicted_key}")

# Simulate an edge server with capacity for 3 objects

edge_cache = LRUCache(capacity=3)

edge_cache.put("/logo.png", b"<binary data>")
edge_cache.put("/style.css", b"body { color: red; }")
edge_cache.put("/app.js", b"console.log('hello')")

# Access logo.png — makes it most recently used

edge_cache.get("/logo.png")

# Adding a new item evicts /style.css (now the LRU)

edge_cache.put("/video.mp4", b"<video data>")

# Output: "Evicted: /style.css"

```


---

## Part 6: Cache Invalidation — The Hard Problem

> *"There are only two hard things in Computer Science: cache invalidation and naming things."*
> — Phil Karlton

We've arrived at the heart of the topic. Cache invalidation is the process of **proactively removing or marking stale data in the cache before its TTL naturally expires**. [web:6] It sounds simple, but the implications are vast.

### Why is it Hard?

Here's the scenario that reveals the pain: You have a product page for "Blue Sneakers — \$99.99" cached across 150 PoPs globally with a 24-hour TTL. At 3 PM, the price drops to \$79.99. Now, for up to 24 hours, users in some regions see the old price and users in others see the new price — a nightmare for customer trust and potentially illegal depending on jurisdiction.

Waiting for TTL expiration is not always acceptable. We need a mechanism to **forcibly invalidate** the cache the moment the origin data changes.

### Strategy 1: TTL-Based Expiration (Passive Invalidation)

The simplest approach: set a reasonable `max-age`. When it expires, the next request triggers a fresh fetch from origin. [web:11]

```svgbob
  Timeline ──────────────────────────────────────────────►
  
  t=0          t=1h          t=24h         t=25h
  │            │             │             │
  ├── Cached ──┤──── Fresh ──┤─── Stale ───┤── Re-fetched ──
               │             │
               Origin       TTL
               Update!      Expires
               (but CDN     (CDN finally
               doesn't      gets new
               know yet)    content)
```

*In Figure 8, the gap between "Origin Update" and "TTL Expires" is the **stale window** — the period where the CDN serves outdated content. For a 24-hour TTL, that window can be nearly a day.*

**The tradeoff is fundamental:**

- **Short TTL** → Less stale data, but more cache misses, more origin load
- **Long TTL** → Better cache hit rates, but potentially very stale data

A common production heuristic: use short TTLs (minutes) for dynamic/user-facing content, long TTLs (days/weeks) for truly static assets (images, fonts, bundled JS with content hashes in the filename).

### Strategy 2: Cache Purging (Active Invalidation)

Most CDN providers expose an **API** that lets you explicitly purge specific URLs from all PoPs globally — typically within seconds to minutes. [web:14]

```python
import requests
import time

class CDNPurgeClient:
    """
    Simulated CDN cache purge client.
    Real implementations use provider-specific APIs 
    (e.g., Cloudflare API, Fastly Instant Purge, CloudFront Invalidations).
    """
    
    def __init__(self, api_key: str, zone_id: str):
        self.api_key = api_key
        self.zone_id = zone_id
        self.base_url = "https://api.cdn-provider.example.com"
    
    def purge_urls(self, urls: list[str]) -> dict:
        """
        Purge specific URLs from the CDN cache.
        Returns job ID for tracking propagation status.
        """
        response = requests.post(
            f"{self.base_url}/zones/{self.zone_id}/purge_cache",
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={"files": urls}
        )
        return response.json()
    
    def purge_by_tag(self, tags: list[str]) -> dict:
        """
        Purge all URLs associated with given surrogate keys/cache tags.
        Much more powerful than URL-based purging for complex sites.
        """
        response = requests.post(
            f"{self.base_url}/zones/{self.zone_id}/purge_cache",
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={"tags": tags}
        )
        return response.json()
    
    def purge_everything(self) -> dict:
        """
        Nuclear option: purge ALL cached content.
        Use only in emergencies (security incidents, major bugs).
        Causes severe cache miss storm — use with caution.
        """
        response = requests.post(
            f"{self.base_url}/zones/{self.zone_id}/purge_cache",
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={"purge_everything": True}
        )
        return response.json()

# In a deployment pipeline:

cdn = CDNPurgeClient(api_key="secret", zone_id="abc123")

# After deploying a new product price:

result = cdn.purge_urls([
    "https://example.com/products/blue-sneakers",
    "https://example.com/products/blue-sneakers.json",
])
print(f"Purge job: {result['id']} — propagating to all PoPs...")
```

**Purge propagation time** is a key operational consideration. Cloudflare claims near-instant propagation (<500ms for most cases). AWS CloudFront can take 1–2 minutes. Fastly offers "Instant Purge" with ~150ms global propagation. [web:14]

### Strategy 3: Surrogate Keys / Cache Tags (The Production Standard)

URL-based purging has a critical limitation: in a large e-commerce site, a single product might appear on dozens of pages — the product detail page, the category listing, search results, homepage recommendations, etc. Purging by URL would require tracking every page a product appears on, which is a maintenance nightmare.

**Surrogate keys** (also called cache tags) solve this elegantly. [web:6] The origin server includes a special response header listing all the "semantic tags" associated with the response:

```python

# Origin server response for a product detail page

response_headers = {
    "Content-Type": "text/html",
    "Cache-Control": "public, s-maxage=86400",
    # Tag this response with all relevant entity IDs
    "Surrogate-Key": "product-42 category-shoes brand-nike homepage-featured",
    # Cloudflare uses "Cache-Tag" header
    # Fastly uses "Surrogate-Key" header  
    # Akamai uses "Edge-Control" header
}
```

Now, when the product's price changes, a single API call purges every cached response tagged with `product-42` — regardless of which URLs contain it:

```python

# Product price updated in database

product_id = 42

# Purge everything tagged with this product's surrogate key

cdn.purge_by_tag(tags=[f"product-{product_id}"])

# This instantly invalidates:

# - /products/blue-sneakers (product detail page)

# - /category/shoes (listing that includes this product)

# - /search?q=sneakers (search results)

# - /recommendations (homepage widget)

# ALL with a single API call

```

*This is the technique used by major e-commerce platforms like Shopify, Varnish-powered sites, and publishing platforms like The Guardian. It's the closest thing to "cache invalidation at scale done right."*

### Strategy 4: Stale-While-Revalidate (The Graceful Degradation Pattern)

The `stale-while-revalidate` Cache-Control extension is a modern, elegant solution to a subtle UX problem. Without it, when a TTL expires, the *first* user to request the stale content must wait for the full origin round-trip — their request is slow while the cache "warms back up." With `stale-while-revalidate`, the CDN:

1. Serves the stale (expired) content immediately to the user
2. *Simultaneously and asynchronously* fetches a fresh copy from origin in the background
3. Updates the cache with the new content for subsequent requests
```
Cache-Control: max-age=3600, stale-while-revalidate=60
```

This means: "Fresh for 1 hour. After that, stale content may be served for up to 60 additional seconds while the cache revalidates in the background."

```svgbob
  t=0          t=3600s       t=3601s                  t=3660s
  ├── Fresh ───────────────┤  │                        │
                              │                        │
                              User A gets stale        Cache is now 
                              response (fast!)         fresh again
                              + background fetch
                              starts asynchronously
```

*In Figure 9, the key insight is that no user experiences a slow request during the revalidation window. The stale response is "good enough" for the brief moment it takes to refresh. This trades a small window of potential staleness for guaranteed low latency during cache transitions.*

### Strategy 5: Cache Versioning (The Immutable Asset Pattern)

This is a paradigm shift: instead of invalidating the cache, we **sidestep invalidation entirely** by making cached content *inherently immutable*.

The approach: embed a **content hash** directly into the asset URL. Every time the file changes, its hash changes, and therefore its URL changes. The CDN doesn't need to invalidate anything because the URL is different.

```python
import hashlib

def generate_versioned_url(
    base_url: str,
    file_path: str,
    file_content: bytes
) -> str:
    """
    Generate a content-addressed URL for a static asset.
    The hash changes only when the content changes.
    """
    content_hash = hashlib.md5(file_content).hexdigest()[:8]
    
    # Insert hash into filename before extension
    # e.g., app.js → app.abc12345.js
    parts = file_path.rsplit(".", 1)
    if len(parts) == 2:
        versioned_path = f"{parts}.{content_hash}.{parts}"
    else:
        versioned_path = f"{file_path}.{content_hash}"
    
    return f"{base_url}/{versioned_path}"

# Old content

old_js = b"console.log('version 1');"
url_v1 = generate_versioned_url("https://cdn.example.com", "app.js", old_js)
print(url_v1)  # https://cdn.example.com/app.e9c6a354.js

# New content

new_js = b"console.log('version 2');"
url_v2 = generate_versioned_url("https://cdn.example.com", "app.js", new_js)
print(url_v2)  # https://cdn.example.com/app.8d7a1bc0.js

# These are different URLs — no cache invalidation needed!

# Old URL still works for old clients; new URL is immediately available

```

This is exactly what **Webpack**, **Vite**, and **Next.js** do in production builds. Build artifacts are fingerprinted, CDN TTLs are set to 1 year (`max-age=31536000, immutable`), and cache invalidation becomes a non-problem for static assets.

The HTML file (which references the hashed JS/CSS URLs) is the *only* file that needs active invalidation — and it typically has a very short TTL or is served directly from origin.

---

## Part 7: Advanced Patterns — What Senior Engineers Know

### Multi-Tier Caching (The Full Stack)

In production systems, there is rarely just a CDN cache. There are **multiple cache layers**:

```svgbob
  User Browser
       │
       │  Layer 1: Browser Cache (private, user-specific)
       │
       ▼
  CDN Edge Server
       │
       │  Layer 2: Edge Cache (shared, geographic)
       │
       ▼
  CDN Origin Shield
       │
       │  Layer 3: Shield Cache (shared, centralized)
       │
       ▼
  Application Layer Cache (e.g., Redis)
       │
       │  Layer 4: In-memory application cache
       │
       ▼
  Database
```

*In Figure 10, notice that a single user request may be satisfied at Layer 1 (browser cache) without ever touching the network. Cache invalidation strategies must account for ALL layers — this is why a CDN purge alone may not fix stale content if the user's browser also has a stale copy.*

### Conditional Requests — Cheap Freshness Checks

When a cached resource's TTL has expired, the CDN doesn't have to download the entire resource again if it hasn't changed. It can use a **conditional GET** request with `ETag` or `Last-Modified`:

```python
def make_conditional_request(
    url: str,
    cached_etag: str | None,
    cached_last_modified: str | None
) -> dict:
    """
    Make a conditional GET request. If the resource hasn't changed,
    the server returns 304 Not Modified (no body!) — saving bandwidth.
    """
    headers = {}
    
    if cached_etag:
        headers["If-None-Match"] = cached_etag
    
    if cached_last_modified:
        headers["If-Modified-Since"] = cached_last_modified
    
    # In a real CDN, this would be the edge-to-origin request
    response = requests.get(url, headers=headers)
    
    if response.status_code == 304:
        print("✓ Resource unchanged — serving existing cache, refreshing TTL")
        return {"status": "cache_refreshed", "bytes_transferred": 0}
    elif response.status_code == 200:
        print("↓ Resource changed — downloading new version")
        return {
            "status": "cache_updated",
            "content": response.content,
            "bytes_transferred": len(response.content)
        }
```

A `304 Not Modified` response has **no body** — just headers. This is dramatically cheaper than re-downloading a 5MB video to discover it hasn't changed.

### Cache Stampede Prevention

We touched on thundering herd earlier. In a single PoP, if a high-traffic resource expires and thousands of requests arrive simultaneously, they can all simultaneously miss the cache and slam the origin. The fix is **probabilistic early expiration** (also called "cache jitter"):

```python
import random
import math

def should_revalidate_early(
    entry_age_seconds: float,
    ttl_seconds: int,
    beta: float = 1.0
) -> bool:
    """
    XFetch algorithm: probabilistically revalidate cache before expiration
    to prevent thundering herd stampedes.
    
    Higher beta = more aggressive early revalidation.
    Returns True if the cache should proactively refresh NOW.
    """
    remaining_ttl = ttl_seconds - entry_age_seconds
    
    # Probabilistic check — more likely to revalidate as expiry approaches
    # Based on research by Vattani, Chierichetti, Lowenstein (2015)
    jitter = beta * math.log(random.uniform(0, 1))
    
    return (remaining_ttl + jitter) <= 0

# Example: A cache entry with 3600s TTL, currently 3550s old

age = 3550
ttl = 3600
beta = 1.0

revalidations = sum(
    1 for _ in range(1000) 
    if should_revalidate_early(age, ttl, beta)
)
print(f"~{revalidations/10}% of edge workers would proactively revalidate")

# Some workers refresh early → origin load spreads over time, not a single spike

```


---

## Part 8: Interview Cheat Sheet — What to Say When

When a system design interviewer asks about CDNs, they're evaluating whether you understand the **trade-off space**, not just the happy path. Here's how to structure your thinking:

### Decision Framework: When to Use a CDN

| Content Type | CDN Appropriate? | Recommended TTL | Invalidation Strategy |
| :-- | :-- | :-- | :-- |
| Static assets (images, fonts) | ✅ Strongly yes | 1 year (`immutable`) | Content hashing (URL versioning) |
| JS/CSS bundles | ✅ Yes | 1 year | Content hashing |
| HTML pages (semi-static) | ✅ Yes | 5-60 minutes | Purge on deploy |
| API responses (public) | ✅ Carefully | 30s-5 min | TTL + selective purge |
| User-personalized content | ⚠️ Avoid or bypass | 0 (`no-store`) | N/A |
| Authenticated content | ❌ No (default) | 0 | N/A |
| Real-time data (prices, stock) | ⚠️ Short TTL only | 1-30s | Event-driven purge + surrogate keys |

### The Five Vocabulary Words That Signal Expertise

1. **Cache Hit Ratio** — the percentage of requests served from cache vs. total requests. A good CDN should achieve 80-95%+ for static workloads.
2. **Origin Shield** — secondary caching layer to prevent thundering herd on origin.
3. **Surrogate Keys** — tag-based invalidation for invalidating groups of related cached objects atomically.
4. **Stale-While-Revalidate** — serve stale content while asynchronously refreshing; eliminates "first request after TTL" latency penalty.
5. **Content Addressing** — embedding content hash in URL to make assets permanently cacheable and eliminating invalidation complexity for static content.

---

## Part 9: Real-World CDN Providers at a Glance

Understanding the major CDN providers and their differentiating features is useful context for interviews at companies that have made infrastructure decisions around them.


| Provider | Key Feature | Best For | Invalidation Speed |
| :-- | :-- | :-- | :-- |
| **Cloudflare** | Anycast network, Workers (edge compute) | Security + performance | ~500ms global |
| **AWS CloudFront** | Deep AWS integration, Lambda@Edge | AWS-native workloads | 1-2 minutes |
| **Fastly** | Instant Purge (~150ms), real-time logs | Publishing, e-commerce | ~150ms |
| **Akamai** | Largest PoP network (4,000+ PoPs) | Enterprise, streaming | Minutes |
| **Google Cloud CDN** | Tight GCP integration, HTTP/3 support | GCP-native workloads | Minutes |

Netflix uses multiple CDNs simultaneously and routes traffic dynamically based on real-time performance telemetry — a strategy called **multi-CDN** that provides both redundancy and performance optimization. This is a pattern worth mentioning in a senior-level interview.

---

## Closing Thoughts — The Mental Model That Ties It All Together

We started in a supermarket. We end back there.

The supermarket chain analogy holds all the way through to invalidation:

- **TTL expiration** = the milk's sell-by date. When it expires, the store discards it and orders fresh stock from the warehouse.
- **Cache purge** = headquarters calling the store and saying, "Throw out all the blue sneakers, we've changed the price. Get new stock now."
- **Surrogate keys** = headquarters saying, "Throw out everything tagged 'Nike' — not just the sneakers, but the jerseys, the socks, the hats, everything with that label."
- **Content hashing** = instead of updating the product, you put a brand-new SKU on the shelf with a different barcode. The old barcode still works for old receipts; the new barcode is immediately available.
- **Stale-while-revalidate** = a customer picks up the almost-expired milk while the stock clerk is *already walking to the back to get fresh stock*. The customer isn't kept waiting.
