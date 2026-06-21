# Debate Execution Protocol

> This file defines the debate's state structure, execution flow, convergence termination logic, and argument ID specification.

---

## 1. State Structure

The debate engine maintains a global `DebateState` passed between each agent node:

```
DebateState:
  question: str               # Decision question
  context: str | null          # Additional background
  current_round: int           # Current round (starts at 1)
  max_rounds: int              # Maximum rounds (default 5)
  convergence_score_target: float  # Convergence threshold (default 0.8)
  rounds: list[DebateRound]    # Historical round records
  argument_registry: dict      # Argument lifecycle registry (see 05-registry.md)
  status: "running" | "converged" | "max_rounds" | "error"
  convergence_reason: str | null
```

Each `DebateRound` contains:
```
DebateRound:
  round_number: int
  advocate: AgentResponse      # Pro-side output
  critic: AgentResponse        # Con-side output
  fact_check: FactCheckResponse # Fact-check output
  moderator: ModeratorResponse  # Ruling output
```

---

## 2. Argument ID Specification

```
Format: {ROLE_PREFIX}-R{round}-{index}

Pro-side: ADV-R1-01, ADV-R1-02, ADV-R2-01, ...
Con-side: CRT-R1-01, CRT-R1-02, CRT-R2-01, ...
```

- `ROLE_PREFIX`: `ADV` (Advocate) or `CRT` (Critic)
- `R{round}`: Round in which the argument was raised
- `{index}`: Sequence number within the round, starting at 01

Rebuttals reference the target argument ID via `target_argument_id`.

---

## 3. Argument Lifecycle

Each argument has 4 possible states with one-way transitions:

```
                    ┌── Effectively rebutted ──→ REBUTTED
                    │
  ACTIVE ──────────┼── Proponent concedes  ──→ CONCEDED
                    │
                    └── Revised and alive   ──→ MODIFIED (still alive)
```

- **ACTIVE**: Alive; not yet effectively rebutted
- **REBUTTED**: Effectively rebutted (Fact-Checker `flawed` verdict also triggers this)
- **CONCEDED**: Proponent voluntarily concedes the opponent's point
- **MODIFIED**: Revised and still alive (counts as survived)

---

## 4. Per-Round Execution Order

```
┌─────────────────────────────────────────────────────┐
│                  Single-Round Debate Flow            │
│                                                     │
│  1. Round Setup — increment round, clear temp data  │
│       ↓                                             │
│  2. Advocate — pro-side speaks                      │
│     · Round 1: present 3–5 core arguments           │
│     · Round 2+: rebut + revise/concede + new args   │
│       ↓                                             │
│  3. Critic — con-side speaks                        │
│     · Review pro-side → rebut + raise con args      │
│     · Respond to pro-side rebuttals against self    │
│       ↓                                             │
│  4. Fact-Checker — verify all arguments/rebuttals   │
│     · Verdicts: valid / flawed / needs_context /    │
│       unverifiable                                  │
│     · flawed → automatically marks arg as REBUTTED  │
│       ↓                                             │
│  5. Moderator — ruling                              │
│     · Summarize → identify divergences →            │
│       compute convergence score → rule continue/end │
│       ↓                                             │
│  6. Conditional branch:                             │
│     · should_continue = true → back to step 1      │
│     · should_continue = false → report generation  │
└─────────────────────────────────────────────────────┘
```

---

## 5. Convergence Termination Conditions

The debate terminates when **any one** of the following conditions is met:

| Condition | Description |
|-----------|-------------|
| Max rounds reached | `current_round >= max_rounds` |
| Moderator rules termination | `should_continue = false` |
| Consecutive high scores | 2 consecutive rounds with `convergence_score >= convergence_score_target` |
| Error state | `status == "error"` |

After termination, the **Report Generator** phase runs to produce the final `DecisionReport`.

---

## 6. Report Generation

After termination, a complete report with **two parts** is generated:

### Part 1: Full Debate Transcript

Renders the complete transcript for every round, including:
- Full output from each agent per round (arguments, rebuttals, concessions, confidence shifts)
- Fact-Checker verdicts and explanations for each argument
- Moderator summary, convergence score visualization, and ruling
- Debate termination status and reason

### Part 2: Summary Analysis

The Report Generator reads the full debate transcript and outputs a `DecisionReport`:

- **executive_summary**: 3–5 sentence summary
- **recommendation**: Recommended direction + confidence (high/medium/low) + preconditions
- **pro/con_arguments**: Only arguments with status `active` or `modified`, sorted by `strength` descending
- **unresolved_disagreements**: Core issues where no consensus was reached
- **next_steps**: Concrete actionable follow-up steps (vague phrases like "further research needed" are not allowed)

The final report is a **self-contained document**: readers can fully understand the decision analysis without having participated in the debate.

---

## 7. Execution Pseudocode

```python
async def run_adversarial_debate(question, context=None, max_rounds=5):
    state = init_state(question, context, max_rounds)

    while True:
        state.current_round += 1

        # 1. Advocate
        advocate_resp = await call_advocate(state)
        update_registry(state, advocate_resp, "advocate")

        # 2. Critic (sees Advocate's output)
        critic_resp = await call_critic(state, advocate_resp)
        update_registry(state, critic_resp, "critic")

        # 3. Fact-Checker (sees both)
        fact_check_resp = await call_fact_checker(state, advocate_resp, critic_resp)
        apply_fact_checks(state, fact_check_resp)

        # 4. Moderator (sees full transcript)
        moderator_resp = await call_moderator(state)
        state.rounds.append(DebateRound(advocate_resp, critic_resp, fact_check_resp, moderator_resp))

        # 5. Terminate?
        if should_terminate(state, moderator_resp):
            break

    # 6. Generate report
    return await call_report_generator(state)
```

---

## 8. Transcript Compression Strategy

To control context window length:

| Round position | Content passed |
|----------------|----------------|
| ≤ `transcript_compression_after_round` (default 2) | Full transcript |
| Earlier history rounds | Only Moderator's `round_summary` (summary replaces full text) |
| Most recent 1 round | Full transcript |
| Completed phases in the current round | Full output |

---

## 9. Error Handling and Retry

```
LLM calls:
  · Max 2 retries (MAX_RETRIES = 2)
  · Pydantic validation failure → append format correction instruction and retry
  · LLM API error → retry directly
  · All retries exhausted → raise RuntimeError

Model fallback:
  · Each agent can configure a fallback model
  · Automatically switches to fallback when primary model fails
```
