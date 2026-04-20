# CLAUDE.md

Context and instructions for Claude Code working in this repo.

---

## What This Repo Is

`sans_learning` is a personal knowledge base — notes captured from learning sessions on random topics. It's not a course, not a structured curriculum. It's thinking made persistent.

The audience is future-me (and anyone else who finds it useful). Notes should be clear enough for a smart stranger to follow, but written with a personal voice — not like a Wikipedia article.

Most notes originate from conversations with Claude.ai. They may contain hallucinations or inaccuracies — treat them as a starting point for understanding, not a ground truth.

---

## Tone & Writing Style

- **Semi-formal** — clear and precise, but not stiff or academic
- Write like you're explaining to a smart colleague, not writing documentation
- Prefer concrete examples over abstract definitions
- Use analogies when they genuinely help (don't force them)
- Short sentences over long ones
- Avoid filler phrases like "It's worth noting that..." or "In conclusion..."

---

## Folder Structure

```
sans_learning/
├── README.md                      # Index — always kept up to date
├── CLAUDE.md                      # This file
├── ai/                            # AI, LLMs, ML concepts
│   ├── single-topic.md            #   Single-file notes live at the domain root
│   └── multi-file-topic/          #   Multi-file notes get their own subfolder
│       ├── main.md                #     Main doc (this is what README links to)
│       └── supporting-doc.md      #     Supporting docs that the main doc links out to
├── web-dev/                       # Frontend, backend, browser APIs
├── system-design/                 # Architecture, infrastructure
└── _templates/
    └── topic-template.md          # Template for concept notes
```

New domain folders may be added as topics grow. See folder conventions below.

Most notes are single-file concept explainers. Multi-file notes (e.g. a case study with supporting data appendices) live in their own subfolder under the domain; the main doc is what `README.md` points to, and supporting docs link back to it.

---

## Rules for New Notes

1. **Use `_templates/topic-template.md`** as the starting structure. The template fits concept explainers; formats that genuinely don't fit (e.g. case studies, comparison reports, experiment writeups) may diverge — **ask first, then tag the note as a template exception** in the Current Notes table below
2. **Always add the back-to-index footer** — `← [Back to Index](../README.md)`
3. **Always update `README.md`** to include the new note under the correct domain section with a one-line description
4. File names should be `kebab-case.md` (e.g. `harness-engineering.md`)
5. Place the file in the correct domain folder. If unsure, ask.
6. **If the note originated from a Claude.ai conversation**, add a source blockquote near the bottom (before the back-to-index footer) in this format:
   ```
   > **Source:** This note was produced from [this discussion on Claude.ai](url). *(AI-generated — may contain inaccuracies. Verify critical details independently.)*
   ```

---

## Folder Conventions

- Use existing domain folders when a topic fits
- If a new folder seems needed, **suggest it and wait for approval** before creating it
- Domain folders are broad (e.g. `ai/` covers LLMs, ML, agents, prompt engineering)
- Don't over-fragment into too many folders

---

## What Claude Code Should Do

- Help draft, refine, and expand notes following the template and tone above
- Update `README.md` index whenever a new note is added
- Suggest structure improvements if something feels off
- Flag if a topic already exists before creating a duplicate

## What Claude Code Should NOT Do

- Create new domain folders without approval
- Rewrite or restructure existing notes without being asked
- Use formal/academic language — keep it semi-formal
- Add notes that aren't based on actual learning sessions or discussions

---

## Current Notes

| File | Domain | Description |
|---|---|---|
| `ai/harness-engineering.md` | AI | Building structured systems around AI models |
| `ai/claude-code-model-comparison/comparison.md` | AI | Same-feature comparison of Claude Sonnet 4.6 vs Opus 4.7 in Claude Code — **template exception** (case study, multi-file note) |
| `ai/claude-code-feature-cost-analysis/opus-4-7-cost-analysis.md` | AI | Cost analysis of a ~16-point feature built on Opus 4.7 across three clean-context sessions — **template exception** (case study, multi-file note) |

*(Keep this table in sync with README.md index)*

---

*Last updated: 2026-04*
