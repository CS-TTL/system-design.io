
# DNS Resolution: How the Internet Finds Its Way

---

## The Bridge: The World's Largest Phone Book

Imagine you just moved to a new city and you want to visit a friend.
You know their name — "Sarah Chen" — but you have no idea where she
lives. You'd pull out a phone book, look up her name, and find her
address. The phone book is the bridge between a human-readable name
and a physical location.

Now scale that problem to billions of devices, spread across every
continent, communicating simultaneously. Every website has a
human-readable name like `google.com`, but computers route traffic
using numerical IP addresses like `142.250.80.46`. The Domain Name
System — DNS — is the internet's phone book. It's the infrastructure
that translates names we understand into addresses machines can route.

But unlike a physical phone book, DNS isn't a single directory
sitting on one server. It's a globally distributed, hierarchical,
and aggressively cached system that handles over **1 trillion queries
per day**. Understanding how it works — not just the steps, but the
*reasoning* behind each design decision — is essential knowledge for
any software engineer working at scale.

We'll walk through it layer by layer, starting from the simplest
atomic unit and building up to the full architecture.

---

## Part 1: The Atomic Unit — What a DNS Query Actually Is

Let's start with the simplest possible scenario.

You open a browser and type `www.example.com`. Your computer has no
idea what IP address that name belongs to. It needs to ask someone.
That "ask" is a **DNS query** — at its core, just a question:

> "What is the IP address for `www.example.com`?"

And the DNS system returns an answer:

> "The IP address is `93.184.216.34`."

Your computer doesn't broadcast this question to the entire internet.
It sends it to a designated server called a **recursive resolver**
(also called a recursive nameserver or DNS resolver). This is
typically provided by your ISP, or it can be a well-known public
resolver like Google's `8.8.8.8` or Cloudflare's `1.1.1.1`.

Think of the recursive resolver as your personal research assistant.
You hand it a question, and it does all the legwork — you don't need
to track down the answer yourself.

```svgbob
+-------------+      DNS Query       +--------------------+
|             |  -----------------> |                    |
|   Browser   |                     | Recursive Resolver |
|  (Client)   |  <----------------- |  (e.g., 1.1.1.1)  |
|             |      IP Address      |                    |
+-------------+                     +--------------------+
```

*Figure 1: The client only speaks to the recursive resolver. This
is the "local" layer — the resolver handles all the complexity behind
the scenes on the client's behalf.*

---

## Part 2: The Problem — One Server Can't Know Everything

Here's where it gets interesting. If the recursive resolver was just
one giant server holding every domain's IP address, we'd face an
engineering crisis:

- There are over **350 million registered domain names** globally.
- New domains are registered every second.
- IP addresses change constantly — CDN failover, server migrations,
load balancing updates.

A single centralized directory would be impossible to keep current,
a catastrophic single point of failure, and an impossible bottleneck
for billions of simultaneous queries.

This is the exact problem the engineers at ARPA faced in the early
1980s. Before DNS existed (formally specified in RFC 1034 and
RFC 1035 in 1987), the entire internet used a single file —
`HOSTS.TXT` — maintained by the Stanford Research Institute. Every
computer periodically downloaded this file to resolve hostnames.

As the internet grew, the system collapsed under its own weight. By
1982, maintaining that single file had become unmanageable. The
volume of updates, the coordination overhead, and the sheer number
of hosts made it unsustainable.

The solution they designed was elegant: **a distributed, hierarchical
tree of authority**, where responsibility for different parts of the
namespace is delegated to different servers around the world.

---

## Part 3: The Hierarchy — A Tree of Delegated Responsibility

DNS is organized as a tree. Each level is responsible for a specific
portion of the namespace, and each level **delegates** authority
downward. Let's build this tree from the top down.

### Level 1: The Root

At the very top of the DNS hierarchy is the **root**, represented by
a single dot (`.`). You rarely see it written explicitly, but every
fully qualified domain name (FQDN) technically ends with it:
`www.example.com.` (trailing dot) is the complete form.

There are **13 logical root nameserver addresses**, operated by
organizations like ICANN, VeriSign, NASA, and the University of
Maryland. They don't know where `www.example.com` lives — but they
know who does. Their job is to answer: *"For `.com` domains, talk
to these servers."*

### Level 2: Top-Level Domain (TLD) Nameservers

Below the root are **TLD nameservers**, responsible for specific
top-level domains: `.com`, `.org`, `.net`, `.io`, `.uk`, and so on.

VeriSign operates the `.com` TLD nameservers. When you ask them about
`example.com`, they don't hold the final answer either — but they
know which **authoritative nameserver** is responsible for it.

### Level 3: Authoritative Nameservers

At the bottom of the delegation chain are **authoritative
nameservers**. These servers hold the actual DNS records for a domain.
When you register `example.com` and set up hosting, you configure
these servers. They are the final authority. When they say "the IP
for `www.example.com` is `93.184.216.34`", that is the ground truth.

```svgbob
                      . (Root)
                     / | \
                    /  |  \
                 .com .org .net        <- TLD Nameservers
                 /        \
           example.com   google.com    <- Authoritative Nameservers
           /       \
         www       mail               <- DNS Records
```

*Figure 2: DNS as a tree. Authority flows strictly downward — the
root delegates to TLDs, TLDs delegate to authoritative nameservers,
and authoritative nameservers hold the actual records. No single node
needs to know everything. This is the key to DNS's extraordinary
scalability.*

---

## Part 4: The Full Resolution Journey — Step by Step

Now we can trace a complete DNS resolution. This is what happens when
you type `www.example.com` into your browser for the very first time,
with no cached results anywhere in the chain.

### Step 0: Check the Local Cache

Before any network traffic leaves your machine, your OS checks its
**local DNS cache**. If you've visited `www.example.com` recently and
the cache entry hasn't expired, the IP address is returned instantly
with zero network round trips.

### Step 1: Ask the Recursive Resolver

If the local cache misses, the query goes to the **recursive
resolver** — configured via DHCP from your router, or manually
set to a public resolver. The recursive resolver also maintains
its own cache. If another user on the same resolver recently looked
up the same domain, the result is returned immediately from cache.

If neither cache has the answer, the resolver starts its iterative
hunt through the hierarchy.

### Step 2: Ask the Root Nameservers

The recursive resolver contacts one of the **root nameservers**. It
already knows their IPs — they're hardcoded into DNS software via
the **root hints file**, a bootstrapping file shipped with every DNS
resolver implementation.

The resolver asks: *"Who handles `.com`?"*

The root server responds with a **referral**: *"I don't have the
final answer, but here are the `.com` TLD nameservers."*

### Step 3: Ask the TLD Nameserver

The resolver now queries a `.com` TLD nameserver and asks: *"Who
handles `example.com`?"*

The TLD server returns another referral: *"Here are the authoritative
nameservers for `example.com`"* — typically something like
`ns1.example.com` and `ns2.example.com`.

### Step 4: Ask the Authoritative Nameserver

The resolver contacts the **authoritative nameserver** for
`example.com` and asks: *"What is the IP for `www.example.com`?"*

The authoritative nameserver returns the actual DNS record — the IP
address — along with a **TTL** value.

### Step 5: Return and Cache

The recursive resolver (1) returns the IP to the client and (2) caches
the result along with its TTL. Your browser opens a TCP connection to
the IP and the page loads.

```svgbob
+--------+  1.Query   +-----------+  2.Ask Root  +---------+
|        | ---------> |           | -----------> |  Root   |
| Client |            | Recursive | <----------- |Namesvr  |
|        |            | Resolver  |  3.Referral  +---------+
|        |            |           |
|        |            |           |  4.Ask TLD   +---------+
|        |            |           | -----------> |   TLD   |
|        |            |           | <----------- |Namesvr  |
|        |            |           |  5.Referral  +---------+
|        |            |           |
|        |            |           |  6.Ask Auth  +---------+
|        |  8.IP Addr |           | -----------> |  Auth   |
|        | <--------- |           | <----------- |Namesvr  |
+--------+            +-----------+  7.IP Answer +---------+
```

*Figure 3: The resolver acts as the client's proxy, shielding it from
the full complexity of the hierarchy. The client makes one request and
gets one answer. Steps 2–7 happen transparently, typically in under
100 milliseconds.*

---

## Part 5: DNS Record Types — What Gets Stored

So far we've been loosely saying the authoritative server returns an
"IP address." In reality, it stores a variety of **DNS record types**,
each serving a distinct architectural purpose. We can group them by
their intent.

### Address Records (Routing Traffic to Servers)

- **A Record** — Maps a hostname to an IPv4 address.
`www.example.com → 93.184.216.34`
- **AAAA Record** — Maps a hostname to an IPv6 address.
`www.example.com → 2606:2800:220:1:248:1893:25c8:1946`


### Delegation and Alias Records (Pointing Elsewhere)

- **CNAME Record** (Canonical Name) — Maps a hostname to *another
hostname*, not an IP directly. `blog.example.com → example.wordpress.com`.
Useful for aliasing subdomains without hardcoding IPs. **Crucially,
you cannot use a CNAME at the zone apex (the root of a domain).**
This is a common interview gotcha — more on this in Part 10.
- **NS Record** — Identifies the authoritative nameservers for a domain.
This is how TLD servers delegate authority downward.
- **MX Record** — Specifies the mail server responsible for receiving
email for the domain. Carries a **priority value** so multiple mail
servers can be ranked for failover.


### Informational and Security Records

- **TXT Record** — Stores arbitrary text. Widely used for domain
ownership verification and email authentication standards like
**SPF**, **DKIM**, and **DMARC**.
- **SOA Record** (Start of Authority) — Contains zone metadata:
primary nameserver, admin contact, serial number, and timing
parameters for zone transfers. Every DNS zone has exactly one SOA.
- **PTR Record** — The inverse of an A record. Maps an IP address
back to a hostname. Used in **reverse DNS lookups**, common for
validating email servers.
- **SRV Record** — Specifies a host and port for specific services.
Used by VoIP, messaging protocols (XMPP), and service discovery
in microservice architectures.


### DNS Lookups in Python

Here's how we'd programmatically resolve DNS records using the
`dnspython` library — a standard tool for building services or
debugging DNS issues in production.

```python
import dns.resolver

def resolve_domain(domain: str, record_type: str = "A") -> list[str]:
    """
    Resolves DNS records for a given domain and record type.

    Args:
        domain: The domain name to resolve (e.g., 'www.example.com')
        record_type: DNS record type ('A', 'AAAA', 'MX', 'TXT', etc.)

    Returns:
        A list of resolved values as strings.
    """
    results = []

    try:
        answers = dns.resolver.resolve(domain, record_type)
        for record in answers:
            results.append(str(record))

    except dns.resolver.NXDOMAIN:
        # Domain does not exist at all
        print(f"[NXDOMAIN] '{domain}' does not exist.")

    except dns.resolver.NoAnswer:
        # Domain exists, but no record of this type is present
        print(f"[NO ANSWER] No {record_type} records found for '{domain}'.")

    except dns.resolver.Timeout:
        # Resolver did not respond within the configured timeout
        print(f"[TIMEOUT] DNS query timed out for '{domain}'.")

    return results


# Resolve A records (IPv4 addresses)
ipv4_addrs = resolve_domain("www.google.com", "A")
print(f"IPv4: {ipv4_addrs}")

# Resolve MX records (mail servers, with priority)
mail_servers = resolve_domain("gmail.com", "MX")
print(f"Mail Servers: {mail_servers}")

# Resolve TXT records (SPF, DKIM, ownership verification)
txt_records = resolve_domain("google.com", "TXT")
for record in txt_records:
    print(f"TXT: {record}")
```

Notice that we handle three distinct failure cases. `NXDOMAIN`,
`NoAnswer`, and `Timeout` each call for a different operational
response — knowing the distinction matters in production systems where
silent DNS failures can cause cascading outages.

---

## Part 6: Caching and TTL — Speed vs. Freshness

Every DNS record carries a **TTL (Time to Live)** value, measured in
seconds. This tells resolvers how long they may serve a cached record
before re-querying the authoritative server.

- `TTL = 300` → Cache for 5 minutes
- `TTL = 86400` → Cache for 24 hours
- `TTL = 60` → Cache for 1 minute


### The Trade-off

This is a classic systems engineering tension: **speed vs. freshness**.

**High TTL (e.g., 86400):**

- ✅ Cache hits reduce latency and load on authoritative servers
- ❌ IP changes take up to 24 hours to propagate globally

**Low TTL (e.g., 60):**

- ✅ Changes propagate within minutes
- ❌ More queries, higher load, slightly higher per-request latency

**Best practice for migrations:** Two days before changing a server's
IP address, lower the TTL to 60 seconds. This drains stale cache
entries across global resolvers. Then make the IP change. Once
confirmed stable, raise the TTL back to a higher value.

### Where Caching Happens

Caching doesn't occur in just one place — it's layered throughout the
resolution path, each layer serving a different scope of users.

```svgbob
+----------+   +----------+   +-----------+   +-------------+
|  Browser |   |    OS    |   | Recursive |   |Authoritative|
|  Cache   |-->|  Cache   |-->| Resolver  |   |  Nameserver |
|          |   |          |   |  Cache    |   |             |
+----------+   +----------+   +-----------+   +-------------+
  Per-tab        Per-device    Shared across    Ground truth,
  private        private       millions of      no caching
                               ISP users
```

*Figure 4: Three caching layers exist before we ever reach the
authoritative server. Each layer serves a progressively wider
audience. The browser cache is the fastest and most narrow; the
resolver cache is the broadest and handles enormous volumes of
repeated lookups.*

---

## Part 7: Iterative vs. Recursive Resolution

We've been describing **recursive resolution** — the resolver does
all the work and returns a final answer. DNS also defines **iterative
resolution**, where a server returns a referral ("go ask this other
server") instead of doing the lookup on the client's behalf.


| Mode | Who Does the Work | Typical User |
| :-- | :-- | :-- |
| Recursive | Resolver follows the chain | Client browsers and apps |
| Iterative | Client follows referrals | Resolvers querying root / TLD servers |

In practice, **clients always send recursive queries** — they don't
want the complexity of following referrals. **Resolvers use iterative
queries** when communicating with root and TLD nameservers, which
deliberately do not support recursive queries for performance and
security reasons.

---

## Part 8: DNS in Distributed Systems — Load Balancing and Failover

For senior-level interviews, this is where DNS becomes architecturally
interesting. DNS isn't just a lookup tool — it's also a primitive but
widely used **traffic management layer**.

### Round-Robin DNS Load Balancing

An authoritative nameserver can return **multiple A records** for the
same hostname. Resolvers and clients typically rotate through them in
a **round-robin** fashion.

```
www.example.com  300  IN  A  10.0.0.1
www.example.com  300  IN  A  10.0.0.2
www.example.com  300  IN  A  10.0.0.3
```

Client A receives `10.0.0.1`. Client B receives `10.0.0.2`. Client C
receives `10.0.0.3`. This distributes traffic roughly evenly.

**The limitation:** DNS round-robin is completely unaware of server
health or load. It will keep returning a failed IP until the record
is changed. This is why production systems typically have DNS point
to a **load balancer's IP** (like AWS ELB or NGINX), not directly to
application servers.

### GeoDNS — Latency-Based Routing

Providers like AWS Route 53 and Cloudflare support **GeoDNS**: they
return different IP addresses based on the geographic location of the
querying resolver. A user in Tokyo gets routed to the nearest edge
server in Asia. A user in London gets Frankfurt.

This is how CDNs like Cloudflare and Akamai route users to the closest
point of presence without any application-level logic — it all happens
at the DNS layer.

### DNS Failover

When a health check on an IP fails, some DNS providers automatically
remove that IP from responses and only return healthy addresses.
Combined with a low TTL, this enables **DNS-level failover** in
roughly 60–120 seconds — fast enough for most availability requirements.

```python
import dns.resolver
import time


def monitor_dns_propagation(
    domain: str,
    expected_ip: str,
    max_retries: int = 10,
    interval_seconds: int = 30
) -> bool:
    """
    Polls DNS resolution until an expected IP propagates,
    or the retry limit is reached. Useful after a DNS change
    to confirm that the new record is live globally.

    Args:
        domain: The domain to monitor (e.g., 'www.example.com')
        expected_ip: The new IP address we expect post-migration
        max_retries: Number of polling attempts before giving up
        interval_seconds: Delay between attempts

    Returns:
        True if propagation is confirmed, False otherwise.
    """
    resolver = dns.resolver.Resolver()
    # Bypass local cache by querying Google's resolver directly
    resolver.nameservers = ["8.8.8.8"]

    for attempt in range(1, max_retries + 1):
        try:
            answers = resolver.resolve(domain, "A")
            resolved_ips = [str(r) for r in answers]

            print(f"Attempt {attempt}/{max_retries} → {resolved_ips}")

            if expected_ip in resolved_ips:
                print(f"✅ Confirmed: {domain} → {expected_ip}")
                return True

        except Exception as e:
            print(f"Attempt {attempt} error: {e}")

        if attempt < max_retries:
            print(f"Waiting {interval_seconds}s...")
            time.sleep(interval_seconds)

    print(f"❌ Propagation not confirmed after {max_retries} attempts.")
    return False


# After migrating your server to 93.184.216.34,
# poll every 30 seconds for up to 5 minutes.
monitor_dns_propagation("www.example.com", "93.184.216.34")
```

We explicitly set `resolver.nameservers = ["8.8.8.8"]` to bypass the
local OS and ISP cache. Otherwise, we'd be checking a stale cached
value and might incorrectly conclude the propagation succeeded.

---

## Part 9: DNS Security — The Vulnerabilities and the Fixes

DNS was designed in the 1980s for a small, trusted academic network.
Security was not a primary concern. As the internet scaled and
adversaries became sophisticated, this created real and exploited
vulnerabilities.

### The Problem: DNS Cache Poisoning

In 2008, security researcher Dan Kaminsky disclosed a fundamental
vulnerability in DNS called **cache poisoning**. An attacker could
inject forged DNS responses into a resolver's cache, silently
redirecting users to malicious servers.

The attack works conceptually like this:

1. The attacker triggers a DNS query on the target resolver for a
domain the attacker controls.
2. Before the legitimate response arrives, the attacker floods the
resolver with forged responses containing guessed transaction IDs.
3. Due to weak randomization in early implementations, the correct
transaction ID could be guessed with reasonable probability.
4. The forged record is cached — every user on that resolver gets
redirected to the attacker's server.

The impact was enormous. Every OS, browser, and resolver was
potentially vulnerable.

### The Fix: DNSSEC

**DNSSEC (DNS Security Extensions)** addresses this by adding
**cryptographic signatures** to DNS records. Each record is signed
with a private key, and resolvers verify the signature using the
corresponding public key, which is itself published in DNS.

With DNSSEC, a forged response cannot pass signature validation — the
attacker doesn't possess the private signing key. The chain of trust
extends from the root all the way down to the authoritative nameserver.

```svgbob
+---------+   signs   +---------+   signs   +-------------+
|  Root   | --------> |   TLD   | --------> |Authoritative|
|  Zone   |           |  Zone   |           |    Zone     |
+---------+           +---------+           +-------------+
     |                     |                      |
     v                     v                      v
Root DNSKEY            TLD DNSKEY           Domain DNSKEY
 (published)           (published)          (published)
```

*Figure 5: DNSSEC creates a verifiable chain of trust. Each level
cryptographically signs the public keys of the level below it. A
validating resolver can trace that chain from the root down to the
authoritative nameserver to confirm that no data was tampered with
in transit.*

**Limitation of DNSSEC:** It guarantees data **integrity** — it
confirms the record hasn't been tampered with. But it does not provide
**confidentiality** — anyone monitoring network traffic can still see
which domains you're querying.

### DNS over HTTPS (DoH) and DNS over TLS (DoT)

These protocols encrypt the DNS query itself, hiding it from network
observers — ISPs, Wi-Fi operators, or surveillance infrastructure.

- **DoT (DNS over TLS)** — Wraps DNS queries in a TLS connection on
port 853. Clean separation of concerns; easy to filter at the
network level.
- **DoH (DNS over HTTPS)** — Sends DNS queries over HTTPS on port
443. DNS traffic becomes indistinguishable from regular web traffic,
making it resistant to blocking.

Both are supported by major browsers (Firefox, Chrome) and public
resolvers. Cloudflare's `1.1.1.1` supports both; Google's `8.8.8.8`
supports DoH.

---

## Part 10: Interview Scenarios and Common Gotchas

These are the patterns that reliably appear in senior and staff-level
engineering interviews.

### "Walk me through what happens when you type a URL in the browser."

DNS resolution is step one of the canonical deep-dive question.
After DNS resolves the IP, the full sequence continues: TCP three-way
handshake → TLS handshake (for HTTPS) → HTTP request → server
processing → HTTP response → browser rendering (HTML parsing,
resource loading, render pipeline). Getting DNS exactly right —
including the caching layers and hierarchy — sets the foundation for
the rest of the answer and signals depth.

### "How does DNS support high availability?"

- Multiple A records with round-robin distribute traffic across servers
- Low TTL combined with health checks enables DNS-level failover
- GeoDNS routes users to the geographically nearest server
- Multiple authoritative nameservers (always deploy at least two)
provide redundancy at the authority layer itself


### "Why don't DNS changes propagate instantly?"

The change is immediate on the authoritative server. But resolvers
across the internet have cached the old record, and they'll serve it
until its TTL expires. This is a feature, not a bug — without caching,
DNS would require billions of queries per second to authoritative
servers. TTL management (lowering before migrations, raising after)
is how we balance propagation speed with cache efficiency.

### "What's a CNAME and why can't you use it at the zone apex?"

A CNAME maps a hostname to another hostname. `www.example.com` can
be a CNAME pointing to `lb.example.net`. But `example.com` itself
**cannot** be a CNAME because RFC 1034 requires the zone apex to
have SOA and NS records, and a CNAME record at any name prohibits
all other record types at that name. Many DNS providers work around
this with proprietary **ALIAS** or **ANAME** record types that behave
like CNAMEs at the zone apex.

### "What is negative caching?"

When a DNS lookup returns `NXDOMAIN` (the domain does not exist),
resolvers also cache that **negative result** for a duration defined
in the SOA record's `MINIMUM` field. This prevents repeated queries
for non-existent domains from hammering authoritative servers — a
concern not just for efficiency but as a defense against amplification
attacks.

```python
import dns.resolver


def classify_dns_response(domain: str) -> dict:
    """
    Performs a DNS lookup and categorizes the outcome, including
    negative responses. Demonstrates the distinction between
    NXDOMAIN (domain absent) and NoAnswer (domain exists,
    record type absent).
    """
    try:
        answers = dns.resolver.resolve(domain, "A")
        return {
            "status": "FOUND",
            "domain": domain,
            "ips": [str(r) for r in answers],
            "ttl": answers.rrset.ttl,
        }

    except dns.resolver.NXDOMAIN:
        # Negative result — also cached by resolvers (negative caching)
        return {
            "status": "NXDOMAIN",
            "domain": domain,
            "note": (
                "Domain does not exist. Resolvers cache this result "
                "for the SOA MINIMUM TTL to prevent repeated queries."
            ),
        }

    except dns.resolver.NoAnswer:
        # Domain exists but has no A record (may have AAAA, MX, etc.)
        return {
            "status": "NO_A_RECORD",
            "domain": domain,
            "note": "Domain exists but carries no A record.",
        }

    except dns.resolver.Timeout:
        return {
            "status": "TIMEOUT",
            "domain": domain,
            "note": "Resolver did not respond in time.",
        }


# Concrete test cases demonstrating each outcome
print(classify_dns_response("www.google.com"))
# → FOUND, with IPv4 addresses and TTL

print(classify_dns_response("this-domain-absolutely-does-not-exist-xyz.com"))
# → NXDOMAIN, with negative caching note

print(classify_dns_response("gmail.com"))
# → may return A record, or NO_A_RECORD depending on configuration
```

The separation between `NXDOMAIN` and `NoAnswer` is subtle but
important in real systems. An `NXDOMAIN` tells you the domain
registration itself doesn't exist. A `NoAnswer` tells you the domain
exists but lacks the specific record type — useful when debugging
misconfigured mail servers or AAAA-only deployments.

---

## Putting It All Together

DNS is elegant precisely because it solves an enormous coordination
problem — mapping hundreds of millions of names to billions of
addresses across a globally distributed system — with a surprisingly
small set of design principles:

- **Delegate authority** so no single server owns everything
- **Cache aggressively** so the network isn't flooded with queries
- **Respect TTLs** to balance freshness with performance
- **Sign records** to protect integrity in an adversarial environment
- **Encrypt queries** to protect privacy from observers

For an interview, what separates a great answer from a good one isn't
memorizing the steps — it's understanding *why* each component exists.
The root servers exist because we needed a globally agreed-upon
starting point that everyone trusts. The recursive resolver exists
because clients shouldn't bear the complexity of iterative lookups.
TTL exists because we can't query authoritative servers for every
single request, but we also can't serve stale data forever. DNSSEC
exists because trust on an unsigned system can be forged. DoH exists
because confidentiality matters even when integrity is guaranteed.

Every design decision in DNS is a response to a real constraint.
When you see the system that way, the architecture stops being a set
of facts to memorize and becomes a set of engineering decisions you
can reason about, debate, and extend — which is exactly how the best
engineers think.

```
