# 02 — Scalability and Load Balancing

*Why adding machines stops working, and how traffic actually finds a server.*

[← back to the handbook](../README.md)

---

## 1. The laws that bound scaling

### 1.1 Amdahl's Law — the serial fraction caps you

If a fraction `s` of the work is inherently serial, the maximum speed-up from `N`
workers is:

```
speedup(N) = 1 / (s + (1 - s) / N)
```

As `N → ∞`, `speedup → 1/s`. A workload that is 5% serial can never exceed a 20×
speed-up, no matter how many machines you buy.

| Serial fraction | Ceiling |
|---|---|
| 1% | 100× |
| 5% | 20× |
| 10% | 10× |
| 25% | 4× |

In a distributed system the "serial fraction" is anything every request must pass
through: a shared primary database, a single ID generator, a global lock, a
coordination service, one leader.

### 1.2 The Universal Scalability Law — it gets worse

Amdahl is optimistic because it ignores the cost of workers *talking to each
other*. The USL adds a coherency term:

```
C(N) = N / (1 + σ(N − 1) + κN(N − 1))

  σ (sigma) = contention   — waiting for a shared resource
  κ (kappa) = coherency    — keeping nodes consistent with each other
```

```mermaid
flowchart LR
    A["N = 10<br/>throughput 8x"] --> B["N = 50<br/>throughput 22x"] --> C["N = 100<br/>throughput 25x<br/>PEAK"] --> D["N = 200<br/>throughput 21x<br/>WORSE than 100"]
    style C fill:#14532d,color:#fff
    style D fill:#7d1128,color:#fff
```

The κ term is quadratic, so **beyond some N, adding nodes reduces throughput**.
Anyone who has watched a database cluster get slower after adding a replica, or a
consensus group degrade as it grew, has seen κ dominate.

### 1.3 What to do about it

| Symptom | Cause | Fix |
|---|---|---|
| Throughput plateaus | σ — contention on a shared resource | Find the shared resource. Shard it, replicate it, or remove it from the hot path |
| Throughput **declines** with more nodes | κ — coordination cost | Reduce cross-node communication: partition so nodes are independent, batch coordination, use gossip instead of all-to-all |
| Latency rises but throughput is flat | Queueing | You are at saturation — add capacity or shed load |

The universal remedy is **independence**. Nodes that never need to agree scale
linearly. Every design decision that lets two nodes act without consulting each
other is a scaling decision.

### 1.4 Little's Law — the one you will use most

```
L = λ × W

  L = average number of requests in the system
  λ = arrival rate (requests/sec)
  W = average time in the system (sec)
```

Rearranged, it answers real questions:

- **How many threads do I need?** `threads = qps × latency`. 1,000 QPS at 50 ms
  needs 50 concurrent workers. At 500 ms it needs 500.
- **How deep will my queue get?** If arrivals exceed service rate by 100/s, depth
  grows by 100 every second. Forever.
- **What happens if latency triples?** Concurrency requirement triples. This is
  exactly how a slow dependency exhausts a connection pool.

### 1.5 Queueing theory, in one graph

Response time does not degrade linearly with utilisation — it degrades
*hyperbolically*, and the knee is much earlier than intuition suggests.

```
For an M/M/1 queue:  W = service_time / (1 − utilisation)

  50% utilisation → 2.0x service time
  70%             → 3.3x
  80%             → 5.0x
  90%             → 10x
  95%             → 20x
  99%             → 100x
```

```mermaid
flowchart LR
    U50["50% util<br/>2x latency"] --> U70["70%<br/>3.3x"] --> U80["80%<br/>5x"] --> U90["90%<br/>10x"] --> U95["95%<br/>20x"] --> U99["99%<br/>100x"]
    style U50 fill:#14532d,color:#fff
    style U70 fill:#14532d,color:#fff
    style U90 fill:#3b0d0d,color:#fff
    style U99 fill:#7d1128,color:#fff
```

**This is why you target 60–70% utilisation, not 95%.** The idle 30% is not waste;
it is the latency budget. It is also the capacity that absorbs a node failure
without cascading.

---

## 2. Load balancing in depth

### 2.1 Where balancing happens

```mermaid
flowchart TD
    C["Client"] --> D["1. DNS<br/>picks a region"]
    D --> A["2. Anycast / global LB<br/>picks a PoP"]
    A --> E["3. Edge L4 LB<br/>picks an ingress node"]
    E --> I["4. Ingress L7 LB<br/>picks a service"]
    I --> M["5. Mesh sidecar / client LB<br/>picks an instance"]
    M --> P["6. Instance<br/>picks a thread"]
    P --> DB["7. DB proxy<br/>picks a replica"]
    style D fill:#0b2545,color:#fff
    style M fill:#134e4a,color:#fff
```

Seven layers of load balancing in a single request is normal in a mature system.
Each has a different failure mode and a different rebalancing latency: DNS takes
minutes, anycast takes seconds, an L7 balancer takes milliseconds.

### 2.2 Algorithm behaviour under stress

The published comparisons of LB algorithms use uniform request costs. Real traffic
is not uniform, and that is where the differences appear.

| Scenario | Round robin | Least connections | Least response time | P2C |
|---|---|---|---|---|
| Uniform requests, uniform nodes | Optimal | Optimal | Optimal | Near-optimal |
| One node is 2× slower | Overloads it | Adapts well | Adapts well | Adapts well |
| One node **fails fast** (instant 500s) | Sends 1/N | **Sends everything** — no connections held | Adapts (errors counted) | Adapts if health-aware |
| Highly variable request cost | Poor | Good | Good | Good |
| 1,000-node fleet | Fine | Needs global state | Needs global state | **Fine — no global state** |

> **The "least connections attracts failure" trap is real.** A node returning
> instant errors has zero active connections, so a least-connections balancer
> routes *more* traffic to it. This is called a **black hole**, and it converts one
> broken node into a fleet-wide error rate. The defence is **outlier detection**:
> eject nodes whose *error rate* deviates from the fleet, independently of the
> health check.

### 2.3 Consistent hashing for affinity

When backends hold per-key state — a local cache, a WebSocket session, a
stream-processing partition — you want the same key to reach the same backend.

```mermaid
flowchart LR
    R["Request key: user:8821"] --> H["hash(key) → ring position"]
    H --> N["First node clockwise"]
    N --> B["Backend 3<br/>already has this user's data cached"]
    B --> HIT["Local cache hit"]
    style HIT fill:#14532d,color:#fff
```

The trade-off is **load imbalance**: consistent hashing distributes *keys* evenly,
not *load*, and one hot key can saturate its owner. **Bounded-load consistent
hashing** fixes this: if the target node is above a load threshold (say 1.25× the
mean), spill to the next node on the ring. You keep most of the affinity and cap
the damage.

### 2.4 Health checking done properly

```mermaid
flowchart TD
    subgraph checks["Three separate endpoints"]
        L["/livez — is the process functioning?<br/>Failure ⇒ RESTART"]
        R["/readyz — can it serve traffic now?<br/>Failure ⇒ REMOVE FROM POOL"]
        S["/startupz — has it finished warming up?<br/>Gates the other two during boot"]
    end
    subgraph danger["The dependency-check trap"]
        D1["/readyz checks the database"] --> D2["Database blips"] --> D3["ALL nodes report not-ready"] --> D4["LB has an empty pool"] --> D5["100% outage from a partial degradation"]
    end
    style danger fill:#7d1128,color:#fff
    style checks fill:#14532d,color:#fff
```

Rules that prevent the trap:

- **Minimum healthy percentage.** Never eject below a floor (commonly 50%). If more
  than half the fleet reports unhealthy, the check is more likely wrong than the
  fleet. Envoy calls this *panic mode* and it sends traffic to everyone.
- **Separate shallow from deep.** Shallow checks gate routing; deep checks emit
  metrics and page a human.
- **Passive health checking.** Use real traffic outcomes — consecutive 5xx, consecutive
  timeouts — rather than only synthetic probes. Real traffic tests the real path.
- **Slow start.** A newly healthy node with cold caches and an unoptimised JIT will
  fall over if it receives its full share instantly. Ramp over 30–60 seconds.

### 2.5 Connection draining

```mermaid
sequenceDiagram
    participant O as Orchestrator
    participant P as Pod
    participant LB as Load balancer
    participant C as In-flight client

    O->>P: SIGTERM
    P->>P: flip readiness to false
    Note over LB: next probe fails → remove from pool<br/>(this takes up to one probe interval)
    P->>P: keep serving in-flight requests
    Note over P,C: drain window — typically 30 s
    C->>P: existing request completes
    P->>P: close idle keep-alive connections
    P->>O: exit 0
    Note over O,P: if still alive after terminationGracePeriod → SIGKILL
```

The classic bug: the pod exits immediately on SIGTERM while the balancer still has
it in the pool, producing a burst of connection-refused errors on every deploy.
Two fixes, both needed — fail readiness **before** you stop accepting, and add a
small `preStop` sleep so the balancer's probe interval elapses before the process
begins shutting down.

---

## 3. Service discovery

```mermaid
flowchart TD
    subgraph cs["Client-side discovery"]
        C1["Client"] --> R1["Registry<br/>Consul / etcd / Eureka"]
        R1 -->|"list of healthy instances"| C1
        C1 -->|"client picks one"| S1["Instance"]
        N1["No extra hop. Smart clients needed<br/>in every language."]
    end
    subgraph ss["Server-side discovery"]
        C2["Client"] --> LB2["Load balancer / DNS name"]
        LB2 --> R2["Registry"]
        LB2 --> S2["Instance"]
        N2["Dumb clients. One extra hop.<br/>The LB is a shared component."]
    end
    subgraph mesh["Service mesh"]
        C3["Client"] --> SC3["Sidecar proxy"]
        SC3 --> CP["Control plane"]
        SC3 --> SC4["Peer sidecar"] --> S3["Instance"]
        N3["Uniform policy, mTLS, retries,<br/>telemetry. Real overhead per hop."]
    end
    style mesh fill:#134e4a,color:#fff
```

**Registration** is either self-registration (the instance announces itself and
heartbeats) or third-party registration (the orchestrator registers it on the
instance's behalf — what Kubernetes does). Third-party is more reliable because a
crashed instance cannot fail to deregister itself.

The subtlety that bites people: **deregistration is not instant**. Between a crash
and its removal from the registry there is a window — typically one or two
heartbeat intervals — during which clients will still be handed a dead address.
Client-side retry with a different instance is what covers that gap. Discovery
alone is not sufficient; discovery plus retry is.

---

## 4. Autoscaling

```mermaid
flowchart TD
    M["Metric"] --> D{"Which signal?"}
    D -->|"CPU"| C["Simple, lagging.<br/>Useless for I/O-bound services."]
    D -->|"RPS per instance"| R["Direct, predictable.<br/>Needs a known per-instance capacity."]
    D -->|"Queue depth / consumer lag"| Q["Best for workers.<br/>Directly measures the backlog."]
    D -->|"p99 latency"| L["Closest to what users feel.<br/>Noisy; scales up late."]
    D -->|"Schedule"| S["Best for known patterns.<br/>Combine with reactive as a floor."]
    style Q fill:#14532d,color:#fff
    style S fill:#14532d,color:#fff
```

The realities that make autoscaling harder than it looks:

- **Scaling is not instant.** Instance boot, image pull, JIT warm-up and cache
  priming can take minutes. A 60-second traffic spike is over before autoscaling
  responds. **Buffer capacity handles spikes; autoscaling handles trends.**
- **Scale up fast, scale down slow.** Aggressive scale-down causes thrashing, and
  the cost of one extra instance for ten minutes is trivial next to an outage.
- **Predictive beats reactive** where the pattern is known. Pre-scale before the
  daily peak, before the marketing email goes out, before the match starts.
- **The database usually cannot follow.** Scaling the stateless tier 5× while the
  database stays fixed just moves the queue. Autoscaling without a matching data
  tier plan is a way to overload your database faster.
- **Watch for scale-to-zero cold starts.** Excellent for cost, hostile to p99.

---

## 5. A practical scaling checklist

```
□ Identify every serialised resource on the request path
□ Compute Little's Law: threads = qps x latency, for each pool
□ Target 60-70% utilisation, not 90%
□ Confirm the LB algorithm handles a fast-failing node (outlier detection on)
□ Health checks: shallow gates routing, deep only alerts
□ Minimum-healthy-percentage floor configured
□ Slow start enabled for new instances
□ Graceful drain: readiness flipped before shutdown, preStop delay present
□ Retry on a DIFFERENT instance, with a retry budget
□ Autoscale on the signal that matches the workload, not on CPU by default
□ Confirm the data tier can absorb the scaled app tier
```

---

[← previous: The method](01-the-method.md) · [back to the handbook](../README.md) · [next: Caching and CDN →](03-caching-and-cdn.md)
