# 06 — Microservices and APIs

*Decomposition, domain boundaries, protocols, versioning, and the shape of a good API.*

[← back to the handbook](../README.md)

---

## 1. The honest case for and against

Microservices solve **organisational** problems primarily and technical problems
secondarily. If your team is eight people deploying twice a week, microservices
will cost you more than they return.

| Genuine reason to split | Not a reason to split |
|---|---|
| Two teams block each other on every deploy | The codebase "feels big" |
| One component needs 50× the instances of the rest | It would be more "modern" |
| One component needs a different language or runtime | A blog post recommended it |
| A component must be isolated for compliance (PCI, HIPAA) | The directory structure is messy |
| A component's failure must not affect the rest | We might need to scale someday |
| Radically different change velocity | It will look good in the architecture diagram |

```mermaid
flowchart LR
    S["Team size"] --> A["1-8 engineers<br/>Monolith. Full stop."]
    S --> B["8-30<br/>Modular monolith, maybe 2-4 services<br/>around genuinely different profiles"]
    S --> C["30-100<br/>Services per team boundary"]
    S --> D["100+<br/>Services, platform team, mesh,<br/>internal developer platform"]
    style A fill:#14532d,color:#fff
    style D fill:#4a044e,color:#fff
```

---

## 2. Finding boundaries with domain-driven design

### 2.1 Bounded contexts

The same word means different things in different parts of a business, and **each
distinct meaning is a boundary**.

```mermaid
flowchart TD
    subgraph sales["Sales context"]
        S["Customer =<br/>name, company, deal stage,<br/>lead score, contact history"]
    end
    subgraph billing["Billing context"]
        B["Customer =<br/>billing address, tax id,<br/>payment methods, invoices"]
    end
    subgraph support["Support context"]
        SU["Customer =<br/>tickets, entitlements,<br/>SLA tier, satisfaction"]
    end
    subgraph shipping["Shipping context"]
        SH["Customer =<br/>delivery address, preferences,<br/>access instructions"]
    end
    ID["Shared identity: customer_id"] -.-> S & B & SU & SH
    N["One 'Customer' table with 80 columns<br/>serving all four is the coupling<br/>you are trying to escape."]
    style N fill:#7d1128,color:#fff
    style ID fill:#0b2545,color:#fff
```

Each context owns its own model. They share only an **identifier**, and they learn
about each other through **events**, not through joins.

### 2.2 The aggregate rule

An **aggregate** is a cluster of objects treated as one unit for consistency — an
Order with its OrderLines, an Account with its Entries. Two rules follow:

1. **One transaction, one aggregate.** If a use case must atomically modify two
   aggregates, either they are actually one aggregate, or the consistency between
   them must become eventual (a saga).
2. **Reference other aggregates by id, never by object.** This is what makes them
   separable later.

Aggregates are the natural granularity of a service. A service that owns whole
aggregates has local transactions; a service that owns half an aggregate has a
distributed transaction problem forever.

### 2.3 Testing a proposed boundary

```mermaid
flowchart TD
    P["Proposed service"] --> T1{"Does it own its data<br/>exclusively?"}
    T1 -->|"no"| F["✗ Not a boundary — it is a layer"]
    T1 -->|"yes"| T2{"Can a typical feature ship<br/>touching only this service?"}
    T2 -->|"no"| F2["✗ Wrong seam — the boundary<br/>cuts through a use case"]
    T2 -->|"yes"| T3{"Can you describe it in one<br/>sentence without 'and'?"}
    T3 -->|"no"| F3["✗ Too broad — split it"]
    T3 -->|"yes"| T4{"Does it have &gt; ~3 synchronous<br/>dependencies to do its job?"}
    T4 -->|"yes"| F4["✗ Too coupled — its availability<br/>is the product of four others"]
    T4 -->|"no"| OK["✓ Plausible boundary"]
    style OK fill:#14532d,color:#fff
    style F fill:#7d1128,color:#fff
```

---

## 3. Inter-service data

The hardest part of microservices is not the network. It is that **service B needs
data that service A owns.**

```mermaid
flowchart TD
    N["Service B needs data owned by A"] --> O{"Options"}
    O --> O1["1. Synchronous call to A"]
    O --> O2["2. Replicate A's data into B<br/>via events"]
    O --> O3["3. Move the logic into A"]
    O --> O4["4. Merge A and B —<br/>the boundary was wrong"]

    O1 --> C1["Always fresh.<br/>B's availability now depends on A's.<br/>Latency adds up. Cascades propagate."]
    O2 --> C2["B is independently available.<br/>Data is eventually consistent.<br/>B must handle staleness and gaps."]
    O3 --> C3["No new coupling.<br/>A grows; may be correct if the<br/>logic genuinely belongs to A's domain."]
    O4 --> C4["Sometimes the right answer.<br/>Merging services is not a defeat."]
    style O2 fill:#134e4a,color:#fff
    style O4 fill:#14532d,color:#fff
```

**Data replication via events** is the pattern that makes microservices actually
work. Service B subscribes to `CustomerUpdated` and keeps a local read-only
projection of the fields it needs. B can then serve requests when A is completely
down. The costs are real and must be accepted explicitly: the projection is stale
by the event lag, B must handle events arriving out of order (use version numbers),
and B must be able to rebuild the projection from a replay.

### 3.1 The forbidden pattern

```mermaid
flowchart LR
    A["Service A"] --> DB[("Shared database")]
    B["Service B"] --> DB
    C["Service C"] --> DB
    DB --> N["Any schema change requires<br/>coordinating three deploys.<br/>You have a distributed monolith:<br/>all the cost, none of the benefit."]
    style N fill:#7d1128,color:#fff
```

There is exactly one legitimate variant: **one writer, many readers with a
read-only replica and a stable, versioned view**. Even then, the view is a
published API contract and must be treated with the same discipline as an HTTP
endpoint.

---

## 4. Protocol selection in practice

### 4.1 gRPC vs REST for internal traffic

| | REST/JSON | gRPC/Protobuf |
|---|---|---|
| Payload size | Baseline | 30–60% smaller |
| Serialisation cost | High (text parsing) | Low (binary) |
| Schema | Optional (OpenAPI, often stale) | Mandatory, compiled |
| Client generation | Add-on | Built in, all languages |
| Streaming | SSE or WebSocket | Native, bidirectional |
| Load balancing | Per-request (easy) | Per-connection — **needs L7 or client-side LB** |
| Debuggability | curl | grpcurl, less convenient |
| Browser support | Native | Needs grpc-web + proxy |

The gRPC gotcha worth stating: **HTTP/2 multiplexes many requests over one
long-lived connection**, so a naive L4 balancer pins all of a client's traffic to
one backend and your load distribution collapses. You need either an L7-aware proxy
(Envoy) or client-side load balancing with periodic connection recycling.

### 4.2 GraphQL — where it wins and where it hurts

```mermaid
flowchart TD
    subgraph win["Where GraphQL wins"]
        W1["Many client types with<br/>different field needs"]
        W2["Deeply nested data<br/>fetched in one round trip"]
        W3["Rapidly evolving frontends<br/>without backend changes"]
    end
    subgraph hurt["Where it hurts"]
        H1["HTTP caching — POST bodies<br/>are not cacheable by URL"]
        H2["N+1 resolvers — one query<br/>can trigger thousands of DB calls"]
        H3["Query cost attacks —<br/>deeply nested queries as a DoS"]
        H4["Observability — every request<br/>is 'POST /graphql'"]
    end
    hurt --> FIX["Mitigations: DataLoader batching,<br/>persisted queries (also restores caching),<br/>query depth and complexity limits,<br/>operation name in metrics"]
    style win fill:#14532d,color:#fff
    style hurt fill:#7d1128,color:#fff
    style FIX fill:#0b2545,color:#fff
```

**Persisted queries** solve two problems at once: the client sends a hash instead of
a query string (so it is a cacheable GET), and the server only accepts queries from
a known allowlist (so query-cost attacks become impossible). If you run GraphQL in
production and are not using persisted queries, that is the highest-value change
available.

---

## 5. API design details that matter

### 5.1 Pagination — why keyset wins

```mermaid
flowchart TD
    subgraph off["OFFSET 100000 LIMIT 20"]
        O1["Database must locate and discard<br/>100,000 rows before returning 20"]
        O2["Cost grows linearly with depth"]
        O3["Rows inserted during paging<br/>cause skipped or repeated items"]
    end
    subgraph key["WHERE (created_at, id) &lt; (:ts, :id) ORDER BY ... LIMIT 20"]
        K1["Index seek straight to the position"]
        K2["Constant cost at any depth"]
        K3["Stable under concurrent inserts"]
        K4["Cannot jump to 'page 500'"]
    end
    style off fill:#7d1128,color:#fff
    style key fill:#14532d,color:#fff
```

Encode the cursor as an opaque base64 blob so clients cannot construct one and you
can change its internals later. Always include the sort key **plus a tiebreaker**
(usually the primary key) — sorting on a non-unique column alone produces
duplicates and gaps at page boundaries.

### 5.2 Partial responses and expansion

```
GET /orders/42?fields=id,total,status          → sparse fieldset
GET /orders/42?expand=customer,items.product   → controlled embedding
```

This gives most of GraphQL's over-fetching benefit with none of its infrastructure.
Cap the expansion depth and the field count, or you have built the same DoS surface.

### 5.3 Long-running operations

```mermaid
sequenceDiagram
    participant C as Client
    participant A as API

    C->>A: POST /exports  {range, format}
    A-->>C: 202 Accepted<br/>Location: /exports/8821<br/>{status: "pending"}
    loop poll with backoff, or use a webhook
        C->>A: GET /exports/8821
        A-->>C: 200 {status: "running", progress: 0.4}
    end
    C->>A: GET /exports/8821
    A-->>C: 200 {status: "done", result_url: "https://..."}
```

Return a **resource that represents the operation**, not a job id in an ad-hoc
shape. The operation resource can then carry progress, errors, timing, and a result
link, and it is cancellable via `DELETE`.

### 5.4 Versioning without pain

```mermaid
flowchart LR
    A["Additive change<br/>new optional field,<br/>new endpoint,<br/>new enum value*"] --> S["Safe — no version bump"]
    B["Breaking change<br/>remove field, rename,<br/>change type, tighten validation,<br/>change default behaviour"] --> V["New version required"]
    V --> P["Run both versions in parallel"]
    P --> M["Measure usage per version"]
    M --> D["Deprecate with a date,<br/>notify by client id,<br/>then sunset"]
    style S fill:#14532d,color:#fff
    style B fill:#7d1128,color:#fff
```

\* New enum values are only safe if clients were told to tolerate unknown values.
Say so in the contract on day one; retrofitting it is not possible.

**Never version for the sake of it.** Every live version is a maintenance burden
multiplied by every downstream service. Two versions is normal; five means you have
no deprecation process.

---

## 6. The gateway and the BFF

```mermaid
flowchart TD
    subgraph clients["Clients"]
        W["Web"]
        M["Mobile"]
        P["Partners"]
    end
    subgraph edge["Edge"]
        GW["API gateway<br/>CROSS-CUTTING ONLY:<br/>TLS, authn, rate limit, routing,<br/>request id, metrics"]
    end
    subgraph bff["BFFs — owned by client teams"]
        BW["Web BFF<br/>aggregates for web screens"]
        BM["Mobile BFF<br/>smaller payloads, fewer round trips"]
    end
    subgraph svc["Domain services"]
        S1["Orders"]
        S2["Catalogue"]
        S3["Pricing"]
    end
    W --> GW --> BW --> S1 & S2 & S3
    M --> GW --> BM --> S1 & S2 & S3
    P --> GW --> S1
    style GW fill:#0b2545,color:#fff
    style bff fill:#134e4a,color:#fff
```

The division of responsibility is the point:

- **Gateway** — owned by the platform team, contains **no business logic**, changes
  rarely.
- **BFF** — owned by the client team, contains presentation-shaped aggregation,
  changes with the UI, and can be deployed without coordinating with anyone.

Without the BFF, aggregation logic accumulates in the gateway, and the gateway
becomes a shared bottleneck that every team must queue behind — the exact
organisational problem microservices were meant to remove.

---

## 7. Service mesh — when it earns its cost

```mermaid
flowchart LR
    subgraph without["Without a mesh"]
        L["Every service implements:<br/>retries, timeouts, circuit breaking,<br/>mTLS, tracing, load balancing"]
        L --> N1["In every language.<br/>Inconsistently.<br/>Updated never."]
    end
    subgraph with["With a mesh"]
        S["Sidecar proxy does it uniformly"]
        S --> N2["+ One policy, all languages<br/>+ mTLS everywhere by default<br/>+ Consistent telemetry<br/>− 1-5 ms added latency per hop<br/>− Memory per pod<br/>− A complex control plane to operate"]
    end
    style with fill:#134e4a,color:#fff
```

A mesh pays for itself somewhere around **20–30 services or 3+ languages**. Below
that, a shared client library gives you most of the benefit for a fraction of the
operational cost. Above it, the library approach fails because you cannot get every
team to upgrade.

---

## 8. Checklist

```
□ Split justified by a named organisational or technical need, not by aesthetics
□ Each service owns its data exclusively — no shared tables
□ Boundaries follow bounded contexts and aggregates, not technical layers
□ Cross-service data need answered deliberately: call, replicate, or move the logic
□ Synchronous dependency depth kept shallow; availability product computed
□ Protocol chosen per use case; gRPC LB is L7 or client-side
□ Pagination is cursor-based with a tiebreaker
□ Errors use a stable machine-readable shape with a request id
□ Idempotency keys accepted on all unsafe, retryable operations
□ Versioning policy written down; additive changes preferred; deprecation has dates
□ Gateway holds no business logic; aggregation lives in a BFF
□ Every service can start when its dependencies are down (degraded, not crashed)
```

---

[← previous: Messaging and async](05-messaging-and-async.md) · [back to the handbook](../README.md) · [next: Distributed transactions and consensus →](07-distributed-transactions-and-consensus.md)
