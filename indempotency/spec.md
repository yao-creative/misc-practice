**Intent: practice idempotency as a distributed-systems invariant, progressing from API semantics → concurrency → persistence → failure recovery.**

I’d do these in order. **Don’t implement a generic idempotency library first**; each problem should force you to discover one invariant.

### Level 1 — Semantics

**1. Payment retry**

Implement:

```python
POST /payments
Idempotency-Key: K
amount=100
```

Requirements:

* First `K` executes the payment.
* Repeated `K` returns the original result.
* Different keys create different payments.
* A client timeout followed by retry must not create two payments.

**Question:** What exactly does “same operation” mean?

---

**2. Key collision**

Suppose:

```text
K = abc

Request 1: charge $100
Request 2: charge $500
```

Both use `K`.

Design the behavior.

You should discover that:

$$
K \mapsto \text{request fingerprint}
$$

is necessary, not merely:

$$
K \mapsto \text{response}
$$

---

### Level 2 — Concurrent requests

**3. Two requests arrive simultaneously**

Two workers receive:

```text
Request A: key=K
Request B: key=K
```

at exactly the same time.

Naive implementation:

```python
if key not in store:
    result = perform_payment()
    store[key] = result
```

Find the race condition.

Then design a state machine:

$$
\text{Absent}
\rightarrow
\text{InProgress}
\rightarrow
\text{Completed}
$$

What should the second request do while `K` is `InProgress`?

---

**4. Crash during execution**

Now:

```text
K → InProgress
     ↓
perform_payment()
     ↓
PROCESS CRASH
```

The database contains:

```text
K = InProgress
```

but you don't know whether the payment actually happened.

Design the recovery semantics.

This is the first **really important** problem.

---

### Level 3 — Database transactions

**5. Idempotency + database**

You have:

```text
idempotency_keys
----------------
key
request_hash
status
response

payments
--------
payment_id
user_id
amount
```

Implement:

```python
create_payment(request, key)
```

with the invariant:

$$
\forall K,\quad
|\operatorname{payments}(K)| \leq 1
$$

Your implementation must remain correct with two concurrent processes.

Think carefully about:

* unique constraints
* transaction boundaries
* isolation
* commit ordering

---

**6. The external side effect problem**

Your database transaction does:

```text
BEGIN
    INSERT idempotency_key
COMMIT

charge_stripe()
```

The process crashes between the two.

Now:

```text
idempotency key exists
payment doesn't know whether Stripe charged
```

Redesign the architecture.

This should lead you toward **transactional outbox / durable state machines / external idempotency**.

---

### Level 4 — TTL and distributed systems

**7. Expiring keys**

You retain idempotency keys for 24 hours.

After:

```text
t = 0      POST K → payment created
t = 24h    K expires
t = 25h    POST K → ?
```

What semantics do you want?

Define precisely what guarantee your TTL provides.

It is **not**:

$$
\text{“K can only ever execute once.”}
$$

It is closer to:

$$
\text{“K can only be reused within retention window } T\text{.”}
$$

---

**8. Redis implementation**

Implement an idempotency store using Redis-like primitives:

```python
SET key value NX
GET key
```

Handle:

```text
Worker A ──┐
           ├── K
Worker B ──┘
```

Then answer:

> Is `SET NX` alone sufficient to implement the entire idempotency protocol?

---

### Level 5 — Production design

**9. HTTP API contract**

Design the complete semantics for:

```http
POST /orders
Idempotency-Key: 7f91...
```

Specify behavior for:

| Situation                   | Response |
| --------------------------- | -------- |
| first request               | ?        |
| exact retry                 | ?        |
| same key, different body    | ?        |
| request currently executing | ?        |
| previous execution failed   | ?        |
| server crashed              | ?        |
| key expired                 | ?        |
| malformed key               | ?        |

You're essentially defining an **algebra of observable request outcomes**.

---

### Level 6 — Hard problem

**10. Exactly-once illusion**

Build:

```text
Client
  ↓
API
  ↓
Idempotency layer
  ↓
Database
  ↓
Message queue
  ↓
Payment worker
  ↓
External payment provider
```

Requirement:

> The client must observe one logical payment even if every component can retry, crash, or process messages more than once.

Define the invariants at **each boundary**.

The key insight you're trying to reach is:

$$
\text{Exactly-once business semantics}
\neq
\text{exactly-once message delivery}
$$

Instead, you construct the former from **at-least-once execution + deduplication + durable identity + transactional boundaries**.

---

### Recommended implementation progression

```text
1. in-memory dict
       ↓
2. concurrent workers
       ↓
3. SQLite + UNIQUE(key)
       ↓
4. crash recovery
       ↓
5. Redis
       ↓
6. queue + worker
       ↓
7. external side effect
       ↓
8. full payment/order API
```

For your style of modeling, **#3 → #6 is the sweet spot**: it forces you to model idempotency as a transition system rather than treating it as “just cache the response.”
