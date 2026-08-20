# 13 — Numbers and Cheat Sheets

*Everything you should be able to recall without looking it up.*

[← back to the handbook](../README.md)

---

## 1. Latency, ordered

| Operation | Time | Notes |
|---|---|---|
| L1 cache reference | 1 ns | |
| Branch mispredict | 3 ns | |
| L2 cache reference | 4 ns | |
| Mutex lock/unlock | 17 ns | Uncontended |
| Main memory reference | 100 ns | ~100× L1 |
| Compress 1 KB with Snappy | 2 µs | |
| Read 1 MB sequentially from memory | 3 µs | ~330 GB/s |
| SSD random read | 16 µs | NVMe is faster still, ~10 µs |
| Read 1 MB sequentially from NVMe | 49 µs | ~20 GB/s |
| Round trip within the same datacenter | 500 µs | |
| Read 1 MB sequentially from disk | 825 µs | |
| Disk seek | 2 ms | Why spinning disks lose to SSD by ~100× |
| Round trip within the same region (cross-AZ) | 1–2 ms | |
| US East → US West round trip | 60–70 ms | |
| US → Europe round trip | 80–100 ms | |
| US → Asia round trip | 150–200 ms | Speed of light + routing |

**Derived rules:**

- Memory is ~100× faster than SSD; SSD is ~100× faster than a disk seek.
- One cross-continent round trip ≈ 10,000 SSD reads ≈ 1,500,000 memory reads.
- Sequential access beats random by 10–100×, on SSD as well as spinning disk.
- If a user-facing request makes two serial cross-region calls, its p99 floor is
  ~300 ms before you have done any work.

---

## 2. Time and size

```
seconds/minute        60
seconds/hour       3,600
seconds/day       86,400   ≈ 10^5   ← the one to memorise
seconds/month  2,592,000   ≈ 2.5 x 10^6
seconds/year  31,536,000   ≈ 3 x 10^7
```

```
1 KB = 10^3     1 KiB = 1,024
1 MB = 10^6     1 MiB = 1,048,576
1 GB = 10^9     1 GiB = 1,073,741,824
1 TB = 10^12
1 PB = 10^15
```

**The QPS anchor:**

```
1 action/user/day for 1 M users     ≈ 12 QPS
1 action/user/day for 100 M users   ≈ 1,157 QPS
10 actions/user/day for 100 M users ≈ 11,574 QPS
```

---

## 3. Availability

| Nines | Per year | Per month | Per week |
|---|---|---|---|
| 90% | 36.5 days | 73 hours | 16.8 hours |
| 99% | 3.65 days | 7.3 hours | 1.68 hours |
| 99.9% | 8.77 hours | 43.8 min | 10.1 min |
| 99.95% | 4.38 hours | 21.9 min | 5.04 min |
| 99.99% | 52.6 min | 4.38 min | 1.01 min |
| 99.999% | 5.26 min | 26.3 s | 6.05 s |

**Composition:**

```
Serial (all must work):     A_total = A1 × A2 × ... × An
Parallel (any must work):   A_total = 1 − (1−A1) × (1−A2) × ... × (1−An)

Four 99.9% services in series  = 99.6%   (35 hours/year)
Two 99% replicas in parallel   = 99.99%  (if failures are independent)
```

---

## 4. Capacity anchors

| Resource | Realistic figure |
|---|---|
| Commodity server | 32–128 cores, 128 GB–2 TB RAM |
| NVMe SSD | 1–8 TB, 100k–1M IOPS, 2–7 GB/s sequential |
| Network interface | 10–100 Gbps |
| HTTP requests per app node (simple) | 10k–50k /s |
| Relational DB per node (simple transactions) | 5k–20k /s |
| Relational DB per node (complex queries) | 100–1k /s |
| Redis per node | 100k–1M ops/s |
| Kafka per broker | 100k–1M msgs/s, hundreds of MB/s |
| Elasticsearch per node | 1k–10k queries/s |
| WebSocket connections per tuned node | 100k–1M |
| Ephemeral ports per source IP per destination | ~28,000 usable (default range) |

| Data | Typical size |
|---|---|
| UUID | 16 B binary, 36 B as text |
| Timestamp | 8 B |
| Tweet-length text | ~300 B |
| JSON API response | 1–10 KB |
| Web page (total) | 2–3 MB |
| Photo (compressed) | 200 KB–5 MB |
| 1 minute of 1080p video | ~30 MB |
| 1 hour of 1080p video | ~2 GB |
| Database row with indexes | payload × ~1.3 |

---

## 5. Formula sheet

```
QPS                  = daily_events / 86,400
peak QPS             = avg QPS × peak_factor        (2-3 global, 5-10 regional)

storage/day          = writes/day × bytes/record
storage/year         = storage/day × 365 × replication × (1 + index_overhead)

bandwidth (bps)      = QPS × bytes_per_message × 8

threads needed       = QPS × latency_seconds        (Little's Law)
queue growth/s       = arrival_rate − service_rate

nodes needed         = peak_QPS / per_node_QPS × (1 + headroom)   headroom 0.3-0.5

response time        = service_time / (1 − utilisation)           (M/M/1)
effective latency    = hit% × cache_lat + (1−hit%) × origin_lat
origin load          = total_QPS × (1 − hit_ratio)

quorum overlap       = R + W > N
consensus tolerance  = (N−1)/2 failures for N nodes

Amdahl ceiling       = 1 / serial_fraction
burn rate            = observed_error_rate / budgeted_error_rate
```

---

## 6. Decision tables

### Data store

| Access pattern | Store |
|---|---|
| Transactions, joins, ad-hoc queries | Relational |
| Point lookups by key, very high throughput | Key-value |
| Nested documents, flexible schema | Document |
| Massive writes, range scans within a partition | Wide-column |
| Relationship traversal | Graph |
| Append-only metrics over time | Time series |
| Full-text relevance | Search engine |
| Nearest-neighbour on embeddings | Vector |
| Analytical scans and aggregation | Columnar / warehouse |
| Large immutable files | Object storage |

### Communication

| Need | Choice |
|---|---|
| Public API, broad clients, cacheable | REST |
| Internal, low latency, typed | gRPC |
| Client-driven field selection | GraphQL + persisted queries |
| Server → client push | SSE |
| Bidirectional low latency | WebSocket |
| Decoupled, durable, retryable | Queue |
| Replayable, multi-consumer, ordered | Log |

### Consistency, per operation

| Operation | Model |
|---|---|
| Money, inventory, permissions | Linearisable |
| Own data right after writing it | Read-your-writes |
| Reply ordering, causality | Causal |
| Counts, feeds, recommendations | Eventual |

### Scaling response

| Symptom | First move |
|---|---|
| Read-heavy, DB saturated | Cache, then read replicas |
| Write-heavy, DB saturated | Batch, queue, then partition |
| Slow response, work is deferrable | Async queue + worker |
| Latency dominated by distance | CDN / edge / regional deployment |
| One shard hot | Better partition key, or sub-key the hot entity |
| One tenant hurting others | Per-tenant quotas, then cells / shuffle sharding |
| CPU-bound on serialisation | Binary protocol, compression, fewer round trips |

### Rate limiting

| Need | Algorithm |
|---|---|
| General purpose, allow bursts | Token bucket |
| Smooth downstream load | Leaky bucket |
| Accuracy with low memory | Sliding window counter |
| Exact, low volume, high value | Sliding log |

### Failure response

| Failure | Response |
|---|---|
| Dependency slow | Timeout + circuit breaker + fallback |
| Dependency down | Fallback, degrade, or fail fast |
| Overloaded | Shed by priority; reject in ~1 ms |
| Node degraded but "healthy" | Outlier ejection, hedged requests |
| Retry storm | Retry budget; suppress retries |
| Region down | Failover; DNS/anycast; verify the control plane is elsewhere |

---

## 7. HTTP status codes

| Code | Meaning | Retryable? |
|---|---|---|
| 200 / 201 / 204 | OK / Created / No content | n/a |
| 202 | Accepted for async processing | n/a |
| 301 / 302 | Moved permanently / found | n/a |
| 304 | Not modified (validator matched) | n/a |
| 400 | Malformed request | **No** |
| 401 | Not authenticated | No (refresh token, then retry once) |
| 403 | Authenticated but not permitted | **No** |
| 404 | Not found | **No** |
| 409 | Conflict (version mismatch, duplicate) | Only after reconciling |
| 422 | Semantically invalid | **No** |
| 429 | Rate limited | **Yes** — honour `Retry-After` |
| 500 | Server error | Yes, with backoff |
| 502 / 503 / 504 | Bad gateway / unavailable / gateway timeout | **Yes**, with backoff and jitter |

---

## 8. Cache header recipes

```
Hashed static assets:
  Cache-Control: public, max-age=31536000, immutable

HTML entry point:
  Cache-Control: no-cache
  ETag: "..."

Public API list endpoint:
  Cache-Control: public, s-maxage=60, stale-while-revalidate=600, stale-if-error=86400

Private user data:
  Cache-Control: private, no-store

Semi-static reference data:
  Cache-Control: public, max-age=300, s-maxage=3600
```

---

## 9. Port and protocol reference

| Port | Service |
|---|---|
| 53 | DNS |
| 80 / 443 | HTTP / HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 9092 | Kafka |
| 27017 | MongoDB |
| 2379 / 2380 | etcd client / peer |
| 9200 / 9300 | Elasticsearch HTTP / transport |
| 11211 | Memcached |

---

## 10. The one-page design template

```
SYSTEM: ______________________________________________

1. REQUIREMENTS
   In scope:      ______________________________________
   Deferred:      ______________________________________
   Latency:       p99 ______   Availability: ______
   Consistency:   strong for ______  eventual for ______
   Scale:         ______ DAU, ______ read:write

2. ESTIMATION
   Write QPS: ______ avg  ______ peak
   Read QPS:  ______ avg  ______ peak
   Storage:   ______ /day  ______ /year (x replication)
   Bandwidth: ______ egress
   CONCLUSION FROM THE NUMBERS: ______________________

3. API
   ______________________________________________
   Sync/async boundary at: ____________________________

4. DATA MODEL
   Access patterns: ___________________________________
   Store choice + why: ________________________________
   Partition key + why: _______________________________

5. ARCHITECTURE  (5-9 boxes, arrows labelled)

6. THE HARD PART: _____________________________________
   Level 1: ___________________________________________
   Level 2: ___________________________________________
   Level 3 (failure mode + fix + cost): ________________

7. BOTTLENECKS
   SPOFs: _____________________________________________
   Serialised resources: ______________________________
   Unbounded growth: __________________________________
   At 10x, first thing to break: ______________________

8. FAILURE BEHAVIOUR
   Component slow: ____________________________________
   Component down: ____________________________________
   Degradation plan: __________________________________

9. TRADE-OFFS
   Chose ______ over ______; cost is ______; would flip if ______
   Chose ______ over ______; cost is ______; would flip if ______
```

---

[← previous: Interview playbook](12-interview-playbook.md) · [back to the handbook](../README.md)
