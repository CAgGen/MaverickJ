---
name: marverick-j-skills
description: Launch a multi-agent adversarial debate on any decision question. Produces a structured decision report through pro/con argumentation, fact-checking, and convergence judgment.
---

# Adversarial Multi-Agent Debate Engine

> Launch a multi-agent adversarial debate on any decision question. Produces a structured decision report through pro/con argumentation, fact-checking, and convergence judgment.

## When to Use This Skill

- The user poses a **"Should we X?"** type of decision question
- A technology choice requires **multi-perspective trade-off analysis**
- A strategic decision needs **structured argumentation** rather than gut instinct
- The user explicitly requests "debate analysis", "pro/con argumentation", or "Devil's Advocate"

## Core Idea

Single-perspective analysis is prone to confirmation bias. By forcing **adversarial pro/con debate + neutral fact-checking + moderator convergence judgment**, every argument must survive multiple rounds of challenge — only arguments that "survive" make it into the final report.

## 30-Second Overview

```
                    Per-Round Debate Flow
  ┌──────────────────────────────────────┐
  │  Advocate (pro) → Critic (con)       │
  │       ↓                ↓             │
  │    Fact-Checker (neutral audit)      │
  │            ↓                         │
  │    Moderator (ruling: continue/end)  │
  └──────────┬───────────────────────────┘
             │ Converged? → Generate DecisionReport
             │ Not yet?   → Next round
             ↓
```

4 Agents · Up to 5 rounds · Argument lifecycle tracking · Structured JSON output · **Full debate transcript + summary analysis** two-part report

## Sub-File Index

Load the following files on demand — **no need to read all at once**:

| File | Purpose | When to load |
|------|---------|--------------|
| [01-identity.md](01-identity.md) | Agent role definitions and responsibility boundaries | **First** — understand the 4 roles |
| [02-protocol.md](02-protocol.md) | Debate execution protocol, state machine, convergence logic | **First** — understand how the debate runs |
| [03-prompts.md](03-prompts.md) | Complete system prompts for all 5 agents | **At execution** — when making actual LLM calls |
| [04-schemas.md](04-schemas.md) | All structured output JSON schemas | **At execution** — when parsing LLM output |
| [05-registry.md](05-registry.md) | Argument registry, lifecycle tracking, scoring logic | **At execution** — when tracking argument survival |
| [06-integration.md](06-integration.md) | Multi-framework integration guide (LangGraph/CrewAI/AutoGen/…) | **At integration** — when connecting to a specific framework |
| [07-config.md](07-config.md) | Config parameters, model selection, usage examples | **At config** — when tuning or checking examples |

### Recommended Loading Order

**Minimum startup set** (understand + execute):
1. `01-identity.md` → who the roles are
2. `02-protocol.md` → how it runs

**Full execution set** (actually run the debate):
3. `03-prompts.md` → LLM prompts
4. `04-schemas.md` → output format
5. `05-registry.md` → argument tracking

**Reference on demand**:
6. `06-integration.md` → framework integration
7. `07-config.md` → tuning & examples
