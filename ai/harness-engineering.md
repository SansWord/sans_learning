# Harness Engineering

> *"You build the harness. Users ride the horse."*

> **Source:** This note was produced from [this discussion on Claude.ai](https://claude.ai/share/a84a11ea-b270-4e35-b59c-f3e083951050). *(AI-generated — may contain inaccuracies. Verify critical details independently.)*

← [Back to Index](../README.md)

---

## What Is It?

**Harness engineering** is the practice of building structured systems around AI models so users can get reliable value from them — without dealing with the raw, unpredictable model directly.

It lives at the intersection of software engineering and AI capability. You still need solid engineering underneath (APIs, databases, auth flows), but harness engineering is the discipline of **bridging structured systems with unstructured reasoning.**

```
[ Software Engineering ]
        ∩
  [ AI Capability ]
        =
[ Harness Engineering ]
```

---

## The Core Metaphor

| Role | What It Is |
|---|---|
| 🐴 **The horse** | The raw AI model — powerful but unpredictable on its own |
| 🪢 **The harness** | Your prompts, tools, workflows, guardrails |
| 🧑‍🦯 **The rider** | Your user — just wants to get somewhere |
| 🔧 **You** | The harness engineer — you enable the ride |

A poorly harnessed horse is dangerous. A poorly harnessed AI gives users wrong answers, breaks unexpectedly, or goes completely off course. Your job is making sure that **never happens to the rider** — even when the horse does something unexpected underneath.

---

## The Key Insight

> Harness engineering isn't about adding AI everywhere. It's about knowing **exactly where determinism ends** — and placing the AI horse precisely at that boundary. Not everywhere. Not as a feature bolted on. Only where judgment, variation, and reasoning are genuinely required.

---

## Core Components

| Component | What It Does |
|---|---|
| **Prompt Architecture** | Designing instructions that reliably elicit specific AI behaviors |
| **Tool & Function Binding** | Connecting AI to external capabilities — APIs, search, databases |
| **Memory & State Management** | Retaining context across sessions (RAG, vector stores, history) |
| **Orchestration & Chaining** | Multi-step pipelines where outputs feed into other models or tools |
| **Evaluation & Guardrails** | Output validators and safety layers that catch and correct failures |
| **Reliability Scaffolding** | Retry logic, structured output parsing, error recovery |

---

## When Do You Actually Need It?

The key question to ask first:

> **"Is there a place in this product where determinism genuinely breaks down?"**

| Condition | Use AI? |
|---|---|
| Input space is too large to hardcode | ✅ Yes |
| The right answer requires judgment, not lookup | ✅ Yes |
| Edge cases outnumber the happy path | ✅ Yes |
| Output needs to flex per user | ✅ Yes |
| Logic is fully rule-based (`if/else`) | ❌ Just build the app |
| Responses are always the same for the same input | ❌ Just build the app |

**If your product works fine with `if/else` logic — just build the app. AI adds cost and unpredictability to things that should be rock solid.**

---

## A Concrete Example

### ❌ Low Harness Value — A Rule-Based Auth Flow
Detecting a browser environment and switching auth strategies is pure logic — deterministic inputs, deterministic outputs. Build it without AI.

### ✅ High Harness Value — YouTube Summarizer
Every video is different. Length, topic, structure, speaker style — you can't hardcode responses for that. The right summary requires judgment. The output needs to flex per user. This product **only exists because AI exists.**

| | Rule-Based Auth Flow | YouTube Summarizer |
|---|---|---|
| Input space | Small, known | Infinite, unknown |
| Logic type | Rules-based | Judgment-based |
| Edge cases | Manageable | Endless |
| AI value | Low | High |

---

## Is It a New Kind of Product or a Software Practice?

**Both — but not equal weight.**

- **Primarily:** A new kind of product design — you're designing products where AI is a *necessary component*, not a feature bolted on
- **Secondarily:** A software engineering practice — but only for that specific class of product

If you treat it as just a software practice, you'll try to apply it everywhere and end up adding AI to things that don't need it.

---

## The "Am I Doing This?" Test

- Am I just *using* an AI tool? → **User**
- Am I configuring an AI tool for others? → **Power user / prompt engineer**
- Am I *building the system* that the AI operates inside of? → **Harness engineer**

The key signal: are you **designing the environment the AI lives in**, or just talking to it?

---

## Further Reading

### Foundations
- [AI Engineering — Chip Huyen](https://www.oreilly.com/library/view/ai-engineering/9781098166298/) — the most direct book on this topic
- [LLM Powered Autonomous Agents — Lilian Weng](https://lilianweng.github.io/posts/2023-06-23-agent/) — deep dive on orchestration and agentic systems

### Practical
- [Building LLM Applications for Production — Chip Huyen](https://huyenchip.com/2023/04/11/llm-engineering.html) — covers reliability and guardrail thinking
- [Prompt Engineering Guide](https://www.promptingguide.ai/) — prompt architecture fundamentals

### Mindset
- [Co-Intelligence — Ethan Mollick](https://www.oneusefulthing.org/) — where AI genuinely adds value vs. where it doesn't

---

*Captured from a learning conversation. Original thinking developed through dialogue, not a textbook.*

← [Back to Index](../README.md)
