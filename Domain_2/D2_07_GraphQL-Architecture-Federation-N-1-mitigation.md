

# GraphQL Architecture: Federation \& N+1 Mitigation

### A Field Guide for Senior Engineer Interviews

> *"The art of architecture is not in building one great wall — it is in knowing how to build a thousand smaller ones so they behave as one."*

***

## Part 1 — The Bridge: A City of Departments

Imagine you are the mayor of a fast-growing city. In the early days, City Hall handled everything: building permits, tax records, marriage licenses, utility bills. One building, one address, one receptionist. Citizens loved it. Simple.

Then the city tripled in size.

Now the single receptionist is a bottleneck. The permit department cannot update their records without restarting the entire building. If the utilities wing catches fire, *all* services go down. The tax team wants to migrate to a new filing system, but cannot — because the whole building shares one ancient mainframe.

Sound familiar? This is *exactly* the story of monolithic GraphQL at enterprise scale.

The solution the city eventually adopts is **federation**: separate department buildings (each owning their own records and staff), connected by a **central directory** that knows which building handles what request. A citizen calls one phone number, the directory routes them. From the citizen's perspective, it is still one city. Under the hood, it is a distributed network of specialized offices.

This metaphor is not decorative. We will return to it throughout the article because every concept in GraphQL Federation maps directly onto it. By the time we finish, you should be able to explain subgraphs, gateways, query planning, and N+1 mitigation using nothing but this city as your mental model.

***

## Part 2 — GraphQL Refresher: The Atomic Unit

Before we scale, we need to be precise about the atomic unit we are scaling. GraphQL is a **query language for APIs** and a **runtime for fulfilling those queries**. Its power over REST lies in three properties:

1. **Declarative fetching**: The client specifies exactly what fields it needs — no more over-fetching.
2. **Single endpoint**: All queries go to one URL (`/graphql`), regardless of the resource.
3. **Strongly typed schema**: The schema is a contract between client and server, introspectable and self-documenting.

Here is the simplest possible GraphQL interaction. A schema defines a `User` type, a client queries it, and a **resolver** fetches the data.

```python
# schema.py — the "contract"
type_defs = """
    type Query {
        user(id: ID!): User
    }

    type User {
        id: ID!
        name: String
        email: String
    }
"""

# resolvers.py — the "fulfillment"
def resolve_user(obj, info, id):
    return db.users.find_one({"_id": id})

resolvers = {
    "Query": {
        "user": resolve_user
    }
}
```

A **resolver** is a function that fetches data for a specific field. GraphQL's execution engine walks the query tree, calling the correct resolver for every field the client requested. Hold onto this idea — resolvers calling functions per field — because it is the seed of the N+1 problem we will explore in Part 4.

***

## Part 3 — The Monolith Breaks: Identifying the Pain Point

Imagine our e-commerce platform starts small. We have one GraphQL server with everything: users, products, orders, reviews, inventory. Life is good.

```svgbob
+------------------------------------------------------+
|               MONOLITHIC GRAPHQL SERVER              |
|                                                      |
|  +----------+  +----------+  +----------+           |
|  |  Users   |  | Products |  |  Orders  |           |
|  | Resolver |  | Resolver |  | Resolver |           |
|  +----------+  +----------+  +----------+           |
|                                                      |
|  +----------+  +----------+                         |
|  | Reviews  |  | Inventory|                         |
|  | Resolver |  | Resolver |                         |
|  +----------+  +----------+                         |
|                                                      |
+------------------------------------------------------+
           |
           v
    Single /graphql endpoint
```

*Figure 1: The monolith — everything in one box. Simple to start, increasingly painful to grow.*

As the company scales, four concrete problems emerge:

**Problem 1 — Organizational Coupling.** The Users team and the Orders team both need to touch the same codebase. Every deployment requires coordination.

**Problem 2 — Independent Scaling is Impossible.** Product search gets 10x the traffic of order mutation. In a monolith, you scale *everything* — wasting compute and money.

**Problem 3 — Technology Lock-in.** The Reviews team wants a vector database. The Inventory team wants to write in Go. In a monolith, everyone uses the same stack.

**Problem 4 — Schema Blast Radius.** A breaking change to the `Product` type requires touching code across the entire codebase.

These four problems are not theoretical. They are why Netflix, Shopify, GitHub, and virtually every company beyond a certain size eventually walks away from the monolithic graph.

***

## Part 4 — GraphQL Federation: The Architecture

GraphQL Federation (popularized by Apollo in 2019 and now being standardized by the GraphQL Foundation) solves the monolith's four problems by splitting the graph into **independently owned services called subgraphs**, each with their own schema, database, and deployment pipeline.[^2]

A specialized orchestration layer — the **Gateway** (or **Router**) — presents one unified schema to the client (the **supergraph**), while routing query execution to the correct subgraphs behind the scenes.[^3]

```svgbob
                    +------------------+
                    |      CLIENT      |
                    +--------+---------+
                             |
                     Single  |  /graphql
                             v
              +--------------+--------------+
              |         GATEWAY / ROUTER    |
              |  (Query Planner + Executor) |
              +----+-------+------+---------+
                   |       |      |
         +---------+  +----+  +---+----------+
         |             |          |
+--------v---+  +------v-----+  +-v----------+
|  Users     |  | Products   |  |  Orders    |
|  Subgraph  |  |  Subgraph  |  |  Subgraph  |
|  :4001     |  |  :4002     |  |  :4003     |
+--------+---+  +------+-----+  +------+-----+
         |             |               |
      User DB      Product DB       Orders DB
```

*Figure 2: The federated architecture. Notice the client sees only one endpoint. The Gateway handles all routing decisions. Each subgraph owns its own schema, logic, and data store.*[^3]

***

### 4.1 — Subgraphs: The Department Buildings

A **subgraph** is a standard GraphQL service with one crucial addition: it is **federation-aware**. It declares which types and fields it owns and exposes a special `_entities` query that allows the Gateway to stitch cross-service data together.[^4]

```python
# users_subgraph/schema.py

type_defs = """
    type Query {
        me: User
    }

    # @key declares the "primary key" of this type
    # Tells the Gateway: "I am the authority on User entities identified by id"
    type User @key(fields: "id") {
        id: ID!
        name: String!
        email: String!
    }
"""

def resolve_user_reference(representation, info):
    # The Gateway calls this when another subgraph only has the user's ID
    return db.users.get(representation["id"])

resolvers = {
    "Query": {"me": resolve_me},
    "User": {"__resolveReference": resolve_user_reference}
}
```

The `@key(fields: "id")` directive is the lynchpin. It says: *"The User type is identified by its `id` field. If another subgraph references a User, provide just the `id`, and I will resolve the full object."* This is the city analogy at work — other departments can reference a citizen by their ID number, and the Citizens Department always knows how to turn that ID into a full record.

***

### 4.2 — Schema Composition: The Supergraph

Each subgraph publishes its schema. The **schema composition** process merges all subgraph schemas into one unified **supergraph schema** — run via Apollo GraphOS or the Rover CLI.[^5]

```python
# products_subgraph — owns the base Product type
products_type_defs = """
    type Product @key(fields: "id") {
        id: ID!
        name: String!
        price: Float!
    }
"""

# reviews_subgraph — EXTENDS Product with reviews
reviews_type_defs = """
    type Product @key(fields: "id") @extends {
        id: ID! @external       # I don't own this, but I need it to join
        reviews: [Review]       # I DO own this
    }

    type Review {
        id: ID!
        body: String!
        rating: Int!
    }
"""

# After composition, the CLIENT sees:
supergraph_view = """
    type Product {
        id: ID!
        name: String!      # Owned by Products subgraph
        price: Float!      # Owned by Products subgraph
        reviews: [Review]  # Owned by Reviews subgraph
    }
"""
```

This is federation's killer feature: **the Reviews team extended the `Product` type without touching the Products team's codebase**. Two teams, two deployments, one seamless schema.

***

### 4.3 — The Gateway \& Query Planning

The Gateway is where the real intelligence lives. When a client sends a query, the Gateway runs a **query planner** that builds a **directed acyclic graph (DAG)** of fetch operations and executes them, potentially in parallel.[^3]

Let us trace a real query:

```svgbob
 Client Query: product(id) { name price reviews { body author { name } } }
      |
      v
+------------------------------------------+
|           QUERY PLANNER                  |
|                                          |
|  Step 1: Fetch from Products Subgraph    |
|    query { product(id:$id) {             |
|        name price __typename id } }      |
|                                          |
|  Step 2: Fetch from Reviews Subgraph     | <-- depends on Step 1
|    _entities(representations:[{          |
|      __typename:"Product", id:"abc" }])  |
|    { ... on Product { reviews { body     |
|      author { id } } } }                 |
|                                          |
|  Step 3: Fetch from Users Subgraph       | <-- depends on Step 2
|    _entities(representations:[{          |
|      __typename:"User", id:"u1" }])      |
+------------------------------------------+
```

*Figure 3: The query execution plan forms a dependency chain. Notice how the Gateway uses `_entities` — the special federated query — to fetch cross-service data using only the entity's key fields.*[^6]

***

### 4.4 — Key Federation Directives

Think of federation directives as the **vocabulary** of the federated schema language. We group them by *behavioral intent*, not alphabetically:

**Ownership Directives** — Who owns what?


| Directive | Meaning |
| :-- | :-- |
| `@key(fields: "id")` | Declares the primary key; marks the "home" subgraph |
| `@external` | This field is owned by another subgraph; I reference it |
| `@extends` | I am adding fields to a type defined in another subgraph |

**Dependency Directives** — What do I need to do my job?


| Directive | Meaning |
| :-- | :-- |
| `@requires(fields: "weight")` | I need this `@external` field to compute my own field |
| `@provides(fields: "name")` | I can provide this field without an extra subgraph hop |

**Schema Evolution Directives** — How do we change things safely?


| Directive | Meaning |
| :-- | :-- |
| `@inaccessible` | Hide from the supergraph (internal use only) |
| `@deprecated` | Mark a field for future removal |
| `@override` | Migrate field ownership from one subgraph to another |


***

## Part 5 — The N+1 Problem: The Other Monster in the Room

We have built our distributed city. Now let us talk about a subtle but devastating performance trap lurking inside every GraphQL server — federated or not. It is called the **N+1 problem**, and it is one of the most common senior-interview topics.

### 5.1 — The Library Analogy

Imagine a librarian asked: *"Give me the author's name for each of these 100 book IDs."*

A naive librarian walks to the shelf, pulls Book \#1, reads the author ID, walks to the author catalog, looks up the name, writes it down. Then walks back, pulls Book \#2, walks to the author catalog again... That is **100 catalog walks for 100 books**. The librarian made 1 request for the book list, then N separate requests for each author. Total: **1 + N = 101 database round trips**.

A smart librarian would collect all 100 author IDs first, then make **one trip to the author catalog** with a list, getting all 100 authors back in a single lookup. Total: **2 database round trips**.

This is not just a thought experiment. It is exactly what naive GraphQL resolvers do.

***

### 5.2 — How N+1 Manifests in GraphQL

```python
# The naive resolver implementation — looks harmless, is destructive
from db import get_all_posts, get_user_by_id

def resolve_posts(obj, info):
    # Query 1: Fetches all posts
    return get_all_posts()   # SELECT * FROM posts -> returns 50 posts

def resolve_post_author(post, info):
    # Called ONCE PER POST — 50 separate DB queries for 50 posts
    return get_user_by_id(post["author_id"])
    # SELECT * FROM users WHERE id = 'u1'
    # SELECT * FROM users WHERE id = 'u2'
    # SELECT * FROM users WHERE id = 'u1'  <- same user, fetched AGAIN
    # ... 50 times total

resolvers = {
    "Query": {"posts": resolve_posts},
    "Post": {"author": resolve_post_author}
}
```

Let us visualize the database activity this generates:

```svgbob
  Client Request
      |
      v
  +--------+
  | posts  |  --> SELECT * FROM posts  (1 query, returns 50 posts)
  +--------+
      |
      +-- Post[^0].author --> SELECT * FROM users WHERE id='u1'
      +-- Post.author --> SELECT * FROM users WHERE id='u2'
      +-- Post[^2].author --> SELECT * FROM users WHERE id='u1'  (DUPLICATE!)
      +-- Post[^3].author --> SELECT * FROM users WHERE id='u3'
      +-- ...
      +-- Post[^49].author --> SELECT * FROM users WHERE id='u7'

  Total DB Queries: 1 + 50 = 51 queries
  Distinct users: maybe 7
  Wasted queries: 43 (duplicates + unnecessary round trips)
```

*Figure 4: The N+1 problem visualized. The single `posts` query spawns N separate `author` queries — one per post. Many are redundant. This scales catastrophically as N grows.*

With 50 posts this might be tolerable. With 500 posts at 10ms per database round trip, that's **5 seconds** of pure database latency just for author resolution.

***

### 5.3 — DataLoader: The Standard Fix

The solution was formalized by Facebook's Lee Byron in 2015 with the **DataLoader** pattern. The core insight is beautifully simple:[^7]

> **Collect all the keys you need within a single "tick" of the event loop, then fetch them all in one batch.**

DataLoader operates as a two-phase system:

```svgbob
  Event Loop Tick

  Resolvers fire concurrently:
  Post[^0].author -> loader.load("u1")  ---|
  Post.author -> loader.load("u2")  ---+--> Keys queued
  Post[^2].author -> loader.load("u1")  ---|    (deduplicated)
  Post[^3].author -> loader.load("u3")  ---|

          |
          v (end of tick — batch fires)

  getUsersByIds(["u1", "u2", "u3"])
  --> SELECT * FROM users WHERE id IN ('u1', 'u2', 'u3')
  --> Returns [User(u1), User(u2), User(u3)]

  DataLoader maps results back to each original Promise:
  loader.load("u1") resolves -> User(u1)
  loader.load("u2") resolves -> User(u2)
  loader.load("u1") resolves -> User(u1)  (from cache, no extra query)
  loader.load("u3") resolves -> User(u3)

  Total DB Queries: 1 + 1 = 2  (instead of 1 + 50)
```

*Figure 5: DataLoader's two-phase operation. All `load()` calls within a single event loop tick are batched into one DB call. Duplicate keys (u1 appears twice) are deduplicated automatically.*[^8]

```python
# dataloader_setup.py
from aiodataloader import DataLoader
from db import get_users_by_ids

class UsersDataLoader(DataLoader):
    async def batch_load_fn(self, user_ids: list[str]):
        # One DB call for ALL ids collected in this tick
        users = await get_users_by_ids(user_ids)
        user_map = {u["id"]: u for u in users}
        # CRITICAL: return in the SAME ORDER as user_ids
        return [user_map.get(uid) for uid in user_ids]


# context.py — attach the loader to GraphQL context PER REQUEST
def create_request_context(request):
    return {
        "user_loader": UsersDataLoader(),  # Fresh instance per request!
        "request": request,
    }

# resolvers.py — the fixed version (identical interface, batched internally)
async def resolve_post_author(post, info):
    return await info.context["user_loader"].load(post["author_id"])
```

The interface is identical. The performance impact is dramatic: 51 queries become 2.[^7]

***

### 5.4 — The "Same Order" Contract: The Most Common DataLoader Bug

There is one rule that DataLoader beginners consistently get wrong. **The batch function must return results in the exact same order as the input keys, and must return exactly the same number of results.** If your database returns results sorted differently, your DataLoader will silently return wrong data — mapping User B's data to User A's post.[^7]

```python
# ❌ BROKEN: DB returns users sorted by name, not by input order
async def batch_load_users_broken(user_ids):
    users = await db.query(
        "SELECT * FROM users WHERE id = ANY($1) ORDER BY name", user_ids
    )
    return users  # WRONG ORDER — silent data corruption!

# ✅ CORRECT: Always re-map to match input order
async def batch_load_users_correct(user_ids):
    users = await db.query("SELECT * FROM users WHERE id = ANY($1)", user_ids)
    user_map = {u["id"]: u for u in users}
    return [user_map.get(uid) for uid in user_ids]  # Preserves input order
```

We recommend writing a unit test specifically for this behavior in any production DataLoader. It is a silent bug that only manifests under specific data conditions, making it especially dangerous.

***

### 5.5 — DataLoader Caching: Per-Request Memoization

DataLoader includes a **per-request memoization cache**. If `loader.load("u1")` is called five times in one request, the batch function only receives `"u1"` once.[^6]

**Important:** DataLoader caches are scoped per request. You must create a **new DataLoader instance for every incoming GraphQL request**. Sharing a DataLoader across requests causes stale data — User A's request would see User B's cached records. This is why we create the loader inside `create_request_context()`, not as a module-level singleton.

***

## Part 6 — The Distributed N+1: Federation's Specific Challenge

In a monolithic GraphQL server, DataLoader solves N+1 at the database layer. In a federated system, there is a second layer where N+1 can occur: **between the Gateway and subgraphs**. This is the distributed N+1 — and it is meaner than the original.[^9]

### 6.1 — How Distributed N+1 Occurs

A naive Gateway, after fetching 100 orders from the Orders subgraph, might call the Users subgraph **100 times** — once per order. Each "query" is now a **network hop across a service boundary**, not just a DB query.[^9]

```svgbob
  Gateway fetches 100 orders from Orders Subgraph (1 network call)
       |
       v
  For each order (naive execution):
    order[^0].user -> HTTP POST to Users Subgraph
    order.user -> HTTP POST to Users Subgraph
    order[^2].user -> HTTP POST to Users Subgraph
    ...
    order[^99].user -> HTTP POST to Users Subgraph

  Total network calls: 1 + 100 = 101
  At 20ms per call: ~2 seconds of pure network latency
  (vs. 20ms if batched into 2 calls)
```

*Figure 6: Distributed N+1. The Gateway makes one call per entity to the Users subgraph instead of batching. At 20ms/call across 100 entities, this generates 2 seconds of pure network overhead.*

***

### 6.2 — Entity Batching: Federation's Built-In Solution

Apollo Federation's query planner solves this through **entity batching** via the `_entities` query. The Gateway collects all user IDs from all 100 orders, then makes **one call** to the Users subgraph with a list of entity representations.[^6]

```python
# What the Gateway sends to Users subgraph (one call for all 100 users):
batched_entities_query = """
    query($representations: [_Any!]!) {
        _entities(representations: $representations) {
            ... on User { id name }
        }
    }
"""

variables = {
    "representations": [
        {"__typename": "User", "id": "u1"},
        {"__typename": "User", "id": "u2"},
        # ... all unique user IDs from all 100 orders
    ]
}
```

For this to work efficiently, our Users subgraph must implement `__resolveReference` with its own DataLoader:[^6]

```python
# users_subgraph/resolvers.py — production-grade
from aiodataloader import DataLoader

class UsersDataLoader(DataLoader):
    async def batch_load_fn(self, user_ids: list[str]):
        users = await db.query(
            "SELECT id, name, email FROM users WHERE id = ANY($1)",
            list(user_ids)
        )
        user_map = {u["id"]: u for u in users}
        return [user_map.get(uid) for uid in user_ids]

async def resolve_user_reference(representation, info):
    # DataLoader batches all _entities calls in this request tick
    return await info.context["users_loader"].load(representation["id"])

resolvers = {
    "User": {"__resolveReference": resolve_user_reference}
}
```

The result: 101 network calls collapse to 2, and 100 database queries collapse to 1. Two layers of optimization working in concert.

***

### 6.3 — The Full Optimized Request Lifecycle

```svgbob
  Client: "Give me 100 orders with user names"
        |
        v
  +-------------------+
  |  GATEWAY          |
  |  (Query Planner)  |
  +-------------------+
        |
        | Call 1: GET /graphql (Orders Subgraph)
        v
  +-------------------+         +------------+
  |  Orders Subgraph  | ------> | Orders DB  |
  |                   | <------ | (1 query)  |
  +-------------------+         +------------+
        |
  Returns 100 orders with userIds
  Gateway collects all unique userIds
        |
        | Call 2: POST /graphql (Users Subgraph)
        | _entities(representations: [100 user refs])
        v
  +-------------------+         +------------+
  |  Users Subgraph   | ------> | Users DB   |
  |  (DataLoader)     | <------ | (1 query)  |
  +-------------------+         +------------+

  TOTAL: 2 network hops, 2 DB queries
  (vs. 101 network hops, 100 DB queries without optimization)
```

*Figure 7: The fully optimized request path. The Gateway batches entity lookups into one `_entities` call. The subgraph's DataLoader batches all DB lookups into one SQL `IN (...)` query.*

***

## Part 7 — Advanced Patterns

### 7.1 — @provides: Skip Network Hops Entirely

The `@provides` directive tells the query planner: *"I already have this data — no need to call the other subgraph."*

```python
# orders_subgraph/schema.py
orders_type_defs = """
    type Order @key(fields: "id") {
        id: ID!
        total: Float!
        # Orders table already has user name denormalized
        user: User @provides(fields: "name")
    }

    type User @key(fields: "id") @extends {
        id: ID! @external
        name: String! @external
    }
"""
```

With `@provides`, querying `user.name` through an Order eliminates the Users subgraph network hop entirely. Use this for high-traffic paths where latency matters most.

***

### 7.2 — @requires: Computed Fields Across Subgraphs

`@requires` tells the planner: *"To compute my field, I need a field from another subgraph first."* The planner automatically fetches the dependency before calling my resolver.

```python
# shipping_subgraph/schema.py
shipping_type_defs = """
    type Product @key(fields: "id") @extends {
        id: ID! @external
        weight: Float! @external       # owned by Products subgraph

        # Gateway will fetch weight from Products first, then call this resolver
        shippingCost: Float! @requires(fields: "weight")
    }
"""

async def resolve_shipping_cost(product, info):
    # product.weight is pre-populated by the Gateway before this fires
    return calculate_shipping(product["weight"])
```


***

### 7.3 — Persisted Queries: Caching the Query Plan

For high-traffic systems, the Gateway's query planning step itself can become overhead. **Persisted queries** allow clients to send a hash of the query; the Gateway caches the parsed and planned query, skipping the planning step on repeat requests entirely.[^10]

```python
# client-side APQ (Automatic Persisted Queries)
import hashlib

def create_query_hash(query: str) -> str:
    return hashlib.sha256(query.encode()).hexdigest()

# First request: send hash + full query
first_request = {
    "extensions": {
        "persistedQuery": {"version": 1, "sha256Hash": create_query_hash(query)}
    },
    "query": query  # Included on first request only
}

# Subsequent requests: send ONLY the hash
cached_request = {
    "extensions": {
        "persistedQuery": {"version": 1, "sha256Hash": create_query_hash(query)}
    }
    # No "query" field — Gateway uses cached plan
}
```

For a query plan running thousands of times per second, this shaves **5–15ms** off every request and significantly reduces Gateway CPU usage.

***

## Part 8 — Interview Mental Models

Hiring managers at top-tier companies rarely ask you to write DataLoader from scratch. They want to see you reason about **tradeoffs, failure modes, and system behavior under load**.

**"What happens when a subgraph goes down in federation?"** — The Gateway can be configured with `@defer` to return partial results. Design your supergraph so a Reviews subgraph failure does not block a user from seeing their Order total. Resilient federation means treating non-critical subgraphs as optional.

**"How do you handle schema evolution without breaking clients?"** — Use `@override` for safe field migration. The Inventory team adds the field with `@override(from: "products")`, both versions coexist temporarily during traffic migration, then Products removes the field. Zero downtime. Zero client changes.[^2]

**"When would you NOT use federation?"** — Small teams (fewer than 3–4 services) should not adopt federation. The operational overhead — schema registry, separate deployments, composition tooling, distributed tracing — is not justified until your team size makes the monolith's coordination costs exceed federation's infrastructure costs. A 5-person startup should not be running a federated graph.[^11]

**"How is DataLoader different from a cache?"** — DataLoader's per-request cache is scoped and ephemeral — it exists only for one GraphQL request's lifetime. It prevents redundant fetches *within* one request. It is not a replacement for Redis or a CDN cache, which persist across requests. DataLoader is a **deduplication and batching tool**, not a persistence strategy.[^8]

***

## Part 9 — The Complete Mental Model: City Hall, Revisited

We promised at the start that every concept would map to our city analogy. Let us close the loop:


| Federation Concept | City Hall Analogy |
| :-- | :-- |
| Subgraph | An independent city department building |
| Supergraph | The city's unified services directory |
| Gateway / Router | The central switchboard / 311 operator |
| `@key` | A citizen's government ID number |
| `__resolveReference` | The department clerk who retrieves a full file given just an ID |
| `_entities` query | "Here are 100 citizen IDs — retrieve all their records at once" |
| DataLoader | The smart clerk who waits for all requests, then pulls all files in one trip |
| N+1 problem | A clerk who makes 100 separate trips to the archives for 100 citizens |
| `@provides` | Department A already has a photocopy of Department B's field; no trip needed |
| `@requires` | Department A needs Department B's data before it can compute its own answer |
| Persisted Queries | Pre-approved, pre-routed requests the switchboard already knows how to handle |

```svgbob
  COMPLETE REFERENCE DIAGRAM
  ==========================

  FEDERATION ARCHITECTURE
  -----------------------
  Client --> Gateway --> [Query Planner]
                              |
              +---------------+---------------+
              |               |               |
         Subgraph A      Subgraph B      Subgraph C
          (@key types)   (@extends)      (@key types)
              |               |               |
           DB / API        DB / API        DB / API


  N+1 MITIGATION STACK
  --------------------
  GraphQL Resolver
       |
       v
  DataLoader.load(key)   <-- Per-request batching + dedup cache
       |
       v  (end of event loop tick)
  Batch Function fires
       |
       v
  DB: SELECT * FROM X WHERE id IN (k1, k2, k3 ...)
       |
       v
  Results mapped back to original load() promises


  FEDERATED N+1 SOLUTION
  ----------------------
  Gateway collects all entity keys from parent results
       |
       v
  Single _entities() call to subgraph with ALL keys
       |
       v
  Subgraph uses DataLoader internally
       |
       v
  Single DB query for all entities
```

*Figure 8: The complete reference diagram. The Gateway batches at the network layer; DataLoader batches at the database layer. Two complementary patterns at two different architectural levels.*


