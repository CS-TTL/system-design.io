
# REST API Design Patterns: Versioning & Pagination


---

## Part I — Versioning: Managing Change Without Breaking the World

---

### The Library Analogy: Why Versioning Exists

Imagine you are the head librarian of the world's most popular library. Millions
of people visit every day. One morning, you decide to reorganize — shelf numbers
change, categories are renamed, and some books are moved to a different floor
entirely. You announce the change and reopen. Chaos. Students who memorized
"Science is on shelf 4B" now find cookbooks. Regular visitors who relied on the
old map are completely lost.

Now imagine instead that you kept the *old* wing exactly as it was — you simply
built a *new* wing with the improved layout. You placed signs at the entrance:

```

Old layout → Wing 1 (Legacy)
New layout → Wing 2 (Current)

```

Regulars keep working. New visitors get the better experience. Nobody is
ambushed. That is REST API versioning in a nutshell.

APIs evolve. New features are added, old fields are removed, data shapes change.
The critical challenge is: **how do we evolve without destroying clients who
depend on the current behavior?** Versioning is our answer. It is not just
a technical detail — it is a *social contract* between the API provider and
its consumers.

---

### The Atomic Unit: What Actually Changes in an API?

Before diving into strategies, we need to understand what "a breaking change"
actually is. Changes to an API fall into two categories:

**Non-breaking (backward-compatible) changes:**
- Adding a new optional field to a response
- Adding a new endpoint
- Adding a new optional query parameter
- Expanding an enum with a new value (with caution)

**Breaking changes:**
- Removing or renaming a field
- Changing a field's data type (e.g., `id` from `int` to `string`)
- Changing the meaning of an existing field
- Removing an endpoint
- Changing authentication requirements

The moment you introduce a *breaking change*, you need versioning. Without it,
every client that calls your API risks crashing silently or loudly.

---

### The Pain Point: "I Just Renamed One Field…"

Here is the broken state. Imagine a simple user endpoint:

```python
# Version 1 response shape
{
  "user_name": "alice",
  "mail": "alice@example.com"
}
```

One day, a developer on the API team rightfully decides to align with industry
naming conventions and renames the fields:

```python
# Version 2 response shape
{
  "username": "alice",
  "email": "alice@example.com"
}
```

This seems trivial. But your mobile app, third-party integrations, and internal
dashboards were all parsing `user_name` and `mail`. The moment v2 goes live, all
of them silently receive `None` for those fields, causing failed logins, empty
displays, or null pointer exceptions — all without a single error code to
diagnose. **This is the broken state.** Now let's sell the fix.

---

## The Four Versioning Strategies

We have four primary strategies in wide industry use. We'll walk through each,
examining their mechanics, tradeoffs, and when each is the right tool for the
job.

---

### Strategy 1 — URI Path Versioning

This is the most common strategy you will encounter in the wild, and almost
certainly the one you'll be asked about first in an interview.

```
https://api.myservice.com/v1/users
https://api.myservice.com/v2/users
```

The version number is embedded directly in the URL path. The client explicitly
chooses which version they are calling. Below is the ASCII diagram of how
requests are routed:

```svgbob
 Client A                     API Gateway                   Services
(using v1)                       |                              |
    |                            |                              |
    |--- GET /v1/users --------->|                              |
    |                            |--- route to v1 handler ----->|
    |                            |                              |
    |<-- { user_name, mail } ----|<-- v1 response --------------|
    |                            |                              |
 Client B                        |                              |
(using v2)                       |                              |
    |                            |                              |
    |--- GET /v2/users --------->|                              |
    |                            |--- route to v2 handler ----->|
    |                            |                              |
    |<-- { username, email } ----|<-- v2 response --------------|
```

*In Figure 1, notice how the gateway acts as the traffic controller. Clients
do not know about each other — they simply speak the version they understand.
The routing logic lives in one centralized place.*

**Implementation in Python (FastAPI):**

```python
from fastapi import FastAPI

app = FastAPI()

# v1 router
@app.get("/v1/users/{user_id}")
def get_user_v1(user_id: int):
    return {
        "user_name": "alice",   # legacy field naming
        "mail": "alice@example.com"
    }

# v2 router
@app.get("/v2/users/{user_id}")
def get_user_v2(user_id: int):
    return {
        "username": "alice",    # modern field naming
        "email": "alice@example.com"
    }
```

**Tradeoffs:**


| Dimension | Assessment |
| :-- | :-- |
| Discoverability | ✅ Excellent — visible in browser, logs, and docs |
| Cacheability | ✅ Excellent — `/v1/users` and `/v2/users` are distinct cache keys |
| REST purity | ⚠️  Debated — purists argue a "resource" shouldn't change URL by version |
| Routing simplicity | ✅ Simple — most gateways (NGINX, Kong) handle path-based routing trivially |
| Client effort | ✅ Minimal — just change a URL string |

**When to use it:** This is the safe default for public-facing APIs. GitHub,
Stripe, Twilio, and PayPal all use URI versioning. If you are designing an
API from scratch and need something unambiguous and easy to document, start
here.

---

### Strategy 2 — Query Parameter Versioning

An alternative that keeps the base URL clean, routing the version as a
parameter:

```
https://api.myservice.com/users?version=1
https://api.myservice.com/users?version=2
```

**Implementation in Python (FastAPI):**

```python
from fastapi import FastAPI, Query, HTTPException

app = FastAPI()

@app.get("/users/{user_id}")
def get_user(user_id: int, version: int = Query(default=1)):
    if version == 1:
        return {"user_name": "alice", "mail": "alice@example.com"}
    elif version == 2:
        return {"username": "alice", "email": "alice@example.com"}
    else:
        raise HTTPException(
            status_code=400,
            detail=f"Unsupported API version: {version}"
        )
```

This approach is common in internal APIs and quick MVPs. However, it has a
meaningful weakness for production systems: **caching behavior**. Many CDN and
proxy layers treat query parameters inconsistently. `GET /users/1` and
`GET /users/1?version=1` may be treated as the same resource by some caches,
serving the wrong version to a client. For high-scale systems, this is a
subtle but real hazard.

---

### Strategy 3 — Header Versioning

This approach keeps versioning entirely out of the URL, using a custom HTTP
header instead:

```
GET /users/1
Accept-Version: v2
```

Or using the `Accept` header with content negotiation (the "media type"
approach, sometimes called **content negotiation versioning**):

```
GET /users/1
Accept: application/vnd.myservice.v2+json
```

**Implementation in Python (FastAPI):**

```python
from fastapi import FastAPI, Header, HTTPException
from typing import Optional

app = FastAPI()

@app.get("/users/{user_id}")
def get_user(
    user_id: int,
    accept_version: Optional[str] = Header(default="v1")
):
    if accept_version == "v1":
        return {"user_name": "alice", "mail": "alice@example.com"}
    elif accept_version == "v2":
        return {"username": "alice", "email": "alice@example.com"}
    else:
        raise HTTPException(
            status_code=400,
            detail=f"Unsupported version header: {accept_version}"
        )
```

**The architectural landscape of header versioning:**

```svgbob
 HTTP Request
+----------------------------------------------+
|  GET /users/42          HTTP/1.1              |
|  Host: api.myservice.com                      |
|  Accept-Version: v2         <-- version here  |
|  Authorization: Bearer abc123                 |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
|         API Gateway / Middleware              |
|   reads Accept-Version header                 |
|   routes to correct handler                   |
+----------------------------------------------+
                    |
         +----------+----------+
         |                     |
         v                     v
  [v1 Handler]           [v2 Handler]
  legacy schema          modern schema
```

*In Figure 2, the URL stays completely stable — `/users/42` is always `/users/42`.
The header is the only signal the gateway needs to dispatch the request correctly.*

**Tradeoffs:**


| Dimension | Assessment |
| :-- | :-- |
| URL cleanliness | ✅ Best — URLs are pure resource identifiers |
| REST fidelity | ✅ Best — aligns with REST's content negotiation model |
| Discoverability | ❌ Poor — you can't test it by typing a URL in a browser |
| Caching | ⚠️  Requires `Vary: Accept-Version` header to work correctly |
| Client complexity | ⚠️  Slightly higher — all clients must set the header |

**Historical note:** The header/content-negotiation approach was popular in
academic discussions in the early 2010s when REST purists were fighting the
URI-versioning camp. In practice, most companies chose URI versioning for its
simplicity. Today, header versioning is often seen in mature, enterprise-grade
APIs with sophisticated clients.

---

### Strategy 4 — Semantic Versioning in Practice

Regardless of which strategy you choose for the URL surface, you should apply
semantic versioning principles to the *version number itself*:

```
v1.0.0  →  Major.Minor.Patch
```

- **Major (v1 → v2):** Breaking changes. Clients must migrate.
- **Minor (v1.1 → v1.2):** New features, fully backward compatible.
- **Patch (v1.1.0 → v1.1.1):** Bug fixes. No API shape change.

In practice, most public APIs only expose the major version in the URL
(`/v1/`, `/v2/`) and communicate minor/patch changes through changelogs and
headers like `X-API-Version: 1.3.2`.

---

### Deprecation: The Graceful Goodbye

Versioning without a deprecation strategy is like having an emergency exit but
no evacuation plan. The full versioning lifecycle looks like this:

```svgbob
  v1 Released
      |
      |<---- v1 Active Period (stable, fully supported) ---->|
      |                                                       |
      |              v2 Released                             |
      |                   |                                  |
      |<--- Sunset Period: v1 marked deprecated, v2 active ->|
      |                   |                                  |
      |    (Clients notified via Deprecation headers,        |
      |     changelogs, email)                               |
      |                   |                                  |
      |                   v                                  |
      |              v1 Sunsetted (returns 410 Gone)         |
```

*In Figure 3, the key takeaway is the* ***sunset period***. *This is the window
during which both versions co-exist. Responsible API providers give clients
at minimum 6–12 months before killing an older version. Stripe is legendary
for maintaining old API versions for years.*

**Communicating deprecation programmatically:**

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse

app = FastAPI()

@app.get("/v1/users/{user_id}")
def get_user_v1(user_id: int):
    response_data = {
        "user_name": "alice",
        "mail": "alice@example.com"
    }

    response = JSONResponse(content=response_data)
    # RFC 8594 — standard deprecation signaling
    response.headers["Deprecation"] = "true"
    response.headers["Sunset"] = "Sat, 01 Jan 2027 00:00:00 GMT"
    response.headers["Link"] = (
        '<https://api.myservice.com/v2/users>; rel="successor-version"'
    )
    return response
```

When clients see the `Deprecation` and `Sunset` headers, they know they have
a deadline to migrate. This is far more graceful than a sudden 404.

---

### Interview Cheat-Sheet: Versioning Decision Tree

```
Q: Should I version my API?
        |
        v
   Is this a public API, or do I have external
   clients I don't control?
        |
   Yes--|-----> USE VERSIONING (URI path is safest default)
        |
   No --+-----> Coordinate directly with all clients;
                consider feature flags instead
                        |
                        v
                Is discoverability & caching critical?
                        |
                   Yes--|-----> URI Path Versioning
                        |
                   No --+-----> Header Versioning (enterprise/internal APIs)
```


---

## Part II — Pagination: Taming the Infinite Feed


---

### The Library Returns: Fetching "All the Books"

Let's go back to our library. A new patron walks in and says, "Give me every
book you have." The librarian stares. There are 4 million books. Should she
hand them all over at once? The patron would collapse under the weight. The
loading dock would be paralyzed. No other patron could get service.

Instead, the smart librarian says: "Here are the first 20 books from shelf A.
Come back when you're ready, and I'll give you the next 20."

That is pagination. It is about **delivering data in digestible, predictable
chunks** rather than dumping entire datasets in one response. Without it, a
single `GET /products` request against an e-commerce database of 10 million
SKUs would either time out, crash the server, or eat the client's memory alive.

---

### The Atomic Unit: A Simple Limit/Offset

The first and most intuitive pagination strategy uses two parameters: how many
items to return, and where to start counting.

```
GET /products?limit=10&offset=0     → items 1–10
GET /products?limit=10&offset=10    → items 11–20
GET /products?limit=10&offset=20    → items 21–30
```

Conceptually this maps to SQL directly:

```python
import sqlite3

def get_products_offset(limit: int, offset: int):
    conn = sqlite3.connect("shop.db")
    cursor = conn.cursor()

    cursor.execute(
        "SELECT id, name, price FROM products ORDER BY id LIMIT ? OFFSET ?",
        (limit, offset)
    )

    rows = cursor.fetchall()
    conn.close()

    return [
        {"id": r, "name": r, "price": r}
        for r in rows
    ]
```

A standard paginated response includes metadata so clients know their position:

```python
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/products")
def list_products(
    limit: int = Query(default=20, ge=1, le=100),
    offset: int = Query(default=0, ge=0)
):
    total = 10_000      # from COUNT(*) query
    items = get_products_offset(limit, offset)

    return {
        "data": items,
        "pagination": {
            "total": total,
            "limit": limit,
            "offset": offset,
            "has_next": (offset + limit) < total,
            "has_prev": offset > 0
        }
    }
```

This is clean, intuitive, and maps perfectly to UI patterns like numbered page
controls ("Page 3 of 47").

---

### The Hidden Problem: The "Drifting Floor" Bug

Offset pagination has a fatal flaw that becomes apparent only under real-world
conditions. We call it the **drifting floor problem**.

Imagine our client is paginating through a live feed of comments:

```
Page 1: Client fetches offset=0, receives comments #1–10
        While reading, 3 new comments are added to the top of the feed.
Page 2: Client fetches offset=10, now receives comments #8–17
        Comments #8, #9, #10 were already seen on Page 1. DUPLICATES!
```

And the reverse is equally dangerous:

```
Page 1: Client fetches offset=0, receives comments #1–10.
        While reading, comment #5 is deleted.
Page 2: Client fetches offset=10. The DB shifts left.
        Comment #11 is now at position #10. Client SKIPS it. SILENT GAP!
```

```svgbob
 Before deletion                     After deletion of item #5
 
 Offset   Item                        Offset   Item
 ------   ----                        ------   ----
   0       #1                            0      #1
   1       #2                            1      #2
   2       #3                            2      #3
   3       #4                            3      #4
   4       #5  <-- deleted              4      #6  (shifted up!)
   5       #6                            5      #7
   ...     ...                          ...     ...
   10      #11 <-- client fetches next  10     #12  (client skips #11 !)
```

*In Figure 4, this is not a bug in the code — it is a fundamental limitation
of offset-based pagination on live data. The offset is a positional index,
and positions shift when data changes.*

This is exactly the "broken state" that motivated a better approach.

---

### Strategy 2 — Cursor-Based Pagination: The Fix

Instead of saying "give me items starting at position 10," cursor-based
pagination says "give me items **after this specific item**." The item is
identified by an opaque token called a **cursor**.

```
GET /comments?limit=10                       → first 10 items
← response includes: next_cursor = "abc123"

GET /comments?limit=10&cursor=abc123         → next 10 items after cursor
← response includes: next_cursor = "def456"
```

The cursor encodes the identity of the last item seen, not its position. When
items are added or deleted, the cursor's anchor point is unaffected.

```svgbob
 First Request                    Second Request
 GET /comments?limit=3            GET /comments?limit=3&cursor=<id_of_3>

 +--------+                       +--------+
 | item 1 |                       | item 4 |
 | item 2 |                       | item 5 |
 | item 3 | <-- cursor anchors    | item 6 |
 +--------+    to this item       +--------+

 Even if new items are added above item 1,
 or item 2 is deleted, item 4 is always
 "the item after item 3" in the ordered set.
```

*In Figure 5, notice that the cursor acts as a* ***bookmark*** *rather than a
page number. The database query is anchored to a real entity, not an
arithmetic position.*

**Implementation in Python (FastAPI + SQLite):**

```python
import base64
import json
import sqlite3
from fastapi import FastAPI, Query, HTTPException
from typing import Optional

app = FastAPI()

def encode_cursor(last_id: int) -> str:
    payload = json.dumps({"id": last_id})
    return base64.urlsafe_b64encode(payload.encode()).decode()

def decode_cursor(cursor: str) -> int:
    try:
        payload = base64.urlsafe_b64decode(cursor.encode()).decode()
        return json.loads(payload)["id"]
    except Exception:
        raise HTTPException(status_code=400, detail="Invalid cursor token")

def fetch_after_cursor(after_id: Optional[int], limit: int):
    conn = sqlite3.connect("shop.db")
    cursor = conn.cursor()

    if after_id is None:
        cursor.execute(
            "SELECT id, name, created_at FROM comments ORDER BY id ASC LIMIT ?",
            (limit + 1,)  # fetch one extra to detect if there's a next page
        )
    else:
        cursor.execute(
            "SELECT id, name, created_at FROM comments WHERE id > ? ORDER BY id ASC LIMIT ?",
            (after_id, limit + 1)
        )

    rows = cursor.fetchall()
    conn.close()
    return rows

@app.get("/comments")
def list_comments(
    limit: int = Query(default=20, ge=1, le=100),
    cursor: Optional[str] = Query(default=None)
):
    after_id = decode_cursor(cursor) if cursor else None
    rows = fetch_after_cursor(after_id, limit)

    has_next = len(rows) > limit
    items = rows[:limit]  # trim the extra item used for has_next detection

    next_cursor = encode_cursor(items[-1]) if has_next else None

    return {
        "data": [
            {"id": r, "name": r, "created_at": r}
            for r in items
        ],
        "pagination": {
            "has_next": has_next,
            "next_cursor": next_cursor
        }
    }
```

Notice two important implementation details in the code above:

1. **The `+1` trick:** We always fetch `limit + 1` rows. If we get back more
than `limit` results, we know there is a next page. We then trim the extra
item before returning.
2. **Opaque encoding:** We Base64-encode the cursor so clients treat it as
a black box. This lets us change what the cursor encodes internally (e.g.,
adding a timestamp or composite key) without breaking the API contract.

---

### Strategy 3 — Keyset Pagination: Cursor at Scale

Cursor-based pagination is conceptually about tracking "where I am" via an
ID. **Keyset pagination** generalizes this to support *sorting by arbitrary
fields* — a critical feature for real-world APIs where users sort by date,
name, price, or relevance.

Imagine our `/products` endpoint supports `?sort=price`. The "last seen" state
is no longer just an ID — it is the combination `(last_price, last_id)` (the
ID is needed as a tiebreaker for equal prices):

```python
from fastapi import FastAPI, Query
from typing import Optional
import sqlite3

app = FastAPI()

@app.get("/products")
def list_products_keyset(
    limit: int = Query(default=20, ge=1, le=100),
    last_price: Optional[float] = Query(default=None),
    last_id: Optional[int] = Query(default=None)
):
    conn = sqlite3.connect("shop.db")
    cur = conn.cursor()

    if last_price is None or last_id is None:
        # First page — no keyset provided
        cur.execute(
            """
            SELECT id, name, price
            FROM products
            ORDER BY price ASC, id ASC
            LIMIT ?
            """,
            (limit + 1,)
        )
    else:
        # Subsequent pages — use keyset to anchor position
        cur.execute(
            """
            SELECT id, name, price
            FROM products
            WHERE (price > ?) OR (price = ? AND id > ?)
            ORDER BY price ASC, id ASC
            LIMIT ?
            """,
            (last_price, last_price, last_id, limit + 1)
        )

    rows = cur.fetchall()
    conn.close()

    has_next = len(rows) > limit
    items = rows[:limit]

    return {
        "data": [{"id": r, "name": r, "price": r} for r in items],
        "pagination": {
            "has_next": has_next,
            "next_last_price": items[-1] if has_next else None,
            "next_last_id": items[-1] if has_next else None,
        }
    }
```

The SQL clause `WHERE (price > ?) OR (price = ? AND id > ?)` is the heart of
keyset pagination. This is directly index-scannable — the database hits a
B-tree index on `(price, id)` and reads forward from the anchor point. There
is no `OFFSET N` scanning thousands of rows to discard.

**Performance visualization — why this matters:**

```svgbob
Offset Pagination (fetching page 5000 of 20 items):

DB Index
+--------------------------------------------------+
| Scan and discard 100,000 rows to reach offset    |
| 100000... then read 20 rows.                     |
+--------------------------------------------------+
   O(N) scan — gets slower with every page

Keyset Pagination (fetching page 5000 of 20 items):

DB Index (B-tree on price, id)
+--------------------------------------------------+
| Seek directly to (price=42.99, id=10050)         |
| Read forward 20 rows. DONE.                      |
+--------------------------------------------------+
   O(log N) seek — stays fast regardless of depth
```

*In Figure 6, the performance gap between offset and keyset pagination grows
exponentially with dataset size. For a table with 10 million rows, fetching
the 500,000th page via offset means the database discards 10 million rows on
every single request. Keyset finds the anchor in microseconds.*

---

### Strategy 4 — Page-Number Pagination

We would be incomplete without mentioning the simplest variant: page-number
pagination. Rather than exposing raw `offset`, we expose a human-friendly
`page` parameter:

```
GET /products?page=3&per_page=20
```

This is simply syntactic sugar over offset pagination where
`offset = (page - 1) * per_page`. The database query is identical, and the
same drifting-floor problem applies. However, it maps directly to UI components
("Page 3 of 47") and is useful for *stable, infrequently-updated datasets*
like a product catalog, a list of countries, or archived data.

```python
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/products")
def list_products_paged(
    page: int = Query(default=1, ge=1),
    per_page: int = Query(default=20, ge=1, le=100)
):
    offset = (page - 1) * per_page
    total = 10_000
    items = get_products_offset(limit=per_page, offset=offset)
    total_pages = (total + per_page - 1) // per_page  # ceiling division

    return {
        "data": items,
        "pagination": {
            "page": page,
            "per_page": per_page,
            "total": total,
            "total_pages": total_pages,
            "has_next": page < total_pages,
            "has_prev": page > 1
        }
    }
```


---

### Pagination Strategy Selection Guide

We now have four strategies. How do we choose? Group them by two dimensions:
data stability and access pattern.

```svgbob
                    Data Access Pattern
                   +--------------------+--------------------+
                   | Sequential only    | Random page access |
+------------------+--------------------+--------------------+
|  Data is STABLE  | Cursor-based       | Offset / Page-     |
|  (rarely changes)|                    | Number Pagination  |
+------------------+--------------------+--------------------+
|  Data is LIVE    | Keyset / Cursor    | No perfect option; |
|  (real-time feed)|                    | cursor + accept    |
|                  |                    | duplication risk   |
+------------------+--------------------+--------------------+
```

*In Figure 7, the top-right quadrant (live data + random page access) has no
perfect solution. Real-time data and arbitrary page jumping are fundamentally
in tension. In practice, we either accept eventual consistency or restrict the
UI to forward-only navigation.*

A more concise lookup:


| Strategy | Best Use Case | Avoid When |
| :-- | :-- | :-- |
| Offset/Limit | Admin dashboards, stable data | Live feeds, huge datasets |
| Page Number | UI with numbered pages | Real-time data |
| Cursor-based | Social feeds, activity streams | Users need random page access |
| Keyset | High-volume sorted queries | Complex multi-field sort orders |


---

### The Response Envelope: Designing for Clients

Great pagination is not just about the query — it is about how you package the
result for the client. A well-designed response envelope is self-describing:
the client should be able to paginate entirely by following the data in the
response, without reading documentation.

**The HATEOAS-inspired approach (Links in response):**

```python
@app.get("/products")
def list_products_hateoas(
    limit: int = Query(default=20),
    cursor: Optional[str] = Query(default=None)
):
    after_id = decode_cursor(cursor) if cursor else None
    rows = fetch_after_cursor(after_id, limit)
    has_next = len(rows) > limit
    items = rows[:limit]

    base_url = "https://api.myservice.com/products"
    next_cursor = encode_cursor(items[-1]) if has_next else None

    links = {
        "self": f"{base_url}?limit={limit}" + (f"&cursor={cursor}" if cursor else ""),
        "next": f"{base_url}?limit={limit}&cursor={next_cursor}" if next_cursor else None
    }

    return {
        "data": [{"id": r, "name": r} for r in items],
        "pagination": {
            "has_next": has_next,
            "next_cursor": next_cursor
        },
        "_links": links  # clients can follow links without building URLs
    }
```

This pattern is used by GitHub's API, Stripe, and Twilio — the `_links`
or `links` object gives clients a full URL to follow for the next page,
abstracting away cursor encoding entirely.

---

### Pagination Error Handling

No article is complete without failure modes. Two critical edge cases:

**1. Invalid cursor:**

```python
# The cursor token may be expired, corrupted, or tampered with
@app.get("/comments")
def list_comments(cursor: Optional[str] = Query(default=None)):
    if cursor:
        try:
            after_id = decode_cursor(cursor)
        except Exception:
            raise HTTPException(
                status_code=400,
                detail={
                    "error": "INVALID_CURSOR",
                    "message": "The pagination cursor is invalid or expired. "
                               "Please restart from the first page.",
                    "docs": "https://api.myservice.com/docs/pagination"
                }
            )
```

**2. Limit abuse (client requests 10,000 items):**

```python
@app.get("/products")
def list_products(
    limit: int = Query(default=20, ge=1, le=100)  # FastAPI enforces max automatically
):
    # ge=1 rejects limit=0
    # le=100 rejects limit=10000
    ...
```

By setting a hard maximum with `le=100`, we prevent a single client from
triggering a massive DB scan. This is a rate-limiting complement that
belongs in the API layer.

---

### Putting It All Together: A Production-Ready Endpoint

Let us write a complete, production-quality paginated endpoint that combines
all the lessons above: cursor-based pagination, a proper response envelope,
opaque cursor tokens, has-next detection, and error handling.

```python
import base64
import json
import sqlite3
from fastapi import FastAPI, Query, HTTPException
from fastapi.responses import JSONResponse
from typing import Optional, List, Dict, Any

app = FastAPI(title="Products API", version="2.0.0")

DATABASE = "shop.db"

def db_connect():
    conn = sqlite3.connect(DATABASE)
    conn.row_factory = sqlite3.Row
    return conn

def encode_cursor(last_id: int, last_name: str) -> str:
    payload = json.dumps({"id": last_id, "name": last_name})
    return base64.urlsafe_b64encode(payload.encode()).decode()

def decode_cursor(token: str) -> Dict[str, Any]:
    try:
        raw = base64.urlsafe_b64decode(token.encode()).decode()
        return json.loads(raw)
    except Exception:
        raise HTTPException(
            status_code=400,
            detail="Invalid pagination cursor. Restart from the first page."
        )

@app.get("/v2/products", tags=["Products"])
def list_products(
    limit: int = Query(default=20, ge=1, le=100, description="Items per page"),
    cursor: Optional[str] = Query(default=None, description="Opaque pagination cursor"),
    sort: str = Query(default="id", regex="^(id|name|price)$")
):
    anchor = decode_cursor(cursor) if cursor else None

    conn = db_connect()
    cur = conn.cursor()

    try:
        if anchor is None:
            cur.execute(
                f"SELECT id, name, price FROM products ORDER BY {sort} ASC, id ASC LIMIT ?",
                (limit + 1,)
            )
        else:
            cur.execute(
                f"""
                SELECT id, name, price FROM products
                WHERE ({sort} > :anchor_val)
                   OR ({sort} = :anchor_val AND id > :anchor_id)
                ORDER BY {sort} ASC, id ASC
                LIMIT :lim
                """,
                {
                    "anchor_val": anchor.get(sort, anchor["id"]),
                    "anchor_id": anchor["id"],
                    "lim": limit + 1
                }
            )
        rows = cur.fetchall()
    finally:
        conn.close()

    has_next = len(rows) > limit
    items = [dict(r) for r in rows[:limit]]

    next_cursor = (
        encode_cursor(items[-1]["id"], items[-1]["name"])
        if has_next else None
    )

    return {
        "data": items,
        "pagination": {
            "limit": limit,
            "has_next": has_next,
            "next_cursor": next_cursor
        },
        "_links": {
            "self": f"/v2/products?limit={limit}" + (f"&cursor={cursor}" if cursor else ""),
            "next": f"/v2/products?limit={limit}&cursor={next_cursor}" if next_cursor else None
        }
    }
```

This is a real-world-grade endpoint. It uses versioning in the URI (`/v2/`),
keyset-based cursor pagination, an opaque token, a `+1` trick for has-next
detection, and a `_links` envelope for HATEOAS clients — all in under 70
lines of Python.

---

## Part III — Interview Synthesis: What They're Really Testing

When an interviewer asks about API versioning or pagination, they are not
asking you to recite definitions. They want to see **systems thinking** — your
ability to reason about tradeoffs and constraints.

### High-Signal Questions \& Model Answers

**Q: "How would you design pagination for a Twitter-like feed?"**

Model answer framing:

- Data is live (new tweets constantly). Offset pagination fails here.
- Access pattern is sequential (infinite scroll). Cursor is ideal.
- Sort is chronological. Use a timestamp + tweet_id composite cursor.
- Consider: cursors expire if a client is idle for days; return a clear error.
- Mention the `+1` trick for has_next detection.

**Q: "When would you NOT version your API?"**

- Very early stage, single consumer you control completely.
- Internal microservices where you can do atomic deployments.
- Use feature flags instead of API versions for gradual rollout.

**Q: "What's the tradeoff between URI and header versioning?"**


| Concern | URI Versioning | Header Versioning |
| :-- | :-- | :-- |
| Caching | Simple | Requires `Vary` header |
| Browser-testable | Yes | No |
| REST purity | Debated | High |
| Client setup | Minimal | Requires header config |
| Industry adoption | Dominant | Enterprise/internal |


---

### The Mental Model Map

Before you leave, here is a one-page mental model connecting everything we
covered:

```svgbob
 API Design Decisions
 +------------------------------------------------------------------+
 |                                                                  |
 |   API CHANGES OVER TIME?                                        |
 |         |                                                        |
 |         v                                                        |
 |    [VERSIONING]                                                  |
 |    URI Path (default) / Header (REST purist) / Query Param      |
 |    Semantic versioning → deprecation headers → sunset date       |
 |                                                                  |
 |   ENDPOINT RETURNS LARGE COLLECTIONS?                           |
 |         |                                                        |
 |         v                                                        |
 |    [PAGINATION]                                                  |
 |         |                                                        |
 |    Is data stable?    Is access random?    Is data huge/live?   |
 |         |                   |                      |            |
 |         v                   v                      v            |
 |   Page-Number          Offset/Limit           Cursor/Keyset     |
 |   (UX-friendly)        (admin UIs)            (feeds/streams)   |
 |                                                                  |
 |   ALWAYS: response envelope with _links, has_next, next_cursor  |
 +------------------------------------------------------------------+
```

*In Figure 8, this map is your decision framework. Every production API will
touch both of these domains. The best engineers don't memorize answers —
they internalize the tradeoffs so they can reason from first principles
under interview pressure.*

---

## Summary of Key Takeaways

- **Versioning** is a social contract. Break it thoughtlessly and you break
your users' trust along with their integrations.
- **URI path versioning** is the industry default for public APIs. Start there
unless you have a compelling reason not to.
- **Deprecation** is part of versioning. Always communicate via headers, give
clients ample time, and return `410 Gone` after sunset.
- **Offset pagination** is intuitive but fragile on live data due to the
drifting floor problem.
- **Cursor-based pagination** solves the drift problem but sacrifices random
page access.
- **Keyset pagination** is cursor-based pagination at database-level performance,
critical for high-volume, sorted datasets.
- **Response envelopes** with `_links`, `has_next`, and `next_cursor` make
paginated APIs self-describing and client-friendly.
- In interviews, lead with tradeoffs, not definitions. The ability to say
"Strategy A is better than B *in this context because...*" signals
senior-level thinking.

---
