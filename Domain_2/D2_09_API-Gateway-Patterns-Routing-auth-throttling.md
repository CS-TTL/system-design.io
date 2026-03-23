
# API Gateway Patterns: Routing, Authentication & Throttling

---

## Part 1: The Hotel Concierge — A Bridge to the Concept

Imagine you have just arrived at a large, sprawling hotel. The hotel has dozens of departments — a restaurant, a spa, a gym, a concierge desk, a valet, room service, housekeeping, and a business center. As a guest, you do not sprint from department to department every time you need something. You walk up to a single desk — the **front desk** — and say, *"I need a dinner reservation,"* or *"I need my car,"* or *"Send towels to Room 412."*

The front desk does three remarkable things, invisibly and instantly:

1. **It verifies who you are.** Are you a registered guest? Do you have a platinum membership that grants access to the executive lounge?
2. **It routes your request** to the correct department — the restaurant team, the valet team, or housekeeping.
3. **It manages the flow of requests.** During peak check-in hours, it controls the queue so that no single department becomes overwhelmed.

Now replace "hotel" with a modern software application, "departments" with backend microservices, and "front desk" with an **API Gateway** — and you have just understood the single most important pattern in distributed systems architecture.

The API Gateway is the *front desk of your software system*. Every client request — from a mobile app, a web browser, or a third-party integration — passes through this single entry point. It decides where the request goes, whether the caller is allowed to make it, and how many requests it will serve per unit of time.

In this article, we will build our understanding of the API Gateway from the ground up. We start with the atomic unit — a simple reverse proxy — and layer in routing intelligence, authentication mechanisms, and throttling algorithms, one concept at a time. By the end, you will have the mental model to design and reason about API Gateways in any system design interview or production architecture.

---

## Part 2: The Simplest Gateway — A Reverse Proxy

Before we talk about routing strategies or throttling algorithms, we need to establish a baseline. What is the most primitive form of an API Gateway?

It is a **reverse proxy**.

A regular proxy sits in front of the *client* — it hides who the requester is. A reverse proxy sits in front of the *servers* — it hides where the service actually lives. When a client sends `GET /users/42`, it does not know (and should not care) whether the User Service lives at `10.0.1.22:3001` or `10.0.1.55:3003`. It just talks to the gateway.

```

                  ┌───────────────────────────────────────┐
    Client          │           API GATEWAY                 │
──────────────► │  - Receives all inbound requests      │
GET /users/42   │  - Forwards to correct upstream       │──► User Service
│  - Returns the upstream response      │
└───────────────────────────────────────┘

```

*In this first figure, the gateway acts purely as a transparent pass-through. The client sees one address; the services remain completely hidden. This is the "atomic unit" we build everything else on top of.*

Here is the most minimal Python implementation, using `httpx` as the proxy transport:

```python
import httpx
from fastapi import FastAPI, Request
from fastapi.responses import Response

app = FastAPI()

SERVICES = {
    "users":    "http://user-service:3001",
    "orders":   "http://order-service:3002",
    "products": "http://product-service:3003",
}

@app.api_route("/{service}/{path:path}", methods=["GET", "POST", "PUT", "DELETE"])
async def reverse_proxy(service: str, path: str, request: Request):
    upstream_url = SERVICES.get(service)
    if not upstream_url:
        return Response(content="Service not found", status_code=404)

    async with httpx.AsyncClient() as client:
        url = f"{upstream_url}/{path}"
        resp = await client.request(
            method=request.method,
            url=url,
            headers=dict(request.headers),
            content=await request.body(),
        )
    return Response(content=resp.content, status_code=resp.status_code, headers=dict(resp.headers))
```

This works. Clients can reach any service through a single port. But we have already hit our first wall.

**The Limitation:** Every service is equally reachable by anyone. There is no authentication. There is no version management. There is no traffic shaping. A single bad actor can send 100,000 requests per second and take down every microservice behind the gateway simultaneously.

We need more. We need *intelligence* in our gateway. Let us add it, layer by layer.

---

## Part 3: Routing — Teaching the Gateway to Think

Routing is the "traffic cop" function of an API Gateway. At its heart, routing answers one question: *"Given this incoming request, which backend service should receive it?"*

We will explore routing in three tiers of sophistication: **path-based**, **header-based**, and **weighted routing**. Each tier solves a problem the previous one could not.

### Tier 1 — Path-Based Routing (The Foundation)

This is the most common routing strategy. The gateway inspects the URL path and uses it as a key to look up the destination service.

```
  Incoming Request            Gateway Routing Table           Upstream Service
  ─────────────────           ──────────────────────          ────────────────
  GET /users/42          ──► /users/*  → user-service    ──► http://users:3001/42
  POST /orders           ──► /orders/* → order-service   ──► http://orders:3002/
  GET /products/laptop   ──► /products/* → product-svc  ──► http://products:3003/laptop
```

*In Figure 2, notice how the gateway strips the service prefix from the path before forwarding. A request for `/users/42` becomes simply `/42` at the User Service's door — the service does not need to know it lives behind a gateway.*

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class RouteConfig:
    prefix: str
    upstream: str
    strip_prefix: bool = True

class PathRouter:
    def __init__(self, routes: list[RouteConfig]):
        # Sort by prefix length descending so more specific routes match first
        self.routes = sorted(routes, key=lambda r: len(r.prefix), reverse=True)

    def resolve(self, path: str) -> Optional[str]:
        for route in self.routes:
            if path.startswith(route.prefix):
                downstream_path = path[len(route.prefix):] if route.strip_prefix else path
                return f"{route.upstream}/{downstream_path.lstrip('/')}"
        return None

# Configuration
router = PathRouter([
    RouteConfig(prefix="/users",    upstream="http://user-service:3001"),
    RouteConfig(prefix="/orders",   upstream="http://order-service:3002"),
    RouteConfig(prefix="/products", upstream="http://product-service:3003"),
])

# Example
print(router.resolve("/users/42"))       # → http://user-service:3001/42
print(router.resolve("/orders/history")) # → http://order-service:3002/history
```

**The Limitation of Path-Based Routing:** It works brilliantly when all clients use the same version of your API. But what happens when you release `v2` of your User Service and need to keep `v1` alive for older clients? You cannot serve both from the same path prefix without leaking version logic into your service URL structure — which is brittle and ugly.

### Tier 2 — Header-Based Routing (API Versioning \& Tenancy)

When we cannot rely on the path alone, we look at the HTTP *headers* — the metadata envelope attached to every request. Header-based routing is the professional's tool for **API versioning**, **multi-tenancy**, and **A/B testing at the infrastructure layer**.

```
  Client A (mobile v1)                  Client B (web v2)
  ─────────────────────                 ─────────────────────
  GET /users/42                         GET /users/42
  X-API-Version: v1                     X-API-Version: v2
         │                                      │
         ▼                                      ▼
  ┌──────────────────────────────────────────────────────┐
  │                   API GATEWAY                        │
  │   Inspect X-API-Version header                       │
  │   v1 → http://user-service-v1:3001                   │
  │   v2 → http://user-service-v2:3002                   │
  └──────────────────────────────────────────────────────┘
```

*Figure 3 shows how the same URL path (`/users/42`) can be served by two completely different backends depending on a single request header. This is the cleanest way to manage API versioning without polluting URL structure.*

```python
from typing import Dict

class HeaderRouter:
    def __init__(self, header_name: str, routes: Dict[str, str], default: Optional[str] = None):
        self.header_name = header_name
        self.routes = routes
        self.default = default

    def resolve(self, headers: dict) -> Optional[str]:
        value = headers.get(self.header_name.lower())
        if value and value in self.routes:
            return self.routes[value]
        return self.default

# API versioning router
version_router = HeaderRouter(
    header_name="X-API-Version",
    routes={
        "v1": "http://user-service-v1:3001",
        "v2": "http://user-service-v2:3002",
    },
    default="http://user-service-v1:3001"  # v1 is the stable default
)

# Multi-tenancy router
tenant_router = HeaderRouter(
    header_name="X-Tenant-ID",
    routes={
        "tenant-alpha": "http://alpha-cluster:3001",
        "tenant-beta":  "http://beta-cluster:3001",
    }
)
```

Header-based routing is so powerful that it underpins entire SaaS multi-tenancy architectures, where a single gateway serves hundreds of isolated customer environments, each identified by a tenant header.

### Tier 3 — Weighted Routing (Canary Deployments \& Blue/Green)

This is where routing becomes genuinely strategic. Weighted routing allows us to split traffic across multiple versions of a service by *percentage*. It is the mechanism that powers **canary deployments** and **blue/green releases** — two of the safest ways to ship software to production.

The concept comes from the old mining practice of sending a canary into a coal mine before humans. We send a small percentage of *real* traffic to a new version of our service ("the canary"). If the canary survives — no elevated error rates, no degraded latency — we gradually shift more traffic over until the rollout is complete.

```
  100 Incoming Requests
        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │            WEIGHTED ROUTER                      │
  │  Random float in [0.0, 1.0) generated           │
  │                                                 │
  │  0.00 ─ 0.90  (90%) → users-stable:3001         │◄── 90 requests
  │  0.90 ─ 1.00  (10%) → users-canary:3001         │◄── 10 requests
  └─────────────────────────────────────────────────┘
```

*Figure 4 shows the probabilistic nature of weighted routing. No individual request is "guaranteed" a destination — instead, over a large enough sample, the distribution converges to the configured weights.*

```python
import random
from dataclasses import dataclass

@dataclass
class WeightedTarget:
    url: str
    weight: int  # relative weight, e.g., 90 and 10 = 90%/10% split

class WeightedRouter:
    def __init__(self, targets: list[WeightedTarget]):
        self.targets = targets
        self.total_weight = sum(t.weight for t in targets)

    def select(self) -> str:
        roll = random.uniform(0, self.total_weight)
        cumulative = 0
        for target in self.targets:
            cumulative += target.weight
            if roll < cumulative:
                return target.url
        return self.targets[-1].url  # fallback

# Canary deployment: 10% of traffic to new version
canary_router = WeightedRouter([
    WeightedTarget(url="http://user-service-stable:3001", weight=90),
    WeightedTarget(url="http://user-service-canary:3001", weight=10),
])

# Simulate 10 requests
for _ in range(10):
    print(canary_router.select())
```

We now have a routing layer that is intelligent, version-aware, and capable of safe, progressive deployments. But we still have a critical gap. The gateway happily routes *any* request, from *anyone*. It is time to install a lock on the front door.

---

## Part 4: Authentication — Who Are You, and Do You Belong Here?

Let us step back to our hotel analogy for a moment. Imagine the front desk never asked for your ID. Anyone could walk in off the street, claim to be in Room 412, and demand free room service. Chaos.

Authentication at the API Gateway is your ID check. It is the layer that answers the question: *"Before I route this request to my backend services, do I trust the caller?"*

There are three dominant patterns for API Gateway authentication, each representing a generation of thinking about the problem. We will study them in historical order.

### Pattern 1 — API Keys (The Hotel Room Card)

API keys are the oldest and simplest mechanism. A client is issued a random string (the "key"), which they include in every request — typically in a header like `X-API-Key`. The gateway looks up the key in a database or cache and either allows or rejects the request.

```
  Client                    API Gateway                  Key Store (Redis)
  ──────                    ───────────                  ─────────────────
  GET /orders               Extract X-API-Key header
  X-API-Key: abc123   ──►   Lookup "abc123" in cache ──► { client: "MobileApp",
                            If found → forward              rate_limit: 1000/hr,
                            If not found → 401              scopes: ["read"] }
```

*In Figure 5, the gateway acts as a fast cache lookup. Storing keys in Redis gives sub-millisecond validation without hitting a database on every request.*

```python
import redis
import json
from fastapi import HTTPException, Header
from typing import Optional

redis_client = redis.Redis(host="localhost", port=6379, decode_responses=True)

async def validate_api_key(x_api_key: Optional[str] = Header(None)) -> dict:
    if not x_api_key:
        raise HTTPException(status_code=401, detail="API key required")

    key_data = redis_client.get(f"apikey:{x_api_key}")
    if not key_data:
        raise HTTPException(status_code=401, detail="Invalid API key")

    return json.loads(key_data)
```

**The Limitation of API Keys:** They are stateless secrets. Once issued, there is no expiry mechanism unless you build one. More critically, they grant monolithic access — there is no built-in concept of *what* the caller is allowed to do, only *who* they are. You cannot say, "This key can read orders but not create them." For that granularity, we need a richer mechanism.

### Pattern 2 — JWT (JSON Web Tokens): The Self-Contained Passport

JWTs solve the limitations of API keys with an elegant insight: *what if the token itself carried all the information needed to authenticate and authorize, without a database lookup?*

A JWT is a digitally signed, base64-encoded string with three parts:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9   ← Header (algorithm)
.
eyJzdWIiOiJ1c2VyXzQyIiwic2NvcGVzIjpbInJlYWQ6b3JkZXJzIl0sImV4cCI6MTc0MDAwMH0
                                         ← Payload (claims: user ID, scopes, expiry)
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV...  ← Signature (HMAC-SHA256 or RS256)
```

The gateway verifies the *signature* using a public key from the identity provider. If the signature is valid, the gateway trusts all the claims in the payload — **without** hitting a database. This is the critical performance advantage.

```
  Client                       API Gateway                   Auth Server
  ──────                       ───────────                   ───────────
  Step 1: Login                                              Issues JWT signed
  username + password    ──────────────────────────────►    with private key
                         ◄──────────────────────────────    returns JWT

  Step 2: Use JWT
  GET /orders
  Authorization:               Decode JWT header
  Bearer eyJhbG...       ──►   Fetch public key from JWKS endpoint (cached)
                               Verify signature locally
                               Check: exp (not expired)?
                               Check: aud (right audience)?
                               Extract scopes → ["read:orders"]
                               Route request ──────────────────────────────►
```

*Figure 6 shows the two-phase JWT flow. Phase 1 (login) happens once; Phase 2 (verification) is entirely local — the gateway never calls the Auth Server again for a given request. This is why JWTs scale beautifully.*

```python
import jwt
from jwt import PyJWKClient
from fastapi import HTTPException, Header
from typing import Optional
import time

JWKS_URL = "https://auth.example.com/.well-known/jwks.json"
jwks_client = PyJWKClient(JWKS_URL)

async def validate_jwt(authorization: Optional[str] = Header(None)) -> dict:
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Bearer token required")

    token = authorization.split(" ")

    try:
        # Fetch the signing key from the JWKS endpoint (cached automatically)
        signing_key = jwks_client.get_signing_key_from_jwt(token)

        # Decode and verify the token
        payload = jwt.decode(
            token,
            signing_key.key,
            algorithms=["RS256"],
            audience="api.example.com",
        )

        # Verify expiry explicitly
        if payload.get("exp", 0) < time.time():
            raise HTTPException(status_code=401, detail="Token expired")

        return payload  # Returns claims: sub, scopes, exp, iat, etc.

    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError as e:
        raise HTTPException(status_code=401, detail=f"Invalid token: {e}")
```

**The Limitation of JWTs:** Because verification is stateless, there is no easy way to *revoke* a JWT before it expires. If a user logs out, or a key is compromised, the JWT remains valid until its `exp` claim is reached. This is the infamous "JWT revocation problem" — typically solved with short expiry times (15 minutes) combined with refresh tokens, or a token blacklist stored in Redis.

### Pattern 3 — OAuth 2.0: The Delegated Authorization Framework

OAuth 2.0 is not just an authentication pattern — it is a full **authorization framework**. It solves a specific problem that API keys and JWTs cannot easily address: *How can a user grant a third-party application access to their data, without handing over their password?*

Think of it as the hotel giving your cleaner a room key that only works between 9 AM and 12 PM on Tuesdays, and *only* opens the door — not the minibar. That granular, delegated, time-scoped access is what OAuth 2.0 provides.

```
  User (Resource Owner)      Client App           Auth Server      API Gateway
  ─────────────────────      ──────────           ───────────      ───────────
  "I want to use App X"
                         ──► Redirect user
                             to login page
                                          ──────────────────────►
  Login + Consent
  "Allow App X to            "Auth Code
   read your orders"         granted"
                         ◄──────────────────────────────────────
                             Exchange code
                             for Access Token  ──────────────────►
                             (Client Secret)
                             Access Token
                         ◄──────────────────────────────────────
  API Call
  GET /orders
  Bearer <access_token>  ──────────────────────────────────────► Validate token
                                                                  Check scopes
                                                                  Route request
```

*Figure 7 maps the full OAuth 2.0 Authorization Code flow. Notice that the user's actual password never reaches the Client App — only the Auth Server ever sees it. This is the security property that makes OAuth 2.0 the standard for third-party integrations.*

The API Gateway's role in OAuth 2.0 is to act as the **Resource Server** — it validates the access token (which is typically a JWT) and enforces *scope-based authorization*:

```python
async def require_scope(required_scope: str, token_claims: dict):
    """
    Enforces OAuth 2.0 scope-based authorization at the gateway level.
    Scopes in the JWT claim look like: "read:orders write:orders admin:all"
    """
    scopes = token_claims.get("scope", "").split()

    if required_scope not in scopes:
        raise HTTPException(
            status_code=403,
            detail=f"Insufficient scope. Required: '{required_scope}'"
        )

# Usage in a route handler
@app.get("/orders/{order_id}")
async def get_order(order_id: str, claims: dict = Depends(validate_jwt)):
    await require_scope("read:orders", claims)
    # ... proxy to order service
```


### Choosing Your Authentication Pattern

| Pattern | Best For | Statefulness | Revocation | Complexity |
| :-- | :-- | :-- | :-- | :-- |
| **API Keys** | Server-to-server, internal services | Stateful (DB/Cache) | Immediate (delete key) | Low |
| **JWT** | Stateless auth, microservices | Stateless | Hard (needs blacklist) | Medium |
| **OAuth 2.0** | Third-party access, user-delegated auth | Stateless (with JWT) | Refresh token revocation | High |

A well-architected gateway typically uses **all three** — API keys for machine-to-machine internal calls, JWTs for end-user sessions, and OAuth 2.0 for third-party integrations.

---

## Part 5: Throttling — Teaching the Gateway to Say "Slow Down"

We have a smart router. We have a secure front door. Now we face the final boss: **traffic abuse**.

Imagine you run a popular travel booking API. On a busy Monday morning, your gateway receives 50,000 requests per second. Somewhere in that traffic is one poorly-written bot hammering your `/flights/search` endpoint 10,000 times a minute. Without throttling, that bot can degrade service for every legitimate user — and potentially bring down your entire fleet of backend services.

Throttling (or rate limiting) is the mechanism by which the gateway enforces a **ceiling on how many requests a given caller can make within a given time window**. But the devil is in the algorithm. Let us study the four dominant approaches.

### Algorithm 1 — Fixed Window Counter (The Stopwatch)

This is the simplest algorithm. We divide time into discrete, fixed windows (e.g., each 60-second interval). For each window, we keep a counter per client. When the counter exceeds the limit, we reject with `429 Too Many Requests`.

```
  Time     ─────────────────────────────────────────────────────────────
           │   Window 1 (0-60s)   │   Window 2 (60-120s)   │
  Requests ─────────────────────────────────────────────────────────────
           ●●●●●●●●●●● (100 req)  │ ●●●●●●●●●● (100 req)   │
           LIMIT HIT at req 101   │ Counter resets to 0     │
           → 429 returned         │ Fresh window begins     │
```

*Figure 8 shows the fundamental flaw in the fixed window approach: a "boundary burst" attack. A client can make 100 requests at 11:59:59 and another 100 at 12:00:01 — that is 200 requests in 2 seconds, perfectly within the rules but wildly over the intended rate.*

```python
import time
import redis

class FixedWindowRateLimiter:
    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window_seconds = window_seconds
        self.redis = redis.Redis(host="localhost", port=6379, decode_responses=True)

    def is_allowed(self, client_id: str) -> tuple[bool, dict]:
        # The window key is tied to the current time bucket
        window_key = int(time.time() // self.window_seconds)
        redis_key = f"ratelimit:{client_id}:{window_key}"

        # Atomic increment using Redis pipeline
        pipe = self.redis.pipeline()
        pipe.incr(redis_key)
        pipe.expire(redis_key, self.window_seconds * 2)  # cleanup buffer
        count, _ = pipe.execute()

        remaining = max(0, self.limit - count)
        reset_at = (window_key + 1) * self.window_seconds

        return count <= self.limit, {
            "X-RateLimit-Limit": str(self.limit),
            "X-RateLimit-Remaining": str(remaining),
            "X-RateLimit-Reset": str(int(reset_at)),
        }
```

**The Limitation:** The boundary burst problem. We need an algorithm that tracks traffic more smoothly across time windows.

### Algorithm 2 — Token Bucket (The Patience Tank)

The token bucket is the most widely deployed algorithm in production API gateways — AWS API Gateway, Kong, and Nginx all use it. The intuition is elegant:

- Imagine a bucket with a maximum capacity of `N` tokens.
- Tokens are added to the bucket at a steady rate (e.g., 10 per second).
- Each request consumes one token.
- If the bucket is empty, the request is rejected.
- If the bucket is full and new tokens arrive, they are discarded (they do not accumulate beyond capacity).

```
  Time 0s:  Bucket [●●●●●●●●●●] (10 tokens full)
  
  Burst at 0s: Client sends 10 requests rapidly
               Bucket [          ] (0 tokens remaining — but burst was served!)
  
  Time 1s:  +10 tokens added
             Bucket [●●●●●●●●●●] (10 tokens)
  
  Time 1s:  5 requests arrive
             Bucket [●●●●●     ] (5 tokens remaining)
  
  Time 2s:  +10 tokens added, capped at 10 (bucket capacity)
             Bucket [●●●●●●●●●●] (10 tokens — capped at max)
```

*Figure 9 illustrates the key property of the token bucket: it **allows controlled bursting**. A well-behaved client that has been idle can immediately use all accumulated tokens for a burst of requests. This is why it is preferred for APIs that need to support legitimate traffic spikes, like a user uploading a batch of files.*

```python
import time
import redis

class TokenBucketRateLimiter:
    def __init__(self, capacity: int, refill_rate: float):
        """
        capacity:    max tokens (also max burst size)
        refill_rate: tokens added per second
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.redis = redis.Redis(host="localhost", port=6379, decode_responses=True)

    def is_allowed(self, client_id: str) -> tuple[bool, dict]:
        now = time.time()
        key_tokens = f"token_bucket:{client_id}:tokens"
        key_last   = f"token_bucket:{client_id}:last_refill"

        pipe = self.redis.pipeline(True)  # WATCH for optimistic locking
        try:
            pipe.watch(key_tokens, key_last)

            tokens     = float(pipe.get(key_tokens) or self.capacity)
            last_refill = float(pipe.get(key_last)  or now)

            # Calculate how many tokens to add since last request
            elapsed = now - last_refill
            new_tokens = elapsed * self.refill_rate
            tokens = min(self.capacity, tokens + new_tokens)

            allowed = tokens >= 1.0
            if allowed:
                tokens -= 1.0

            pipe.multi()
            pipe.set(key_tokens, tokens, ex=3600)
            pipe.set(key_last, now, ex=3600)
            pipe.execute()

            return allowed, {
                "X-RateLimit-Tokens-Remaining": str(int(tokens)),
                "X-RateLimit-Capacity": str(self.capacity),
            }
        except redis.WatchError:
            return False, {}  # Retry in calling code
```


### Algorithm 3 — Leaky Bucket (The Output-Smoothing Queue)

Where the token bucket asks *"Does the client have permission to send now?"*, the leaky bucket asks *"Can we process this request at our current output rate?"*

Think of the namesake: a physical bucket with a small hole in the bottom. Water pours in from the top (incoming requests). It drips out at a fixed rate from the hole (processing rate). If too much water pours in, the bucket overflows and the excess is discarded.

The critical difference from token bucket: **the leaky bucket smooths output to a constant rate**. It is ideal for protecting downstream services with strict capacity constraints, not for accommodating client bursts.

```
  ┌────────────────────────────────────────────┐
  │  Incoming Requests (variable rate)         │
  │  ●●●●●●●●●● ←── bursty arrivals           │
  └─────────────────────────────────────────── │
                           │ Overflow? DROP ●  │
                           ▼                   │
                  ┌────────────────┐           │
                  │  QUEUE (bucket)│ capacity=N│
                  │   ●●●●●        │           │
                  └───────┬────────┘           │
                          │ 1 request/interval │
                          ▼                    │
               Downstream Service (steady)     │
```

*Figure 10 reveals the leaky bucket's personality: it cares about the **downstream**, not the client. The queue absorbs bursts; the output rate is strictly enforced. If you need to protect a fragile legacy service that can only handle 100 req/s, the leaky bucket is your tool.*

```python
import asyncio
import time
from collections import deque

class LeakyBucketRateLimiter:
    def __init__(self, capacity: int, leak_rate: float):
        """
        capacity:  max items in the queue (bucket size)
        leak_rate: requests processed per second (leak speed)
        """
        self.capacity = capacity
        self.leak_rate = leak_rate
        self.queue: deque = deque()
        self.last_leak_time = time.time()

    def add_request(self, request_id: str) -> bool:
        self._leak()  # Process pending requests first

        if len(self.queue) < self.capacity:
            self.queue.append((request_id, time.time()))
            return True  # Accepted into queue
        else:
            return False  # Bucket full — request dropped

    def _leak(self):
        now = time.time()
        elapsed = now - self.last_leak_time
        leaked = int(elapsed * self.leak_rate)

        for _ in range(min(leaked, len(self.queue))):
            if self.queue:
                self.queue.popleft()  # Process at fixed rate

        if leaked > 0:
            self.last_leak_time = now
```


### Algorithm 4 — Sliding Window Counter (The Hybrid)

The sliding window counter is the production-preferred algorithm at scale companies. It combines the memory efficiency of the fixed window with the boundary-burst resistance of a full sliding window log.

The elegant trick: instead of storing every timestamp, we store **two counters** — the current window and the previous window — and compute a weighted estimate of the current load.

```
  Time now: 12:00:45 (45s into current 60s window)
  
  Previous window (12:00:00 → 12:00:59):  60 requests
  Current window  (12:00:00 → current):   30 requests
  
  Weight of previous window = (60s - 45s) / 60s = 0.25
  
  Estimated count = (60 × 0.25) + 30 = 15 + 30 = 45 requests
  
  If limit is 100/min → 45 < 100 → REQUEST ALLOWED ✓
```

*Figure 11 shows the weighted interpolation that makes sliding window counters work. We use the fraction of the previous window that "overlaps" with our current sliding window to get a smooth, accurate estimate of request density — without storing individual timestamps.*

```python
import time
import redis

class SlidingWindowRateLimiter:
    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window_seconds = window_seconds
        self.redis = redis.Redis(host="localhost", port=6379, decode_responses=True)

    def is_allowed(self, client_id: str) -> tuple[bool, dict]:
        now = time.time()
        current_window = int(now // self.window_seconds)
        previous_window = current_window - 1

        current_key  = f"sw:{client_id}:{current_window}"
        previous_key = f"sw:{client_id}:{previous_window}"

        # How far into the current window are we? (0.0 to 1.0)
        window_progress = (now % self.window_seconds) / self.window_seconds

        pipe = self.redis.pipeline()
        pipe.get(current_key)
        pipe.get(previous_key)
        results = pipe.execute()

        current_count  = int(results or 0)
        previous_count = int(results or 0)

        # Weight of the previous window's contribution
        previous_weight = 1.0 - window_progress
        estimated_count = (previous_count * previous_weight) + current_count

        if estimated_count >= self.limit:
            return False, {
                "X-RateLimit-Limit": str(self.limit),
                "X-RateLimit-Remaining": "0",
                "Retry-After": str(int(self.window_seconds - (now % self.window_seconds))),
            }

        # Increment current window counter
        pipe = self.redis.pipeline()
        pipe.incr(current_key)
        pipe.expire(current_key, self.window_seconds * 2)
        pipe.execute()

        remaining = max(0, self.limit - int(estimated_count) - 1)
        return True, {
            "X-RateLimit-Limit": str(self.limit),
            "X-RateLimit-Remaining": str(remaining),
        }
```


### Choosing Your Rate Limiting Algorithm

| Algorithm | Burst Handling | Memory Use | Smoothness | Best For |
| :-- | :-- | :-- | :-- | :-- |
| **Fixed Window** | Poor (boundary burst) | O(1) | Low | Simple services, internal tools |
| **Token Bucket** | Excellent (controlled burst) | O(1) | Medium | Public APIs, user-facing endpoints |
| **Leaky Bucket** | Drops bursts | O(N) queue | High | Protecting fragile downstreams |
| **Sliding Window** | Good (smooth estimate) | O(1) | High | High-scale production APIs |


---

## Part 6: Assembling the Complete Gateway

We have studied each layer in isolation. Now let us see how they compose into a single, coherent gateway with all three capabilities working together.

```
                         ┌──────────────────────────────────────────────────────┐
  Incoming Request        │               API GATEWAY PIPELINE                   │
  ─────────────────────► │                                                       │
                         │  Step 1: TLS Termination                              │
                         │  ─────────────────────────────────────────────────   │
                         │  Decrypt HTTPS, work in plaintext internally          │
                         │                                                       │
                         │  Step 2: Authentication Middleware                    │
                         │  ─────────────────────────────────────────────────   │
                         │  -  API Key? → Redis lookup                            │
                         │  -  JWT Bearer? → Signature verify + claims extract    │
                         │  -  Missing auth? → 401 Unauthorized                  │
                         │                                                       │
                         │  Step 3: Throttling Middleware                        │
                         │  ─────────────────────────────────────────────────   │
                         │  -  Check sliding window counter for client_id         │
                         │  -  Exceeded limit? → 429 Too Many Requests            │
                         │  -  Add X-RateLimit-* headers to response              │
                         │                                                       │
                         │  Step 4: Routing Middleware                           │
                         │  ─────────────────────────────────────────────────   │
                         │  -  Path match → determine upstream service            │
                         │  -  Header routing → version selection                 │
                         │  -  Weighted routing → canary/stable split             │
                         │                                                       │
                         │  Step 5: Request Forwarding                           │
                         │  ─────────────────────────────────────────────────   │
                         │  -  Strip sensitive headers (Authorization)            │
                         │  -  Inject internal headers (X-User-ID from JWT sub)  │
                         │  -  Forward to upstream service                        │
                         └──────────────────────────────────────────────────────┘
```

*Figure 12 shows the complete middleware pipeline. Notice the deliberate order: authentication before throttling (we cannot rate-limit by client ID if we do not know who the client is), and throttling before routing (no point resolving an upstream address if the request will be rejected anyway). Order matters enormously.*

Here is the complete gateway assembled as a FastAPI middleware pipeline:

```python
from fastapi import FastAPI, Request, Response
from fastapi.responses import JSONResponse
import httpx

app = FastAPI()

# Instantiate our components
path_router    = PathRouter(routes=[...])
jwt_validator  = JWTValidator(jwks_url="https://auth.example.com/.well-known/jwks.json")
rate_limiter   = SlidingWindowRateLimiter(limit=100, window_seconds=60)

@app.middleware("http")
async def gateway_pipeline(request: Request, call_next):
    # ── Step 1: Authentication ──────────────────────────────────────────
    auth_header = request.headers.get("authorization", "")
    if not auth_header.startswith("Bearer "):
        return JSONResponse(status_code=401, content={"error": "Bearer token required"})

    try:
        claims = jwt_validator.validate(auth_header.split(" "))
    except Exception:
        return JSONResponse(status_code=401, content={"error": "Invalid or expired token"})

    client_id = claims.get("sub", request.client.host)

    # ── Step 2: Throttling ──────────────────────────────────────────────
    allowed, rate_headers = rate_limiter.is_allowed(client_id)
    if not allowed:
        response = JSONResponse(
            status_code=429,
            content={"error": "Rate limit exceeded. See Retry-After header."}
        )
        for k, v in rate_headers.items():
            response.headers[k] = v
        return response

    # ── Step 3: Routing ─────────────────────────────────────────────────
    upstream_url = path_router.resolve(request.url.path)
    if not upstream_url:
        return JSONResponse(status_code=404, content={"error": "Route not found"})

    # ── Step 4: Forward Request ─────────────────────────────────────────
    headers = dict(request.headers)
    headers.pop("authorization", None)       # strip credentials before forwarding
    headers["X-User-ID"]    = client_id      # inject enriched identity
    headers["X-User-Scopes"] = " ".join(claims.get("scopes", []))

    async with httpx.AsyncClient() as client:
        response = await client.request(
            method=request.method,
            url=upstream_url,
            headers=headers,
            content=await request.body(),
            params=dict(request.query_params),
            timeout=10.0,
        )

    gateway_response = Response(
        content=response.content,
        status_code=response.status_code,
        headers=dict(response.headers),
    )

    # Attach rate limit headers to all responses
    for k, v in rate_headers.items():
        gateway_response.headers[k] = v

    return gateway_response
```


---

## Part 7: Interview Patterns \& Design Considerations

In a system design interview, the API Gateway is one of the most frequently asked about components. Here are the key talking points and trade-offs to demonstrate mastery.

### The "Where Should Auth Live?" Debate

A common interview trap is the question: *"Should authentication happen at the gateway, or in each microservice?"*

The answer is: **both, for different reasons**.

- The **gateway handles authentication** — verifying *who* the caller is (JWT signature verification, API key validation). This is a cross-cutting concern that should not be duplicated across services.
- Each **service handles authorization** — deciding *what* the authenticated caller is allowed to do within its domain. A User Service should decide if `user_42` can see `user_99`'s data; that is not the gateway's business.

The pattern of "auth at the gateway, authz in the service" is called the **Outer and Inner Ring** model — and it is the correct answer for MAANG-level design interviews.

### Distributed Rate Limiting

Our single-node Redis implementation works for one gateway instance. But in a production system, you have dozens of gateway replicas behind a load balancer. How do you ensure rate limits are enforced globally, not per-instance?

The standard answer has three tiers:

1. **Centralized Redis**: All gateway nodes share a single Redis cluster. Every rate-limit check is a network call to Redis. Simple and correct, but adds latency.
2. **Local + Sync (Redis Async)**: Each gateway node maintains a local in-memory counter. Periodically (every 100ms), it syncs the delta to Redis. Slightly inaccurate but extremely low latency.
3. **Gossip Protocol**: Gateway nodes communicate peer-to-peer (like Apache Kafka's ZooKeeper election), sharing rate limit state. Complex but avoids the Redis bottleneck entirely.

For most interviews, describing option 2 with an acknowledgment of the consistency trade-off will satisfy the interviewer.

### Circuit Breakers: The Gateway's Emergency Brake

A healthy gateway does more than rate-limit clients. It also protects clients from *unhealthy upstreams*. If the Order Service starts returning HTTP 500 errors at 90% rate, blindly forwarding requests is wasteful and damaging.

The **Circuit Breaker** pattern (named by Michael Nygard in his seminal book *Release It!*) adds a state machine at the gateway:

```
  CLOSED (normal) → too many failures → OPEN (reject all) → timeout → HALF-OPEN
       ▲                                                                   │
       └───────────────── success in half-open ────────────────────────────┘
```

When the circuit is **OPEN**, the gateway immediately returns a `503 Service Unavailable` to clients without ever touching the unhealthy backend — protecting the downstream service from being overwhelmed during recovery.

### Observability: What Every Gateway Should Emit

A gateway sitting at the entry point of your entire system is uniquely positioned to emit rich telemetry. Every production gateway should produce:

- **Structured access logs**: `client_id`, `route`, `upstream`, `latency_ms`, `status_code`, per request
- **Metrics**: `requests_per_second` (labeled by route and client tier), `p50/p95/p99 latency`, `4xx/5xx error rates`, `rate_limit_rejections`
- **Distributed traces**: Inject a `X-Trace-ID` header so requests can be tracked end-to-end across all services (OpenTelemetry is the standard)

A gateway without telemetry is a black box. A gateway with telemetry is a *control plane*.

---

## Part 8: Putting It All Together — The Mental Model

Let us compress everything we have learned into a single mental model you can carry into any interview or design session.

An API Gateway is a **programmable reverse proxy** that enforces three core concerns:

1. **Routing**: "Where does this request go?" — answered by path, header, and weighted routing strategies, enabling versioning, multi-tenancy, and safe deployments.
2. **Authentication**: "Who is making this request?" — answered through API keys (simple, stateful), JWTs (scalable, stateless), and OAuth 2.0 (delegated, scope-based), each suited for different trust relationships.
3. **Throttling**: "How much should this client be allowed to consume?" — answered by rate-limiting algorithms that range from simple fixed-window counters to sophisticated sliding-window hybrids, with token buckets remaining the industry workhorse.

The gateway is the hotel front desk: it authenticates your identity, routes your request to the right department, and ensures no single guest monopolizes the staff. Everything behind it — the microservices, the databases, the message queues — gets to focus on *doing its job*, free from the concern of who is knocking on the door.

Mastering this single pattern — the API Gateway — gives you leverage over a surprisingly large portion of modern system design. Authentication, load balancing, versioning, abuse prevention, observability: they all flow through this one chokepoint. Design it well, and the rest of your architecture gets to breathe.

```
