# Argument Registry and Scoring

> This file defines the management logic, lifecycle tracking, and argument strength scoring algorithm for the ArgumentRegistry.

---

## 1. ArgumentRecord

Each argument in the registry is stored as an `ArgumentRecord`:

```
ArgumentRecord:
  argument: Argument           # The argument itself (id, claim, reasoning, evidence, status)
  raised_in_round: int         # Round in which the argument was raised
  raised_by: "advocate" | "critic"  # Which side raised it
  rebuttals: list[Rebuttal]    # List of rebuttals received
  fact_checks: list[FactCheck] # List of fact-check results received
  modification_history: list[str]  # Revision history log
```

---

## 2. Registry Operations

### register(argument, round, agent)

Register a new argument in the registry.

```
registry[arg.id] = ArgumentRecord(
    argument=arg,
    raised_in_round=round,
    raised_by=agent,
    rebuttals=[],
    fact_checks=[],
    modification_history=[]
)
```

### add_rebuttal(arg_id, rebuttal)

Append a rebuttal record to the target argument.

```
if arg_id in registry:
    registry[arg_id].rebuttals.append(rebuttal)
```

### add_fact_check(arg_id, check)

Append a fact-check result to the target argument. If the verdict is `flawed`, automatically marks the argument status as `REBUTTED`.

```
if arg_id in registry:
    registry[arg_id].fact_checks.append(check)
    if check.verdict == "flawed":
        registry[arg_id].argument.status = "rebutted"
        registry[arg_id].modification_history.append(f"Fact-check: {check.explanation}")
```

### update_status(arg_id, new_status, reason)

Manually update an argument's status (used for concessions and similar scenarios).

```
if arg_id in registry:
    registry[arg_id].argument.status = new_status
    if reason:
        registry[arg_id].modification_history.append(reason)
```

### get_active_arguments(side=None)

Retrieve all surviving arguments (status `active` or `modified`), optionally filtered by side.

```
results = [r for r in registry.values() if r.argument.status in ("active", "modified")]
if side:
    results = [r for r in results if r.raised_by == side]
return results
```

---

## 3. Argument Strength Scoring Algorithm

When generating the final report, a `strength` score is calculated for each surviving argument:

```
def calculate_strength(record: ArgumentRecord) -> int:
    score = 5                                    # Base score

    # +1 for each rebuttal challenge survived (surviving challenges shows resilience)
    score += len(record.rebuttals)

    # Adjustments from Fact-Checker verdicts
    for fc in record.fact_checks:
        if fc.verdict == "valid":
            score += 1                           # Verified as valid +1
        elif fc.verdict == "flawed":
            score -= 3                           # Identified as flawed -3

    # Clamp to [1, 10]
    return max(1, min(10, score))
```

**Scoring rationale**:
- Base score of 5; surviving more rebuttals demonstrates the argument is more battle-tested
- Fact-Checker `valid` adds 1; `flawed` deducts 3 (heavy penalty for logical flaws)
- `needs_context` and `unverifiable` do not affect the score

---

## 4. Survival Statistics

```
def get_survivor_stats(registry) -> dict:
    total = len(registry)
    active = count(r for r in registry.values() if r.status in ("active", "modified"))
    rebutted = count(r for r in registry.values() if r.status == "rebutted")
    conceded = count(r for r in registry.values() if r.status == "conceded")
    return {
        "total_raised": total,
        "survived": active,
        "rebutted": rebutted,
        "conceded": conceded,
    }
```

---

## 5. Registry Update Timing per Node

| Execution phase | Registry operation |
|----------------|-------------------|
| Advocate Node | `register()` new arguments + `add_rebuttal()` pro-side rebuttals + `update_status()` concessions |
| Critic Node   | `register()` new arguments + `add_rebuttal()` con-side rebuttals + `update_status()` concessions |
| Fact-Checker Node | `add_fact_check()` for all verdicts (flawed automatically triggers status change) |
| Moderator Node | Read-only; does not modify the registry |
| Report Node   | Read `get_active_arguments()` + `calculate_strength()` to generate scored arguments |
