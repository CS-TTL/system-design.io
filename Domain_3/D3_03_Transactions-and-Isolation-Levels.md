
# Transactions and Isolation Levels

## The Bank Teller Problem

Imagine it's Monday morning at a busy bank branch. Two tellers — Alice and Bob — are serving customers simultaneously. A couple, Jamie and Sam, share a joint account with exactly **$500** in it.

At 9:00 AM sharp:
- Jamie walks to Alice's window and requests to **withdraw $400**.
- Sam, at the exact same moment, walks to Bob's window and requests to **withdraw $300**.

Both tellers glance at the balance simultaneously. Both see **$500**. Both decide: "Yes, sufficient funds — approved." Alice processes $400 out. Bob processes $300 out. The account is now at **-$200**.

The bank just lost $200. Not because of fraud. Not because of a bug. But because two operations that were never supposed to overlap did exactly that — they *read the same data simultaneously and wrote back incompatible results.*

This, in essence, is the problem that **database transactions and isolation levels** were built to solve. And as we'll discover, the solution is surprisingly nuanced. The database can't simply "lock everything and slow down" — it needs to offer fine-grained control over *how much* operations can overlap, and *what trade-offs* we're willing to accept.

Let's build up our understanding from the atomic unit.

---

## The Atomic Unit: What is a Transaction?

Before we can talk about isolation, we need to understand what we're isolating. A **transaction** is the smallest logical unit of work in a database — a sequence of operations (reads and writes) that must be treated as a single, indivisible action.

Think of a transaction like a **bank transfer**: to move $200 from Account A to Account B, you need two separate operations:

1. Deduct $200 from Account A
2. Add $200 to Account B

These two operations are individually meaningless in isolation. If step 1 completes but step 2 fails — say, because the server crashes — you've just made $200 vanish into the ether. That's catastrophic. A transaction wraps both steps together and makes a guarantee: **either both happen, or neither happens.**

In SQL, a transaction looks like this:

```sql
BEGIN;
    UPDATE accounts SET balance = balance - 200 WHERE id = 'account_A';
    UPDATE accounts SET balance = balance + 200 WHERE id = 'account_B';
COMMIT;
```

The `BEGIN` opens the transaction. The `COMMIT` finalizes it. If anything goes wrong between those two statements, a `ROLLBACK` undoes everything as if the transaction never started.

```
  ┌─────────────────────────────────────────────────┐
  │               TRANSACTION LIFECYCLE              │
  │                                                  │
  │   BEGIN ──▶ Operation 1 ──▶ Operation 2 ──▶ ...  │
  │      │                                    │      │
  │      │         SUCCESS PATH               │      │
  │      │                                 COMMIT    │
  │      │                                    │      │
  │      │         FAILURE PATH               │      │
  │      └──────────────────────────────▶ ROLLBACK   │
  │                                                  │
  └─────────────────────────────────────────────────┘
```

In Figure 1, notice that there are only two exit paths from any transaction: a successful `COMMIT` that makes changes permanent, or a `ROLLBACK` that erases them entirely. There is no "partial commit." That all-or-nothing guarantee is the foundation of everything we'll build on.

---

## The Four Pillars: ACID

The transaction lifecycle we just described is governed by four foundational properties known as **ACID**. These aren't optional features — they are the contract that reliable databases make with every application. Let's unpack them one at a time, building from the simplest to the most complex.

### Atomicity — All or Nothing

We've already seen this one. **Atomicity** guarantees that a transaction is treated as a single unit. Every operation within it either succeeds completely, or the entire transaction is rolled back. This is what prevents our bank transfer from leaving the system in a half-updated state.

The implementation mechanism is typically a **Write-Ahead Log (WAL)**: before any change is written to the actual data files, it's first recorded in a log. If a crash occurs, the database can replay or undo operations from the log on restart.

```
  Account A: $500         Account A: $300
  Account B: $200         Account B: $400
       │                       │
       ▼                       ▼
  [BEGIN TX]             [COMMITTED]
    deduct $200  ──────▶   +$200 applied
    add $200     ──────▶   -$200 applied
                                │
           ─── OR ───           │
                                ▼
                        [ROLLBACK on crash]
                        Account A: $500 (restored)
                        Account B: $200 (restored)
```


### Consistency — Rules Must Hold

**Consistency** means that a transaction can only bring the database from one *valid* state to another. It cannot violate the business rules or schema constraints we've defined — things like "a balance can never go negative" or "every order must reference an existing customer."

If a transaction tries to break a constraint, the database rejects it entirely. Consistency is the bridge between the technical guarantee (atomicity) and the business guarantee (correctness).

### Isolation — Transactions Don't Step on Each Other

**Isolation** is the property at the heart of this article. It means that concurrently executing transactions should behave *as if* they were running serially — one at a time, in some order. The intermediate state of one transaction should not be visible to other transactions.

This is the hard one. It's easy to say "act as if they're serial" — but in practice, we might be processing thousands of transactions per second. True, perfect isolation would destroy performance. Which is why, as we'll see, databases offer a *spectrum* of isolation levels, each making different trade-offs.

### Durability — Committed Means Permanent

**Durability** guarantees that once a transaction is committed, it stays committed — even in the event of a system crash, power failure, or catastrophic server meltdown. This is achieved through the same Write-Ahead Log: once the commit record is flushed to disk, the data is permanent.

```
  ┌───────────────────────────────────────────────────────────┐
  │                  THE ACID PROPERTIES                      │
  │                                                           │
  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────┐  │
  │  │  ATOMIC   │  │CONSISTENT │  │ ISOLATED  │  │DURABLE│  │
  │  │           │  │           │  │           │  │       │  │
  │  │ All or    │  │ Rules must│  │ Txns don't│  │Commit │  │
  │  │ Nothing   │  │ be upheld │  │ interfere │  │sticks │  │
  │  │           │  │           │  │           │  │forever│  │
  │  └───────────┘  └───────────┘  └───────────┘  └───────┘  │
  │                                                           │
  └───────────────────────────────────────────────────────────┘
```

Now we have a foundation. We understand what a transaction is, and we understand the *ideal* we're aiming for. But here's where it gets interesting: **Isolation, in its perfect form, is expensive.** To understand why, we need to introduce some chaos.

---

## The Concurrency Problem: When Transactions Collide

Modern databases serve thousands of concurrent users. Enforcing strict, perfect isolation would mean serializing every transaction — processing them one by one. That's untenable for a production system handling millions of requests per day.

So databases make a pragmatic choice: **they allow some degree of concurrency, and in doing so, they open the door to certain anomalies.** Our job, as engineers, is to understand these anomalies and choose the level of isolation that prevents the ones we care about.

There are three classic concurrency anomalies. Let's meet them in order of severity, from the least alarming to the most insidious.

### Anomaly 1: The Dirty Read

A **dirty read** occurs when Transaction A reads data that Transaction B has *modified but not yet committed*. If Transaction B then rolls back, Transaction A has read data that effectively never existed.

```
  Timeline ──────────────────────────────────────────────────▶

  Txn B:  BEGIN ──▶ UPDATE balance = $50 ──────────────▶ ROLLBACK
                                │
  Txn A:            BEGIN ──────┴──▶ SELECT balance
                                      reads $50 ← DIRTY!
                                      (This $50 was never real)
```

In Figure 3, notice that Txn A reads the value `$50` — a value that Transaction B was in the *middle* of computing and ultimately discarded. Txn A now makes decisions based on a ghost value. This is like reading someone's draft email before they've decided whether to send it.

**Real-world impact:** An order management system might read an inventory update that gets rolled back, leading it to believe an item is out of stock when it's actually not — or worse, available when it isn't.

### Anomaly 2: The Non-Repeatable Read

A **non-repeatable read** occurs when Transaction A reads the same row *twice* within the same transaction and gets *different values* — because Transaction B committed a change between the two reads.

```
  Timeline ──────────────────────────────────────────────────▶

  Txn A:  BEGIN ──▶ SELECT balance  ─────────────▶ SELECT balance
                    returns $100                   returns $150 ← DIFFERENT!

  Txn B:              BEGIN ──▶ UPDATE balance = $150 ──▶ COMMIT
```

In Figure 4, Transaction A's two reads are both looking at committed data — so neither is "dirty." But the data shifted underneath Txn A between its two reads. This is the classic "rug-pull" problem. Txn A is computing something based on `$100`, and by the time it reads again, the ground has shifted.

**Real-world impact:** A reporting query that sums up multiple columns and reads each row multiple times might produce an inconsistent aggregate — one that doesn't correspond to any real state of the database.

### Anomaly 3: The Phantom Read

The **phantom read** is the most subtle. Transaction A executes a query with a *range condition* (e.g., `WHERE age > 25`) and gets a set of rows. Meanwhile, Transaction B inserts or deletes rows that match that condition and commits. When Transaction A re-executes the same query, it gets a *different set of rows* — new rows have "appeared" like phantoms.

```
  Timeline ──────────────────────────────────────────────────▶

  Txn A:  BEGIN ──▶ SELECT * WHERE dept='Sales'  ──▶  SELECT * WHERE dept='Sales'
                    returns 3 rows                     returns 5 rows ← PHANTOMS!

  Txn B:              BEGIN ──▶ INSERT 2 rows (dept='Sales') ──▶ COMMIT
```

The key distinction from a non-repeatable read: a non-repeatable read is about a *specific row changing*. A phantom read is about the *set of rows changing* — rows appearing or disappearing entirely.

**Real-world impact:** A concert ticketing system might calculate that 5 seats are available, then proceed to reserve them — only to find that 3 of those seats were just claimed by another transaction, leaving the booking in an invalid state.

Now that we understand the three enemies, let's meet the four levels of armor.

---

## The Four Isolation Levels

SQL defines four standard isolation levels. Think of them as a dial — turning it up gives you more protection against anomalies but costs you concurrency performance. Turning it down gives you speed but leaves you exposed to certain anomalies.

```
  PERFORMANCE                                    CORRECTNESS
       │                                              │
       │                                              │
  READ          READ           REPEATABLE      SERIALIZABLE
  UNCOMMITTED   COMMITTED      READ
       │             │              │                 │
       ▼             ▼              ▼                 ▼
  [Fastest]    [Default in     [Default in       [Slowest]
               PostgreSQL]      MySQL]
```


### Level 1: READ UNCOMMITTED — The Wild West

**READ UNCOMMITTED** is the lowest rung on the isolation ladder. At this level, transactions can read uncommitted changes from other transactions. All three anomalies — dirty reads, non-repeatable reads, and phantom reads — are possible.

In practice, very few modern databases even fully implement this level. PostgreSQL, for instance, treats `READ UNCOMMITTED` as `READ COMMITTED` internally, because the database maintainers considered dirty reads too dangerous to enable.

```
  ┌────────────────────────────────────────────────────────┐
  │             READ UNCOMMITTED                           │
  │                                                        │
  │  Dirty Reads:        ✗  ALLOWED                        │
  │  Non-Repeatable:     ✗  ALLOWED                        │
  │  Phantom Reads:      ✗  ALLOWED                        │
  │  Performance:        ★★★★★  (Highest)                  │
  └────────────────────────────────────────────────────────┘
```

**When would you ever use this?** Honestly, almost never. The one legitimate use case is running *approximate analytics* on a massive dataset where you need maximum speed and are fine with results being slightly stale or inconsistent. Think: a real-time dashboard estimating order volume, where being off by a few records is totally acceptable.

```python
import psycopg2
from psycopg2.extensions import ISOLATION_LEVEL_READ_UNCOMMITTED

conn = psycopg2.connect(dbname="analytics_db", user="reader")

# Acceptable for rough approximations only

conn.set_isolation_level(ISOLATION_LEVEL_READ_UNCOMMITTED)

cursor = conn.cursor()
cursor.execute("SELECT COUNT(*) FROM orders WHERE status = 'pending'")
approx_count = cursor.fetchone()
print(f"Approximate pending orders: {approx_count}")

conn.commit()
conn.close()
```


### Level 2: READ COMMITTED — The Practical Default

**READ COMMITTED** prevents dirty reads. Every read within a transaction will only see data that has been *committed* by the time the read occurs. It does not, however, prevent non-repeatable reads or phantom reads.

This is the default isolation level in PostgreSQL and Oracle, and for good reason: it eliminates the most dangerous anomaly (dirty reads) while keeping concurrency high.

```
  ┌────────────────────────────────────────────────────────┐
  │             READ COMMITTED                             │
  │                                                        │
  │  Dirty Reads:        ✓  PREVENTED                      │
  │  Non-Repeatable:     ✗  ALLOWED                        │
  │  Phantom Reads:      ✗  ALLOWED                        │
  │  Performance:        ★★★★☆  (High)                     │
  └────────────────────────────────────────────────────────┘
```

The mechanism here is straightforward: each individual `SELECT` statement sees a snapshot of all committed data *at the moment that statement begins*. But if you issue two `SELECT` statements within the same transaction, each one gets its own fresh snapshot — which means another transaction could have committed changes between your two reads.

```python
import psycopg2
from psycopg2.extensions import ISOLATION_LEVEL_READ_COMMITTED

def get_product_inventory(product_id: int) -> int:
    """
    READ COMMITTED is ideal here: we want committed data,
    and we don't need the count to be frozen mid-transaction.
    """
    conn = psycopg2.connect(dbname="store_db", user="app_user")
    conn.set_isolation_level(ISOLATION_LEVEL_READ_COMMITTED)

    try:
        cursor = conn.cursor()
        cursor.execute(
            "SELECT quantity FROM inventory WHERE product_id = %s",
            (product_id,)
        )
        quantity = cursor.fetchone()
        conn.commit()
        return quantity
    except Exception as e:
        conn.rollback()
        raise e
    finally:
        conn.close()
```

**When to use it:** Almost all standard web applications — social feeds, e-commerce product listings, content management systems. Anywhere that the consequence of a slightly stale read is low, and throughput matters.

### Level 3: REPEATABLE READ — The Snapshot

**REPEATABLE READ** takes READ COMMITTED one step further: not only does it prevent dirty reads, but it also guarantees that if you read a row within a transaction, *every subsequent read of that same row within the same transaction will return the same value* — regardless of what other transactions commit in the meantime.

The transaction gets a **snapshot** of the database at the moment it begins, and all reads within that transaction see the world as it was at that snapshot.

```
  ┌────────────────────────────────────────────────────────┐
  │             REPEATABLE READ                            │
  │                                                        │
  │  Dirty Reads:        ✓  PREVENTED                      │
  │  Non-Repeatable:     ✓  PREVENTED                      │
  │  Phantom Reads:      ✗  ALLOWED (in standard SQL)      │
  │  Performance:        ★★★☆☆  (Medium)                   │
  └────────────────────────────────────────────────────────┘
```

Note: PostgreSQL's implementation of REPEATABLE READ actually prevents phantom reads too, due to how it uses MVCC (Multi-Version Concurrency Control). MySQL/InnoDB uses gap locks. But in the SQL standard, phantom reads are technically still allowed at this level — so in interviews, always clarify you're discussing the SQL standard versus a specific engine's implementation.

```
  Timeline: REPEATABLE READ in action
  ─────────────────────────────────────────────────────────

  T=0: Txn A begins. Snapshot ID = 42.
  
  T=1: Txn A reads account balance → $500 (from snapshot 42)
  
  T=2: Txn B commits: UPDATE balance = $800
  
  T=3: Txn A reads account balance → STILL $500 ← from snapshot 42!
  
  T=4: Txn A commits.
  
  Result: Txn A saw a consistent view throughout.
```

This is the default isolation level in **MySQL/InnoDB** — a deliberate choice for a database often used in transactional systems where read consistency matters.

```python
import psycopg2
from psycopg2.extensions import ISOLATION_LEVEL_REPEATABLE_READ

def generate_account_summary(user_id: int) -> dict:
    """
    Calculating a summary across multiple related tables.
    We need ALL reads to reflect the same consistent snapshot.
    REPEATABLE READ is ideal for this read-heavy, multi-query report.
    """
    conn = psycopg2.connect(dbname="finance_db", user="reporting_user")
    conn.set_isolation_level(ISOLATION_LEVEL_REPEATABLE_READ)

    try:
        cursor = conn.cursor()

        cursor.execute(
            "SELECT SUM(balance) FROM accounts WHERE user_id = %s",
            (user_id,)
        )
        total_balance = cursor.fetchone()

        cursor.execute(
            "SELECT COUNT(*) FROM transactions WHERE user_id = %s AND status='completed'",
            (user_id,)
        )
        total_txns = cursor.fetchone()

        # Both queries above read from the SAME snapshot.
        # No risk of a concurrent update skewing one number
        # relative to the other.

        conn.commit()
        return {
            "user_id": user_id,
            "total_balance": total_balance,
            "total_transactions": total_txns
        }
    except Exception as e:
        conn.rollback()
        raise e
    finally:
        conn.close()
```

**When to use it:** Financial reporting, balance summaries, any multi-query operation where you need a coherent view across several reads. Perfect for read-only transactions.

### Level 4: SERIALIZABLE — The Iron Fist

**SERIALIZABLE** is the highest and most restrictive isolation level. It guarantees that the outcome of executing transactions concurrently is identical to some *serial execution* — as if each transaction ran one after another, in some order.

This prevents *all* three anomalies, including phantom reads.

```
  ┌────────────────────────────────────────────────────────┐
  │             SERIALIZABLE                               │
  │                                                        │
  │  Dirty Reads:        ✓  PREVENTED                      │
  │  Non-Repeatable:     ✓  PREVENTED                      │
  │  Phantom Reads:      ✓  PREVENTED                      │
  │  Performance:        ★★☆☆☆  (Lower — retry overhead)   │
  └────────────────────────────────────────────────────────┘
```

The trade-off is **serialization failures**. When two serializable transactions would conflict in a way that prevents a valid serial ordering, one of them is aborted and must retry. Your application code *must* handle this.

```
  Two Serializable Transactions — Conflict Example:
  ─────────────────────────────────────────────────

  Txn A reads "seats available" = 3
  Txn B reads "seats available" = 3
  Txn A reserves 2 seats → writes back 1 remaining
  Txn B reserves 2 seats → SERIALIZATION FAILURE ← aborted!

  One of these transactions must retry, finding only 1 seat left.
  Correct behavior is enforced.
```

Because retries are now part of the contract, your code needs a retry loop:

```python
import psycopg2
from psycopg2 import errors
from psycopg2.extensions import ISOLATION_LEVEL_SERIALIZABLE
import time
import logging

logger = logging.getLogger(__name__)

def execute_serializable(operation_fn, max_retries: int = 5):
    """
    Wraps a database operation in serializable isolation with
    exponential backoff retry logic. Any operation passed to
    this function will be retried on serialization failure.
    """
    attempt = 0
    while attempt < max_retries:
        conn = psycopg2.connect(dbname="banking_db", user="tx_user")
        conn.set_isolation_level(ISOLATION_LEVEL_SERIALIZABLE)

        try:
            result = operation_fn(conn)
            conn.commit()
            return result

        except errors.SerializationFailure:
            conn.rollback()
            wait = 0.1 * (2 ** attempt)   # exponential backoff
            logger.warning(f"Serialization failure. Retry {attempt + 1} in {wait:.2f}s")
            time.sleep(wait)
            attempt += 1

        except Exception as e:
            conn.rollback()
            raise

        finally:
            conn.close()

    raise RuntimeError(f"Transaction failed after {max_retries} retries.")


def transfer_funds(from_id: int, to_id: int, amount: float):
    """
    A critical financial operation: must use SERIALIZABLE.
    We cannot tolerate any anomalies when moving money.
    """
    def _transfer(conn):
        cursor = conn.cursor()

        cursor.execute(
            "SELECT balance FROM accounts WHERE id = %s FOR UPDATE",
            (from_id,)
        )
        source_balance = cursor.fetchone()

        if source_balance < amount:
            raise ValueError(f"Insufficient funds: have {source_balance}, need {amount}")

        cursor.execute(
            "UPDATE accounts SET balance = balance - %s WHERE id = %s",
            (amount, from_id)
        )
        cursor.execute(
            "UPDATE accounts SET balance = balance + %s WHERE id = %s",
            (amount, to_id)
        )
        return True

    return execute_serializable(_transfer)
```

**When to use it:** Banking transfers, inventory reservation systems, ticket booking, any domain where the cost of an anomaly (missing funds, oversold tickets) is catastrophic.

---

## The Full Picture: Anomaly Matrix

We now have all four levels in view. Let's consolidate the relationship between isolation levels and the anomalies they prevent:

```
  ┌─────────────────┬─────────────┬───────────────────┬──────────────┐
  │ Isolation Level │ Dirty Read  │ Non-Repeatable    │ Phantom Read │
  │                 │             │ Read              │              │
  ├─────────────────┼─────────────┼───────────────────┼──────────────┤
  │ READ UNCOMMITTED│   Possible  │     Possible      │   Possible   │
  │ READ COMMITTED  │  Prevented  │     Possible      │   Possible   │
  │ REPEATABLE READ │  Prevented  │    Prevented      │  Possible*   │
  │ SERIALIZABLE    │  Prevented  │    Prevented      │  Prevented   │
  └─────────────────┴─────────────┴───────────────────┴──────────────┘

  * PostgreSQL's REPEATABLE READ also prevents phantom reads via MVCC.
    MySQL uses gap locks for the same effect.
    The SQL standard technically allows them at this level.
```

Think of this table as a cliff-face: every row you climb up gives you more safety nets, but makes the ascent slower.

---

## Under the Hood: How Isolation is Implemented

We've talked about *what* isolation levels do. Now let's briefly peek behind the curtain at *how* databases enforce them. There are two dominant approaches: **locking** and **Multi-Version Concurrency Control (MVCC)**.

### The Locking Approach

The traditional approach is simple: if Transaction A is reading a row, put a **shared lock** on it so that Transaction B can't write to it until A is done. If A is writing, put an **exclusive lock** so nobody can read or write until A commits.

```
  LOCK-BASED ISOLATION (Simplified)

  Txn A: READ(row 5) ──▶ acquires SHARED LOCK on row 5
                                     │
  Txn B: WRITE(row 5) ──▶ blocked until Txn A releases lock
                                     │
  Txn A: COMMIT ──────────────▶ LOCK RELEASED
                                     │
  Txn B: ─────────────────────────── ▼ ──▶ WRITE proceeds
```

The problem with pure locking: it can lead to **deadlocks** (Txn A waits for Txn B, while Txn B waits for Txn A — a circular dependency that neither can escape). Databases detect deadlocks and kill one of the transactions to break the cycle.

### The MVCC Approach

Modern databases like PostgreSQL and MySQL use **Multi-Version Concurrency Control (MVCC)** — a far more elegant solution. Instead of blocking writers with locks, the database maintains *multiple versions* of every row.

When Txn A reads a row, it gets the version that was current *at the time the transaction started* (or at the time of the read, depending on isolation level). When Txn B writes to the same row, it creates a *new version*, leaving Txn A's snapshot untouched.

```
  MVCC — Multiple Versions of the Same Row

  Physical Storage:
  ┌──────────────────────────────────────────────────┐
  │  Row ID 5                                        │
  │  ┌────────────────┬──────────────────────────┐   │
  │  │ Version 1      │ balance=$500, txn_id=10  │   │
  │  │ (committed)    │ valid from txn 10        │   │
  │  ├────────────────┼──────────────────────────┤   │
  │  │ Version 2      │ balance=$800, txn_id=14  │   │
  │  │ (committed)    │ valid from txn 14        │   │
  │  ├────────────────┼──────────────────────────┤   │
  │  │ Version 3      │ balance=$600, txn_id=17  │   │
  │  │ (in-flight)    │ not yet committed        │   │
  │  └────────────────┴──────────────────────────┘   │
  └──────────────────────────────────────────────────┘

  Txn 15 (READ COMMITTED): reads Version 2 ($800)
  Txn 11 (REPEATABLE READ, started before txn 14):
         reads Version 1 ($500) — its snapshot
```

In Figure 8, notice that readers and writers are never blocked by each other — each transaction simply reads from its appropriate version. This is the key insight: **MVCC trades storage space for concurrency.** Stale versions are periodically cleaned up by a background process (called `VACUUM` in PostgreSQL).

---

## Database Defaults and Practical Differences

Different databases ship with different defaults and have subtle implementation quirks. Here's a reference to keep close during system design discussions:

```
  ┌───────────────┬──────────────────┬─────────────────────────────────┐
  │ Database      │ Default Level    │ Notable Behavior                 │
  ├───────────────┼──────────────────┼─────────────────────────────────┤
  │ PostgreSQL    │ READ COMMITTED   │ MVCC; REPEATABLE READ also       │
  │               │                  │ prevents phantom reads           │
  ├───────────────┼──────────────────┼─────────────────────────────────┤
  │ MySQL/InnoDB  │ REPEATABLE READ  │ Gap locks prevent phantom reads  │
  │               │                  │ at REPEATABLE READ level         │
  ├───────────────┼──────────────────┼─────────────────────────────────┤
  │ Oracle        │ READ COMMITTED   │ No READ UNCOMMITTED support;     │
  │               │                  │ Snapshot Isolation available     │
  ├───────────────┼──────────────────┼─────────────────────────────────┤
  │ SQL Server    │ READ COMMITTED   │ Offers Snapshot Isolation as     │
  │               │                  │ an alternative to SERIALIZABLE   │
  ├───────────────┼──────────────────┼─────────────────────────────────┤
  │ CockroachDB   │ SERIALIZABLE     │ Defaults to highest; designed    │
  │               │                  │ for distributed correctness      │
  └───────────────┴──────────────────┴─────────────────────────────────┘
```

CockroachDB's choice to default to SERIALIZABLE is a noteworthy philosophical stance: the designers believed that correctness should be the default, and performance optimization should be an explicit opt-in, not the other way around.

---

## Choosing the Right Isolation Level

Now we bring it all together. When designing a system, we ask ourselves three questions:

```
  DECISION FLOWCHART

  Q1: Does my application read then act on
      that data within the same transaction?
          │
       YES │             NO
          ▼              ▼
  Q2: Do I re-read    READ COMMITTED
      the same rows    is probably fine
      multiple times?
          │
       YES │             NO
          ▼              ▼
  Q3: Are phantom    READ COMMITTED
      rows a risk?    is fine
          │
       YES │             NO
          ▼              ▼
  SERIALIZABLE       REPEATABLE READ
  (with retries)
```

Let's map this to real-world domains:

- **Social media / content platforms:** `READ COMMITTED`. A slightly stale like count is tolerable.
- **E-commerce product listings:** `READ COMMITTED`. Showing inventory that's off by 1 is acceptable — the checkout process handles final validation.
- **Financial reporting / dashboards:** `REPEATABLE READ`. Summaries must be internally consistent across all queries.
- **Bank transfers / payment processing:** `SERIALIZABLE`. Absolute correctness. Money cannot disappear.
- **Ticket reservation / hotel booking:** `SERIALIZABLE`. Overselling a seat or a room is not an option.
- **Approximate analytics (log aggregation, telemetry):** `READ UNCOMMITTED`. Raw speed, minor inconsistency is tolerable.

---

## A Common Trap: Lost Updates

Before we wrap up, there's one more anomaly that's easy to miss in interviews: the **lost update problem**. This one is insidious because it can happen even at `REPEATABLE READ`, if you're not careful.

Here's the scenario: two transactions both read the same value, compute something based on it, and write back — the second write silently overwrites the first.

```
  THE LOST UPDATE PROBLEM

  T=0: balance = $100

  Txn A: READ balance → $100
  Txn B: READ balance → $100
  
  Txn A: WRITE balance = $100 + $50 → $150  (deposit $50)
  Txn B: WRITE balance = $100 - $30 → $70   (withdraw $30)

  Final: $70  ← Txn A's deposit is completely LOST!
  Expected: $120  ($100 + $50 - $30)
```

The fix is to use **`SELECT FOR UPDATE`** — a locking read that signals intent to modify. This forces Txn B to wait until Txn A commits before it can read the row:

```python
import psycopg2
from psycopg2.extensions import ISOLATION_LEVEL_READ_COMMITTED

def apply_deposit(account_id: int, deposit_amount: float):
    """
    Using SELECT FOR UPDATE to prevent the lost update problem.
    Even at READ COMMITTED, this lock ensures no concurrent
    transaction can modify the row between our read and write.
    """
    conn = psycopg2.connect(dbname="banking_db", user="app_user")
    conn.set_isolation_level(ISOLATION_LEVEL_READ_COMMITTED)

    try:
        cursor = conn.cursor()

        # FOR UPDATE acquires an exclusive lock on this row.
        # Any other transaction attempting to read/modify
        # this row will BLOCK until we commit.
        cursor.execute(
            "SELECT balance FROM accounts WHERE id = %s FOR UPDATE",
            (account_id,)
        )
        current_balance = cursor.fetchone()
        new_balance = current_balance + deposit_amount

        cursor.execute(
            "UPDATE accounts SET balance = %s WHERE id = %s",
            (new_balance, account_id)
        )
        conn.commit()
        return new_balance

    except Exception as e:
        conn.rollback()
        raise e
    finally:
        conn.close()
```

The `FOR UPDATE` lock doesn't require you to use `SERIALIZABLE` — it's a surgical lock on a specific row, giving you the protection you need without the global overhead.

---

## ACID vs. BASE: A Quick Aside

It's worth knowing that ACID isn't the only paradigm. Many NoSQL and distributed databases (Cassandra, DynamoDB, Riak) use a different philosophy called **BASE**:

- **B**asically **A**vailable — the system appears to work most of the time
- **S**oft state — the state may change over time, even without input
- **E**ventually consistent — data will eventually converge to a consistent state

BASE is not a bug — it's a deliberate trade-off for systems that prioritize availability and horizontal scale over strict consistency. The CAP theorem (Consistency, Availability, Partition Tolerance) formalizes this: you can't have all three simultaneously in a distributed system.

For interview purposes: **ACID for relational, transactional systems. BASE for distributed, high-availability NoSQL systems.** Know the trade-offs for each.

---

## Interview Cheat Sheet

Here are the questions you're most likely to encounter, with the angles that interviewers are actually testing:

**"What is a database transaction?"**
→ A logical unit of work with ACID guarantees. All-or-nothing, consistent, isolated, durable. The canonical example is a bank transfer.

**"Explain the four isolation levels."**
→ Lead with the anomaly table. Describe READ UNCOMMITTED → SERIALIZABLE as a spectrum between performance and correctness. Mention your database's default.

**"What's the difference between a non-repeatable read and a phantom read?"**
→ Non-repeatable = same row, different value. Phantom = same query, different set of rows. One is about row mutation, the other is about row set mutation.

**"When would you use SERIALIZABLE isolation?"**
→ Financial transfers, ticket booking, inventory reservation — anything where anomalies have real monetary or safety consequences.

**"What is MVCC?"**
→ Multi-Version Concurrency Control. Multiple versions of rows allow readers and writers to never block each other. Each transaction reads from a snapshot appropriate to its isolation level.

**"What is the lost update problem and how do you fix it?"**
→ Two transactions read-modify-write the same row, and one overwrites the other. Fix: `SELECT FOR UPDATE` or use `SERIALIZABLE` isolation.

---

## Putting It All Together

We started at a bank branch, watching two tellers unknowingly destroy account integrity. We've now traced the full arc — from the atomic unit of a transaction, through the ACID contract, into the concurrency anomalies that make isolation necessary, across all four isolation levels with their trade-offs and implementations, down into the machinery of MVCC that powers it all.

The core insight to carry into every system design: **isolation is a dial, not a switch.** Every application has different risk tolerances, different performance requirements, and different anomaly budgets. The engineer's job isn't to always use `SERIALIZABLE` or always use `READ COMMITTED` — it's to understand *exactly what can go wrong* at each level and make a deliberate, informed choice.

When you're in the interview room and the whiteboard asks about a financial system or a booking platform, the language of transactions and isolation levels is how you demonstrate that you think like a systems engineer — not just a developer who writes queries.

```
  FINAL MENTAL MODEL

  Low Risk Domain          │   High Stakes Domain
  (Social, Analytics)      │   (Finance, Booking)
                           │
  READ UNCOMMITTED         │   SERIALIZABLE
         │                 │         │
  READ COMMITTED  ─────────┼─────────┤
         │                 │         │
  REPEATABLE READ          │   + SELECT FOR UPDATE
         │                 │         │
  ←──── Performance ──────►│◄─ Correctness ─────►
```

The best engineers don't memorize this table. They *understand the trade-offs deeply enough to derive it from first principles* — and then explain it clearly to anyone in the room.

```

***

This article  covers the full arc from transaction basics to ACID properties, all four isolation levels, and MVCC implementation details. The anomaly descriptions (dirty read, non-repeatable read, phantom read) align precisely with PostgreSQL's official documentation, and the isolation level trade-offs and database-specific defaults are sourced from CockroachDB's detailed analysis  and the OneUptime implementation guide.