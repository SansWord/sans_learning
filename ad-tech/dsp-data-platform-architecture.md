# DSP Data Platform Architecture

A system-design sketch of a **DSP data platform** that handles real-time bidding, event ingestion, attribution, and reconciliation with MMPs. The core idea: a **dual real-time + batch pipeline (Lambda architecture)** on a Kafka backbone. Pairs with [`programmatic-advertising-fundamentals.md`](programmatic-advertising-fundamentals.md), which covers the ad-tech concepts behind it.

---

## 1. The requirements (start here)

Two forces pulling in opposite directions — name the tension up front:

- **Low latency, high throughput, "good enough" accuracy** — bidder responds in **~100ms**; dashboards & bid-optimization need conversion signal in **seconds**; volume is **billions of events/day**.
- **Eventual, exact accuracy** — billing, MMP reconciliation, and reported metrics must be **complete and correct**, handling late-arriving conversions and full cross-source dedup.

One pipeline can't satisfy both → that's *why* the design is dual-path. Lead with this justification.

---

## 2. Architecture diagram

```
                         ┌────────────────────────────────────────────────┐
   SSP / Ad Exchanges ──▶│  RTB BIDDER  (~100ms budget)                   │
   (bid requests)        │  reads: budget, freq-cap, audience, ML model   │
                         │  from low-latency KV (Redis/Aerospike)         │
                         └───────────────┬────────────────────────────────┘
                                         │ emits bid/win/imp logs
   user device ─ impressions, clicks ───▶│
   MMP ─ conversion postbacks ──────────▶│
                                         ▼
                          ┌───────────────────────────┐
                          │          KAFKA            │  ← central event log
                          │ topics: bids, imps,       │     (durable, replayable)
                          │ clicks, conversions       │
                          └───┬───────────────────┬───┘
              HOT PATH        │                   │        COLD PATH
        (seconds, approx)     ▼                   ▼     (hourly/daily, exact)
              ┌────────────────────────┐   ┌───────────────────────────┐
              │ STREAM PROCESSOR       │   │ S3 DATA LAKE (raw Parquet)│ ← source of truth
              │ (Flink / Spark SS)     │   └────────────┬──────────────┘
              │ • real-time attribution│                │
              │ • spend / freq counters│                ▼
              │ • feeds bid model      │     ┌───────────────────────────┐
              └───┬──────────────┬─────┘     │ AIRFLOW → SPARK (batch)   │
                  │              │           │ • authoritative re-attrib │
       ┌──────────▼──┐   ┌───────▼────────┐  │ • dedup + MMP reconcile   │
       │ KV stores   │   │   PINOT        │  │ • billing, log export     │
       │ (bidder     │   │ live advertiser│  └──────┬──────────┬─────────┘
       │  counters)  │   │  dashboards)   │         │          │
       └─────────────┘   └────────────────┘         ▼          ▼
                                            ┌──────────┐  ┌───────────────┐
                                            │SNOWFLAKE │  │ PINOT (offline│
                                            │(finance, │  │ segments —    │
                                            │ internal)│  │ corrected)    │
                                            └──────────┘  └───────────────┘
                                  Trino queries across lake + warehouse (ad-hoc)
```

---

## 3. The flows

**Bid path (hot, <100ms):** exchange sends a bid request → bidder reads budget remaining, frequency cap, audience membership, ML bid score from **low-latency KV stores** (Redis/Aerospike — *not* the warehouse; analytics DBs are too slow for the bid path) → responds. Every bid/win/impression emitted as an event.

**Event ingestion:** impressions, clicks, and **MMP conversion postbacks** all land in **Kafka**. Kafka is the backbone — decouples producers from many consumers, absorbs the firehose, and is **replayable** (reprocess if a downstream job has a bug).

**MMP integration sits on Kafka:** a tracking service consumes click/impression events and fires **real-time tracking URLs to the MMP** (DSP→MMP); a postback-receiver endpoint ingests **MMP→DSP conversion postbacks** and republishes them into Kafka.

**Hot path (stream processor — Flink / Spark Structured Streaming):**
- **Real-time attribution** — stateful, windowed join of conversions against recent clicks/impressions.
- **Spend & frequency counters** — written back to the bidder's KV stores so budgets pace and caps hold *in real time*.
- **Feeds conversion signal back to the bid model / feature store** for live optimization.
- Sinks fresh aggregates to **Pinot** → **advertiser-facing live dashboards** (low latency, high concurrency — Pinot's sweet spot; ingests straight from Kafka).

**Cold path (batch — Airflow orchestrating Spark):**
- Raw events land in the **S3 data lake** (Parquet) — the immutable **source of truth**.
- Nightly/hourly Spark jobs do **authoritative re-attribution** with all late-arriving conversions + **full dedup**, **reconcile against MMP daily files** (claimed vs. awarded, discrepancy reports), compute **billing**, produce **log-level exports** for advertisers.
- Corrected results load into **Snowflake** (finance/internal) and **Pinot offline segments** (overwrite the approximate real-time numbers).
- **Trino** on top for ad-hoc SQL across lake + warehouse without moving data.

### Audience data: loading & eviction (DMP/CDP → bidder)

The bidder's `audience` lookup (in the KV store) is fed by the **DMP/CDP**, but *how* that data gets there is a problem in itself — driven by two constraints: a **user-ID space in the billions** (too big for one box's RAM) and a **sub-millisecond lookup budget** (can't call the DMP synchronously at bid time).

**Loading strategy — pre-load + incremental, never synchronous:**
- **Bulk / batch sync** — DMP delivers segment data as **file drops** (daily S3 dumps of `segment → users`, or deltas); a pipeline loads them into the KV store. Establishes the baseline.
- **Streaming / incremental updates** — as users enter/exit segments, deltas flow over **Kafka** to keep the store fresh between bulk loads. (Same batch-baseline + stream-freshness split as the rest of the platform.)
- **Never** a synchronous DMP call during the auction — it must already be in the fast store.

**Can you load everything into memory? No — and you shouldn't:**
- Won't fit economically → use a **distributed, tiered store**: **Aerospike** (RAM index + SSD data, billions of keys, sub-ms) is the canonical ad-tech choice; or sharded **Redis**. Shard by user ID; no single box holds everything.
- Most stored users are dead weight → you only bid on users who **appear in bid requests**, so the set worth keeping warm ≈ **recently-active users**, far smaller than the universe of IDs. *This is what makes eviction correct, not just necessary.*

**Eviction — TTL primary, LRU as backstop:**
- **TTL (primary)** — segment memberships have **business-defined recency** (a "cart abandoner" segment has a 7/30-day window), so TTL = the segment's recency window. Stale memberships expire on their own; churned/dead cookies clear naturally. This is *semantically correct* eviction.
- **LRU/LFU (secondary)** — a memory-pressure safety valve so the store can't overflow.
- **Why TTL over LRU as primary:** you don't know in advance who you'll see — a user can reappear after a gap. LRU might evict someone right before they show up while keeping recent-but-stale data. TTL keys on the *business meaning* of the data, which is the right axis.

**Memory optimizations (fit more for less):**
- **Roaring bitmaps / bitsets** — encode `segment → user IDs` (or `user → segment bitset`) compactly; huge savings vs. string lists + fast set ops.
- **Bloom filters** — answer "is this user in *any* targeted segment?" in a few bits, short-circuiting the common not-in-segment miss.
- **Tiered storage** (Aerospike RAM-index over SSD) — don't pay DRAM prices for cold data.

**Consistency:** eventually consistent, and **slight staleness is acceptable** — bidding on someone who left a segment 10 min ago costs one mistargeted bid, not correctness. Tolerating that is exactly what permits the async pre-load model over synchronous DMP calls.

---

## 4. The crux: how the two paths reconcile

This is the heart of the design — and the answer to "DSP vs MMP, who's right?":

- **Real-time path = fast but approximate.** Attributes on data it has *now*; misses late conversions, can't do full cross-source dedup. Good enough for dashboards + optimization within seconds.
- **Batch path = slow but exact.** Recomputes the *truth* daily with complete data + MMP reconciliation, then **overwrites the real-time approximation** in the serving layer.

Dashboards show live (approximate) numbers during the day → **reconciled to authoritative batch numbers overnight.** Batch is the **system of record for billing**; real-time is for **steering**. That's the DSP-vs-MMP reconciliation problem expressed as architecture.

---

## 5. Key design decisions / trade-offs (say these unprompted)

- **Why not one path?** Batch-only can't optimize bids in real time; stream-only can't guarantee billing-grade accuracy with late data. Lambda buys both — at the cost of **maintaining attribution logic twice** (downside; **Kappa** = stream-only with replay is the alternative to name-drop).
- **Why KV stores in the bid path, not the warehouse?** 100ms budget; analytics stores can't serve point lookups that fast. Separate **operational serving** from **analytical** layers.
- **Why Kafka in the middle?** Decoupling + **replayability** — when an attribution bug ships, reprocess from the log instead of losing data.
- **Why Pinot for dashboards, not Snowflake?** Advertiser-facing = high concurrency + sub-second + fresh-from-stream. Snowflake is for internal/finance analytical queries, not thousands of advertisers hitting live dashboards.
- **Lineage & governance** — every reported/billed number must trace back through the lake to raw events, so you can **defend it when the advertiser disputes it against the MMP**.
- **Idempotency / exactly-once** — postbacks and events get retried/duplicated; dedup keys + idempotent writes prevent double-counting conversions.

---

## 6. The one-paragraph verbal version

*"I'd build a Lambda-style dual pipeline on a Kafka backbone. The bidder serves at ~100ms off low-latency KV stores. All events — impressions, clicks, MMP postbacks — flow into Kafka. A real-time stream processor does approximate attribution, updates spend/frequency counters back to the bidder, feeds the bid model, and pushes live numbers to Pinot for advertiser dashboards. In parallel, raw events land in an S3 lake; Airflow-orchestrated Spark jobs do authoritative re-attribution with late data, full dedup, and MMP reconciliation, then load corrected numbers into Snowflake and overwrite Pinot. The real-time path is for steering and live dashboards; the batch path is the billing-grade source of truth, and it reconciles nightly. Kafka gives me replayability, lineage lets me defend every billed number against the MMP, and idempotent writes keep retried postbacks from double-counting."*

---

## Glossary recap (terms feeding this design)

| Term | One-liner |
|---|---|
| **DSP** | Buy-side bidding brain; wins impressions, returns an ad tag (pointer), not the creative. |
| **MMP** | Neutral mobile attribution arbiter (AppsFlyer/Adjust/Branch); dedupes across all sources, credits one. |
| **Postback** | MMP→DSP "you won this conversion" (real-time); also DSP→MMP click/impression tracking (real-time). |
| **Attribution window / model** | Time bound + rule (last-click, view-through) for crediting a conversion to an ad. |
| **Kafka** | Durable, replayable event log — the real-time backbone. |
| **Spark** | Distributed batch/stream processing — heavy ETL + re-attribution. |
| **Airflow** | Workflow orchestrator (DAGs) for the batch pipeline. |
| **Pinot** | Real-time OLAP — low-latency, high-concurrency advertiser dashboards. |
| **Snowflake** | Cloud data warehouse — internal/finance analytics. |
| **Trino/Presto** | Federated SQL query engine over lake + warehouse. |
| **S3 data lake** | Cheap raw-event storage; immutable source of truth. |
| **Lineage** | Traceable map of data flow — lets you defend reported numbers. |

---

← [Back to Index](../README.md)
