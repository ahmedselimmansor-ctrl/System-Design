# 05 — Messaging and Async

*Queues, logs, delivery semantics, the outbox, ordering, and backpressure.*

[← back to the handbook](../README.md)

---

## 1. Choosing the messaging primitive

```mermaid
flowchart TD
    Q["What are you actually doing?"]
    Q -->|"handing work to a pool of workers"| W["Task queue<br/>SQS, RabbitMQ, Celery, Sidekiq"]
    Q -->|"publishing facts others may care about"| E["Event log<br/>Kafka, Pulsar, Kinesis"]
    Q -->|"request/response, but not right now"| RR["Queue + reply-to / correlation id"]
    Q -->|"notify many, no durability needed"| PS["Pub/sub fan-out<br/>Redis pub/sub, SNS"]
    Q -->|"process a continuous stream with state"| ST["Stream processing<br/>Flink, Kafka Streams"]
    Q -->|"schedule work for later"| SC["Delay queue / scheduler<br/>SQS delay, Temporal, cron"]
    style W fill:#0b2545,color:#fff
    style E fill:#134e4a,color:#fff
```

The distinction that matters most: **a command has exactly one intended handler; an
event has zero or more interested observers.**

```
Command:  ChargeCard          → "do this"      → a queue, one consumer, may be rejected
Event:    OrderPlaced         → "this happened" → a log, N consumers, cannot be rejected
```

Getting this backwards produces the two classic architectural smells: a "queue" that
five services all read (and each processes a fraction of the messages, silently
dropping the rest) and an "event" that the publisher expects a specific consumer to
act on, creating hidden coupling that no diagram shows.

---

## 2. Log mechanics in depth

### 2.1 Anatomy

```mermaid
flowchart TD
    subgraph P["Partition 0 — an append-only file"]
        S0["offset 0"] --> S1["offset 1"] --> S2["offset 2"] --> S3["offset 3"] --> S4["offset 4<br/>← log end"]
    end
    subgraph consumers["Independent consumer groups"]
        G1["group: billing<br/>committed offset 3"]
        G2["group: analytics<br/>committed offset 1"]
        G3["group: search-indexer<br/>committed offset 4"]
    end
    P --> G1 & G2 & G3
    N["Each group reads at its own pace.<br/>Retention is by TIME or SIZE, never by consumption.<br/>A new group can start at offset 0 and replay history."]
    style N fill:#422006,color:#fff
```

Retention decoupled from consumption is the whole point. It means:

- A bug in a consumer is recoverable — fix it, reset the offset, reprocess.
- A new service can be added later and backfill from history.
- A projection (search index, cache, materialised view) can be rebuilt from scratch.

### 2.2 Partitioning and its consequences

```mermaid
flowchart LR
    K["Message key"] --> H["partition = hash(key) mod partition_count"]
    H --> P["Same key ⇒ same partition ⇒ ordered"]
    P --> C1["✓ per-entity ordering"]
    P --> C2["✗ no ordering across entities"]
    P --> C3["✗ changing partition_count breaks the mapping"]
    P --> C4["✗ a hot key overloads one partition"]
    style C3 fill:#7d1128,color:#fff
    style C4 fill:#7d1128,color:#fff
```

| Decision | Guidance |
|---|---|
| **Partition count** | Set it for future parallelism — increasing later breaks key affinity. A common heuristic: `max(target_throughput / per_partition_throughput, target_consumers)`, then round up generously |
| **Key choice** | The entity whose ordering must be preserved: `order_id`, `user_id`, `account_id`. **Never** a random value if ordering matters, **never** a low-cardinality value like `country` |
| **Null key** | Round-robin distribution, no ordering. Correct for genuinely independent events |
| **Hot key** | Sub-key it (`user_id:bucket`) and accept partial ordering, or handle it out of band |

### 2.3 Consumer group rebalancing

```mermaid
sequenceDiagram
    participant C1 as Consumer 1
    participant C2 as Consumer 2
    participant B as Group coordinator

    Note over C1,B: steady state — C1 owns P0 and P1, C2 owns P2 and P3
    C1->>B: heartbeat
    C2->>B: heartbeat
    Note over C2: processing a message takes 6 minutes<br/>max.poll.interval.ms = 5 minutes
    B->>B: C2 considered dead
    B->>C1: REBALANCE — stop, revoke all partitions
    Note over C1,C2: ENTIRE GROUP STALLS during reassignment
    B->>C1: assign P0,P1,P2,P3
    C2->>B: (finally finishes) commit offset
    B-->>C2: rejected — you were fenced
    C2->>B: rejoin → ANOTHER rebalance
    Note over C1,C2: rebalance storm — throughput collapses
```

This is one of the most common production incidents with log-based consumers, and
it presents as "the broker is broken" when it is an application problem. The fixes:

- **Keep per-message processing well under the poll interval.** If work is slow,
  hand it to a thread pool and keep polling — but then you own the offset-commit
  correctness yourself.
- **Raise `max.poll.interval.ms` and lower `max.poll.records`** so a batch cannot
  exceed the interval.
- **Use cooperative/incremental rebalancing** so only the moved partitions pause
  rather than the whole group.
- **Use static group membership** so a rolling restart does not trigger a full
  reshuffle.

---

## 3. Delivery semantics, precisely

```mermaid
flowchart TD
    subgraph fail["Where duplicates come from"]
        F1["Producer sends, ack lost → producer retries"]
        F2["Consumer processes, crashes before committing the offset"]
        F3["Rebalance reassigns a partition mid-batch"]
        F4["A queue's visibility timeout expires while still processing"]
        F5["An upstream retry at the HTTP layer"]
    end
    fail --> R["At-least-once is the physical reality.<br/>Design the consumer for it."]
    style R fill:#14532d,color:#fff
```

### 3.1 The consumer's commit choice

```mermaid
flowchart LR
    subgraph before["Commit BEFORE processing"]
        B1["read → commit offset → process"] --> B2["crash after commit ⇒ MESSAGE LOST<br/>= at-most-once"]
    end
    subgraph after["Commit AFTER processing"]
        A1["read → process → commit offset"] --> A2["crash before commit ⇒ REPROCESSED<br/>= at-least-once"]
    end
    style before fill:#7d1128,color:#fff
    style after fill:#14532d,color:#fff
```

Always commit after. Then make processing idempotent. There is no third option that
does not involve a transaction spanning the broker and your database — which is
exactly what the outbox/inbox pattern approximates without the distributed
transaction.

### 3.2 The inbox pattern

```mermaid
sequenceDiagram
    participant Q as Broker
    participant C as Consumer
    participant DB as Database

    Q->>C: message (id = msg-8821)
    C->>DB: BEGIN
    C->>DB: INSERT INTO processed_messages(id) VALUES ('msg-8821')
    alt already present — UNIQUE violation
        DB-->>C: conflict
        C->>DB: ROLLBACK
        C->>Q: ack (already done, safely skip)
    else first time
        C->>DB: apply the business effect
        C->>DB: COMMIT
        C->>Q: ack
    end
```

The effect and the dedup record commit atomically, so the duplicate is impossible
to double-apply regardless of when the crash happens. Keep `processed_messages`
partitioned by time and drop old partitions — it is the cheapest way to prevent
unbounded growth.

---

## 4. Ordering

Ordering is the requirement people forget to state and then discover the hard way.

```mermaid
flowchart TD
    Q["Do you need ordering?"] --> A{"Across what scope?"}
    A -->|"globally, all messages"| G["Single partition, single consumer.<br/>Throughput ceiling = one consumer.<br/>Almost never actually required."]
    A -->|"per entity"| E["Partition by entity id.<br/>Full parallelism across entities,<br/>strict order within one. ✓"]
    A -->|"none"| N["Any partition, max parallelism."]
    E --> W["Warning: retries break it.<br/>If message 2 fails and is retried<br/>while message 3 succeeds,<br/>the order is gone."]
    style E fill:#14532d,color:#fff
    style W fill:#7d1128,color:#fff
```

The retry/ordering conflict is subtle and important. If you need strict per-entity
ordering **and** retries, you must either:

- **Block the partition** on failure — stop consuming until the message succeeds or
  is parked. Correct, but one poison message halts every entity in that partition.
- **Park the entity, not the partition** — move the failing entity's messages to a
  side queue and continue with others. More work, much better behaviour.
- **Make handlers order-insensitive** — use version numbers and ignore stale
  updates (`if incoming.version <= current.version: skip`). Often the best answer.

---

## 5. The outbox pattern, in production form

```sql
CREATE TABLE outbox (
    id             BIGSERIAL PRIMARY KEY,
    aggregate_type TEXT        NOT NULL,   -- 'order'
    aggregate_id   TEXT        NOT NULL,   -- '8821'
    event_type     TEXT        NOT NULL,   -- 'OrderPlaced'
    payload        JSONB       NOT NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at   TIMESTAMPTZ            -- NULL until relayed
);

CREATE INDEX outbox_unpublished ON outbox (id) WHERE published_at IS NULL;
```

```mermaid
flowchart TD
    subgraph tx["One database transaction"]
        T1["INSERT INTO orders ..."]
        T2["INSERT INTO outbox ..."]
        T3["COMMIT"]
    end
    tx --> R{"Relay strategy"}
    R -->|"polling"| P["SELECT ... WHERE published_at IS NULL<br/>ORDER BY id LIMIT 100<br/>simple; adds query load; ~1 s latency"]
    R -->|"CDC (preferred)"| C["Tail the WAL with Debezium<br/>no query load; ms latency;<br/>guaranteed commit order"]
    P & C --> B["Publish to broker"]
    B --> M["Mark published / delete row"]
    M --> N["Crash between publish and mark<br/>⇒ republished ⇒ consumers must be idempotent"]
    style C fill:#14532d,color:#fff
    style N fill:#422006,color:#fff
```

Details that separate a working outbox from a demo one:

- **Order by `id`, and process serially per aggregate.** Publishing out of commit
  order defeats the purpose.
- **Delete or archive published rows.** An outbox table that grows forever will
  eventually make the partial index useless and the vacuum expensive.
- **The partial index is not optional.** Without `WHERE published_at IS NULL`, the
  relay's query scans an ever-growing table.
- **Include an idempotency key in the payload** — usually the outbox `id` — so
  consumers have a natural dedup key.
- **Monitor relay lag.** `now() - min(created_at) WHERE published_at IS NULL` is the
  single most important metric; if it grows, events are silently not happening.

---

## 6. Dead letter queues

```mermaid
flowchart TD
    M["Message"] --> H["Handler"]
    H -->|"success"| OK["ack"]
    H -->|"error"| CLS{"Classify"}
    CLS -->|"transient<br/>timeout, 503, deadlock"| RT{"attempts &lt; max?"}
    CLS -->|"permanent<br/>schema mismatch, unknown type,<br/>referenced entity deleted"| DLQ
    RT -->|"yes"| BO["backoff + jitter, requeue"] --> H
    RT -->|"no"| DLQ["Dead letter queue"]
    DLQ --> A["ALERT — depth &gt; 0 is an incident"]
    DLQ --> I["Inspect: why did it fail?"]
    I --> F1["Bug in the consumer → fix, replay"]
    I --> F2["Bad producer data → fix upstream, discard or repair"]
    I --> F3["Genuinely unprocessable → document and drop deliberately"]
    style DLQ fill:#7d1128,color:#fff
```

The operational rules people skip:

- **Store the failure reason and the full original message**, not just the payload.
  A DLQ entry without the exception and the headers is nearly useless.
- **Build the replay tool before you need it.** During an incident is the wrong time
  to write one, and an untested replay path can double-apply effects.
- **Cap DLQ retention and alert on age**, not only on count. A message that has sat
  in the DLQ for two weeks is a decision nobody made.

---

## 7. Backpressure and flow control

```mermaid
flowchart TD
    P["Producer: 10,000 msg/s"] --> Q["Queue"] --> C["Consumer: 6,000 msg/s"]
    Q --> G["Depth grows by 4,000/s<br/>= 14.4 M messages/hour"]
    G --> R{"Response"}
    R -->|"1. Autoscale consumers"| R1["Best if the workload parallelises<br/>and the downstream can take it"]
    R -->|"2. Throttle the producer"| R2["Correct for internal pipelines.<br/>Requires the producer to be able to wait"]
    R -->|"3. Shed load"| R3["Correct for user-facing.<br/>Reject fast with 429/503"]
    R -->|"4. Degrade — sample or drop"| R4["Correct for telemetry and analytics.<br/>Drop by priority, not at random"]
    R -->|"5. Do nothing"| R5["Queue grows until broker limits,<br/>disk, or memory. Then everything fails<br/>at once, including what was working."]
    style R5 fill:#7d1128,color:#fff
```

**Every buffer must be bounded.** An unbounded queue does not absorb overload — it
converts a visible, immediate, partial failure into an invisible, delayed, total
one. When the bound is hit, the system must have a defined behaviour: block, shed,
or drop. "Undefined" means OOM.

### 7.1 Consumer lag is the health metric

Measure lag in **two units**, because they answer different questions:

| Unit | Answers |
|---|---|
| Messages behind | How much work is outstanding |
| **Seconds behind** (age of the oldest unprocessed message) | **How stale is the data downstream** — this is the one users feel |

Alert on seconds-behind with a burn-rate style rule: a lag that is growing is an
incident even if the absolute number is still small, because the trend determines
whether it recovers.

---

## 8. Scheduled and delayed work

| Need | Mechanism | Notes |
|---|---|---|
| Retry in 30 s | Delay queue / visibility timeout | Native in SQS, RabbitMQ (via TTL + DLX) |
| Send a reminder in 3 days | Scheduler table + poller, or a durable workflow engine | Broker delay limits are usually ~15 minutes |
| Nightly batch | Cron / workflow orchestrator | Needs idempotency — reruns happen |
| Long multi-step process with waits | **Durable workflow engine** (Temporal, Step Functions) | Persists execution state; survives restarts; handles timers, retries and compensation natively |

For anything with more than two steps, human approval, or waits measured in days, a
durable workflow engine is dramatically less work than hand-rolling a state machine
across queues and a database — and it makes the process *inspectable*, which is
what you will want at 3 am.

---

## 9. Checklist

```
□ Command vs event distinguished; queue vs log chosen accordingly
□ Partition key chosen from the ordering requirement, not by default
□ Partition count set with future parallelism in mind
□ Consumer commits AFTER processing
□ Consumers are idempotent (dedup key + inbox table, or naturally idempotent effects)
□ Outbox used for any "write to DB and publish an event" flow
□ Outbox relay lag monitored and alerted
□ Retry classification: transient vs permanent
□ Backoff has jitter; total attempts capped
□ DLQ exists, is alerted on, and has a TESTED replay path
□ Every buffer is bounded, with defined behaviour at the bound
□ Consumer lag measured in seconds, not just messages
□ The UX for "work is pending" is designed, not an afterthought
```

---

[← previous: Data and consistency](04-data-and-consistency.md) · [back to the handbook](../README.md) · [next: Microservices and APIs →](06-microservices-and-apis.md)
