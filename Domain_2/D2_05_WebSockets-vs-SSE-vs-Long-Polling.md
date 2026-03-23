
# WebSockets vs. SSE vs. Long Polling

## A Complete Guide to Real-Time Web Communication


***

## Part 1: The Bridge — Rethinking "Communication"

Before we write a single line of code, let's anchor this topic in something deeply familiar: **how humans wait for information**.

Picture this: You've applied for your dream job at a top tech company. The results are expected sometime today. How do you find out if you got the offer?

**Scenario A: The Impatient Caller**
You call the HR office every 30 minutes. "Any updates?" Each call is a complete interaction — you dial, they answer, you ask, they respond, you hang up. Most calls return "Nothing yet." You're burning your own energy and annoying the receptionist.

**Scenario B: The Patient Waiter**
You call HR, but instead of hanging up when there's no news, you stay on the line. HR says "Hold please." You wait in silence. When the hiring manager makes a decision, HR picks the line back up and tells you immediately. Then you hang up — and immediately call back to wait again.

**Scenario C: The Newsletter Subscriber**
You sign up for HR's automated notification system. They push updates to you as they happen. The communication is one-directional: HR → You. You can't "reply" through this channel, but you get instant updates without any effort.

**Scenario D: The Two-Way Radio**
Both you and HR have walkie-talkies. Either party can speak at any moment. The channel stays open. Communication flows both ways, simultaneously.

These four scenarios map almost perfectly to the real-time strategies we're going to deeply understand in this article:


| Human Scenario | Web Technology |
| :-- | :-- |
| Repeated calls | Traditional Short Polling |
| Stay on the line, wait | Long Polling |
| Newsletter subscription | Server-Sent Events (SSE) |
| Two-way radio | WebSockets |

By the end, we'll have the mental model and vocabulary to answer even the trickiest system design interview questions about real-time communication. Let's build it layer by layer.

***

## Part 2: The Core Problem — HTTP Was Built for a Snapshot World

To understand why long polling, SSE, and WebSockets exist, we need to understand the single limitation they were all invented to solve.

HTTP — the protocol that powers the web — was designed around a fundamentally **stateless, request-response model**. The client asks a question; the server answers; the connection closes. That's the entire contract.

```svgbob
  Client                        Server
    |                              |
    |--- HTTP GET /data ---------->|
    |                              |
    |<-- HTTP 200 OK (response) ---|
    |                              |
  [Connection Closed]
```

*Figure 1: The classic HTTP request-response cycle. The client initiates every single interaction. The server can only respond — it can never reach out first. Once the response is delivered, the connection is torn down. This model is stateless by design.*

This model is brilliant for loading web pages. You ask for a page, you get it, done. But the modern web is full of applications that require **the server to push data to the client as events happen**:

- A stock ticker showing live price updates
- A chat application where messages arrive from other users
- A live sports scoreboard
- A collaborative document editor (like Google Docs)
- Real-time build pipeline progress (like GitHub Actions)
- Live auction bidding systems

In all of these cases, **the client doesn't know when new data will arrive**. It can't initiate a request at the right moment because it doesn't know when "the right moment" is. The server knows — but HTTP gives it no way to reach out proactively.

This is the core pain point. Three different solutions emerged to solve it, each making a different trade-off. Let's meet the first one.

***

## Part 3: Long Polling — The Clever Hack

### Show the Broken State First

Before we had elegant solutions, developers did something called **short polling** — the "impatient caller" from our analogy. The client would send an HTTP request every few seconds to ask the server for updates.

```svgbob
  Client                        Server
    |                              |
    |--- GET /updates ------------>|   t=0s
    |<-- 200 OK (no data) ---------|
    |                              |
    |--- GET /updates ------------>|   t=3s
    |<-- 200 OK (no data) ---------|
    |                              |
    |--- GET /updates ------------>|   t=6s
    |<-- 200 OK (new message!) ----|
    |                              |
```

*Figure 2: Short polling. The client hammers the server at fixed intervals. Notice that the first two requests return empty — this is wasted work. In a real system with thousands of clients, the overwhelming majority of requests return empty responses, creating enormous unnecessary server load.*

Short polling has three obvious, painful problems:

1. **Latency**: Updates can only be delivered at the next polling interval — if you poll every 5 seconds, your maximum update delay is 5 seconds
2. **Wasted requests**: The vast majority of requests return empty responses, burning bandwidth and CPU
3. **Server load**: Hundreds of clients polling every few seconds creates thundering herd effects

### The Fix: Hold the Line

Long polling is elegant in its simplicity: instead of hanging up when there's nothing new, the server **holds the connection open** until it has something to say.

```svgbob
  Client                        Server
    |                              |
    |--- GET /updates ------------>|   t=0s
    |                              |   [Server holds connection open...]
    |                              |   [Server holds connection open...]
    |                              |   [New event occurs at t=8s]
    |<-- 200 OK (new message!) ----|   t=8s
    |                              |
    |--- GET /updates ------------>|   t=8s  (immediately reconnects)
    |                              |   [Server holds connection open...]
```

*Figure 3: Long polling. The client sends one request, and the server keeps the connection open until an event occurs or a timeout is reached. The client immediately sends a new request after receiving a response, creating a near-continuous listening state. Critically, there are virtually no "no data" responses — the server only responds when it has something to say.*

The key insight: long polling **simulates** a persistent connection using standard HTTP. It works everywhere — through proxies, firewalls, corporate networks — because to the network, it looks like an ordinary (albeit slow) HTTP request.

### How It Works — Step by Step

1. Client sends a request to `GET /events`
2. If the server has new data immediately, it responds with `200 OK` right away
3. If there's no data, the server **holds the connection open** — the request just waits
4. A safety timeout (usually 30–60 seconds) prevents connections from hanging forever. If nothing happens, the server responds with `204 No Content`
5. Upon receiving any response, the client **immediately sends a new request** to re-establish the listening state
6. This creates a virtuous loop that keeps the client updated with minimal latency

### Python Implementation

Let's build a simple long-polling server using `aiohttp`:

```python
import asyncio
import json
from aiohttp import web
from collections import deque

# Shared event queue — in production, replace with Redis pub/sub
event_queue = deque()
clients_waiting = []  # List of asyncio.Future objects

async def add_event(request):
    """Endpoint for pushing new events (simulates backend triggers)."""
    data = await request.json()
    event = {
        "message": data.get("message"),
        "timestamp": asyncio.get_event_loop().time()
    }
    event_queue.append(event)

    # Wake up all waiting client futures
    for future in clients_waiting[:]:
        if not future.done():
            future.set_result(event)
    clients_waiting.clear()

    return web.json_response({"status": "event queued"})

async def poll(request):
    """
    Long-polling endpoint.
    Holds the connection open until an event is available or timeout fires.
    """
    # If there's already a queued event, return it immediately
    if event_queue:
        return web.json_response(event_queue.popleft())

    # No event yet — create a future and suspend this coroutine
    loop = asyncio.get_event_loop()
    future = loop.create_future()
    clients_waiting.append(future)

    try:
        # Wait up to 30 seconds for an event
        event = await asyncio.wait_for(asyncio.shield(future), timeout=30.0)
        return web.json_response(event)
    except asyncio.TimeoutError:
        # Timeout reached — return empty so client immediately reconnects
        if future in clients_waiting:
            clients_waiting.remove(future)
        return web.Response(status=204)  # No Content

app = web.Application()
app.router.add_get('/poll', poll)
app.router.add_post('/events', add_event)

if __name__ == '__main__':
    web.run_app(app, port=8080)
```

And here's a Python client demonstrating the "reconnect immediately" pattern:

```python
import requests
import time

def long_poll_client(server_url: str):
    """Client that continuously long-polls for events."""
    print("Starting long-poll client...")

    while True:
        try:
            # This request BLOCKS until the server responds (up to 35s)
            response = requests.get(f"{server_url}/poll", timeout=35)

            if response.status_code == 200:
                event = response.json()
                print(f"[EVENT RECEIVED] {event}")
            elif response.status_code == 204:
                # Server timeout — no events. Reconnect immediately.
                print("[TIMEOUT] No events. Reconnecting...")

        except requests.exceptions.Timeout:
            print("[CONNECTION TIMEOUT] Reconnecting...")
        except requests.exceptions.ConnectionError as e:
            print(f"[CONNECTION ERROR] {e}. Retrying in 5s...")
            time.sleep(5)

if __name__ == "__main__":
    long_poll_client("http://localhost:8080")
```


### Long Polling's Trade-offs

| Strength | Weakness |
| :-- | :-- |
| Works through all firewalls and proxies | Each cycle incurs full HTTP header overhead |
| No special server infrastructure needed | High connection churn (constant connect/disconnect) |
| Compatible with any HTTP framework | Message ordering requires careful client bookkeeping |
| Maximum network compatibility | "Thundering herd" risk when many events fire simultaneously |

Long polling was the **dominant real-time solution from roughly 2005–2012**. Libraries like Comet popularized the pattern, and early chat applications like Meebo used it extensively. It's less common today for new systems, but still appears in legacy codebases and in environments with extremely restrictive network policies.

***

## Part 4: Server-Sent Events — The Elegant Stream

### The Pain Point Long Polling Left Behind

Long polling has a subtle inefficiency we haven't fully addressed: it's still fundamentally a series of **individual HTTP requests**. Each request carries HTTP headers — typically 400–800 bytes — all resent with every reconnection. If the server generates events at high frequency, this overhead becomes significant.

More importantly, if events arrive faster than the client processes them, you get a queuing problem. And tracking "which events has this client already seen?" requires custom bookkeeping that every developer has to reinvent.

We need a smarter model.

### The Fix: A Dedicated Event Stream

In 2006, Opera implemented a browser API that was eventually standardized in HTML5: `EventSource`. The idea is simple but powerful — **let the server open one persistent HTTP connection and stream events down it continuously**, using a lightweight, human-readable text format.

This is Server-Sent Events (SSE).

```svgbob
  Client                          Server
    |                                |
    |--- GET /events --------------->|
    |    Accept: text/event-stream   |
    |                                |
    |<-- HTTP 200 OK ----------------|
    |    Content-Type: text/event-stream
    |                                |
    |<-- data: {"price": 102.4} -----|   t=2s
    |                                |
    |<-- data: {"price": 103.1} -----|   t=5s
    |                                |
    |<-- data: {"price": 101.8} -----|   t=7s
    |                                |
    |     [connection stays open indefinitely]
```

*Figure 4: The SSE communication model. A single HTTP connection is established and kept alive. The server pushes data as discrete "events" over this persistent stream. Notice that unlike long polling, the client never sends another request after the initial one — data just flows down continuously. All the connection churn of long polling is eliminated.*

### The SSE Wire Format

SSE uses a beautifully simple text protocol. Events are plain UTF-8 text, separated by double newlines:

```
data: Hello World\n\n

data: {"user": "alice", "message": "Hi there!"}\n\n

event: user_joined\n
data: {"username": "bob"}\n\n

id: 42\n
event: stock_update\n
data: {"ticker": "NVDA", "price": 942.50}\n
retry: 3000\n\n
```

Each field has a specific purpose:

- **`data:`** — The actual payload. Repeat this prefix for multi-line payloads
- **`event:`** — A named event type. The client can listen for specific event names
- **`id:`** — A unique event ID, critical for reconnection and "last event seen" tracking
- **`retry:`** — Tells the client how many milliseconds to wait before reconnecting if the

