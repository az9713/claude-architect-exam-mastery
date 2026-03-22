# Domain 1 Cheat Sheet: Agentic Architecture & Orchestration
**Weight: 27% | Task Statements: 1.1 – 1.7 | Highest-weighted domain**

---

## Key Concepts

| Concept | Definition | Exam Relevance |
|---------|-----------|----------------|
| Agentic loop | Request → inspect `stop_reason` → execute tool → return result → repeat | Terminate on `"end_turn"`, continue on `"tool_use"` |
| `stop_reason` | `"tool_use"` = execute tool; `"end_turn"` = done | Primary loop control signal |
| Hub-and-spoke | Coordinator manages all inter-subagent communication | All routing through coordinator; no direct subagent-to-subagent calls |
| Isolated subagent context | Subagents do NOT inherit coordinator's conversation history | Must explicitly pass context in prompt |
| `Task` tool | Mechanism to spawn subagents | `allowedTools` must include `"Task"` for coordinator to spawn |
| `AgentDefinition` | Config with description, system prompt, tool restrictions per subagent | Defines the scope and role of each subagent |
| Parallel spawning | Multiple `Task` tool calls in ONE coordinator response | Enables parallel subagents; separate turns = sequential |
| `fork_session` | Creates independent branch from shared analysis baseline | Divergent approaches from same starting point |
| `--resume <name>` | Continues a named prior session | Use when prior context is still valid |
| Programmatic enforcement | Code-level prerequisites that block tool calls | Deterministic; prompt instructions are probabilistic |
| PostToolUse hook | Intercepts tool results before model processes them | Normalize heterogeneous formats |
| PreToolCall hook | Intercepts outgoing tool calls | Block policy-violating actions (e.g., refunds > $500) |
| Prompt chaining | Fixed sequential pipeline of steps | Predictable multi-aspect tasks (per-file then cross-file) |
| Dynamic decomposition | Adaptive subtasks based on what is discovered | Open-ended investigation tasks |
| Structured handoff | Summary with customer ID, root cause, refund amount, recommended action | Human agents lack conversation transcript |

---

## Decision Matrix

### Programmatic Enforcement vs. Prompt Instructions

| Scenario | Mechanism |
|----------|-----------|
| Must verify customer identity before refund (financial consequence) | Programmatic prerequisite |
| Preferred but not critical tool ordering | Prompt instruction |
| Refund amount cap ($500 limit) | PreToolCall hook |
| Format preference for output | Prompt instruction |
| Any rule with zero-failure-rate requirement | Programmatic / hook |

### Session Resumption vs. Fresh Start

| Condition | Approach |
|-----------|----------|
| Prior context still valid | `--resume <session-name>` |
| Files have changed since last session | Resume + inform about specific file changes |
| Prior tool results are stale | Fresh session with injected structured summary |
| Exploring two approaches from shared baseline | `fork_session` |

### Prompt Chaining vs. Dynamic Decomposition

| Task Characteristic | Approach |
|---------------------|----------|
| Predictable steps, known scope | Prompt chaining (sequential) |
| Open-ended, findings drive next steps | Dynamic decomposition |
| Code review (per-file + cross-file) | Prompt chaining |
| "Add comprehensive tests to legacy codebase" | Dynamic decomposition |

---

## Code/Config Reference

```python
# Agentic loop — correct pattern
while True:
    response = client.messages.create(
        model="claude-opus-4-6",
        tools=tools,
        messages=messages
    )
    if response.stop_reason == "end_turn":
        break  # Done
    if response.stop_reason == "tool_use":
        # Execute tool, append result
        tool_result = execute_tool(response)
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": [tool_result]})

# Parallel subagent spawning — single coordinator response
# NOT across multiple turns
coordinator_response = client.messages.create(
    tools=tools_including_Task,
    tool_choice={"type": "any"},
    messages=[{"role": "user", "content": "Research AI market: spawn search + analysis agents"}]
)
# If coordinator emits two Task calls in one response → parallel execution

# PostToolUse hook — normalize before model sees result
def post_tool_use_hook(tool_name, result):
    if tool_name == "get_customer":
        # Normalize timestamp format
        result["created_at"] = convert_to_iso8601(result.get("unix_timestamp"))
    return result
```

---

## Anti-Patterns (Common Wrong Answers)

- **Loop termination by text parsing**: checking for "I've completed" in assistant text — wrong; use `stop_reason`
- **Arbitrary iteration caps as primary stop**: always stopping at N iterations — wrong; use `stop_reason`
- **Direct subagent-to-subagent communication**: skipping coordinator — reduces observability, breaks error handling
- **Subagents inherit parent context automatically**: false; always explicitly pass needed context
- **Prompt-only enforcement for critical financial rules**: probabilistic, not guaranteed
- **Sequential subagent spawning**: emitting Task calls in separate turns — use multiple Task calls in one response for parallel
- **Narrow decomposition**: assigning too-specific subtopics causes incomplete coverage of broad topics

---

## If You Remember Nothing Else

1. `stop_reason == "tool_use"` → execute tool and continue; `stop_reason == "end_turn"` → done
2. Subagents have NO automatic access to coordinator's context — pass it explicitly in the prompt
3. Programmatic enforcement (hooks, prerequisites) = deterministic; prompt instructions = probabilistic — critical rules require code-level enforcement
4. Multiple `Task` tool calls in ONE response = parallel subagents; separate turns = sequential
5. `fork_session` creates independent branches from a shared baseline; `--resume` continues an existing session
