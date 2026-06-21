# Multi-Framework Integration Guide

> This file explains how to integrate the debate Skill into different agent frameworks.

---

## 1. Claude Code / Claude Cowork (Single LLM, Self Role-Switching)

**Simplest mode**: No extra framework needed — Claude itself plays each of the 4 roles in sequence.

### Integration Method A: As a System Prompt

Inject the contents of `SKILL.md` + `01-identity.md` + `02-protocol.md` into the System Prompt. Claude follows the execution protocol and completes multi-round debate within a single conversation.

### Integration Method B: As a Skill / Command

```
# .claude/commands/debate.md or .github/skills/debate/SKILL.md
Triggers: "debate analysis", "pro/con argumentation", "should we..."

Execution flow:
1. Load 01-identity.md and 02-protocol.md (understand roles and flow)
2. Load 03-prompts.md (obtain prompt templates)
3. Load 04-schemas.md (clarify output format)
4. Follow the execution order in 02-protocol.md, playing Advocate → Critic → Fact-Checker → Moderator in turn
5. Loop until convergence
6. Generate DecisionReport
```

### Single-LLM Execution Notes

- When switching roles, always **explicitly declare the current role** to avoid role confusion
- Use `---` to separate each role's output for easy reader distinction
- The Moderator's `should_continue` decision must be honest — do not force continuation just to show more rounds

---

## 2. LangGraph (Native StateGraph Mapping)

The debate protocol maps naturally onto LangGraph's StateGraph:

```python
from langgraph.graph import StateGraph, END

workflow = StateGraph(DebateState)

# Nodes
workflow.add_node("round_setup", round_setup_node)
workflow.add_node("advocate", partial(advocate_node, router=router))
workflow.add_node("critic", partial(critic_node, router=router))
workflow.add_node("fact_checker", partial(fact_checker_node, router=router))
workflow.add_node("moderator", partial(moderator_node, router=router))
workflow.add_node("report", partial(report_node, router=router))

# Edges
workflow.set_entry_point("round_setup")
workflow.add_edge("round_setup", "advocate")
workflow.add_edge("advocate", "critic")
workflow.add_edge("critic", "fact_checker")
workflow.add_edge("fact_checker", "moderator")

# Conditional edge: continue or terminate
workflow.add_conditional_edges(
    "moderator",
    should_continue,    # returns "continue" or "terminate"
    {"continue": "round_setup", "terminate": "report"}
)
workflow.add_edge("report", END)

app = workflow.compile()
```

**Key implementation details**:
- Each node function signature: `async def xxx_node(state: DebateState, router: ModelRouter) -> dict`
- Nodes return a `dict` of incremental state updates (LangGraph merges automatically)
- `router` is injected via `functools.partial()`

---

## 3. OpenAI Agents SDK

```python
from agents import Agent, Runner

advocate = Agent(
    name="Advocate",
    instructions=ADVOCATE_SYSTEM_PROMPT,
    output_type=AgentResponse,
)

critic = Agent(
    name="Critic",
    instructions=CRITIC_SYSTEM_PROMPT,
    output_type=AgentResponse,
)

fact_checker = Agent(
    name="FactChecker",
    instructions=FACT_CHECKER_SYSTEM_PROMPT,
    output_type=FactCheckResponse,
)

moderator = Agent(
    name="Moderator",
    instructions=MODERATOR_SYSTEM_PROMPT,
    output_type=ModeratorResponse,
)

# Orchestrator controls round rotation
orchestrator = Agent(
    name="DebateOrchestrator",
    instructions="Orchestrate debate rounds in the order Advocate → Critic → FactChecker → Moderator",
    handoffs=[advocate, critic, fact_checker, moderator],
)

result = await Runner.run(orchestrator, input=question)
```

---

## 4. CrewAI

```python
from crewai import Agent, Task, Crew, Process

advocate = Agent(
    role="Advocate",
    goal="Build strongest pro-side case for the decision",
    backstory=ADVOCATE_SYSTEM_PROMPT,
    llm=your_llm,
)

critic = Agent(
    role="Critic",
    goal="Systematically challenge pro-side and build con-side case",
    backstory=CRITIC_SYSTEM_PROMPT,
    llm=your_llm,
)

fact_checker = Agent(
    role="FactChecker",
    goal="Evaluate logical consistency and factual accuracy",
    backstory=FACT_CHECKER_SYSTEM_PROMPT,
    llm=your_llm,
)

moderator = Agent(
    role="Moderator",
    goal="Judge convergence and guide debate focus",
    backstory=MODERATOR_SYSTEM_PROMPT,
    llm=your_llm,
)

# Define per-round Tasks
def create_round_tasks(round_num):
    return [
        Task(description=f"Round {round_num}: Present pro-side arguments", agent=advocate),
        Task(description=f"Round {round_num}: Challenge and rebut", agent=critic),
        Task(description=f"Round {round_num}: Fact-check all arguments", agent=fact_checker),
        Task(description=f"Round {round_num}: Judge convergence", agent=moderator),
    ]

crew = Crew(
    agents=[advocate, critic, fact_checker, moderator],
    tasks=create_round_tasks(1),  # Dynamically append subsequent rounds
    process=Process.sequential,
)
```

---

## 5. AutoGen

```python
from autogen import AssistantAgent, GroupChat, GroupChatManager

advocate = AssistantAgent("Advocate", system_message=ADVOCATE_SYSTEM_PROMPT)
critic = AssistantAgent("Critic", system_message=CRITIC_SYSTEM_PROMPT)
fact_checker = AssistantAgent("FactChecker", system_message=FACT_CHECKER_SYSTEM_PROMPT)
moderator = AssistantAgent("Moderator", system_message=MODERATOR_SYSTEM_PROMPT)

groupchat = GroupChat(
    agents=[advocate, critic, fact_checker, moderator],
    messages=[],
    max_round=20,  # 4 agents × 5 rounds
    speaker_selection_method="round_robin",  # Fixed rotation order
)

manager = GroupChatManager(groupchat=groupchat, llm_config=llm_config)
manager.initiate_chat(message=f"Decision question: {question}")
```

---

## 6. Universal Integration Checklist

Regardless of framework, ensure the following elements are in place:

| Element | Description |
|---------|-------------|
| ✅ 4 agent roles | Advocate, Critic, Fact-Checker, Moderator |
| ✅ Fixed execution order | A → C → FC → M (per round) |
| ✅ Structured output | Each agent's output conforms to the JSON schemas in 04-schemas.md |
| ✅ Argument ID system | ADV-R{n}-{nn} / CRT-R{n}-{nn} format |
| ✅ Argument registry | Track each argument's lifecycle (see 05-registry.md) |
| ✅ Convergence termination logic | Moderator ruling + consecutive high scores + max-rounds fallback |
| ✅ Report generation | Generate structured DecisionReport after termination |
