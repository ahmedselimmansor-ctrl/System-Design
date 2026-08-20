# 08 — Resilience and Failure

*Timeouts, retries, breakers, bulkheads, shedding, and the failure modes that surprise people.*

[← back to the handbook](../README.md)

---

## 1. The premise

> **Everything fails, all the time.** The question is never whether a component will
> fail, but whether its failure is contained.

Resilience engineering is the discipline of ensuring that the failure of any one
component produces **degradation proportional to that component's importance** —
and nothing more. The alternative, which is the default without deliberate design,
is that any failure anywhere produces total failure everywhere.

---

## 2. Timeouts

### 2.1 Choosing the value

```mermaid
flowchart TD
    M["Measure the dependency's<br/>actual latency distribution"] --> P["Set timeout ≈ p99.9 + margin"]
    P --> W1["Too short: you abandon requests<br/>that would have succeeded,<br/>and add retry load"]
    P --> W2["Too long: threads are held<br/>during an outage, and your<br/>own latency inherits theirs"]
    P --> C["Then verify: does the caller's<br/>deadline actually allow this?"]
    style P fill:#14532d,color:#fff
```

The frequent mistake is a single global timeout. A 30-second default applied to a
call whose p99 is 40 ms means that during a dependency failure, each request holds
a thread for 30 seconds. At 1,000 QPS with a 200-thread pool, the pool is exhausted
in 0.2 seconds and the service is completely down — because of a dependency that
was merely slow.

### 2.2 Deadline propagation

```mermaid
sequenceDiagram
    participant U as User
    participant G as Gateway
    participant A as Service A
    participant B as Service B
    participant D as Database

    U->>G: request (client deadline 3000 ms)
    G->>G: deadline = now + 2900 ms (reserve 100 ms for own work)
    G->>A: call, grpc-timeout: 2900m
    A->>A: 400 ms of own work → 2500 ms remain
    A->>B: call, grpc-timeout: 2500m
    B->>B: 200 ms → 2300 ms remain
    B->>D: query with statement_timeout = 2300 ms
    Note over D: if the query needs 4 s, it is cancelled at 2.3 s —<br/>no work is done that the caller has already abandoned
```

Deadline propagation is the difference between a service that wastes capacity on
dead requests and one that does not. Without it, a timed-out user request continues
consuming database CPU for another 30 seconds — during an overload, exactly when
you can least afford it.

gRPC propagates deadlines natively. For HTTP, pass a header (`X-Request-Deadline`
or the standard `Deadline`) and honour it.

---

## 3. Retries done correctly

### 3.1 The backoff formula

```
# Full jitter — the recommended default
delay = random(0, min(cap, base * 2^attempt))

# Decorrelated jitter — better throughput under contention
delay = min(cap, random(base, previous_delay * 3))
```

```mermaid
flowchart TD
    subgraph nojit["Without jitter"]
        N1["1000 clients fail at t=0"] --> N2["All retry at t=1s"] --> N3["All fail again"] --> N4["All retry at t=2s"] --> N5["Synchronised waves forever"]
    end
    subgraph jit["With full jitter"]
        J1["1000 clients fail at t=0"] --> J2["Retries spread uniformly over 0-1s"] --> J3["Load is smooth; the service<br/>recovers under partial load"]
    end
    style nojit fill:#7d1128,color:#fff
    style jit fill:#14532d,color:#fff
```

### 3.2 Retry budgets — the thing that actually stops storms

Backoff spreads retries in time but does not bound their **total volume**. A retry
budget does:

```
allowed_retries = max(min_retries_per_sec, retry_ratio × successful_requests_per_sec)

typical: retry_ratio = 0.1  (retries may add at most 10% to the load)
```

When the budget is exhausted, retries are **abandoned immediately** — the request
fails fast rather than adding load. This is what breaks the feedback loop: as the
dependency's success rate falls, the retry budget falls with it, so a service in
trouble receives *less* extra traffic, not more.

```mermaid
flowchart LR
    S["Success rate falls to 20%"] --> B["Budget = 0.1 x successes<br/>= a much smaller absolute number"]
    B --> R["Retries are suppressed"]
    R --> L["Load on the failing service drops"]
    L --> REC["It has room to recover"]
    style REC fill:#14532d,color:#fff
```

### 3.3 Where to retry

```mermaid
flowchart TD
    C["Client"] -->|"3 attempts"| G["Gateway"] -->|"3 attempts"| A["Service A"] -->|"3 attempts"| B["Service B"]
    B --> N["A single user request becomes<br/>27 requests to Service B.<br/>During a partial outage this is<br/>the amplification that kills it."]
    style N fill:#7d1128,color:#fff
```

**Retry at exactly one layer.** The usual correct choice is the layer closest to the
failure that still has enough context to know the operation is idempotent — often
the immediate caller. Everything above it should fail fast and let the user decide.

---

## 4. Circuit breakers in practice

### 4.1 Configuration that works

| Parameter | Sensible starting point | Notes |
|---|---|---|
| Failure threshold | 50% over a rolling window | Percentage, not a count — a count is meaningless at varying traffic |
| Minimum request volume | 20 requests in the window | Prevents tripping on 1 failure out of 2 during low traffic |
| Rolling window | 10 seconds | Long enough to be stable, short enough to react |
| Open duration | 30 seconds | Then probe |
| Half-open probes | 1–5 concurrent | Too many and you re-overload the recovering service |
| Success threshold to close | 5 consecutive successes | Avoid closing on a single lucky request |

### 4.2 What counts as a failure

This matters more than the thresholds. **A 404 is not a failure of the dependency**
— it is a correct answer. Counting business-level errors as circuit failures means
a burst of legitimate "not found" responses will trip the breaker and take down a
healthy dependency.

| Count as failure | Do not count |
|---|---|
| Connection refused / reset | 400, 401, 403, 404, 422 |
| Timeout | 409 conflict |
| 500, 502, 503, 504 | Any application-level "no" |
| TLS handshake failure | Validation errors |
| 429 (debatable — usually count as a *separate* signal that triggers throttling, not the breaker) | |

### 4.3 Fallbacks

An open circuit still has to return something. Design the fallback deliberately:

```mermaid
flowchart TD
    O["Circuit open"] --> F{"Fallback strategy"}
    F --> F1["Cached / stale value<br/>best when available"]
    F --> F2["Static default<br/>e.g. non-personalised bestsellers"]
    F --> F3["Reduced functionality<br/>hide the section entirely"]
    F --> F4["Queue for later<br/>if the operation is deferrable"]
    F --> F5["Explicit error<br/>correct when a wrong answer is worse"]
    F5 --> N["For payments, permissions and inventory,<br/>fail closed. A wrong 'yes' is worse<br/>than an honest 'try again'."]
    style F1 fill:#14532d,color:#fff
    style N fill:#7d1128,color:#fff
```

---

## 5. Isolation

### 5.1 Bulkhead sizing

```mermaid
flowchart TD
    T["Total capacity: 300 threads"] --> A["Critical dependency A: 150"]
    T --> B["Important dependency B: 90"]
    T --> C["Optional dependency C: 30"]
    T --> R["Reserve, unallocated: 30"]
    R --> N["The reserve absorbs bursts and<br/>keeps health checks answerable<br/>even when every pool is saturated"]
    style R fill:#14532d,color:#fff
```

Size each pool from `Little's Law`: `threads = expected_qps × p99_latency`. Then
verify that the sum plus reserve does not exceed what the machine can actually run
— 1,000 threads on 4 cores is not isolation, it is context-switch thrashing.

### 5.2 Cells and shuffle sharding

```mermaid
flowchart TD
    subgraph plain["Plain cells — 8 cells"]
        P1["Tenant → cell 3"]
        P2["Cell 3 fails ⇒ 1/8 of tenants<br/>are completely down"]
    end
    subgraph shuffle["Shuffle sharding — 8 cells, 2 per tenant"]
        S1["Tenant A → cells {1,4}"]
        S2["Tenant B → cells {1,7}"]
        S3["Tenant C → cells {2,5}"]
        S4["Cell 1 fails ⇒ A and B lose<br/>HALF their capacity, not all of it.<br/>C is unaffected."]
        S5["C(8,2) = 28 distinct assignments.<br/>A poison tenant damages only<br/>the ~1/28 who share BOTH its cells."]
    end
    style shuffle fill:#14532d,color:#fff
    style plain fill:#3b0d0d,color:#fff
```

The combinatorics are the point: with `C(n,k)` possible assignments, the probability
that another tenant shares *all* of a given tenant's cells falls off dramatically.
Eight cells with two per tenant already isolates any single bad actor from ~96% of
customers. This is how AWS limits the blast radius of a poison request in shared
services.

---

## 6. Load shedding

### 6.1 Shedding beats queueing

```mermaid
flowchart LR
    subgraph q["Accept everything"]
        Q1["Queue grows"] --> Q2["Latency grows"] --> Q3["Clients time out"] --> Q4["Work is completed for<br/>requests nobody is waiting for"] --> Q5["Effective throughput → 0"]
    end
    subgraph s["Shed early"]
        S1["Reject above capacity in ~1 ms"] --> S2["Accepted requests stay fast"] --> S3["Throughput stays at capacity"] --> S4["Clients get a clear, fast 503<br/>and can back off"]
    end
    style q fill:#7d1128,color:#fff
    style s fill:#14532d,color:#fff
```

**Congestive collapse** is the formal name for the left-hand path: a system doing
maximum work and delivering zero value, because every completed request has already
been abandoned by its caller. Shedding is the only exit.

### 6.2 Deciding what to shed

```mermaid
flowchart TD
    R["Incoming request"] --> C{"Current load"}
    C -->|"&lt; 70%"| A["Accept everything"]
    C -->|"70-85%"| S1["Shed priority 3<br/>prefetch, analytics, A/B events"]
    C -->|"85-95%"| S2["Shed priority 2<br/>recommendations, related items,<br/>non-essential enrichment"]
    C -->|"&gt; 95%"| S3["Shed priority 1<br/>browse, search<br/>KEEP: login, cart, checkout, payment"]
    style S3 fill:#7d1128,color:#fff
    style A fill:#14532d,color:#fff
```

Implementing this requires that requests **carry a priority**, assigned at the edge
from the endpoint and the user's context. Retries should be shed *before* first
attempts, and requests whose deadline has already passed should be dropped without
being processed at all — checking the deadline at dequeue time is one of the
cheapest, highest-value pieces of overload protection available.

### 6.3 Adaptive concurrency limits

Rather than a fixed limit, infer capacity continuously from observed latency — the
same idea as TCP congestion control:

```
if latency is rising → reduce the concurrency limit
if latency is stable and the limit is saturated → increase it
```

This adapts automatically to changing instance sizes, dependency performance, and
request mix, and removes the perpetually-stale "max connections" constant that
nobody has revisited since the service was written.

---

## 7. The failure modes that surprise people

### 7.1 Grey failure

```mermaid
flowchart TD
    G["A node is 'up' but degraded:<br/>10x latency, 5% error rate,<br/>failing only certain requests"] --> D["Health check passes —<br/>it returns 200 on /healthz"]
    D --> L["Load balancer keeps sending traffic"]
    L --> P["Every caller's p99 is poisoned"]
    P --> C["Callers' thread pools fill"]
    C --> W["A partial degradation<br/>becomes a total outage"]
    style W fill:#7d1128,color:#fff
```

Grey failure is harder than crash failure precisely because every automatic
mechanism designed for crashes fails to notice it. The defences:

- **Outlier detection on real traffic** — eject a node whose error rate or latency
  deviates significantly from its peers, independently of health checks.
- **Client-side hedging** — for read-only idempotent requests, send a second request
  to a different backend if the first has not answered by p95, and take whichever
  returns first. This caps tail latency at roughly p95 for a few percent extra load.
- **Deep health checks that alert**, even where they must not evict.

### 7.2 Metastable failure

```mermaid
flowchart TD
    T["Trigger: brief spike / deploy / cache flush"] --> O["System enters overload"]
    O --> R["Retries amplify the load"]
    R --> O
    O --> C["Caches stay cold because<br/>nothing completes"]
    C --> O
    T -.->|"trigger is REMOVED"| S["System stays broken"]
    S --> N["Capacity is now insufficient because of the<br/>work created by being overloaded.<br/>Adding servers often does not help."]
    style S fill:#7d1128,color:#fff
    style N fill:#7d1128,color:#fff
```

Recognising metastability matters because the instinctive response — add capacity —
frequently fails. The exit is to **remove load**: shed aggressively, drain queues,
disable retries, warm caches, then readmit traffic gradually. Some systems need a
full restart with traffic held off, which is unpalatable but faster than the
alternative.

### 7.3 Correlated failure

Redundancy assumes independence. Real systems violate it constantly:

| Assumed independent | Actually shares |
|---|---|
| Three replicas | The same rack, PDU, or top-of-rack switch |
| Three availability zones | The same region's control plane |
| A whole fleet | One config push, one bad deploy, one expired certificate |
| Multiple services | One DNS provider, one identity provider, one CDN |
| Primary and backup | The same buggy software version |
| N instances | The same leap-second bug, the same certificate expiry date |

**Certificate expiry** deserves a special mention: it is perfectly correlated across
every instance using that certificate, it is scheduled in advance, and it is one of
the most common causes of large multi-service outages. Monitor expiry as a
first-class metric with a 30-day alert.

---

## 8. Testing for failure

```mermaid
flowchart LR
    U["Unit tests<br/>logic"] --> I["Integration tests<br/>contracts"] --> L["Load tests<br/>capacity limits"] --> F["Fault injection<br/>latency, errors, partitions"] --> G["Game days<br/>humans + runbooks"] --> C["Continuous chaos<br/>in production, bounded"]
    style G fill:#14532d,color:#fff
    style C fill:#4a044e,color:#fff
```

Start where the value is highest per unit of effort:

1. **Inject latency, not just errors.** Most systems handle a dependency returning
   500 correctly and handle it taking 30 seconds catastrophically.
2. **Kill an instance during a deploy.** This is where draining bugs live.
3. **Fill a disk. Exhaust a connection pool. Expire a certificate in staging.**
   These are the boring failures that cause real outages.
4. **Run a game day with the runbook you actually have.** The gap between the
   runbook and reality is the thing you are testing.
5. **Only then automate continuous chaos**, and only with a defined blast radius and
   an automatic stop condition.

---

## 9. Checklist

```
□ Every network call has an explicit timeout derived from measured p99
□ Deadlines propagate across hops; work stops when the caller has given up
□ Retries: idempotent operations only, transient failures only
□ Backoff has full jitter; attempts are capped
□ A retry budget exists and is enforced (retries ≤ ~10% of traffic)
□ Retry happens at exactly one layer
□ Circuit breakers use failure PERCENTAGE with a minimum volume
□ Business errors (4xx) do not trip breakers
□ Every breaker has a defined fallback, and fail-closed where correctness demands
□ Bulkheads isolate dependency pools; sizes derived from Little's Law
□ Load shedding exists, is priority-aware, and rejects in ~1 ms
□ Requests past their deadline are dropped at dequeue, not processed
□ Outlier detection is enabled for grey failure
□ Certificate expiry is monitored with 30-day lead time
□ Failure injection includes LATENCY, not only errors
□ Game days run on a schedule, against the real runbook
```

---

[← previous: Distributed transactions and consensus](07-distributed-transactions-and-consensus.md) · [back to the handbook](../README.md) · [next: Observability and operations →](09-observability-and-operations.md)
