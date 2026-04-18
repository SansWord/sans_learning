# Freeform Method — Claude Sonnet 4.6 (default) vs Opus 4.7

> *Same feature, same prompt, two models, back-to-back. 7.8× the API bill. 1.67× the plan-limit cost. One integration point caught, one missed.*

← [Back to Index](../../README.md)

---

Same feature implemented twice in Claude Code, back-to-back, using two different models. This doc compares code output, process, token cost, and plan-limit consumption.

## TL;DR

- **API dollar cost:** $9 (Sonnet 4.6) vs $73 (Opus 4.7) — **7.8×**.
- **Wall-clock time:** ~36 min (Sonnet 4.6) vs ~34 min (Opus 4.7) — **essentially tied**.
- **Claude Code plan-limit cost:** 9% (Sonnet 4.6) vs 15% (Opus 4.7) of a Max 5x session — **~1.7×**.
- **Unique contributions:** Core code byte-identical. Sonnet 4.6 surfaced a refactoring opportunity (data-driven method registry) during brainstorm that I deferred to `future.md`; Opus 4.7 added a URL-routing fix (`#trends?method=freeform`) not in the prompt but consistent with the app. Each model caught something the other missed.
- **Execution delegation:** When executing tasks, Sonnet 4.6 offered to delegate simple jobs to cheaper agents; Opus 4.7 silently did everything itself — part of why costs landed 8× apart.

|                         | 4.6 (Sonnet, scoped)       | 4.7 (Opus, scoped)         |
| ----------------------- | -------------------------- | -------------------------- |
| Window                  | 12:12:41 → 12:48:21 PDT    | 11:37:21 → 12:10:54 PDT    |
| Duration                | 35m 40s                    | 33m 33s                    |
| Tokens (main)           | 11.95M                     | 27.35M                     |
| API cost (main)         | $5.98                      | $73.06                     |
| API cost (+ subagents)  | $9.28                      | $73.06 ¹                   |
| Plan-limit % ²          | 9%                         | 15%                        |

¹ Opus made no subagent calls in this run, so main = all-in.
² Read from Claude Code's live status indicator (stopwatch-style), not extracted from JSONL.

Both branches ship the core feature and pass all tests. Core source files — [`src/methods/freeform.ts`](https://github.com/SansWord/sans_cube/blob/freeform-4.7/src/methods/freeform.ts), the [method registry](https://github.com/SansWord/sans_cube/blob/freeform-4.7/src/methods/index.ts), [`MethodSelector`](https://github.com/SansWord/sans_cube/blob/freeform-4.7/src/components/MethodSelector.tsx), [`SolveHistorySidebar`](https://github.com/SansWord/sans_cube/blob/freeform-4.7/src/components/SolveHistorySidebar.tsx) — are **byte-identical**. The differences sit at the edges and in the process.

- **Opus 4.7** — completer. Produced a [107-line design spec](https://github.com/SansWord/sans_cube/blob/freeform-4.7/docs/superpowers/specs/2026-04-17-freeform-method-design.md), an [819-line plan](https://github.com/SansWord/sans_cube/blob/freeform-4.7/docs/superpowers/plans/2026-04-17-freeform-method.md), a [devlog entry](https://github.com/SansWord/sans_cube/blob/freeform-4.7/docs/devlog.md), and a [`future.md` strikethrough](https://github.com/SansWord/sans_cube/blob/freeform-4.7/future.md) on the freeform backlog item. Also inferred one extension that wasn't in the prompt but fit the app's existing URL-param pattern — adding `'freeform'` to the [`#trends?method=` allowlist](https://github.com/SansWord/sans_cube/blob/freeform-4.7/src/hooks/useHashRouter.ts) — which Sonnet missed. Took **~33 min** and **7.8× the API-dollar cost** of Sonnet.
- **Sonnet 4.6** — pragmatist. Also produced a [design spec (127 lines — longer than Opus's)](https://github.com/SansWord/sans_cube/blob/freeform-4.6/docs/superpowers/specs/2026-04-17-freeform-method-design.md) and a [439-line plan](https://github.com/SansWord/sans_cube/blob/freeform-4.6/docs/superpowers/plans/2026-04-17-freeform-method.md), plus surfaced a meta-insight Opus did not: that adding a 4th method will require touching ≥5 files and so warrants a "data-driven method registry" refactor, which it appended as a [new backlog item in `future.md`](https://github.com/SansWord/sans_cube/blob/freeform-4.6/future.md). Shipped the feature cheaper and slightly faster. Missed the hash-router whitelist, so `#trends?method=freeform` silently falls back to "all". Skipped devlog and did not strikethrough the original freeform backlog item.
- **Cost** — $73 (Opus) vs $9 (Sonnet) in raw API-dollar terms. But on a Claude Code plan, the ratio is much closer: **15% vs 9% of a Max 5x session limit** (~1.67× in practical plan-usage terms).

*One-task anecdote, not a benchmark. API pricing as of Apr 2026. See [Methodology](#methodology) and [Caveats](#caveats-and-limitations) below.*

## Project context

[`sans_cube`](https://github.com/SansWord/sans_cube) is a Rubik's cube solve analyzer for speedcubers. It connects to smart cubes via Web Bluetooth, records solves in real time, and breaks each solve down phase-by-phase (CFOP and Roux). Stack: React 19 + TypeScript + Vite + Three.js, with optional Firebase Firestore cloud sync.

![Solve-detail view in sans_cube: 3D cube replay on the left, phase-by-phase timing breakdown for a CFOP solve on the right.](./sans_cube.png)

> **▶ Try it live:** [sansword.github.io/sans_cube](https://sansword.github.io/sans_cube/#solve--2) opens an example solve — phase breakdown, 3D replay, and move history, no smart cube needed. Chromium browser (Chrome / Edge) required.

## The task

The starting point was a single line in the project's [`future.md` backlog](https://github.com/SansWord/sans_cube/blob/main/future.md). Both runs used the **same verbatim user prompt** as the very first message:

> I want to create a new method: freeform that has only one Phase: the cube is solved. And use this method in method filter we're using for record list and stats, and also able to recompute with this method hence we can update other solves into this one or allow this update to other methods. I'm not sure if this is the complete list. let me know if you need more information

In concrete terms, that meant adding a third solving method called **Freeform** alongside the existing CFOP and Roux:

- A single phase labeled "Solved" that covers the whole solve (no sub-phase analysis).
- Selectable in the global method selector and per-solve in the detail modal.
- Filterable in the record-list sidebar and the Trends stats modal.
- Skipped by the method-mismatch detector (user deliberately opted out of phase analysis).

Both sessions ran with the [Superpowers plugin](https://github.com/obra/superpowers) installed, which enforces a brainstorm → plan → execute workflow.

## Models compared

- **freeform-4.6** branch — Claude **Sonnet 4.6** (`claude-sonnet-4-6`). This is Claude Code's **default model** (what a typical user gets without opting in to Opus).
- **freeform-4.7** branch — Claude **Opus 4.7** (`claude-opus-4-7`). Flagship-tier, opt-in via `/model`.

So the comparison is **default vs. upgraded**, not older vs. newer of the same tier.

## Methodology

Both runs followed the same procedure so results are directly comparable.

**Environment (identical for both runs):**

- Claude Code CLI with the [Superpowers plugin](https://github.com/obra/superpowers) loaded. Superpowers enforces a brainstorm → plan → execute workflow and adds the `✅ Phase complete!` handoff convention.
- Working directory: `sans_cube` checked out on a clean `main` branch.
- Project [`CLAUDE.md`](https://github.com/SansWord/sans_cube/blob/main/CLAUDE.md) loaded, including its conventions around devlog entries, `future.md` strikethroughs, and design specs under `docs/superpowers/specs/`.
- `MEMORY.md` checked beforehand — no freeform-related entries existed, so both sessions started cold on the task.

**Procedure (identical for both runs):**

1. Open a fresh Claude Code session on `main`.
2. Set the model — `/model claude-opus-4-7` for the Opus run, left at the default `claude-sonnet-4-6` for the Sonnet run. No other configuration changed.
3. Paste the exact user prompt shown in "The task" above as the very first message. Nothing sent before it.
4. Let the superpowers workflow run through all three phases (brainstorm → plan → execute). Answered clarifying questions as they came up; approved design decisions.
5. Stop when the model signalled completion (see Session windows below).
6. Preserve the resulting work on a branch — `freeform-4.7` for the Opus run, `freeform-4.6` for the Sonnet run.

**Sequencing.** Runs were sequential, not parallel: Opus ran first (started 11:37 PDT), then Sonnet (started 12:12 PDT) after the Opus run finished. Same human driving both sessions. No conversation context carried over between them, and no code changes from one run were visible to the other (each branched independently from `main`).

**What was recorded.** Live plan-limit % readings from Claude Code's status indicator, plus running notes about qualitative differences (when the model paused, what decisions it auto-made, which integration points it surfaced). Everything else — token counts, tool calls, durations, commit history, code diffs — was extracted after the fact from the Claude Code session JSONL files and `git log` / `git diff` on the two branches.

**What was not controlled.** Single-task sample, sequential runs, no randomization, no retries, one author. The comparison is an anecdote from one feature, not a benchmark.

## Session windows (scoped to actual work)

| | Start (user's first prompt) | End (superpowers workflow complete) | Duration |
|---|---|---|---|
| **4.6 (Sonnet)** | 2026-04-17 **12:12:41 PDT** | 2026-04-17 **12:48:21 PDT** (`Implementation complete. What would you like to do?`) | **35 min 40 sec** |
| **4.7 (Opus)** | 2026-04-17 **11:37:21 PDT** | 2026-04-17 **12:10:54 PDT** (`✅ Phase complete!` handoff) | **33 min 33 sec** |

Runs were sequential (Opus first, Sonnet second). All token counts, tool-call counts, and costs below are scoped to these windows. Activity after these markers (trailing "thanks" / idle session) contributed <2% of total tokens in either session.

## Branches

| Branch | Model | Commits | Compare link (SHA-pinned) |
|---|---|---|---|
| [`freeform-4.6`](https://github.com/SansWord/sans_cube/tree/freeform-4.6) | Sonnet 4.6 | 10 | [main…freeform-4.6](https://github.com/SansWord/sans_cube/compare/55f5209...f9672d2) |
| [`freeform-4.7`](https://github.com/SansWord/sans_cube/tree/freeform-4.7) | Opus 4.7 | 12 | [main…freeform-4.7](https://github.com/SansWord/sans_cube/compare/55f5209...abd1aef) |

Compare links are pinned to `main@55f5209` (the state of main at the time of this writeup) so future commits on main won't alter the diff.

## Code output comparison

### The one user-visible gap: hash router

On the Sonnet branch, the URL `#trends?method=freeform` silently falls back to "all" — the allowlist in [`src/hooks/useHashRouter.ts`](https://github.com/SansWord/sans_cube/blob/freeform-4.6/src/hooks/useHashRouter.ts) wasn't extended to include `'freeform'`. Opus noticed this integration point during planning, widened both the `TrendsHashParams` type and the runtime validator, and added a dedicated test ([`tests/hooks/useHashRouter.test.ts`](https://github.com/SansWord/sans_cube/blob/freeform-4.7/tests/hooks/useHashRouter.test.ts)). Verified during manual QA against Opus's checklist — the "Hash route: `#trends?method=freeform` opens Trends with Freeform selected" case passes on the Opus branch and fails on the Sonnet branch.

This is the only functional difference a user could actually trip on. Every other production-code difference is cosmetic or defensive (table below).

### All production-code differences

Seven files were touched in production code across the two branches. Five are byte-identical. The three that differ:

| File | `freeform-4.6` (Sonnet) | `freeform-4.7` (Opus) |
|---|---|---|
| [`src/hooks/useHashRouter.ts`](https://github.com/SansWord/sans_cube/blob/freeform-4.7/src/hooks/useHashRouter.ts) | No change. `#trends?method=freeform` silently rejected | Adds `'freeform'` to the `method` whitelist in both the `TrendsHashParams` type and the runtime validator |
| [`src/utils/detectMethod.ts`](https://github.com/SansWord/sans_cube/blob/freeform-4.7/src/utils/detectMethod.ts) | `if (solve.method === 'freeform') continue` | `if ((solve.method ?? 'cfop') === 'freeform') continue` (defensive — handles undefined) |
| [`src/components/TrendsModal.tsx`](https://github.com/SansWord/sans_cube/blob/freeform-4.7/src/components/TrendsModal.tsx) | `buildColorMap` lists FREEFORM first in the `'all'` merge | Lists ROUX before FREEFORM. Ordering-only — keys don't collide across methods |

### Scope / process differences (artifacts)

| | `freeform-4.6` (Sonnet) | `freeform-4.7` (Opus) |
|---|---|---|
| Design spec file | [`docs/superpowers/specs/2026-04-17-freeform-method-design.md`](https://github.com/SansWord/sans_cube/blob/freeform-4.6/docs/superpowers/specs/2026-04-17-freeform-method-design.md) — 127 lines | [`docs/superpowers/specs/2026-04-17-freeform-method-design.md`](https://github.com/SansWord/sans_cube/blob/freeform-4.7/docs/superpowers/specs/2026-04-17-freeform-method-design.md) — 107 lines |
| Implementation plan | [`2026-04-17-freeform-method.md`](https://github.com/SansWord/sans_cube/blob/freeform-4.6/docs/superpowers/plans/2026-04-17-freeform-method.md) — 439 lines | [`2026-04-17-freeform-method.md`](https://github.com/SansWord/sans_cube/blob/freeform-4.7/docs/superpowers/plans/2026-04-17-freeform-method.md) — 819 lines (1.9×) |
| Devlog entry | — | v1.22.0 entry appended to [`docs/devlog.md`](https://github.com/SansWord/sans_cube/blob/freeform-4.7/docs/devlog.md) |
| [`future.md`](https://github.com/SansWord/sans_cube/blob/main/future.md) edit | **Added** a new backlog item ("Data-driven method registry" refactor) | **Struck through** the existing freeform backlog item (per project convention) |
| URL route whitelist | — | Done |
| Manual QA in plan | In design doc but not linked into implementation tasks | Same position: design doc only, not in tasks |
| New test files | recomputePhases, filterSolves, detectMethod | Same three, plus [`tests/hooks/useHashRouter.test.ts`](https://github.com/SansWord/sans_cube/blob/freeform-4.7/tests/hooks/useHashRouter.test.ts) |

Doc output is similar in breadth — both wrote specs and plans — but the conventions differ. Opus followed the `CLAUDE.md` conventions literally (strikethrough done item, write devlog entry). Sonnet did neither, but instead surfaced a forward-looking observation as a new backlog item, which Opus did not. Sonnet's spec is actually slightly longer than Opus's; Opus's plan is ~2× Sonnet's.

### Diffstat and commits

At a glance: Sonnet shipped in **10 commits** touching **14 files** (658+/12-); Opus in **12 commits** touching **17 files** (1046+/15-). The three files Opus touched that Sonnet didn't are [`docs/devlog.md`](https://github.com/SansWord/sans_cube/blob/main/docs/devlog.md) (updated per project convention), `src/hooks/useHashRouter.ts` (the integration point above), and its corresponding test.

Full diffstat, per-commit history, and notes on commit shape → **[branches.md](./branches.md)**.

## Process comparison

### Workflow shape (from session transcripts)

Both models went through the Superpowers spec → plan → execute flow. The notable shape difference is in the **plan phase**:

- **Sonnet 4.6** ended its plan with a trailing `## Manual Smoke Test` section — 5 numbered bullets of things to verify in a browser, but sitting outside the task list as documentation/afterthought.
- **Opus 4.7** made manual QA an actual **Task 9** in the plan, with 4 numbered scenarios, checkbox steps, and the same shape as the code tasks (start dev server → run scenario → expected result). The design spec didn't require this — Opus decided on its own to promote manual QA into an executable task.

**My take:** Opus's decision to put manual QA into the task list (even though the design doc didn't ask for it) is the thing I value most here. It makes the development feel stable — by the time execution finishes I've already walked through the scenarios, so I'm confident the feature is actually done and I won't have to come back and debug. Sonnet's trailing Smoke Test list is the same information on paper, but because it's not a task I'm guided through during execution, it's easy to skip, and then bugs surface later when I'm reviewing in the browser and have to re-enter the context.

### Brainstorm-phase framing

Both models went through the Superpowers brainstorm flow and asked clarifying questions, but **what they asked about was very different**.

**Sonnet 4.6** — 2 clarifying questions (scoping) + 3 design approaches:

1. Should Freeform be selectable as the **global timer method**, or only per-solve in the detail modal?
2. In the **Trends modal**, should the phase chart still show a single bar (A) or hide the breakdown entirely (B)?

Then proposed three approaches: (A) minimal direct addition, (B) data-driven method registry refactor, (C) per-solve only. Recommended (A). Sonnet **did not ask about the phase name or color** — it defaulted to "Solve" (label) and grey in its draft design, and the user had to volunteer *"I want the phase name as 'Solved' and the color is in green"* mid-design for them to be corrected.

**Opus 4.7** — 3 clarifying questions (all small specifics), no "approaches" step:

1. **Phase label** — Solve / Solved / Full Solve / other?
2. **detectMethod behavior** — skip entirely (a) / skip auto-suggest only (b) / full participation (c)? *(with recommendation)*
3. **Phase color** — grey / green / user-supplied hex? *(with recommendation)*

Opus skipped proposing 2–3 high-level approaches and went straight from the three decision questions into presenting the full design. It pre-empted the exact details (name + color) that Sonnet missed.

**Reading:** Sonnet framed the question at a higher level (scoping, UI strategy); Opus framed it at the concrete-decision level (name this, pick this color, handle this edge case). Both arrived at an approved spec, but Opus's framing caught the two specifics that Sonnet only learned about via correction.

**My take:** If clarifying questions have a budget, I'd rather spend it on approach/scope questions than on names and colors — those are cheap to fix. What I *do* value about Opus's detail questions is that asking forces me to **consciously opt in** to the name/color choice. Sonnet picking them unilaterally in the draft spec means I might gloss over them during review and only notice in QA. So the ideal for me is: lead with scope/approach, but still surface small specifics as quick A/B/C picks rather than silently defaulting.

The bigger cost of Opus skipping the "propose 2–3 approaches" step is that it never exposed the "data-driven method registry refactor" as a smarter-but-bigger fork. Sonnet *did* surface it (as Option B), which gave me the chance to consciously defer it to `future.md` as later-scope work. The approach-enumeration step isn't just a formality — it's the step that makes this kind of observation visible to the user.

### Execute-phase delegation (subagents vs inline)

The Superpowers `executing-plans` skill normally offers two execution modes once the plan is approved: **Subagent-Driven** (dispatch a fresh subagent per task, review between tasks) and **Inline Execution** (run all tasks in the main conversation). Both models had equal access to the `Agent` tool — subagent-driven is not a model-gated capability.

- **Sonnet 4.6** presented the fork explicitly: *"Two execution options: 1. Subagent-Driven (recommended) — I dispatch a fresh subagent per task, review between tasks, fast iteration. 2. Inline Execution — Execute tasks in this session using executing-plans, batch execution with checkpoints. Which approach?"*. I picked 1, and Sonnet spent the entire execute phase dispatching implementer + reviewer subagents per task (20 Agent calls total).
- **Opus 4.7** skipped the fork entirely — it went straight from *"I'm using the executing-plans skill to implement this plan"* → *"Starting Task 1"* with zero subagent calls. I wasn't prompted to choose, and by the time I noticed, the execute phase was already done inline.

**What Sonnet's subagents actually did.** Checking the subagent JSONLs, Sonnet's 20 calls weren't a blanket "offload to cheaper models" — they followed a deliberate per-task pattern: most tasks got an **implementer** (general-purpose agent on **Haiku 4.5**) → a **spec reviewer** (also Haiku) → a **code-quality reviewer** (`superpowers:code-reviewer` on **Sonnet 4.6**), plus one final code review at the end. Mechanical implementation and spec-checking went to Haiku (cheaper); judgment-heavy code quality review stayed at Sonnet tier. This is as much a **quality mechanism** as a cost-saving one: every task passed through two review gates before the main Sonnet loop committed. Opus's inline execution has no equivalent per-task review gate — it's just Opus self-checking as it goes.

**Where subagents fired.** All 20 calls were inside the execute phase (first at 19:30:44, right after I picked subagent-driven; last at 19:46:03, the final review). Brainstorm and plan were inline for both models — neither delegated context collection, nor plan writing. So the 20-vs-0 gap is **entirely** from one skill-interpretation moment: whether the model presented the execute-mode fork or skipped it.

**My take:** This surprised me. Subagent-driven execution is common practice among Superpowers users — I've always had that choice in previous sessions — and I didn't notice 4.7 had silently skipped offering it until I reviewed the transcripts later. Same skill, same prompt, same access to `Agent`, but Sonnet read the skill as "present the mode choice" and Opus read it as "just execute inline." That's a meaningful behavioral difference to flag, because it changes the cost profile: staying in the main loop means every token is billed at Opus rates, whereas dispatching implementer subagents could have routed a lot of the mechanical work to cheaper models. Part of the 8× cost gap is per-token price, but part of it is this delegation decision.

**Workaround.** If you want Opus 4.7 to always present the fork rather than silently defaulting to inline, you can pin the behavior in your project's `CLAUDE.md`:

```md
## Superpowers execution mode

When using the `executing-plans` skill, always present the subagent-driven vs inline execution choice to the user before starting execution. Do not default to inline silently.
```

### What each model noticed / missed

| Integration point | 4.6 (Sonnet) | 4.7 (Opus) |
|---|---|---|
| Hash router (`#trends?method=` whitelist) | ❌ missed | ✅ noticed and added |
| Devlog entry at session end | ❌ | ✅ |
| Surfaced a refactoring opportunity during brainstorm | ✅ (data-driven method registry, as Option B in approaches) | ❌ (skipped the approaches step) |
| Defensive `??` on nullable method field | ❌ (plain equality) | ✅ |
| Design spec under `docs/superpowers/specs/` | ✅ (127 lines) | ✅ (107 lines) |
| Manual-QA checklist in design doc | ✅ | ✅ |
| Manual-QA promoted into the implementation plan | ❌ (trailing Smoke Test section, not a task) | ✅ (Task 9 with 4 checkbox scenarios) |
| Offered subagent-driven vs inline execution choice | ✅ (presented both modes, user picked subagent-driven) | ❌ (silently went inline) |

### Tool-call patterns (scoped window)

| Tool | 4.6 (Sonnet) | 4.7 (Opus) | Reading |
|---|---|---|---|
| **Agent** (delegate to subagent) | **20** | 0 | Sonnet offloaded 20 chunks of work to cheaper Haiku/Sonnet subagents; Opus did everything in the main loop |
| Bash | 7 | **50** | Opus ran far more commands (tests, git, checks, inspection) |
| Edit | 4 | **25** | Opus did more incremental edits |
| Grep | 4 | **17** | Opus explored code more |
| Read | 24 | 32 | Similar exploration depth |
| TaskCreate / TaskUpdate | 15 / 29 | 10 / 20 | Sonnet tracked more fine-grained todos |
| Skill | 4 | 4 | Both engaged superpowers skills equally |
| Write | 2 | 5 | Opus wrote more files (spec, plan, devlog, tests) |
| **Total tool calls** | **120** | **167** | |

The single most striking signal: **Sonnet's 20 `Agent` calls vs Opus's 0**. Sonnet delegated heavily — 14 general-purpose agents and 6 code-reviewer agents, with 187 Haiku turns and 95 Sonnet turns across them. Opus did every Read, Edit, Grep, and Bash itself in the main loop at full price. This is the main reason Sonnet's effective cost is so much lower than a naive "Opus is 5× Sonnet" comparison would suggest.

## Cost comparison

### API dollar cost

Scoped to the superpowers window and applied against Anthropic's published per-token rates (April 2026):

- **freeform-4.6 (Sonnet 4.6 main + Sonnet/Haiku subagents): ~$9.28**
- **freeform-4.7 (Opus 4.7 only): ~$73.06**
- **Ratio: 7.87×**

Full token breakdown, per-million-token rates by model, and the line-by-line derivation → **[cost.md](./cost.md)**.

### Plan-limit consumption (Claude Code)

Observed directly from the Claude Code session-limit indicator:

- **4.6 (Sonnet):** 48% → 57% = **9% of the Max 5x session limit consumed**
- **4.7 (Opus):** 33% → 48% = **15% of the Max 5x session limit consumed**
- **Ratio: ~1.67×**

<details>
<summary><strong>Why plan-limit % ≠ API-dollar ratio</strong> (click to expand)</summary>

The two metrics diverge by ~5×. Both are correct; they measure different things.

- **API dollars** (7.87× ratio) is what you'd pay if you bought tokens directly from Anthropic's API at published rates. Opus input tokens literally cost 5× Sonnet input tokens; output 5×; cache read 5×. Combined with Opus using 2.3× more tokens on this task, the result is ~8× raw cost.
- **Plan-limit %** (1.67× ratio) is what your Claude Code Max plan charges against your session bucket (a 5-hour rolling window on Max 5x). Anthropic normalizes Opus consumption with a weighting much more favorable than the API rate card — effectively ~2× per token instead of 5×. This is the mechanism by which paid Claude Code plans make Opus affordable to use regularly.

Use the appropriate metric for your question:

- "Can I afford to run this task in my current 5-hour session?" → plan-limit %.
- "If I moved to pay-per-token API usage or a different provider, what would this feature have cost?" → API dollars.

</details>

### Cost of the experiment itself

Running this comparison produced the feature exactly once that you needed, plus a throwaway second implementation. The marginal cost of the experiment = cost of whichever branch you don't keep.

- If you keep **Opus's branch** (the more complete one): experiment cost ≈ **$9 API / 9% plan-limit** (the Sonnet run is discarded)
- If you keep **Sonnet's branch**: experiment cost ≈ **$73 API / 15% plan-limit** (the Opus run is discarded)

**Takeaway.** The "discard" cost can be smaller than it looks. If both branches ship working code, you can keep one and still harvest insights (refactor opportunities, integration points the other model noticed) from the branch you didn't keep. In this run, my plan is to keep 4.7 as `main` and separately apply the data-driven method registry refactor that 4.6 surfaced during brainstorm — so the final codebase ends up better than either single run would have produced.

## Hybrid worth trying next time

Use **Opus for brainstorm + plan** (where the hash-router integration point would have been caught during planning, before any code was written), then `/clear` and run **Sonnet for execute-plan** (where most of the Opus cost lived: 83% of Opus's tokens were cache reads during execution).

Rough estimated cost of a hybrid run on a comparable task: ~$15–20 API / ~10–12% plan-limit. You'd get Opus-quality scope at near-Sonnet execution cost.

<a name="caveats-and-limitations"></a>
<details>
<summary><strong>Caveats and limitations</strong> (fine print — click to expand)</summary>

- **Single-task sample.** One feature of roughly this size (~50 LoC production code + tests + docs). Don't generalize to much larger or much smaller tasks.
- **Sequential, not parallel.** Opus ran first (11:37 PDT), Sonnet second (12:12 PDT), in separate Claude Code sessions. No in-conversation context carried between them.
- **Memory effect negligible.** `MEMORY.md` was checked for freeform-related entries before the Sonnet run — none existed. Sonnet started cold on this task.
- **Data provenance.** Token counts, tool-call counts, durations, API dollar costs, and model identities are all derived from the Claude Code session JSONL files on disk. **Plan-limit percentages (15% / 9%) are NOT in the JSONLs** — they were read live from Claude Code's status indicator during each run and relayed by the user. Plan-limit weighting behavior is inferred from those two data points, not from Anthropic's published docs, so treat the 1.67× figure as an observation specific to this task and plan, not a universal ratio.
- **Pricing rates** are Anthropic's public API rate card at the time of writing (Opus 4.x: $15/$75/$1.50/$30 per MTok for input/output/cache-read/1h-cache-write; Sonnet 4.x: $3/$15/$0.30/$6; Haiku 4.5: $1/$5/$0.10/$2).
- **Qualitative observations** (manual QA pass/fail, who noticed what, /clear behavior) were recorded live during each run by the user and cross-referenced against the JSONL transcripts after the fact.
- **Source.** Metrics are derived from Claude Code's own session transcripts (JSONL files it writes locally per session, including subagent transcripts), captured on 2026-04-17 alongside the resulting git branches: [`freeform-4.6`](https://github.com/SansWord/sans_cube/tree/freeform-4.6) and [`freeform-4.7`](https://github.com/SansWord/sans_cube/tree/freeform-4.7). *(AI-assisted analysis — may contain inaccuracies. Verify critical details independently.)*

</details>

---

## About

**Project:** [github.com/SansWord/sans_cube](https://github.com/SansWord/sans_cube) — try the live app at [sansword.github.io/sans_cube](https://sansword.github.io/sans_cube/#solve--2) (opens an example solve).

**Author:** SansWord — [LinkedIn](https://www.linkedin.com/in/sansword/). Reach out if you'd like to talk about this comparison, speedcubing, or web apps built with Claude Code.

← [Back to Index](../../README.md)
