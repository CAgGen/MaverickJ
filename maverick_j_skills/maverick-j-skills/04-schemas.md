# Structured Output Schemas

> This file defines the JSON schemas for all agent inputs/outputs, used for LLM structured output parsing.

---

## 1. Argument

```json
{
  "type": "object",
  "required": ["id", "claim", "reasoning"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Argument ID, e.g. ADV-R1-01 or CRT-R1-01",
      "pattern": "^(ADV|CRT)-R\\d+-\\d+$"
    },
    "claim": {
      "type": "string",
      "description": "The argument claim"
    },
    "reasoning": {
      "type": "string",
      "description": "Reasoning process"
    },
    "evidence": {
      "type": ["string", "null"],
      "description": "Supporting evidence (optional)"
    },
    "status": {
      "type": "string",
      "enum": ["active", "rebutted", "conceded", "modified"],
      "default": "active",
      "description": "Argument status"
    }
  }
}
```

---

## 2. Rebuttal

```json
{
  "type": "object",
  "required": ["target_argument_id", "counter_claim", "reasoning"],
  "properties": {
    "target_argument_id": {
      "type": "string",
      "description": "ID of the argument being rebutted"
    },
    "counter_claim": {
      "type": "string",
      "description": "Counter-claim"
    },
    "reasoning": {
      "type": "string",
      "description": "Rebuttal reasoning"
    }
  }
}
```

---

## 3. FactCheck

```json
{
  "type": "object",
  "required": ["target_argument_id", "verdict", "explanation"],
  "properties": {
    "target_argument_id": {
      "type": "string",
      "description": "ID of the argument being checked"
    },
    "verdict": {
      "type": "string",
      "enum": ["valid", "flawed", "needs_context", "unverifiable"],
      "description": "Fact-check verdict"
    },
    "explanation": {
      "type": "string",
      "description": "Explanation of the verdict"
    },
    "correction": {
      "type": ["string", "null"],
      "description": "Suggested correction (optional)"
    },
    "fallacy_type": {
      "type": ["string", "null"],
      "description": "Fallacy type (optional; should be filled when verdict=flawed)"
    }
  }
}
```

---

## 4. AgentResponse (Advocate / Critic output)

```json
{
  "type": "object",
  "required": ["agent_role", "arguments"],
  "properties": {
    "agent_role": {
      "type": "string",
      "enum": ["advocate", "critic"],
      "description": "Agent role"
    },
    "arguments": {
      "type": "array",
      "items": { "$ref": "#/Argument" },
      "description": "Arguments raised this round"
    },
    "rebuttals": {
      "type": "array",
      "items": { "$ref": "#/Rebuttal" },
      "default": [],
      "description": "Rebuttals against the opponent's arguments"
    },
    "concessions": {
      "type": "array",
      "items": { "type": "string" },
      "default": [],
      "description": "Points conceded to the opponent"
    },
    "confidence_shift": {
      "type": "number",
      "minimum": -1,
      "maximum": 1,
      "default": 0,
      "description": "Confidence change in own position this round"
    }
  }
}
```

---

## 5. FactCheckResponse (Fact-Checker output)

```json
{
  "type": "object",
  "required": ["checks", "overall_assessment"],
  "properties": {
    "checks": {
      "type": "array",
      "items": { "$ref": "#/FactCheck" },
      "description": "All fact-check results"
    },
    "overall_assessment": {
      "type": "string",
      "description": "Overall assessment of argumentation quality this round"
    }
  }
}
```

---

## 6. ModeratorResponse (Moderator output)

```json
{
  "type": "object",
  "required": ["round_summary", "key_divergences", "convergence_score", "should_continue"],
  "properties": {
    "round_summary": {
      "type": "string",
      "description": "Summary of this round"
    },
    "key_divergences": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Current key unresolved divergences"
    },
    "convergence_score": {
      "type": "number",
      "minimum": 0,
      "maximum": 1,
      "description": "Convergence score"
    },
    "should_continue": {
      "type": "boolean",
      "description": "Whether to continue the debate"
    },
    "guidance_for_next_round": {
      "type": ["string", "null"],
      "description": "Focus guidance for the next round (optional)"
    }
  }
}
```

---

## 7. DecisionReport (Final decision report)

```json
{
  "type": "object",
  "required": ["question", "executive_summary", "recommendation", "pro_arguments", "con_arguments", "debate_stats"],
  "properties": {
    "question": {
      "type": "string",
      "description": "Decision question"
    },
    "executive_summary": {
      "type": "string",
      "description": "3–5 sentence summary"
    },
    "recommendation": {
      "type": "object",
      "required": ["direction", "confidence", "conditions"],
      "properties": {
        "direction": { "type": "string", "description": "Recommended direction" },
        "confidence": { "type": "string", "enum": ["high", "medium", "low"] },
        "conditions": { "type": "array", "items": { "type": "string" }, "description": "Preconditions for the recommendation to hold" }
      }
    },
    "pro_arguments": {
      "type": "array",
      "items": { "$ref": "#/ScoredArgument" },
      "description": "Pro-side arguments, sorted by strength descending"
    },
    "con_arguments": {
      "type": "array",
      "items": { "$ref": "#/ScoredArgument" },
      "description": "Con-side arguments, sorted by strength descending"
    },
    "resolved_disagreements": {
      "type": "array",
      "items": { "type": "string" },
      "default": [],
      "description": "Issues where consensus was reached"
    },
    "unresolved_disagreements": {
      "type": "array",
      "items": { "type": "string" },
      "default": [],
      "description": "Issues still in dispute"
    },
    "risk_factors": {
      "type": "array",
      "items": { "type": "string" },
      "default": [],
      "description": "Risk factors"
    },
    "next_steps": {
      "type": "array",
      "items": { "type": "string" },
      "default": [],
      "description": "Concrete follow-up actions"
    },
    "debate_stats": {
      "type": "object",
      "required": ["total_rounds", "arguments_raised", "arguments_survived", "convergence_achieved"],
      "properties": {
        "total_rounds": { "type": "integer" },
        "arguments_raised": { "type": "integer" },
        "arguments_survived": { "type": "integer" },
        "convergence_achieved": { "type": "boolean" },
        "total_tokens": { "type": "integer", "default": 0 },
        "total_cost_usd": { "type": "number", "default": 0 }
      }
    }
  }
}
```

### ScoredArgument

```json
{
  "type": "object",
  "required": ["claim", "strength", "survived_challenges"],
  "properties": {
    "claim": { "type": "string", "description": "Argument claim" },
    "strength": { "type": "integer", "minimum": 1, "maximum": 10, "description": "Argument strength score" },
    "survived_challenges": { "type": "integer", "description": "Number of challenges survived" },
    "modifications": { "type": "array", "items": { "type": "string" }, "default": [], "description": "Revision history" },
    "supporting_evidence": { "type": ["string", "null"], "description": "Supporting evidence" }
  }
}
```
