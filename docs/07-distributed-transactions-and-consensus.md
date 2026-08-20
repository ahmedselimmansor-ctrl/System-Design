# 07 — Distributed Transactions and Consensus

*2PC, sagas, Raft in detail, quorums, leases, clocks, and the impossibility results.*

[← back to the handbook](../README.md)

---

## 1. Why this is hard

A single-node transaction is atomic because one component decides. Across nodes,
the decision itself must be agreed upon, and **agreement over an unreliable network
is the fundamental problem of distributed systems.**

Two impossibility results bound what is achievable:

**FLP (1985)** — in a fully asynchronous system where even one process may fail by
crashing, no deterministic algorithm guarantees consensus in bounded time. The
proof rests on the inability to distinguish a crashed process from a slow one.

**The Two Generals Problem** — no finite exchange of messages over a lossy channel
can produce common knowledge of an agreement. Every message needs an
acknowledgement, which needs an acknowledgement, forever.

Practical systems escape both by weakening the requirement: they use timeouts (a
**failure detector**, which may be wrong) and randomisation, trading *guaranteed*
termination for termination *with probability 1*. That is sufficient in practice
and is why Raft and Paxos work despite FLP.

---

## 2. Two-phase commit

### 2.1 The protocol

```mermaid
sequenceDiagram
    participant C as Coordinator
    participant P1 as Participant 1
    participant P2 as Participant 2

    Note over C,P2: Phase 1 — voting
    C->>C: log BEGIN
    C->>P1: PREPARE
    C->>P2: PREPARE
    P1->>P1: do the work, hold locks,<br/>force-write a PREPARED record
    P2->>P2: same
    P1-->>C: YES
    P2-->>C: YES

    Note over C,P2: Phase 2 — decision
    C->>C: force-write COMMIT — the point of no return
    C->>P1: COMMIT
    C->>P2: COMMIT
    P1->>P1: commit, release locks
    P2->>P2: commit, release locks
    P1-->>C: ACK
    P2-->>C: ACK
    C->>C: log END
```

Once a participant votes YES it has **surrendered its autonomy**: it must be able to
commit if told to, so it holds locks and cannot unilaterally abort. That single
property is the source of every 2PC problem.

### 2.2 The failure matrix

| Failure point | Consequence |
|---|---|
| Participant fails **before** voting | Coordinator times out → global ABORT. Clean |
| Participant fails **after** voting YES | On recovery it must ask the coordinator for the outcome. It may block indefinitely |
| **Coordinator fails before writing the decision** | Participants time out → they may abort. Recoverable |
| **Coordinator fails after writing COMMIT but before sending it** | Participants are stuck in PREPARED, holding locks, unable to decide. **This is the blocking case** |
| Network partition during phase 2 | Same as above for the isolated participants |

```mermaid
stateDiagram-v2
    [*] --> Working
    Working --> Prepared: vote YES,<br/>locks held, cannot abort alone
    Prepared --> Committed: COMMIT received
    Prepared --> Aborted: ABORT received
    Prepared --> Blocked: coordinator unreachable
    Blocked --> Committed: coordinator recovers, says commit
    Blocked --> Aborted: coordinator recovers, says abort
    note right of Blocked
        Locks held the entire time.
        Other transactions queue behind them.
        Throughput collapses.
    end note
```

**Three-phase commit** adds a pre-commit phase to make the protocol non-blocking —
but only under the assumption of a synchronous network with bounded message delay
and a perfect failure detector. Those assumptions do not hold on real networks, so
3PC is essentially unused in practice. The modern answer to 2PC's blocking problem
is to make the *coordinator itself* fault-tolerant via consensus — which is exactly
what Spanner and CockroachDB do: 2PC across shards, with each shard's state
replicated by Paxos/Raft, so no single coordinator failure can block anything.

---

## 3. Sagas in production

### 3.1 Orchestrated saga with a state machine

```mermaid
stateDiagram-v2
    [*] --> OrderCreated
    OrderCreated --> InventoryReserved: reserve OK
    OrderCreated --> Cancelled: reserve failed
    InventoryReserved --> PaymentAuthorised: auth OK
    InventoryReserved --> ReleasingInventory: auth failed
    PaymentAuthorised --> PaymentCaptured: capture OK
    PaymentAuthorised --> VoidingAuth: capture failed
    PaymentCaptured --> Shipped: handoff to carrier
    PaymentCaptured --> Refunding: fulfilment failed
    Shipped --> [*]

    ReleasingInventory --> Cancelled
    VoidingAuth --> ReleasingInventory
    Refunding --> ReleasingInventory

    Cancelled --> [*]
```

The saga's state must be **persisted after every transition**, not held in memory.
The orchestrator is then stateless and any instance can pick up any saga after a
crash by reading its current state and re-driving the next step. This is precisely
what a durable workflow engine gives you for free.

### 3.2 The isolation problem

Sagas have **no isolation**. Intermediate states are visible to everyone.

```mermaid
sequenceDiagram
    participant S as Saga
    participant DB as Order DB
    participant R as Reporting
    participant U as User

    S->>DB: create order, status = PENDING
    R->>DB: read — sees a PENDING order in revenue reports
    U->>DB: read — sees "processing" in their order list
    S->>S: payment fails
    S->>DB: compensate → status = CANCELLED
    Note over R: reporting counted revenue that never existed<br/>(a "dirty read" across the saga)
    Note over U: user saw an order that vanished
```

Countermeasures, from the saga literature:

| Countermeasure | What it does |
|---|---|
| **Semantic lock** | A status flag (`PENDING`) that tells everyone else this entity is mid-flight. Readers decide how to treat it |
| **Commutative updates** | Design operations so order does not matter (increment/decrement rather than set) |
| **Pessimistic view** | Reorder steps so the risky, visible-effect step happens as late as possible |
| **Re-read value** | Verify the data has not changed before acting on it — optimistic concurrency inside the saga |
| **By value** | Route low-value requests through a saga and high-value ones through a real distributed transaction |

The practical minimum is the **semantic lock plus a UI that reflects it**: show
"processing" rather than "confirmed", and exclude `PENDING` from financial reports.

### 3.3 What makes compensations hard

```mermaid
flowchart TD
    C["Compensating action"] --> P1["Must be idempotent<br/>— it will be retried"]
    C --> P2["Must be able to fail and retry forever<br/>— there is no compensation for a compensation"]
    C --> P3["Must handle 'nothing to compensate'<br/>— the forward step may not have applied"]
    C --> P4["Cannot assume the forward step's<br/>preconditions still hold"]
    C --> P5["Some things are not compensatable:<br/>sent email, shipped goods, published post"]
    P5 --> S["Put non-compensatable steps LAST,<br/>after the last point of failure"]
    style P2 fill:#7d1128,color:#fff
    style S fill:#14532d,color:#fff
```

The "compensation cannot fail" property is what makes sagas operationally demanding:
a compensation that permanently fails leaves the system in an inconsistent state
that only a human can resolve. Every saga therefore needs a **manual intervention
queue** and someone who watches it.

---

## 4. Raft in detail

### 4.1 The three sub-problems

Raft decomposes consensus into leader election, log replication, and safety.

```mermaid
flowchart LR
    A["Leader election<br/>exactly one leader per term"] --> B["Log replication<br/>leader appends, followers copy"] --> C["Safety<br/>committed entries are never lost<br/>or reordered"]
    style C fill:#14532d,color:#fff
```

### 4.2 Terms and elections

**Terms** are a logical clock: a monotonically increasing integer, at most one
leader per term. Every message carries a term, and any node seeing a higher term
immediately reverts to follower. This single rule makes stale leaders harmless.

```mermaid
sequenceDiagram
    participant F1 as Node 1
    participant F2 as Node 2
    participant F3 as Node 3

    Note over F1,F3: term 4, leader = Node 1
    Note over F1: Node 1 crashes
    Note over F2: election timeout fires (randomised 150-300 ms)
    F2->>F2: term = 5, become CANDIDATE, vote for self
    F2->>F3: RequestVote(term=5, lastLogIndex=42, lastLogTerm=4)
    F3->>F3: my log is not more up-to-date → grant
    F3-->>F2: VOTE GRANTED
    Note over F2: 2 of 3 = majority → LEADER for term 5
    F2->>F3: AppendEntries (heartbeat, term=5)
    Note over F1: Node 1 recovers, still thinks term 4
    F1->>F2: AppendEntries(term=4) as leader
    F2-->>F1: reject, current term is 5
    F1->>F1: sees higher term → step down to FOLLOWER
```

Two rules do the heavy lifting:

- **Randomised election timeouts** (typically 150–300 ms) make simultaneous
  candidacies rare, so split votes resolve within a round or two without any extra
  protocol.
- **The election restriction**: a voter refuses a candidate whose log is less
  up-to-date than its own (compare last log term, then last log index). This
  guarantees that any elected leader already contains every committed entry — which
  is what makes committed entries durable across leader changes.

### 4.3 Log replication and the commit rule

```mermaid
flowchart TD
    A["Client sends command"] --> B["Leader appends to its log (uncommitted)"]
    B --> C["Leader sends AppendEntries to all followers"]
    C --> D{"Majority acked?"}
    D -->|"yes"| E["Leader marks the entry COMMITTED,<br/>applies it to the state machine,<br/>replies to the client"]
    D -->|"no"| F["Retry indefinitely — the leader keeps<br/>sending until each follower catches up"]
    E --> G["commitIndex piggybacks on the next<br/>heartbeat; followers then apply too"]
    style E fill:#14532d,color:#fff
```

The **log matching property** is what keeps everything consistent: each
`AppendEntries` includes the index and term of the *preceding* entry. A follower
rejects the request if it does not have a matching entry there, and the leader then
walks backwards until it finds the point of agreement and overwrites the follower's
divergent suffix. By induction, identical index+term implies identical history.

One subtle safety rule: **a leader may only commit entries from its own term
directly.** Entries from previous terms are committed indirectly, by being followed
by a committed current-term entry. Without this, a specific interleaving of leader
failures can un-commit an already-committed entry.

### 4.4 Cost and placement

```mermaid
flowchart LR
    A["Write arrives at leader"] --> B["Append locally + fsync"] --> C["Parallel AppendEntries to followers"] --> D["Wait for majority fsync"] --> E["Apply + respond"]
    E --> F["Cost = 1 network RTT to the median follower<br/>+ disk fsync latency"]
    F --> G["Same DC: ~1-3 ms ✓<br/>Same region, multi-AZ: ~2-5 ms ✓<br/>Cross-continent: 100-300 ms ✗"]
    style G fill:#422006,color:#fff
```

Placement guidance:

- **3 or 5 nodes.** 3 tolerates 1 failure, 5 tolerates 2. Even numbers add cost
  without adding fault tolerance.
- **Spread across availability zones, not regions**, unless you specifically need to
  survive a region loss and can afford the write latency.
- **Keep it off the data hot path.** Use consensus for cluster membership,
  configuration, leader election and locks. For data, use it per-shard (as
  CockroachDB and TiKV do) so the coordination cost is bounded to one shard's
  replicas rather than the whole cluster.

### 4.5 Raft vs Paxos vs the rest

| Protocol | Notes |
|---|---|
| **Paxos** | The original. Single-decree Paxos agrees on one value; Multi-Paxos chains it. Notoriously difficult to specify completely — most "Paxos" implementations are subtly different |
| **Raft** | Same guarantees, designed for understandability: strong leader, terms, log matching. Dominates new systems |
| **Zab** | ZooKeeper's protocol. Similar to Raft, with primary-order guarantees suited to its API |
| **Viewstamped Replication** | Predates Paxos in publication of the practical form; very close to Raft in structure |
| **EPaxos / Flexible Paxos** | Leaderless or relaxed-quorum variants; lower latency in the common case, considerably more complex |

---

## 5. Leases and fencing

A **lease** is a lock with an expiry — it grants exclusive access for a bounded
time, so a crashed holder cannot block forever.

```mermaid
sequenceDiagram
    participant A as Holder A
    participant B as Holder B
    participant S as Lease service
    participant R as Resource

    A->>S: acquire(lease, ttl=10 s)
    S-->>A: granted, epoch = 17
    loop while working
        A->>S: renew (every 3 s)
    end
    Note over A: A pauses — GC, VM migration, network loss
    Note over S: no renewal for 10 s → lease expires
    B->>S: acquire
    S-->>B: granted, epoch = 18
    B->>R: write(epoch = 18)
    R->>R: accept, record highest epoch = 18
    A->>R: write(epoch = 17)
    R->>R: epoch 17 is below 18 → REJECT
```

The critical insight, from Martin Kleppmann's analysis of distributed locking: **the
lease service alone cannot provide mutual exclusion**, because a process can be
paused arbitrarily long between checking the lease and using it. Safety must come
from the **resource** rejecting stale epochs. If your resource cannot do that (a
plain file, a third-party API with no versioning), then the lock is advisory and
your operation must tolerate concurrent execution.

**Clock skew makes leases subtler still.** The holder and the service must agree
that time has passed. The standard defence is that the holder treats its lease as
expiring *earlier* than the service does — a safety margin larger than the maximum
plausible clock drift plus one round trip.

---

## 6. Time in distributed systems

```mermaid
flowchart TD
    subgraph phys["Physical clocks"]
        P1["Wall clock — NTP synchronised"]
        P2["Can jump backwards, drift, be wrong<br/>by seconds. NEVER use for ordering."]
        P3["Monotonic clock — measures elapsed time<br/>Use for timeouts and durations."]
    end
    subgraph log["Logical clocks"]
        L1["Lamport timestamp — a counter<br/>a → b implies L(a) &lt; L(b)<br/>but NOT the converse"]
        L2["Vector clock — one counter per node<br/>detects concurrency exactly<br/>size grows with node count"]
        L3["Hybrid logical clock (HLC) —<br/>physical time + logical counter<br/>close to wall time AND causally correct"]
    end
    subgraph bounded["Bounded uncertainty"]
        B1["TrueTime (Spanner) — returns an INTERVAL<br/>[earliest, latest], typically ~7 ms wide"]
        B2["Commit-wait: sleep out the interval<br/>so no two transactions can overlap<br/>⇒ external consistency"]
    end
    style P2 fill:#7d1128,color:#fff
    style L3 fill:#14532d,color:#fff
    style B2 fill:#0b2545,color:#fff
```

The practical rules:

- **Never order events across machines by wall-clock timestamp.** NTP skew of tens
  of milliseconds is routine; seconds happen. Last-write-wins on wall clocks loses
  data silently.
- **Use monotonic clocks for durations.** A wall-clock-based timeout can become
  negative or enormous after an NTP correction.
- **Hybrid logical clocks are the pragmatic default** for systems that want
  timestamps that are both meaningful to humans and causally consistent. CockroachDB
  and MongoDB both use them.
- **TrueTime's insight is honesty about uncertainty.** Rather than pretending the
  clock is exact, expose the error bound and wait it out. The cost is a few
  milliseconds per commit; the benefit is globally consistent snapshots without
  coordination.

---

## 7. Choosing an approach

```mermaid
flowchart TD
    Q["Need multiple things to change together"] --> A{"Same database?"}
    A -->|"yes"| T["Local transaction. Done.<br/>Do not overthink this."]
    A -->|"no"| B{"Can the data be co-located?"}
    B -->|"yes"| C["Move it. Reshaping ownership beats<br/>a distributed protocol."]
    B -->|"no"| D{"Do you control all participants<br/>and need real atomicity?"}
    D -->|"yes, and latency permits"| E["2PC over a consensus-replicated<br/>coordinator (distributed SQL)"]
    D -->|"no"| F{"Is eventual consistency acceptable<br/>with visible intermediate states?"}
    F -->|"yes"| G["Saga + outbox + idempotency.<br/>The standard answer."]
    F -->|"no"| H["Reconsider the boundaries.<br/>The requirement and the<br/>decomposition are in conflict."]
    style T fill:#14532d,color:#fff
    style G fill:#134e4a,color:#fff
    style H fill:#7d1128,color:#fff
```

That last branch is not a cop-out. If a business invariant genuinely requires
atomicity across two services, the invariant is telling you those two things belong
to one aggregate. Merging them is usually cheaper and more correct than building a
protocol to simulate what a single transaction would have given you for free.

---

[← previous: Microservices and APIs](06-microservices-and-apis.md) · [back to the handbook](../README.md) · [next: Resilience and failure →](08-resilience-and-failure.md)
