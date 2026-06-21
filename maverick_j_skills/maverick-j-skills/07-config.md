# Configuration Parameters and Usage Examples

> This file contains runtime parameter configuration, model selection recommendations, and practical usage examples.

---

## 1. Core Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max_rounds` | 5 | Maximum debate rounds. 3 rounds suffice for simple questions; set 5–7 for complex decisions |
| `convergence_score_target` | 0.8 | Convergence score threshold. Lower = easier to terminate; higher = deeper debate |
| `convergence_threshold` | 2 | Number of consecutive qualifying rounds required before termination. Prevents premature ending from a single accidental high score |
| `language` | auto | Output language. `auto` = follow the question's language, `zh` = Chinese, `en` = English |
| `transcript_compression_after_round` | 2 | Compress history transcript after N rounds to reduce context consumption |

---

## 2. Model Selection Recommendations

### Recommendations by Role

| Role | Temperature | Recommended characteristics | Example models |
|------|-------------|----------------------------|----------------|
| Advocate | 0.7 | Creative reasoning, argument construction | Claude Sonnet, GPT-4o |
| Critic | 0.7 | Critical thinking, flaw identification | Claude Sonnet, GPT-4o |
| Fact-Checker | 0.3 | Precise analysis, logical judgment | GPT-4o-mini, Claude Haiku |
| Moderator | 0.5 | Balanced judgment, convergence assessment | Claude Haiku, GPT-4o-mini |
| Report Gen | 0.5 | Structured writing | Claude Sonnet, GPT-4o |

### Cost Optimization Strategies

**Unified (low cost)**: All roles use the same fast model
```yaml
default_provider: claude
default_model: claude-haiku-4-5-20251001
```

**Mixed configuration (recommended)**: Strong model for key roles, fast model for supporting roles
```yaml
agents:
  advocate:
    provider: claude
    model: claude-sonnet-4-20250514
    temperature: 0.7
  critic:
    provider: openai
    model: gpt-4o
    temperature: 0.7
  fact_checker:
    provider: openai
    model: gpt-4o-mini
    temperature: 0.3
  moderator:
    provider: claude
    model: claude-haiku-4-5-20251001
    temperature: 0.5
  report_generator:
    provider: claude
    model: claude-sonnet-4-20250514
    temperature: 0.5
```

### Cost Reference

| Model | Input ($/1M tokens) | Output ($/1M tokens) | 3-round debate estimate |
|-------|---------------------|----------------------|------------------------|
| Claude Haiku 4.5 | $0.80 | $4.00 | ~$0.05 |
| GPT-4o-mini | $0.15 | $0.60 | ~$0.02 |
| Claude Sonnet 4 | $3.00 | $15.00 | ~$0.20 |
| GPT-4o | $2.50 | $10.00 | ~$0.15 |

---

## 3. Usage Examples

### Example 1: Technology Choice

```
Question: "Should we migrate our existing Java backend services to Go?"

Context: "50-person backend team, Java + Spring Boot stack running for 3 years.
Pain points:
1. High JVM memory usage, high deployment costs
2. Slow cold start, affecting Serverless scenarios
3. Some team members are interested in Go
4. Services are primarily API Gateways and microservices
5. Annual revenue ~$7M, tech budget ~$1.1M"

Rounds: 3
```

### Example 2: Business Decision

```
Question: "Should we build our own data analytics platform or buy a third-party solution?"

Context: "B2B SaaS company, 200+ enterprise customers, 2 TB/month data volume,
currently using Mixpanel at $180k/year, customers requesting custom dashboards and data exports"

Rounds: 5
```

### Example 3: Team Management

```
Question: "Should we switch from hybrid remote to fully remote?"

Context: "120-person tech company, currently 3 days/week in office.
San Francisco office rent $900k/year. 20% remote employees across 3 time zones."

Rounds: 3
```

---

## 4. Quick Trigger Template

Use this format in Claude Code / Cowork to trigger a debate:

```
Please perform a multi-agent adversarial debate analysis on the following decision:

Question: [your decision question]
Context: [optional background]
Rounds: [optional, default 3]
Language: [optional, default follows question language]
```

---

## 5. Follow-up Mode

After the first debate concludes, you can continue asking follow-up questions based on the conclusions. The system automatically injects the previous debate result as context:

```
Context automatically carried over in follow-ups:
  - Previous debate question
  - Debate conclusion summary
  - Recommended direction and confidence
  - Top 3 pro arguments
  - Top 3 con arguments
  - Top 3 unresolved disagreements
```

Example follow-up:
```
Based on the above debate results, if we decide to migrate to Go, what is the best incremental migration strategy?
```

---

## 6. Report Output Format

The final report is a two-part Markdown document:

```
# Decision Analysis Report

# Part 1: Full Debate Transcript          ← Complete dialogue from all 4 agents per round
  ## Round 1
    ### 🟢 Advocate                       ← Arguments + rebuttals + concessions + confidence shift
    ### 🔴 Critic
    ### 🔍 Fact-Checker                   ← Verdict per argument
    ### ⚖️ Moderator's Ruling             ← Summary + convergence score bar + decision
  ## Round 2
  ...

# Part 2: Summary Analysis               ← LLM-generated structured analysis
  ## Executive Summary
  ## Recommendation (direction + confidence + preconditions)
  ## Pro Arguments (sorted by strength)
  ## Con Arguments (sorted by strength)
  ## Resolved / Unresolved Disagreements
  ## Risk Factors
  ## Next Steps
  ## Debate Statistics
```

---

## 7. Tuning Guide

| Symptom | Adjustment |
|---------|-----------|
| Debate too shallow, arguments too surface-level | Increase `max_rounds`, use stronger models (Sonnet/GPT-4o) |
| Debate too expensive | Decrease `max_rounds`, use lightweight models for Fact-Checker/Moderator |
| Converges too fast, insufficient discussion | Raise `convergence_score_target` (e.g., 0.85) |
| Never converges, keeps looping | Lower `convergence_score_target`, inspect Moderator prompt |
| Arguments repeat | Ensure the "do not repeat already-refuted arguments" rule is enforced in Advocate/Critic prompts |
| Role confusion (single-LLM mode) | Strengthen role declaration at each role switch, use `---` as separator |
