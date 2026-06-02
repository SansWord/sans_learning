# sans_learning

Notes from random things I'm learning. No curriculum, no fixed order — just captured thinking on topics worth remembering.

> **Disclaimer:** Most notes here originate from conversations with Claude.ai. They may contain hallucinations or inaccuracies. Treat them as a starting point for understanding, not as ground truth — verify anything critical independently.

---

## 📚 Notes Index

### Ad-Tech
- [Programmatic Advertising Fundamentals](ad-tech/programmatic-advertising-fundamentals.md) — the players (DSP/SSP/exchange/MMP), how an ad actually gets served (RTB → ad tag → creative fetch, VAST for video/audio), and how conversions get measured and disputed
- [DSP Data Platform Architecture](ad-tech/dsp-data-platform-architecture.md) — system-design sketch of a DSP's dual real-time + batch (Lambda) pipeline on Kafka: bidding, attribution, and MMP reconciliation

### AI
- [Harness Engineering](ai/harness-engineering.md) — what it means to build systems around AI, and when you actually need it
- [Claude Code: Sonnet 4.6 vs Opus 4.7](ai/claude-code-model-comparison/comparison.md) — same feature, same prompt, two models, back-to-back: 7.8× the API bill, one integration point caught vs missed
- [Cost analysis for a feature on Opus 4.7](ai/claude-code-feature-cost-analysis/opus-4-7-cost-analysis.md) — same model, bigger feature: per-story-point dollars stayed flat, attention nearly tripled

### System Design
- [Group Chat System (≤1000 users/group)](system-design/chat-system-design.md) — interview-oriented walkthrough: user-sharded routing, Kafka-partitioned ingestion, per-group seq, two-layer gap detection, active-only Pub/Sub
- [GCP vs AWS: Major Services](system-design/gcp-vs-aws-services.md) — quick-reference table mapping core cloud concepts (compute, storage, DBs, messaging, etc.) between the two clouds

---

## How This Repo Works

- Each note lives in a domain folder (`ai/`, `web-dev/`, `system-design/`, etc.)
- New notes follow the format in [`_templates/topic-template.md`](_templates/topic-template.md)
- Every note links back to this index at the bottom
- This index is updated whenever a new note is added

---

*By SansWord — learning in public, one topic at a time.*
