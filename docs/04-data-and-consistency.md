# 04 — Data and Consistency

*CAP properly stated, PACELC, the consistency hierarchy, session guarantees, and CRDTs.*

[← back to the handbook](../README.md)

---

## 1. CAP, stated precisely

The popular version — "pick two of three" — is wrong in a way that causes real
design errors. The precise statement:

> **In an asynchronous network where messages may be lost, it is impossible for a
> read/write register to be simultaneously available and linearisable.**

Three corrections follow:

1. **Partition tolerance is not a choice.** Networks drop packets. A system that is
   not partition-tolerant is a system that is broken during a partition, which is
   not an architecture.
2. **The choice only applies during a partition.** A healthy CP system is available.
   A healthy AP system is consistent-ish. CAP says nothing about normal operation.
3. **"Consistency" in CAP means linearisability specifically** — not the C in ACID,
   which is about integrity constraints. They are unrelated concepts that share a
   letter.

```mermaid
flowchart TD
    N["Network is healthy"] --> B["Both C and A are achievable.<br/>CAP has nothing to say."]
    P["Partition occurs"] --> C{"A client on the minority side<br/>issues a write"}
    C -->|"CP: refuse it"| CP["Consistency preserved.<br/>That client sees an error."]
    C -->|"AP: accept it"| AP["Availability preserved.<br/>Two divergent histories now exist.<br/>Reconciliation is now YOUR problem."]
    style B fill:#14532d,color:#fff
    style CP fill:#0b2545,color:#fff
    style AP fill:#134e4a,color:#fff
```

### 1.1 The proof, in three lines

Two nodes N1 and N2, both holding `x = 0`, partitioned from each other.

- A client writes `x = 1` to N1. N1 cannot tell N2.
- A client reads `x` from N2.
- N2 must either return `0` (**not linearisable** — it missed a completed write) or
  refuse to answer (**not available**). There is no third option.

---

## 2. PACELC — the formulation to actually use

> **if (P)artition then (A)vailability or (C)onsistency, (E)lse (L)atency or (C)onsistency**

The `else` branch is the one that governs daily life. Even with a perfect network,
making a read linearisable means confirming with a quorum, and confirming with a
quorum costs a round trip. **Most consistency decisions are latency decisions.**

```mermaid
flowchart LR
    W["Write arrives"] --> Q{"Wait for quorum ack?"}
    Q -->|"yes — EC"| S["Durable and linearisable<br/>+1 network RTT<br/>cross-region: +50-200 ms"]
    Q -->|"no — EL"| F["Fast local ack<br/>Reader on another replica<br/>may not see it yet"]
    style S fill:#0b2545,color:#fff
    style F fill:#134e4a,color:#fff
```

| System | P → | E → | Practical reading |
|---|---|---|---|
| Spanner | C | C | Always consistent; TrueTime commit-wait costs latency |
| CockroachDB | C | C | Same family; consensus per range |
| DynamoDB (eventual reads) | A | L | Fast, cheap, may be stale |
| DynamoDB (strong reads) | C | C | Opt-in per call, ~2× read cost |
| Cassandra | A | L or C | Tunable per query via consistency level |
| MongoDB | C | C or L | `writeConcern` / `readConcern` per operation |
| Riak | A | L | Availability-first by design |
| MySQL / Postgres, single primary | C | L | Primary consistent, async replicas lag |

The important skill is not memorising this table. It is recognising that **most
modern stores let you choose per operation**, and that a design which picks one
global setting is leaving both performance and correctness on the table.

---

## 3. The consistency hierarchy

```mermaid
flowchart TD
    A["Strict serialisability<br/>= serialisability + linearisability"] --> B["Serialisability<br/>transactions equivalent to SOME serial order"]
    A --> C["Linearisability<br/>single objects, REAL-TIME order"]
    B --> D["Snapshot isolation<br/>consistent snapshot; write skew possible"]
    C --> E["Sequential consistency<br/>a global order exists,<br/>but not necessarily real-time"]
    E --> F["Causal consistency<br/>if A happened-before B,<br/>everyone sees A before B"]
    F --> G["Session guarantees<br/>per-client promises"]
    G --> H["Eventual consistency<br/>replicas converge if writes stop"]

    style A fill:#0b2545,color:#fff
    style F fill:#134e4a,color:#fff
    style H fill:#3b0d0d,color:#fff
```

### 3.1 Distinguishing the two that get confused

**Linearisability** is about *single objects* and *real time*. If write W completes
at 10:00:00.000 and read R starts at 10:00:00.001, R must see W. It is a
recency guarantee.

**Serialisability** is about *multiple objects in transactions*. It requires the
result to be equivalent to *some* serial order — not necessarily the real-time
order. A serialisable system may execute your transaction "as if" it ran before one
that actually finished earlier.

**Strict serialisability** is both, and it is what people usually mean when they say
"strongly consistent". Spanner provides it. Most single-node databases running
`SERIALIZABLE` provide it too, since one node makes real-time order trivial.

### 3.2 Causal consistency — the sweet spot

Causal consistency is the strongest model achievable **without giving up
availability during a partition**. It preserves the orderings people actually
notice:

```mermaid
sequenceDiagram
    participant A as Alice
    participant S as System
    participant B as Bob

    A->>S: post "I passed the exam!"   (event e1)
    B->>S: read e1
    B->>S: reply "Congratulations!"    (e2, causally depends on e1)
    Note over S: any replica showing e2 MUST already show e1
    Note over S: Carol's unrelated post (e3) may appear in any order —<br/>no causal relationship, so no constraint
```

The implementation uses **vector clocks** or **dependency tracking**: each write
carries the set of writes it observed, and a replica delays applying a write until
its dependencies are present. The cost is metadata that grows with the number of
writers, which is why practical systems compress it (dotted version vectors,
per-datacentre counters, or hybrid logical clocks).

### 3.3 Session guarantees — the cheap wins

These are what users perceive, and they are far cheaper than global consistency
because they only constrain **one client's own view**.

| Guarantee | Formal meaning | Implementation |
|---|---|---|
| **Read-your-writes** | A client sees its own prior writes | Route to the leader for N seconds after a write; or carry the write's LSN and require the replica to be caught up |
| **Monotonic reads** | Successive reads never go backwards in time | Pin the session to one replica; or track the highest LSN seen and never read from a replica behind it |
| **Monotonic writes** | A client's writes apply in issue order | Same-session ordering token; or serialise per session |
| **Writes-follow-reads** | A write that follows a read is ordered after what was read | Attach the read's version to the write |

The **version-token** technique deserves highlighting because it generalises: the
client keeps the highest version it has observed (an LSN, a hybrid logical
timestamp, a vector) and sends it with every request. Any replica can serve the
request as long as it has caught up to that token; otherwise it waits briefly or
forwards to one that has. You get per-client consistency with full replica
parallelism.

---

## 4. Conflict resolution

When two replicas accept conflicting writes, something must reconcile them.

```mermaid
flowchart TD
    C["Concurrent conflicting writes detected"] --> S{"Resolution strategy"}
    S --> LWW["Last-write-wins<br/>compare timestamps"]
    S --> MV["Keep siblings<br/>return all versions to the app"]
    S --> CRDT["CRDT<br/>merge is defined by the data type"]
    S --> APP["Application logic<br/>domain-specific merge"]

    LWW --> LWWN["Simple. SILENTLY LOSES DATA.<br/>Clock skew decides your business outcomes."]
    MV --> MVN["Correct, no loss.<br/>Pushes the problem to the application."]
    CRDT --> CRDTN["Automatic, provably convergent.<br/>Only for types that admit a merge."]
    APP --> APPN["Most correct.<br/>Most work."]

    style LWWN fill:#7d1128,color:#fff
    style CRDTN fill:#14532d,color:#fff
```

**Last-write-wins is more dangerous than it looks.** With clock skew between nodes,
"last" is not "latest" — a write made at 10:00:05 on a node whose clock is 3
seconds slow will lose to a write made at 10:00:03 on a fast node. The user who
wrote second sees their change vanish, silently, with no error anywhere. Only use
LWW where losing a concurrent update is genuinely acceptable (a cache, a
last-seen-at timestamp, a presence indicator).

### 4.1 CRDTs

A **conflict-free replicated data type** has a merge function that is commutative,
associative and idempotent. That means replicas converge regardless of the order or
number of times updates arrive — no coordination, no conflicts, ever.

| Type | Operation | Merge rule |
|---|---|---|
| **G-Counter** | increment only | Per-replica counters; merge takes the max of each, sum for the value |
| **PN-Counter** | increment and decrement | Two G-Counters (P and N); value = sum(P) − sum(N) |
| **G-Set** | add only | Union |
| **2P-Set** | add, remove once | Two G-Sets (added, removed) |
| **LWW-Element-Set** | add, remove | Timestamped membership; latest wins per element |
| **OR-Set** | add, remove, re-add | Unique tag per add; remove targets specific tags |
| **RGA / Logoot / YATA** | ordered sequence | Position identifiers between neighbours — the basis of collaborative text editing |

```mermaid
flowchart LR
    subgraph r1["Replica A"]
        A1["counter: {A:5, B:0, C:0}"]
        A2["A increments twice<br/>{A:7, B:0, C:0}"]
    end
    subgraph r2["Replica B"]
        B1["counter: {A:5, B:0, C:0}"]
        B2["B increments three times<br/>{A:5, B:3, C:0}"]
    end
    A2 --> M["merge: element-wise max<br/>{A:7, B:3, C:0} → value 10"]
    B2 --> M
    M --> N["Converges regardless of<br/>message order or duplication"]
    style M fill:#14532d,color:#fff
```

The limitation is real: **CRDTs give you convergence, not correctness.** A CRDT set
will happily converge to a state where a seat is booked twice. Anything requiring a
global invariant — non-negative inventory, unique usernames, account balances that
must not go below zero — needs coordination. CRDTs remove the need to coordinate;
they do not remove the need to be right.

---

## 5. Replication mechanics

### 5.1 What actually gets shipped

| Method | Ships | Pros | Cons |
|---|---|---|---|
| **Statement-based** | The SQL text | Compact | Non-deterministic functions (`NOW()`, `RAND()`), triggers and auto-increment diverge |
| **Write-ahead log shipping** | Physical page changes | Exact, cheap | Replicas must run the identical storage engine version |
| **Logical / row-based** | Row before/after images | Version-independent, enables CDC and heterogeneous targets | Larger volume |
| **Trigger-based** | Application-level captures | Fully flexible | Slow, invasive, fragile |

Logical replication is the one to know, because it is what makes **change data
capture** possible — streaming a database's changes into Kafka, a search index, a
warehouse, or a cache invalidator, in commit order, with no polling.

### 5.2 Replication lag and its symptoms

```mermaid
sequenceDiagram
    participant U as User
    participant P as Primary
    participant R as Replica

    U->>P: UPDATE profile SET bio = 'new'
    P-->>U: 200 OK
    P->>R: replicate (async, 200 ms behind)
    U->>R: GET /profile   (load balancer sent the read to a replica)
    R-->>U: bio = 'old'   ⚠ "my change didn't save"
    Note over P,R: 200 ms later the replica catches up.<br/>The user has already refreshed twice and filed a bug.
```

Fixes, cheapest first:

1. **Read your own writes from the primary** for a short window after any write —
   set a cookie or session flag with a timestamp.
2. **Carry the LSN.** The write returns its log position; the read requires a
   replica at or beyond it.
3. **Monitor and cap lag.** Eject replicas beyond a lag threshold from the read
   pool.
4. **Semi-synchronous replication** so at least one replica is always current.

### 5.3 Failover hazards

```mermaid
flowchart TD
    F["Primary fails"] --> D{"Was replication synchronous?"}
    D -->|"async"| L["Un-replicated committed writes are LOST.<br/>The client was told 'success'."]
    D -->|"semi-sync"| S["At least one replica has them.<br/>Promote that one."]
    L --> Z{"Old primary returns"}
    Z --> SB["SPLIT BRAIN: two primaries,<br/>divergent histories, conflicting ids"]
    SB --> FIX["Fencing: the old primary must be<br/>refused by storage and clients<br/>before the new one is promoted"]
    style L fill:#7d1128,color:#fff
    style SB fill:#7d1128,color:#fff
    style FIX fill:#14532d,color:#fff
```

The nastiest concrete instance: an async replica is promoted, starts issuing
auto-increment ids from where *it* left off, and reuses ids the old primary had
already handed out — so ids that were unique become duplicates pointing at
different rows in the application, the cache, and every downstream system. This is
a genuine, documented production failure class, and it is the reason external ids
should not be a bare database sequence.

---

## 6. Quorum arithmetic

```
N = replicas
W = nodes that must acknowledge a write
R = nodes that must respond to a read

R + W > N  ⇒  read and write sets intersect ⇒ at least one node in every read
             has the latest write
W > N/2    ⇒  writes cannot conflict with each other (no two disjoint write quorums)
```

| Config | Behaviour |
|---|---|
| N=3, W=3, R=1 | Fast reads. Any node down blocks writes |
| N=3, W=1, R=3 | Fast writes. Any node down blocks reads |
| **N=3, W=2, R=2** | Tolerates one failure for both. **The standard** |
| N=5, W=3, R=3 | Tolerates two failures |
| N=3, W=1, R=1 | No overlap — eventual only, maximum availability |

**Sloppy quorums** (Dynamo-style) relax this: if the natural replicas are
unreachable, write to *any* N available nodes and hand the data back later via
**hinted handoff**. This buys availability during a partition and gives up the
`R + W > N` guarantee while the hints are outstanding. It is the right trade for a
shopping cart and the wrong one for a bank balance.

---

## 7. A decision procedure

```mermaid
flowchart TD
    Q["For THIS operation..."] --> A{"Can a stale read cause<br/>financial, legal or safety harm?"}
    A -->|"yes"| S["Linearisable read.<br/>Route to leader or read at quorum.<br/>Accept the latency."]
    A -->|"no"| B{"Would the USER notice<br/>and be confused?"}
    B -->|"yes — their own data"| RY["Read-your-writes.<br/>Session pinning or LSN token."]
    B -->|"yes — ordering matters"| CC["Causal consistency.<br/>Dependency tracking."]
    B -->|"no"| E["Eventual.<br/>Read any replica, cache freely."]
    style S fill:#0b2545,color:#fff
    style RY fill:#134e4a,color:#fff
    style E fill:#14532d,color:#fff
```

Applied to one product, this produces a table like:

| Operation | Model | Why |
|---|---|---|
| Account balance at withdrawal | Linearisable | Money |
| Permission check | Linearisable, fail closed | Security |
| Placing an order | Linearisable on inventory | Cannot oversell |
| Own profile after editing | Read-your-writes | Otherwise "it didn't save" |
| Reply appearing under its parent | Causal | Otherwise nonsense |
| Follower count | Eventual | Nobody is harmed by a 10 s lag |
| Recommendations | Eventual, cached for hours | Freshness is nearly irrelevant |

**One system, six different answers.** That is the correct outcome — a single global
consistency setting means you are either paying too much or being wrong somewhere.

---

[← previous: Caching and CDN](03-caching-and-cdn.md) · [back to the handbook](../README.md) · [next: Messaging and async →](05-messaging-and-async.md)
