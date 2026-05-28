# Designing a Group Chat System (≤1000 users/group)

> *Partition by `group_id`, shard front-ends by `user_id`, persist before publish — and Kafka's consumer-group protocol gives single-writer-per-group for free.*

← [Back to Index](../README.md)

---

A system design walkthrough covering message routing, fan-out, durability, ordering, and scaling. Built up as a series of design decisions, each addressing a concrete concern.

**Severity legend:**
- 🔴 **Critical** — breaks correctness or causes data loss; must be addressed
- 🟠 **High** — significant performance, availability, or UX impact under realistic load
- 🟡 **Medium** — affects efficiency or operational smoothness; tune as system matures
- 🟢 **Low** — edge case or polish; worth knowing, often deferred

**Each decision has a 💡 Interview Recommendation** — a 3-sentence answer to lead with, plus framing for the alternatives you'd reject if pushed.

---

## TL;DR

| # | Question | Decision | Why | Seniority signal |
|---|---|---|---|---|
| [D1](#decision-1-shard-front-end-by-user_id-not-group_id) | How do we route user connections across the front-end fleet? | Shard by `user_id` (consistent hashing + sticky sessions), not by `group_id` | Users belong to many groups; group-pinning forces multi-WebSocket or cross-instance proxying, concentrates load, and amplifies failure blast radius | **Senior** — the naive answer (group-pinning for "free fan-out") is tempting; recognizing the many-to-many membership inverts it |
| [D3](#decision-3-redis-pubsub-for-fan-out-durable-store-as-source-of-truth) | What's the relationship between durable storage and Pub/Sub fan-out? | Persist to the durable store first, **then** publish to Pub/Sub with payload inline | Pub/Sub is at-most-once and allowed to lose messages; reverse order risks "ghost" messages users see live but can't reload | **Mid** — durability-before-notification is a well-known principle |
| [D4](#decision-4-per-group-monotonic-sequence-numbers) | How do we generate ordering identifiers for messages? | Per-group monotonic `seq`; `(group_id, seq)` uniquely identifies a message | Gap detection needs strict `+1` successor semantics — Snowflake "rough order" isn't enough; per-group scope avoids global contention | **Senior** — recognizing why timestamp-style IDs fail for gap detection is non-obvious |
| [D5](#decision-5-two-layer-gap-detection-client--instance) | How do we recover dropped or out-of-order messages? | Two-layer gap detection (client + instance) + idle reconciliation for invisible gaps | Pub/Sub drops are silent; client-only detection misses gaps when no follow-up message arrives; per-message acks don't scale | **Senior** — invisible-gap reconciliation is the part most candidates miss |
| [D6](#decision-6-kafka-as-ingestion-buffer-partitioned-by-group_id) | How do we serialize writes per group without a coordinator? | Kafka topic partitioned by `group_id`; one consumer per partition is the sole writer for every group on it | Partition assignment + consumer-group protocol gives single-writer-per-group for free — replaces what would otherwise be a placement service or distributed lock | **Senior** — using Kafka's protocol *as* the coordination primitive is the elegant move |
| [D7](#decision-7-long-lived-consumers-scaled-on-kafka-lag) | What signal do we auto-scale the consumer fleet on? | Kafka consumer lag, with asymmetric thresholds and `cooperative-sticky` assignment | I/O-bound consumers can be at low CPU yet falling behind; lag is the actual SLO. Asymmetric thresholds keep median consumer lifetime in hours so warmup amortizes | **Mid** — scaling on the SLO instead of a proxy is a standard principle |
| [D8](#decision-8-active-only-pubsub-subscriptions-per-user) | How do we keep Pub/Sub fan-out tractable as users × groups grows? | Subscribe to a group's channel only for users with that conversation actively open (5-min hysteresis); background groups use push notifications + on-demand catch-up | The bottleneck isn't subscription count — it's aggregate fan-out: every message hits every instance with even one user in the group, regardless of whether that user is looking. Cuts traffic 10–50× | **Senior** — recognizing the bottleneck is workload-shaped, not backend-shaped (so don't switch to NATS) |

---

## System Diagram

```
                          ┌──────────────────────────────────┐
                          │           Clients (apps)          │
                          │  user_id-stamped WebSocket conns  │
                          └────┬─────────────────────────┬────┘
                               │ WebSocket (long-lived)  │
                               │ + REST (history fetch)  │
                               ▼                         ▼
                    ┌─────────────────────────────────────────┐
                    │        Front-end Instance Fleet         │
                    │  (sharded by user_id, sticky sessions)  │
                    │                                         │
                    │  • Owns user WebSocket connections      │
                    │  • Subscribes to Pub/Sub for ACTIVE     │
                    │    groups of locally connected users    │
                    │  • Tracks last-seen seq per group       │
                    │  • Produces send-events to Kafka        │
                    └────┬────────────────────────────┬───────┘
                         │ produce                    ▲ subscribe
                         │ (key=group_id)             │ (channel=group:X)
                         ▼                            │
                    ┌─────────────────┐         ┌──────────────────┐
                    │   Kafka topic   │         │  Redis Pub/Sub   │
                    │ chat-messages   │         │  (fan-out hint   │
                    │ partitioned by  │         │   + payload)     │
                    │   group_id      │         └────────▲─────────┘
                    └────────┬────────┘                  │ publish
                             │ consume                   │
                             ▼                           │
                    ┌────────────────────────────────────┴─────┐
                    │           Writer Consumer Fleet          │
                    │   (1 consumer per partition at a time)   │
                    │                                          │
                    │  For each message:                       │
                    │   1. counter[group_id]++  (in-memory)    │
                    │      (lazy init from store on first hit) │
                    │   2. persist(group_id, seq, payload)     │
                    │   3. publish to Pub/Sub                  │
                    │   4. commit Kafka offset                 │
                    └────────────────┬─────────────────────────┘
                                     │ writes + reads (max seq, history)
                                     ▼
                    ┌──────────────────────────────────────────┐
                    │           Durable Message Store          │
                    │     (Cassandra / DynamoDB / similar)     │
                    │                                          │
                    │  Primary key: (group_id, seq)            │
                    │  Patterns:                               │
                    │   • Append on write                      │
                    │   • Range scan for catch-up / gap fill   │
                    │   • max(seq) for counter init / recon    │
                    └──────────────────────────────────────────┘
```

**Hot path (send → receive):**
Client → front-end (produce) → Kafka → consumer (assign seq, persist, publish) → Redis Pub/Sub → front-end instances → recipient WebSockets.

**Cold path (catch-up / gap fill):**
Client / front-end detects missing seq → REST / direct read against the durable store → fill gap.

---

## Decision Log

### Decision 1: Shard front-end by `user_id`, not `group_id`

**Choice:** Each user's WebSocket connection lives on one front-end instance, chosen by consistent hashing on `user_id`. Instances subscribe to Redis Pub/Sub channels for any group with a locally connected (and active) user.

**Concerns this addresses:**

| Concern | Severity | Notes |
|---|---|---|
| Users belong to many groups simultaneously | 🔴 Critical | Group-pinning would require either multiple WebSockets per user or cross-instance proxying — reinventing pub/sub badly |
| Hot instances from large active groups | 🟠 High | Group-pinning concentrates load; user-sharding spreads it naturally |
| Failure blast radius | 🟠 High | Group-pinning means 1000 simultaneous reconnects on instance death; user-sharding spreads reconnect storms |
| Deploy / rebalance churn | 🟡 Medium | Migrating whole groups during rolling deploys is painful; user-level sharding makes drains gentler |
| Routing layer complexity | 🟡 Medium | Group→instance maps need synchronization; user-level hashing is stateless |

**Alternatives considered:**

- **Group-pinning (all users of group X on one instance):** Tempting because fan-out becomes a local memory iteration. Rejected because users belong to many groups — you'd need either multi-WebSocket per user or cross-instance proxying. Only viable for broadcast rooms (Twitch model) where users participate in one room at a time.

> **💡 Interview Recommendation:**
> "I'd shard front-end instances by user_id with consistent hashing, not by group_id. Group-pinning is tempting because it makes fan-out a local memory iteration, but users belong to many groups — group-pinning forces either multiple WebSockets per user or cross-instance proxying, which reinvents pub/sub. User-sharding also spreads hot-group load and reconnect storms naturally across the fleet."
>
> **If pushed on group-pinning:** "It's the right answer for broadcast rooms — Twitch chat, live-stream events — where users watch one stream at a time and fan-out cost dominates everything else. For conversation chat where users are in 50+ groups, it doesn't work."

---

### Decision 2: Session affinity via consistent hashing on `user_id`

**Choice:** Once authenticated, requests and reconnects route to a deterministic instance based on `hash(user_id)`. Combined with cookie-based stickiness on the load balancer for the WebSocket upgrade handshake.

**Concerns this addresses:**

| Concern | Severity | Notes |
|---|---|---|
| Presence registry races | 🟠 High | Without affinity, the registry frequently points at a stale instance during reconnect storms |
| Local cache reuse (group memberships, recent msgs) | 🟡 Medium | Sticky reconnects reuse cached state; cold reconnects pay re-warm cost |
| Reduced fine-grained load shedding | 🟢 Low | LB can't reroute individual users away from a hot instance — accept this; rebalance at connection establishment instead |

**Alternatives considered:**

- **IP hash:** Simple, no cookie. Rejected because NAT collapses many users to one IP and mobile clients change IPs frequently.
- **Cookie-only stickiness without user_id hashing:** Works for HTTP but doesn't give you a deterministic mapping — useful for the upgrade handshake, but you still want consistent hashing for reconnection routing.
- **No affinity (random routing):** Acceptable for stateless services but creates presence registry races and forces every reconnect to rebuild local cache.

> **💡 Interview Recommendation:**
> "Session affinity via consistent hashing on user_id, combined with cookie-based stickiness on the load balancer for the WebSocket upgrade handshake. The hashing gives a deterministic instance for reconnects, which keeps the presence registry consistent and lets the instance reuse cached state like group memberships. The tradeoff is reduced fine-grained load shedding — you can't reroute individual users away from a hot instance — but for WebSockets, that's acceptable since the connection itself is inherently sticky."
>
> **If pushed on IP hash:** "Breaks under NAT and changes when mobile clients move networks; user_id hashing is more stable and meaningful."

---

### Decision 3: Redis Pub/Sub for fan-out, durable store as source of truth

**Choice:** Persist message to durable store with monotonic per-group `seq`, *then* publish to Redis Pub/Sub. Pub/Sub carries the full payload for small messages (text); large attachments go to object storage with a reference in the Pub/Sub message.

**Concerns this addresses:**

| Concern | Severity | Notes |
|---|---|---|
| Pub/Sub is at-most-once (no buffering, no retry) | 🔴 Critical | Without durable backing, any subscriber blip loses messages permanently |
| Hot-path latency to recipients | 🟠 High | Payload-in-Pub/Sub avoids the extra round-trip per message |
| Durable store overload from per-message reads | 🟠 High | Notification-only design hits the store on every fan-out × every subscribed instance |
| Large payload bloat in Pub/Sub | 🟡 Medium | Mitigated by sending object-storage references for attachments |
| Persist-then-publish ordering | 🔴 Critical | Reverse order risks "ghost" messages users see live but can't reload |

**Alternatives considered:**

- **Pub/Sub-only (no durable store):** Unacceptable — at-most-once means any subscriber blip loses messages permanently.
- **Pub/Sub as notification only, subscribers fetch from store:** Adds a round-trip per message and inverts read/write ratio on the store badly. Acceptable if messages are very large and Pub/Sub bandwidth is the constraint, but for typical chat text, hot-path latency wins.
- **Redis Streams instead of Pub/Sub:** Provides at-least-once delivery natively with consumer groups, but adds memory cost and slightly more complex consumer logic. Worth considering if you don't already have a durable store doing the same job.
- **Kafka for fan-out (instead of Redis Pub/Sub):** Higher latency per fan-out hop; Kafka is sized for throughput, not low-latency broadcast. Better in the ingestion layer.

> **💡 Interview Recommendation:**
> "Persist to a durable store with a monotonic per-group seq, then publish to Redis Pub/Sub with the full payload — the store is source of truth, Pub/Sub is the low-latency delivery layer that's allowed to lose messages. Pub/Sub-only doesn't work because it's at-most-once, but notification-only Pub/Sub forces every subscriber to fetch from the store on every message, which kills latency and overloads the store. Carrying the payload in Pub/Sub for small messages and a reference for large attachments is the right balance."
>
> **If pushed on Redis Streams:** "Streams give at-least-once natively, but I already have a durable store doing that job — adding Streams duplicates the durability guarantee. I'd consider it if I didn't have a separate durable store."

---

### Decision 4: Per-group monotonic sequence numbers

**Choice:** `seq` is monotonic *within* each `group_id`. The tuple `(group_id, seq)` uniquely identifies a message. Generated in-memory by a single writer per group (see Decision 6).

**Concerns this addresses:**

| Concern | Severity | Notes |
|---|---|---|
| Detecting missing messages | 🔴 Critical | Without strict +1 successor semantics, gap detection is impossible |
| Global counter contention | 🟠 High | Per-group scope eliminates cross-group contention |
| Concurrent writers reordering same-group messages | 🟠 High | Two INCR operations can race; resolved by serializing writes per group (Decision 6) |
| "Burned" seq numbers from failed writes | 🟡 Medium | Clients treat fetch results as truth; advance last_seen to actual max returned |

**Alternatives considered:**

- **Global monotonic seq:** Pointless — no one needs the ordering across groups, and you create a system-wide contention point.
- **Snowflake-style timestamped IDs:** "Roughly ordered" is insufficient — gap detection requires exact +1 successor relationships.
- **Per-group Postgres sequences:** Don't scale to millions of groups; `CREATE SEQUENCE` per group is operationally absurd.
- **Cassandra LWT (`IF NOT EXISTS`):** 4x latency of normal write due to Paxos, and contention forces retries. Fine for low write rates only.
- **Database-assigned auto-increment with row-level lock:** Works but contends on the counter row. Acceptable at small scale.

> **💡 Interview Recommendation:**
> "Sequence numbers are per-group monotonic — the tuple (group_id, seq) uniquely identifies a message. Global ordering would be a contention bottleneck and isn't useful since clients only need ordering within a group, and Snowflake-style IDs aren't enough because gap detection requires exact +1 successor semantics. The counter is generated by a single writer per group, which I'll get to in the Kafka section."
>
> **If pushed on Snowflake IDs:** "Snowflake gives rough ordering, but the client can't tell '5 and 7 arrived, did I miss 6 or is there no 6?' — gap detection needs strict successor semantics."

---

### Decision 5: Two-layer gap detection (client + instance)

**Choice:** Every recipient layer tracks `last_seen_seq` per group.
- **Client:** on receiving `seq > last_seen + 1`, buffer the new message, fetch missing range from store via REST, deliver in order.
- **Instance:** on receiving Pub/Sub message with `seq > last_delivered + 1` for a group, fetch missing range from store, fan out in order.
- **Idle reconciliation:** periodic check (client heartbeat or server-side poll of `max(seq)`) catches "invisible gaps" where no follow-up message exposes the loss.

**Concerns this addresses:**

| Concern | Severity | Notes |
|---|---|---|
| Pub/Sub silently dropped a message | 🔴 Critical | Instance-layer detection recovers before clients ever notice |
| Client missed messages during a network blip | 🔴 Critical | Client-layer detection + buffered out-of-order delivery |
| Invisible gap (no follow-up message arrives) | 🟠 High | Without reconciliation, users see nothing missing — for hours |
| Out-of-order display jank | 🟡 Medium | Buffer the K+1 message until gap fills; deliver K, then K+1 |
| Duplicate messages from retries | 🟡 Medium | Drop messages with `seq ≤ last_seen` |
| Reconnect catch-up cost | 🟢 Low | Single range scan per group; cheap with `(group_id, seq)` primary key |

**Alternatives considered:**

- **Client-only gap detection:** Misses the "invisible gap" case where the instance dropped a message and no follow-up arrives — clients have no way to know.
- **Acks for every message:** Conceptually simpler, but at high fan-out it's an ack storm. Pull-on-gap-detect is more efficient because it amortizes recovery cost over many messages.
- **Total ordering at the Pub/Sub layer (e.g., Kafka):** Removes the need for gap detection but trades off latency and adds operational complexity to the fan-out path.

> **💡 Interview Recommendation:**
> "Gap detection happens at two layers. Each message carries a per-group seq; clients track last-seen seq per group, and on receiving seq > last_seen + 1 they buffer the new message and fetch the gap from the durable store before delivering in order. Instances do the same thing for their Pub/Sub subscriptions — that way a Pub/Sub drop between Redis and the instance is recoverable without the client noticing. For invisible gaps where no follow-up message arrives, periodic reconciliation via heartbeat covers it."
>
> **If pushed on per-message acks:** "Acks scale poorly at high fan-out — you'd be inverting the read/write ratio. Pull-on-detection only pays the recovery cost when something actually goes wrong."

---

### Decision 6: Kafka as ingestion buffer, partitioned by `group_id`

**Choice:** Front-end instances produce send-events to a Kafka topic partitioned by `group_id`. A consumer fleet reads partitions, assigns `seq` from in-memory counters, persists, and publishes to Pub/Sub.

**The single-writer property falls out for free:**
- All messages for group X hash to one partition.
- Kafka's consumer group protocol guarantees one consumer per partition at a time.
- Therefore one consumer is the sole writer for group X — no coordination service, no leases, no placement service.

**Concerns this addresses:**

| Concern | Severity | Notes |
|---|---|---|
| Concurrent writes per group race on seq | 🔴 Critical | Partition order + single-consumer assignment serializes by construction |
| Durable buffer if store is slow / unavailable | 🟠 High | Kafka absorbs back-pressure; front-ends don't block users |
| Replay after consumer crash | 🟠 High | Offset commit protocol resumes from last successful processing |
| Decoupling user-facing latency from store write latency | 🟡 Medium | Front-ends ack on Kafka produce; persist happens async |
| Consumer rebalance duplicate processing | 🟡 Medium | Mitigated by `(group_id, seq)` uniqueness on persist (idempotency) |
| Added latency from extra hops (~20-50ms) | 🟢 Low | Acceptable for chat; humans don't notice <100ms |

**Alternatives considered:**

- **Direct front-end → durable store writes:** Lower latency but no durable buffer when the store is slow, and you need a separate placement service to enforce single-writer-per-group. Lots of complexity to replace what Kafka gives free.
- **Redis INCR for seq + multiple writers:** Atomic but lets concurrent writers interleave — produces reorder events that trigger spurious gap-fill fetches. Works, but not cleanly.
- **Coordination service (etcd/ZooKeeper) for group ownership:** Explicit but adds another moving part. Kafka's consumer group protocol is the same idea, simpler to operate.
- **Kafka with no per-group partitioning (random partition):** Loses ordering guarantee; can't serialize seq assignment.

> **💡 Interview Recommendation:**
> "Kafka as the ingestion layer with group_id as the partition key. That gives single-writer-per-group for free — Kafka's consumer group protocol guarantees one consumer per partition, so the consumer handling a partition is the sole writer for every group hashed to it. Seq is an in-memory counter on the consumer, persisted to the durable store, and then published to Pub/Sub. Front-end instances ack the sender on Kafka produce success, which decouples user-facing latency from store write latency."
>
> **If pushed on direct writes (no Kafka):** "Direct writes need a separate placement service to enforce single-writer-per-group, plus a back-pressure mechanism when the store is slow — Kafka gives me both for free, at the cost of ~20-50ms added latency."

---

### Decision 7: Long-lived consumers, scaled on Kafka lag

**Choice:** Consumers run for hours to days (deploy-to-deploy). Auto-scale on Kafka consumer lag with asymmetric thresholds. Over-partition the topic upfront (2-3× peak consumer count).

**Why long-lived matters:** the in-memory counter map is a cache derived from the store. Restarts cost a lazy warm-up window. Amortized over millions of messages, this is negligible — *if* the consumer lives long enough.

**Lifetime guidance:**

| Lifetime | Verdict | Notes |
|---|---|---|
| Hours to days | ✅ Ideal | Warmup and rebalance are negligible overhead |
| 30+ minutes | ✅ Healthy | Small overhead, not a concern |
| 5-30 minutes | ⚠️ Acceptable but suboptimal | Investigate why churn is happening |
| 1-5 minutes | 🟠 Problematic | Overhead is significant; system feels twitchy |
| < 1 minute | 🔴 Broken | Rebalancing more than processing |

**Concerns this addresses:**

| Concern | Severity | Notes |
|---|---|---|
| Consumer thrash from aggressive auto-scaling | 🟠 High | Asymmetric thresholds + cooldowns keep median lifetime in hours |
| Partition count as hard ceiling on parallelism | 🟠 High | Over-partition upfront — adding partitions later changes hash mapping |
| Wrong scaling signal (CPU vs. lag) | 🟠 High | I/O-bound consumers can be at low CPU yet falling behind; lag is truth |
| Counter map memory growth over weeks | 🟡 Medium | LRU-evict cold groups; next message lazy-reinits |
| Rebalance latency during scale events | 🟡 Medium | Use `cooperative-sticky` assignor — only moving partitions are paused |
| Consumer crash → counter rebuild stall | 🟡 Medium | Lazy init means only active groups pay the cost; spreads organically |
| Hot partitions (one very chatty group) | 🟢 Low | Rarely an issue at 1000-user cap; over-partitioning helps spread |

**Alternatives considered:**

- **CPU-based scaling:** I/O-bound consumers can be at 30% CPU and still falling behind. Kafka lag directly measures whether consumers are keeping up.
- **Reactive minute-level scaling:** Causes consumer thrash — short lifetimes mean rebalance overhead dominates useful work.
- **Eager warmup of all counters on startup:** Slower startup; most groups in a partition are idle at any moment, so lazy is cheaper.
- **Persisting counters in Redis/external store:** Removes the warm-up cost on restart but adds a network round-trip per message. Lazy init from `max(seq)` is simpler and only pays the cost once per group per consumer lifetime.

**Recommended config:**
- Partitions: 2-3× peak consumer count (e.g., 200 partitions for ~50-80 peak consumers)
- Scaling signal: per-partition Kafka lag (KEDA Kafka scaler)
- Scale up: +30% capacity if lag > 5K msgs/partition for 60s
- Scale down: -10% capacity if lag < 500/partition for 10 min
- Stabilization window: 5-15 min after any scale event
- Assignor: `cooperative-sticky`

> **💡 Interview Recommendation:**
> "Consumers are long-lived — typically deploy-to-deploy, so days to weeks. The scaling signal is Kafka consumer lag, not CPU, because I/O-bound consumers can be at low CPU and still falling behind. I'd use asymmetric thresholds — scale up aggressively, scale down conservatively with a 10-15 minute stabilization window — to keep median consumer lifetime in the hours range, since shorter lifetimes mean rebalance and warmup overhead start dominating useful work."
>
> **If pushed on CPU-based scaling:** "CPU is a noisy proxy; lag is the actual SLO. Same reason you wouldn't scale a database on CPU instead of query latency."

---

### Decision 8: Active-only Pub/Sub subscriptions per user

**Choice:** Instances subscribe to a group's Pub/Sub channel only when a local user has that conversation actively open or recently active (5-min hysteresis). Background conversations rely on push notifications and on-demand catch-up via the durable store.

**Why this matters:** With users sharded by user_id and each user in ~50 groups, an instance with 100K users would nominally subscribe to millions of channels. But the real cost isn't subscription count — it's **aggregate Pub/Sub fan-out**: every message gets delivered to every instance with even one user in that group, regardless of whether that user has the conversation open. Most fan-outs are wasted work.

**Concerns this addresses:**

| Concern | Severity | Notes |
|---|---|---|
| Subscription cardinality per instance | 🟠 High | Reduces by 10-50x (50 groups → 3-5 active per user) |
| Pub/Sub aggregate fan-out cost | 🟠 High | Inactive users don't drive Pub/Sub traffic |
| Per-instance inbound message rate | 🟠 High | Most messages go to instances that actually have active users for that group |
| Background message delivery latency | 🟡 Medium | Tradeoff: push notifications + catch-up replace live delivery |
| Subscription churn during foreground/background transitions | 🟡 Medium | 5-min unsubscribe delay prevents thrashing |
| Push notification system as additional dependency | 🟡 Medium | Most chat apps need this anyway |

**Alternatives considered:**

- **Group-aware routing (place users of same group on same instance):** Requires a placement service that knows group memberships and rebalances as memberships change — cross-cuts the "stateless instance" property and adds significant operational complexity. Benefit is unclear because users belong to many groups, so optimizing for one group pessimizes another. Rejected.
- **Gateway tier between Pub/Sub and edge instances:** Right answer at very large fleet sizes (hundreds of edge instances) where active-only subscriptions still aren't enough. Adds a whole new component to deploy, monitor, and scale, plus a new protocol between gateways and edges. Defer until active-only hits its ceiling.
- **Different Pub/Sub backend (NATS, Kafka for fan-out):** Gives server-side filtering and more efficient subscription handling, but solves the wrong problem — the bottleneck is workload pattern (instances care about too many groups), not the backend itself. Rejected unless rearchitecting for unrelated reasons.

> **💡 Interview Recommendation:**
> "With users sharded by user_id and ~50 groups per user, instances would nominally subscribe to millions of channels — but the real cost isn't subscription count, it's aggregate Pub/Sub fan-out, since every message gets delivered to every instance that has even one user in that group. The fix is active-only subscriptions: instances subscribe to a group's channel only when a local user has the conversation foregrounded or recently active, with short hysteresis to avoid churn — background conversations rely on push notifications plus on-demand catch-up from the durable store when reopened. This cuts subscription cardinality and per-instance inbound traffic by 10-50x, since users typically have ~3-5 active conversations at any moment out of their 50 memberships."
>
> **If pushed on gateway tier:** "It's the right next step at very large fleet sizes — WhatsApp's chatd works similarly — but I'd defer it until active-only subscriptions actually hit their ceiling. Adding it preemptively introduces a new component for a problem the simpler optimization already solves."
>
> **If pushed on group-aware routing:** "It tries to co-locate users of the same group on fewer instances, but users are in many groups simultaneously — optimizing for one group's locality pessimizes another's. The benefit is unclear, the cost (placement service, rebalancing logic) is significant."
>
> **If pushed on switching to NATS or Kafka for fan-out:** "That solves a backend problem; the fan-out cardinality issue is a workload problem. Active-only subscriptions address the workload at much lower architectural cost."

---

## Cross-Cutting Concerns

### Idempotency

| Concern | Severity | Mitigation |
|---|---|---|
| Consumer crash between persist and offset commit → reprocess on restart | 🔴 Critical | `(group_id, seq)` uniqueness constraint; or store an idempotency key (Kafka offset / client `message_id`) on persist |
| Client retry on send (network flake) | 🟠 High | Client-assigned `message_id` deduped at ingestion or persist layer |
| Duplicate Pub/Sub delivery | 🟢 Low | Subscribers drop `seq ≤ last_seen` |

> **💡 Interview Recommendation:**
> "Idempotency is enforced at the persist step using either a (group_id, seq) uniqueness constraint or a client-assigned message_id as a dedup key. The main case I care about is consumer crashes between persist and Kafka offset commit — on restart, the consumer re-reads and tries to persist again, and the uniqueness check makes the second write a no-op."

### Failover & Recovery

| Concern | Severity | Mitigation |
|---|---|---|
| Front-end instance death → 1000s of WebSocket reconnects | 🟠 High | User-level sharding + sticky hashing spreads reconnect load across surviving fleet |
| Consumer death → partition reassignment + counter rebuild | 🟡 Medium | Lazy-init handles it gracefully; new consumer queries `max(seq)` per group on first message |
| Redis Pub/Sub instance death | 🟡 Medium | Acceptable: durable store is source of truth; clients/instances detect gap on next message and recover |
| Kafka broker death | 🟢 Low | Replication factor ≥ 3; producers retry; consumers rebalance |
| Durable store partial failure | 🔴 Critical | Multi-AZ replication; Kafka acts as buffer for transient unavailability |

### Latency Budget (typical, sender → recipient)

| Hop | Cost | Notes |
|---|---|---|
| Client → front-end (produce send) | 5-10ms | Network |
| Front-end → Kafka produce ack | 5-10ms | `acks=all` with replication |
| Consumer reads from Kafka | <5ms | Long-poll, in-cluster |
| Persist to durable store | 5-20ms | Cassandra/DynamoDB write |
| Publish to Redis Pub/Sub | 1-2ms | In-memory, single round-trip |
| Front-end → recipient WebSocket | 5-10ms | Network |
| **End-to-end** | **30-60ms** | Comfortably under 100ms |

---

## Scaling Bands: When the Design Breaks

The decisions above target the conversation-chat use case — groups up to ~10K users with high participation. Beyond that, the design needs adjustments:

| Group Size | What Holds | What Changes |
|---|---|---|
| 1K-10K | Everything | None |
| 10K-100K | Architecture intact | Shard Pub/Sub across Redis cluster; batch persists in writer; jittered reconnect backoff |
| 100K-1M+ | Persistence and consumer scaling | **Structural shift:** hierarchical fan-out tier, room-sharded routing, relaxed ordering. Run two architectures side by side — conversation chat (this design) for small/medium groups, broadcast chat (Twitch model) for large public rooms. |

> **💡 Interview Recommendation when asked to remove the user cap:**
> "Up to 10K users per group, the design holds with no changes. From 10K to 100K, I'd shard Pub/Sub across a Redis cluster and batch persists in the writer to handle higher per-group throughput — same architecture, tuned harder. Beyond 100K, the model fundamentally changes: fan-out cost dominates everything else, and the assumption that 'users belong to many groups' inverts at broadcast scale where viewers participate in one room at a time. So I'd run two architectures side by side — conversation chat for the long tail of small/medium groups, broadcast chat with hierarchical fan-out and room-sharded routing for large public rooms. The decision boundary is around 100K members."

---

## Evaluation: How Well Does This Hold Up?

### Strengths

- **Correctness under partial failure.** The durable store + per-group seq + two-layer gap detection means no message is permanently lost from any single component failure. Recovery is automatic.
- **Single-writer-per-group without explicit coordination.** Kafka's partition assignment protocol replaces what would otherwise be a placement service — significant complexity savings.
- **Decoupled write path from read path.** Front-end instances handle WebSockets; consumer fleet handles persistence and fan-out. Each scales independently.
- **Bounded fan-out.** 1000-user-per-group cap means Pub/Sub fan-out per message is predictable and small.
- **Generalizes to other patterns.** "Partitioned log + long-lived consumers with local state" is the same pattern Kafka Streams / Flink / Samza use — the design isn't bespoke.

### Weaknesses & Open Questions

| Concern | Severity | Notes |
|---|---|---|
| Added latency from Kafka hop (~20-50ms vs. direct write) | 🟢 Low | Fine for chat; would matter more for trading or gaming |
| Cross-region presence and message delivery not addressed | 🟠 High | Single-region design assumed; multi-region requires regional Kafka clusters + cross-region replication strategy |
| Read-after-write consistency for the sender's own messages | 🟡 Medium | Sender sees their message via Pub/Sub round-trip, not local echo — small but noticeable lag |
| No discussion of message editing, deletion, reactions | 🟡 Medium | Same per-group seq mechanism extends; CRDTs may be needed for concurrent edits |
| End-to-end encryption not addressed | 🟠 High | If E2EE required, server can't see message contents — fan-out and search require redesign |
| Client offline → push notifications | 🟠 High | Real systems need a notification fan-out separate from WebSocket delivery (and Decision 8 depends on it) |
| Spam, abuse, rate limiting | 🟡 Medium | Front-end ingress is the right place; per-user and per-group limits needed |
| Message ordering across groups (a user sees msgs from groups A and B) | 🟢 Low | No global ordering attempted — usually fine, occasionally surprising |

### What Would Break This Design

- **Group size growing to 100K+.** Pub/Sub fan-out cost dominates; would need group-pinned broadcast architecture (Twitch model).
- **Strict global ordering requirements.** Can't be satisfied with per-group seq alone.
- **Sub-10ms end-to-end SLAs.** Kafka hop is the wrong tradeoff; would need direct write paths with separate durability mechanism.
- **Multi-region active-active with low cross-region latency.** Per-group single-writer doesn't trivially extend to "writer in region A and writer in region B simultaneously"; conflict resolution strategy required.

---

## Interview Cheat Sheet

When walking through this design, the order that builds well:

1. **Routing:** "User-level sharding, not group-level — users are in many groups."
2. **Fan-out:** "Pub/Sub for low-latency delivery, durable store as source of truth."
3. **Ordering & gaps:** "Per-group monotonic seq with two-layer gap detection."
4. **Single writer:** "Kafka partitioned by `group_id` gives us this for free."
5. **Scaling consumers:** "Long-lived consumers, scale on lag, over-partition upfront."
6. **Subscription cost:** "Active-only subscriptions to keep Pub/Sub fan-out tractable."
7. **Failure modes:** Walk through what happens when each component dies.
8. **Scale beyond cap:** Two architectures — conversation chat and broadcast chat — split around 100K members.

The senior-level signal is **naming the tradeoff explicitly** at each step — not presenting the chosen design as obvious. For each decision, the framing is consistent:
1. Acknowledge what the alternative does well
2. Name the cost (complexity, deferred suitability, or wrong problem)
3. State the trigger condition for revisiting

That structure signals you understand **the design space, not just one design** — which is what distinguishes senior from mid-level system design answers.

---

## Further Reading

- [*Designing Data-Intensive Applications* — Martin Kleppmann](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/) — Ch. 11 on stream processing is the textbook version of "partitioned log + long-lived consumers with local state"; Ch. 5 covers replication and ordering tradeoffs that show up in every decision here
- [Apache Kafka documentation — Consumer Groups](https://kafka.apache.org/documentation/#intro_consumers) — primary source for the partition-assignment guarantees Decision 6 leans on
- [Jepsen — Kafka analyses](https://jepsen.io/analyses/kafka) — empirical look at Kafka's actual durability and ordering guarantees under failure
- [Redis Pub/Sub vs Streams](https://redis.io/docs/latest/develop/interact/pubsub/) — official docs covering the at-most-once semantics that motivate Decision 3

---

← [Back to Index](../README.md)
