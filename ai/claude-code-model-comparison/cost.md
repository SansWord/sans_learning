# Freeform branch cost breakdown — token counts, rates, derivation

> Supporting detail for [the main comparison report](./comparison.md#api-dollar-cost). This doc shows how the API-dollar totals were derived from scoped token counts and published per-token rates — skip it if you only care about the totals.

← [Back to the main report (API dollar cost section)](./comparison.md#api-dollar-cost)

---

## Token breakdown (scoped to superpowers window)

| | 4.6 (Sonnet) main | 4.6 subagents | 4.7 (Opus) |
|---|---|---|---|
| Model(s) | Sonnet 4.6 | 95× Sonnet + 187× Haiku turns across 20 subagents | Opus 4.7 |
| Input tokens | 245 | 4,621 | 466 |
| Output tokens | 78,459 | 33,113 | 178,281 |
| Cache-read tokens | 11.65M | 6.44M | 26.51M |
| Cache-write (1h) tokens | 218k | 0 | 664k |
| Cache-write (5m) tokens | 0 | 881k | 0 |
| Total billable tokens | **11.95M** | **7.36M** | **27.35M** |

## Published per-token rates (Anthropic API, April 2026)

Rates used (per million tokens):

| Model | Input | Output | Cache read | Cache write (5m) | Cache write (1h) |
|---|---|---|---|---|---|
| Opus 4.x | $15.00 | $75.00 | $1.50 | $18.75 | $30.00 |
| Sonnet 4.x | $3.00 | $15.00 | $0.30 | $3.75 | $6.00 |
| Haiku 4.5 | $1.00 | $5.00 | $0.10 | $1.25 | $2.00 |

## Applied to scoped token counts

| | 4.6 main | 4.6 subagents (Sonnet) | 4.6 subagents (Haiku) | 4.7 main |
|---|---|---|---|---|
| Input | $0.00 | $0.00 | $0.00 | $0.01 |
| Output | $1.18 | $0.18 | $0.10 | $13.37 |
| Cache read | $3.50 | $0.59 | $0.45 | $39.76 |
| Cache write | $1.31 | $1.31 | $0.66 | $19.92 |
| **Subtotal** | **$5.98** | **$2.08** | **$1.22** | **$73.06** |

## API totals

- **freeform-4.6 (Sonnet main + Sonnet subagents + Haiku subagents): ~$9.28**
- **freeform-4.7 (Opus only): ~$73.06**
- **Ratio: 7.87×**

---

← [Back to the main report (API dollar cost section)](./comparison.md#api-dollar-cost)
