# Programmatic Advertising Fundamentals

Ad-tech fundamentals from the perspective of a DSP data-platform: who the players are, how an ad actually gets served, and how conversions get measured and disputed. Pairs with [`dsp-data-platform-architecture.md`](dsp-data-platform-architecture.md), which covers the system-design side (the data stack lives there).

---

## 1. The ecosystem players

| Player | Side | Job |
|---|---|---|
| **DSP** (Demand-Side Platform, e.g. The Trade Desk, DV360) | Buy | The bidding brain. Decides whether/how much to bid, wins impressions on behalf of advertisers. |
| **SSP** (Supply-Side Platform) | Sell | Publisher's side. Manages and auctions their ad inventory. |
| **Ad Exchange** | Marketplace | The auction venue connecting DSPs (demand) and SSPs (supply). |
| **Ad Server** | Either | Stores & delivers the creative; does creative-level decisioning + tracking. Can be the DSP's own or a 3rd-party (Google CM360, Flashtalking). |
| **CDN** | Delivery | Serves the heavy creative bytes (video, images) from an edge near the user. |
| **MMP** (Mobile Measurement Partner) | Measurement | Neutral attribution arbiter for mobile (AppsFlyer, Adjust, Branch). |
| **DMP / CDP** | Data | Audience/segment data feeding targeting. |

### How the players interact (the request lifecycle)

It's a **two-sided market**: the **DSP buys** for advertisers, the **SSP sells** for publishers, and the **ad exchange** is the auction floor between them. The whole bid cycle (steps 2–5) runs in **~100ms** while the page loads.

```
        DEMAND SIDE                                      SUPPLY SIDE
   ┌───────────────────┐                          ┌───────────────────┐
   │    Advertiser     │                          │     Publisher     │
   │ (brand / agency)  │                          │ (site / app / CTV)│
   └─────────┬─────────┘                          └─────────┬─────────┘
             │ campaign: budget,                            │ lists ad
             │ targeting, creatives                         │ inventory (1)
             ▼                                              ▼
   ┌───────────────────┐    (2) bid request     ┌───────────────────┐
   │        DSP        │◀───────────────────────│        SSP         │
   │  (buy-side bid    │                         │ (sell-side yield  │
   │   decisioning)    │───(3) bid ───┐          │   management)     │
   └─────────┬─────────┘              │          └─────────┬─────────┘
        ▲    │                        ▼              (1) inventory
   (audience)│             ┌──────────────────┐          │
   ┌─────────┴──┐          │   AD EXCHANGE    │◀─────────┘
   │  DMP / CDP │          │  (runs auction)  │
   └────────────┘          └────────┬─────────┘
                            (4) winner notified
                                     │
             ┌──(5) winning ad markup (tag)──┘
             ▼
   ┌───────────────────────────────────────────────┐
   │        USER'S BROWSER / APP / PLAYER            │
   │             (renders the ad slot)               │
   └───┬─────────────────────────────────────▲──────┘
       │ (6) fetch creative                   │ creative bytes
       ▼                                      │
   ┌────────────────┐                         │
   │  AD SERVER+CDN  │─────────────────────────┘
   └────────────────┘
                                     (7) user clicks / converts later
                                     ▼
   ┌──────────────────────────────────────────────┐
   │  Advertiser site/app                          │
   │  (8) conversion pixel fires → DSP             │
   │      MMP arbitrates credit (mobile)           │
   └──────────────────────────────────────────────┘
```

**Step by step:**
1. **Publisher lists inventory** with its SSP; the SSP connects to ad exchanges.
2. **User loads a page** → SSP sends the impression to the **ad exchange** → exchange fans out **bid requests** to many DSPs.
3. **Each DSP decides & bids** — using the advertiser's campaign settings plus **DMP/CDP** audience data to value the impression.
4. **Exchange runs the auction** and notifies the **winning DSP**.
5. **Winning DSP returns ad markup** (a pointer/tag, *not* the creative — see §2).
6. **User's device fetches the creative** from the **ad server + CDN** and renders it.
7. **User clicks / later converts** on the advertiser's site or app.
8. **Conversion pixel reports to the DSP**; on mobile the **MMP** arbitrates which channel gets credit (see §4–5).

Steps 2–5 are **real-time bidding (RTB)**. Step 6 is creative serving (§2). Steps 7–8 are measurement/attribution (§3–6).

---

## 2. Who serves the ad? (RTB → creative fetch)

Key insight: **the DSP wins the impression but doesn't serve the creative bytes.** The winning bid returns a **pointer (ad tag/markup)**, not the ad. The user's device then fetches the real creative from an ad server + CDN.

Why: the auction must finish in ~100ms — you can't ship a 5MB video through bidding. Ship a tiny tag, win, *then* stream the heavy creative separately.

**The flow:**
1. User opens a publisher page/app with an empty ad slot.
2. Publisher's **SSP / exchange** auctions the impression → bid requests to DSPs.
3. **DSP decides to bid, wins.** Bid response carries **ad markup** (the pointer).
4. User's **browser / video player / app SDK** reads the markup → **fetches the creative from ad server / CDN.**
5. As it renders, the client fires **tracking pixels/URLs** (impression, then clicks, completions).

**The literal answer:** the **ad server + CDN** deliver the creative, triggered when the client renders the winning ad tag.

### Format examples

**Text / images (display banners):** bid response returns **HTML markup** (or an image URL in OpenRTB's `adm` field). Browser renders it, fetches the `.jpg/.png/.gif` from a CDN. Lightweight.

**Video / audio:** uses **VAST** (Video Ad Serving Template) — an XML contract between players and ad servers.
- Bid markup contains a **VAST tag (URL)**, not the video.
- Player calls the VAST URL → ad server returns **VAST XML** with: media file URLs (`.mp4` at several bitrates, on a CDN), tracking URLs (start, 25/50/75%, complete, click), click-through destination.
- Player picks a bitrate, **streams the video from the CDN**, fires tracking as it plays.
- Related: **VPAID/SIMID** (interactive), **VMAP** (ad pods — pre/mid/post-roll), **OMID/OM SDK** (viewability).
- **Audio** (podcasts, Spotify) uses **VAST 4.x** too (older **DAAST** folded in); often **SSAI** (server-side ad insertion) stitches the ad into the content stream server-side.

### DSP creative-serving nuance
A DSP usually has its **own integrated creative hosting/ad serving** (advertisers upload creatives). But advertisers can bring **3rd-party ad server tags** (e.g. CM360) — then the DSP's winning markup is a **redirect** to that tag. So the DSP can be both bidder *and* creative server, or just the bidder pointing elsewhere.

---

## 3. Conversion rate: who calculates, provide vs. prove

**Conversion** = post-ad action the advertiser cares about (purchase, signup, install). 
`CVR = conversions / clicks` (post-click) or `/ impressions` (view-through).
- **Post-click** — user clicked then converted.
- **Post-view (view-through)** — user *saw* (didn't click) then converted. DSPs love these (more credit); advertisers are skeptical.

**Who calculates:** the **DSP** — advertiser places a **conversion pixel/tag** on their confirmation page; the DSP's **attribution** (window + model, usually last-touch) matches conversions to ads it served, then reports CVR.

**Does the DSP have to *provide* metrics?** Effectively yes — **reporting *is* the product.** Delivered via dashboards/APIs. Caveats:
- The DSP can only report what its pixel can **measure** (broken pixel = no data, that's on the advertiser).
- **Transparency varies** — black-box aggregates vs. **log-level raw data** access (sophisticated advertisers demand the latter for their own analysis — exactly what a data platform team builds).

**Does the DSP have to *prove* metrics?** Not legally, but commercially yes — because it's **grading its own homework**, advertisers don't take it on faith (see §5).

---

## 4. Multiple DSPs & the double-counting problem

Brands run **multiple DSPs** (different strengths — CTV vs display, inventory, data; avoid dependency; test performance) — plus search/social in parallel.

**The problem:** each DSP self-attributes off its own pixel. If a user saw ads from DSP A *and* B before buying, **both claim the same conversion.** Naively summing self-reported numbers over-counts (worse with view-through).

**How they resolve which DSP gets credit** — always the same pattern: **a single neutral arbiter that sees all touchpoints and credits once.**
- **Advertiser's own analytics** (GA4, internal warehouse) = single source of truth for web; one model across all channels, counted once.
- **MMP** = neutral arbiter for mobile (the cleanest case). Every DSP sends click/impression data to the MMP; the **MMP** (not the DSPs) applies last-click and credits **exactly one**. DSPs only get paid for installs the MMP awards.
- **3rd-party ad server** (CM360) — if all creatives route through its tags, it sees full cross-DSP exposure and dedupes centrally.
- **Data clean rooms / MTA** — fractional credit across touchpoints; privacy-safe matching (Google ADH, Amazon Marketing Cloud).

Underneath: needs a **common identifier** (device ID, hashed email) + a **single attribution authority**. Post-cookie, matching is lossy — the hard current problem.

---

## 5. DSP vs. MMP tension — who does the client trust?

**They almost always disagree, predictably:** DSP reports **more** conversions than the MMP. Not (usually) lying — different rules:
- DSP counts **view-through**; MMP often **last-click only**.
- DSP may use a **longer window**.
- **Dedup scope:** MMP sees *every* source and credits one; DSP only sees its own ads, blind to other channels' last touch.

**Who the client trusts → follow the incentives:**
- **MMP is neutral** — paid the same regardless of which DSP wins; hired by the advertiser. No incentive to inflate. → **Source of truth / system of record for billing.**
- **DSP is conflicted** — gets paid/renewed if it claims the conversion. → Over-claims.
- So the **DSP reconciles *to* the MMP**, not vice versa.

**But it's not "trust one, ignore the other"** — different jobs:
- **MMP number** → truth for payment & channel value.
- **DSP number** → **optimization signal** (the bid model needs its own conversion feedback in real time; valid for steering even if inflated in absolute terms).

**Reconciliation in practice:**
1. **Align settings** (same window + model) — most of the gap evaporates (it was apples-to-oranges).
2. **Look at the residual:** ~10–15% normal (identity loss, timing); large & persistent = real problem (misconfigured pixel/SDK, double-firing, fraud) → investigate.
3. Neutral party authoritative for what gets paid.

**Caveat — MMP isn't omniscient:** it can only arbitrate players who **share data** with it. DSPs do (postbacks), so it referees them cleanly. **Walled gardens (Google, Meta) self-report** and don't hand over user-level data → MMP's cross-garden view is partial (clean rooms help).

---

## 6. Postback timing — real-time or batch?

Two directions (people conflate them):
- **DSP → MMP:** click/impression tracking ("I touched this device").
- **MMP → DSP:** conversion **postbacks** ("you won this conversion").

**The attribution-critical path is real-time, event-driven:**
- DSP→MMP clicks/impressions fire an HTTP call to the MMP's tracking URL **instantly, per-event**, not batched.
- **Why it must be real-time:** attribution is **time-ordered** — the click record must reach the MMP *before* the install fires (installs can happen seconds after a click). Batching clicks hourly → installs arrive first → attribution **silently breaks** and the DSP loses earned credit.
- MMP→DSP conversion postbacks: also **real-time / near-real-time** (seconds), because DSPs feed them into **live bid optimization**.

**What *is* batched:** aggregated reports (hourly/daily), reconciliation & billing (daily file drops to S3), bulk log-level exports. Volume-efficient, not latency-sensitive.

| Path | Data | Cadence | Why |
|---|---|---|---|
| Real-time / event | clicks, impressions, conversion postbacks | per-event, sub-second–seconds | attribution ordering + live optimization |
| Batch | aggregated reports, reconciliation, billing, log exports | hourly / daily | volume efficiency |

**Modern exception — iOS SKAN:** Apple's SKAdNetwork postbacks are **deliberately delayed (24–48h+ randomized timer) and aggregated**, user-level detail stripped for privacy. So iOS-SKAN conversion signal is **not** real-time, **not** per-user. DSPs optimize around it.

This maps to the **Lambda architecture** in the companion note: real-time stream (Kafka → bidder/Pinot) for the hot path + batch (Spark/Airflow → warehouse) for accurate reconciliation.

---

## 7. The two conceptual axes (everything sorts into these)

- **OLTP vs. OLAP** — transactional (operational, point reads/writes) vs. analytical (aggregations over lots of rows). Bidder counters = OLTP-ish; dashboards/reports = OLAP.
- **Batch vs. streaming** — accurate-but-delayed bulk processing vs. low-latency-but-approximate event processing. The whole DSP platform runs both side by side.

*(Data-stack glossary — Snowflake, Pinot, TiDB, Spark, Airflow, Kafka, Trino, lineage — is in the [companion architecture note](dsp-data-platform-architecture.md#glossary-recap-terms-feeding-this-design).)*

---

← [Back to Index](../README.md)
