# CLAUDE.md

Context and instructions for Claude Code working in this repo.

---

## What This Repo Is

`sans_learning` is a personal knowledge base — notes captured from learning sessions on random topics. It's not a course, not a structured curriculum. It's thinking made persistent.

The audience is future-me (and anyone else who finds it useful). Notes should be clear enough for a smart stranger to follow, but written with a personal voice — not like a Wikipedia article.

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
├── README.md                  # Index — always kept up to date
├── CLAUDE.md                  # This file
├── ai/                        # AI, LLMs, ML concepts
├── web-dev/                   # Frontend, backend, browser APIs
├── system-design/             # Architecture, infrastructure
└── _templates/
    └── topic-template.md      # Template for all new notes
```

New domain folders may be added as topics grow. See folder conventions below.

---

## Rules for New Notes

1. **Always use `_templates/topic-template.md`** as the starting structure
2. **Always add the back-to-index footer** — `← [Back to Index](../README.md)`
3. **Always update `README.md`** to include the new note under the correct domain section with a one-line description
4. File names should be `kebab-case.md` (e.g. `harness-engineering.md`)
5. Place the file in the correct domain folder. If unsure, ask.

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

*(Keep this table in sync with README.md index)*

---

*Last updated: 2026-04*
