# Freeform branch diff — `freeform-4.6` vs `freeform-4.7`

> Supporting detail for [the main comparison report](./comparison.md#diffstat-and-commits). Start there if you haven't read it — this doc is the raw enumeration (every commit, every changed file) behind the summary. Most readers won't need it.

← [Back to the main report (Diffstat and commits)](./comparison.md#diffstat-and-commits)

---

## Branches

| Branch | Model | Commits | Compare link (SHA-pinned) |
|---|---|---|---|
| [`freeform-4.6`](https://github.com/SansWord/sans_cube/tree/freeform-4.6) | Sonnet 4.6 | 10 | [main…freeform-4.6](https://github.com/SansWord/sans_cube/compare/55f5209...f9672d2) |
| [`freeform-4.7`](https://github.com/SansWord/sans_cube/tree/freeform-4.7) | Opus 4.7 | 12 | [main…freeform-4.7](https://github.com/SansWord/sans_cube/compare/55f5209...abd1aef) |

Compare links are pinned to `main@55f5209` (the state of main at the time of this writeup) so future commits on main won't alter the diff.

## Files changed (git diffstat)

**`freeform-4.6` vs `main`** — 14 files, 658 insertions, 12 deletions

```
docs/superpowers/plans/2026-04-17-freeform-method.md        | 439 ++++++
docs/superpowers/specs/2026-04-17-freeform-method-design.md | 127 +++
future.md                                                   |   2 +
src/components/MethodSelector.tsx                           |   4 +-
src/components/SolveHistorySidebar.tsx                      |   1 +
src/components/TrendsModal.tsx                              |   6 +-
src/methods/freeform.ts                                     |  14 +
src/methods/index.ts                                        |   4 +-
src/types/solve.ts                                          |   4 +-
src/utils/detectMethod.ts                                   |   1 +
tests/components/SolveHistorySidebar.test.tsx               |   4 +-
tests/filterSolves.test.ts                                  |  25 +-
tests/utils/detectMethod.test.ts                            |  12 +
tests/utils/recomputePhases.test.ts                         |  27 +-
```

**`freeform-4.7` vs `main`** — 17 files, 1046 insertions, 15 deletions

```
docs/devlog.md                                              |  22 +
docs/superpowers/plans/2026-04-17-freeform-method.md        | 819 +++++++++
docs/superpowers/specs/2026-04-17-freeform-method-design.md | 107 +++
future.md                                                   |   2 +-
src/components/MethodSelector.tsx                           |   4 +-
src/components/SolveHistorySidebar.tsx                      |   1 +
src/components/TrendsModal.tsx                              |   6 +-
src/hooks/useHashRouter.ts                                  |   6 +-
src/methods/freeform.ts                                     |  14 +
src/methods/index.ts                                        |   4 +-
src/types/solve.ts                                          |   4 +-
src/utils/detectMethod.ts                                   |   1 +
tests/components/SolveHistorySidebar.test.tsx               |   4 +-
tests/filterSolves.test.ts                                  |  23 +-
tests/hooks/useHashRouter.test.ts                           |   7 +
tests/utils/detectMethod.test.ts                            |  20 +
tests/utils/recomputePhases.test.ts                         |  17 +
```

Three files not present on `freeform-4.6` but added on `freeform-4.7`: `docs/devlog.md` (updated), `src/hooks/useHashRouter.ts`, and `tests/hooks/useHashRouter.test.ts`. The hash-router pair is the functional gap the main report centers on.

## Commit-by-commit

### `freeform-4.6` (Sonnet 4.6) — 10 commits

1. [`707fa0c`](https://github.com/SansWord/sans_cube/commit/707fa0c) docs: freeform method design spec + future.md data-driven registry item
2. [`5e7474a`](https://github.com/SansWord/sans_cube/commit/5e7474a) docs: freeform method implementation plan
3. [`dfb0ebf`](https://github.com/SansWord/sans_cube/commit/dfb0ebf) feat: add FREEFORM method definition and register in index
4. [`c09233a`](https://github.com/SansWord/sans_cube/commit/c09233a) fix: update SolveRecord.method comment to include freeform
5. [`0b0344d`](https://github.com/SansWord/sans_cube/commit/0b0344d) test: FREEFORM recompute produces one Solved phase
6. [`639b63c`](https://github.com/SansWord/sans_cube/commit/639b63c) test: filterSolves handles freeform method correctly
7. [`19371ae`](https://github.com/SansWord/sans_cube/commit/19371ae) feat: skip freeform solves in detectMethodMismatches
8. [`3d4b21c`](https://github.com/SansWord/sans_cube/commit/3d4b21c) feat: add Freeform to MethodSelector
9. [`7a48fc5`](https://github.com/SansWord/sans_cube/commit/7a48fc5) feat: add Freeform option to method filter in SolveHistorySidebar
10. [`f9672d2`](https://github.com/SansWord/sans_cube/commit/f9672d2) feat: add Freeform to TrendsModal method filter and color map

### `freeform-4.7` (Opus 4.7) — 12 commits

1. [`be8d11a`](https://github.com/SansWord/sans_cube/commit/be8d11a) docs: freeform method design spec
2. [`3b6a8a3`](https://github.com/SansWord/sans_cube/commit/3b6a8a3) docs: correct freeform spec timing language (hardware clock, not wall clock)
3. [`508874b`](https://github.com/SansWord/sans_cube/commit/508874b) docs: freeform method implementation plan
4. [`114631a`](https://github.com/SansWord/sans_cube/commit/114631a) feat: add Freeform method (single 'Solved' phase)
5. [`6183ff1`](https://github.com/SansWord/sans_cube/commit/6183ff1) test: FREEFORM method produces single Solved phase
6. [`d152074`](https://github.com/SansWord/sans_cube/commit/d152074) feat: widen MethodFilter to include 'freeform'
7. [`e6c00ff`](https://github.com/SansWord/sans_cube/commit/e6c00ff) feat: add Freeform to method selector dropdown
8. [`fc2e48f`](https://github.com/SansWord/sans_cube/commit/fc2e48f) feat: add Freeform option to sidebar filter
9. [`58522dd`](https://github.com/SansWord/sans_cube/commit/58522dd) feat: add Freeform to Trends filter and color map
10. [`1f7b751`](https://github.com/SansWord/sans_cube/commit/1f7b751) feat: accept method=freeform in #trends URL param
11. [`3015ecf`](https://github.com/SansWord/sans_cube/commit/3015ecf) feat: skip Freeform solves in method mismatch detection
12. [`abd1aef`](https://github.com/SansWord/sans_cube/commit/abd1aef) docs: v1.22.0 devlog entry + future.md update for Freeform method

### Observations on commit shape

- **Opus split the docs phase into two commits** (spec, then a self-correction to the spec's timing language — `3b6a8a3`) before moving to the plan. Sonnet bundled spec + `future.md` insight into a single commit (`707fa0c`).
- **Opus landed devlog + `future.md` strikethrough together** in one final commit (`abd1aef`). Sonnet never wrote a devlog entry; the `future.md` edit is Sonnet's *new* backlog item, committed alongside the spec rather than at the end.
- Both models used `feat:` / `test:` / `docs:` prefixes without being asked. This is project convention but neither model was given that rule explicitly — both inferred it from the git history.

---

← [Back to the main report (Diffstat and commits)](./comparison.md#diffstat-and-commits)
