# Domain 1: Agentic Architecture & Orchestration
## Claude Certified Architect – Foundations Exam Study Notes
**Weight: 27% of scored content**

> Domain 1 is the single largest domain on the exam. Nearly three in ten scored questions test your ability to design, implement, and reason about agentic systems. Expect scenario-based questions drawn from the Customer Support Resolution Agent (Scenario 1), Multi-Agent Research System (Scenario 3), and Developer Productivity (Scenario 4) contexts.

---

## Table of Contents

1. [Task Statement 1.1: Design and Implement Agentic Loops](#task-statement-11)
2. [Task Statement 1.2: Orchestrate Multi-Agent Systems](#task-statement-12)
3. [Task Statement 1.3: Configure Subagent Invocation, Context Passing, and Spawning](#task-statement-13)
4. [Task Statement 1.4: Implement Multi-Step Workflows with Enforcement and Handoff Patterns](#task-statement-14)
5. [Task Statement 1.5: Apply Agent SDK Hooks for Tool Call Interception and Data Normalization](#task-statement-15)
6. [Task Statement 1.6: Design Task Decomposition Strategies for Complex Workflows](#task-statement-16)
7. [Task Statement 1.7: Manage Session State, Resumption, and Forking](#task-statement-17)
8. [Key Terminology Quick Reference](#key-terminology-quick-reference)
9. [Decision Matrix](#decision-matrix)
10. [If You're Short on Time](#if-youre-short-on-time)

---

## Task Statement 1.1

# Task Statement 1.1: Design and Implement Agentic Loops for Autonomous Task Execution

---

### What You Need to Know

- The **agentic loop lifecycle** consists of four repeating steps: (1) send a request to Claude, (2) inspect the response's `stop_reason` field, (3) if `stop_reason == "tool_use"`, execute the requested tool(s), and (4) append tool results to the conversation history and loop back to step 1.
- The loop **terminates** when `stop_reason == "end_turn"`, which signals that Claude has finished reasoning and no further tool calls are needed.
- **Tool results are appended to conversation history** — not sent separately. Each tool result becomes a new message in the ongoing `messages` array, preserving the full reasoning chain so the model can incorporate new information at each iteration.
- **Model-driven decision-making** means Claude reasons about which tool to call next based on accumulated context. This is fundamentally different from **pre-configured decision trees** or hard-coded tool sequences where the orchestrating code determines what runs next based on business logic.
- The two valid `stop_reason` values that matter for agentic loops are `"tool_use"` (continue) and `"end_turn"` (stop). Other values like `"max_tokens"` or `"stop_sequence"` indicate edge conditions that require separate handling.
- Claude can emit **both a text block and one or more tool_use blocks in the same response**. The presence of a text block does NOT mean the loop should end — only `stop_reason == "end_turn"` means that.

---

### What You Need to Be Able to Do

- Implement agentic loop control flow that **continues** when `stop_reason == "tool_use"` and **terminates** when `stop_reason == "end_turn"`.
- Correctly **append tool results** to the `messages` array between iterations so the model retains full context across all loop turns.
- Recognize and avoid the following anti-patterns:
  - Parsing natural language in Claude's text output to decide whether to continue the loop.
  - Setting an arbitrary iteration cap (e.g., `max_iterations = 10`) as the **primary** stopping mechanism.
  - Checking for the presence of text content in the assistant message as a completion signal.
- Handle the `tool_use` content blocks correctly: extract `id`, `name`, and `input` from each block, execute the corresponding tool, and package the result as a `tool_result` content block referencing the same `id`.

---

### Deep Dive

#### The Agentic Loop in Practice

The agentic loop is the fundamental execution model for all Claude-based autonomous agents. At its core, it is a `while` loop governed by the `stop_reason` field returned in every Claude API response. Understanding what drives loop continuation and termination — and what does not — is the single most important concept in this task statement.

The canonical agentic loop skeleton:

```python
while True:
    response = client.messages.create(model=..., tools=tools, messages=messages)
    messages.append({"role": "assistant", "content": response.content})  # append FIRST

    if response.stop_reason == "end_turn":
        return extract_text(response)  # done

    if response.stop_reason == "tool_use":
        tool_results = [{"type": "tool_result", "tool_use_id": b.id,
                         "content": execute_tool(b.name, b.input)}
                        for b in response.content if b.type == "tool_use"]
        messages.append({"role": "user", "content": tool_results})
        # loop continues

    else:
        raise RuntimeError(f"Unexpected stop_reason: {response.stop_reason}")
```

#### stop_reason Decision Table

| stop_reason | Loop action | Notes |
|---|---|---|
| `"tool_use"` | Continue — execute tools and loop | Claude wants to invoke one or more tools |
| `"end_turn"` | Terminate — task is complete | The only correct primary termination signal |
| `"max_tokens"` | Handle as edge condition | Response was cut off; may need to resume or raise an error |
| `"stop_sequence"` | Handle as edge condition | A custom stop sequence was hit; context-dependent handling |

Only the first two values drive normal loop control flow.

#### Why Conversation History Is the Memory Layer

The `messages` array is not just an API formality — it is the agent's working memory. Every tool result appended to this array becomes part of the context that Claude reasons over on the next API call. If a tool returns a customer's verified ID, that ID is available in the next turn because it exists in `messages`. If you forget to append the tool result, Claude will be reasoning in the dark, potentially calling the same tool again or making decisions without the information it retrieved.

This is why the correct order of operations matters: (1) append the assistant's response (which may contain tool use blocks), then (2) execute the tools, then (3) append the tool results as a new user message. Skipping step 1 means Claude's own reasoning is missing from history when it resumes.

#### Model-Driven vs Pre-Configured Execution

A pre-configured decision tree looks like this: "If the user mentions a refund, call `lookup_order`, then call `process_refund`." The business logic in the calling code drives what runs next. A model-driven agentic loop looks like this: "Here are the available tools. Claude, figure out what to call." Claude inspects the available tools, the conversation history, and the user's request, then decides autonomously which tool to invoke and in what order.

The exam tests whether you understand this distinction. Model-driven loops are more flexible and can handle novel situations the pre-configured path did not anticipate, but they require that Claude has adequate context and clear tool descriptions to make good decisions. Pre-configured sequences provide stronger determinism but require business logic to be maintained in the orchestration code.

#### Iteration Caps as Safety Nets

While arbitrary iteration caps should not be the *primary* stopping mechanism, they are a legitimate *safety net*. A loop that runs to 1,000 iterations because a tool keeps failing is a bug, not a feature. The correct pattern is: terminate on `end_turn` (primary), handle `max_tokens` gracefully (secondary), and use an iteration cap as an emergency circuit breaker with appropriate logging (tertiary). The exam distinguishes between using a cap as the sole stopping condition versus using it defensively alongside proper `stop_reason` handling.

---

### Exam Tip

Questions testing this task statement typically present a code snippet with a flawed loop and ask you to identify the problem. Watch for:
- A loop that terminates when it finds "text" content in the response instead of checking `stop_reason`.
- A loop that checks a counter as the only termination condition.
- A loop that executes tools but appends results in the wrong format (e.g., as a plain string instead of a `tool_result` content block).
- A loop that skips appending the assistant's response before appending tool results.

The correct answer will always be the one that uses `stop_reason == "end_turn"` as the definitive termination signal.

---

### Anti-Pattern Alert

| Anti-Pattern | Why It Fails |
|---|---|
| `if "I have completed" in response.text: break` | Natural language is not a reliable loop control signal. Claude may say "I have completed" mid-reasoning without being done. |
| `for i in range(10): ...` as sole control | The loop may exit before the task is done (e.g., after 10 turns on a 12-turn task) or may hard-stop valid work. |
| `if response.content[0].type == "text": break` | A text block appearing in the response does not mean the task is complete — Claude often includes reasoning text alongside tool_use blocks in the same response. |
| Forgetting to append assistant message before tool results | The conversation history becomes malformed — Claude loses its own prior reasoning, leading to repeated tool calls or contradictions. |
| Not matching `tool_use_id` in `tool_result` | Claude cannot correlate results to requests. The API requires the `tool_use_id` in each `tool_result` to match the `id` of the corresponding `tool_use` block. |

---

### Cross-Domain Connection

- **Domain 2 (Task 2.2)**: Error responses from MCP tools are returned as `tool_result` content — understanding the agentic loop is prerequisite to understanding how errors propagate back to Claude.
- **Domain 4 (Prompt Engineering)**: The system prompt and tool descriptions directly influence which tools Claude calls in the loop. Poor descriptions lead to loop inefficiency or incorrect tool selection.
- **Domain 5 (Context Management)**: Long-running loops accumulate conversation history. Context window management strategies (summarization, truncation) apply when loops run many iterations.

---

## Task Statement 1.2

# Task Statement 1.2: Orchestrate Multi-Agent Systems with Coordinator-Subagent Patterns

---

### What You Need to Know

- **Hub-and-spoke architecture**: A coordinator agent sits at the center and manages all inter-subagent communication, error handling, and information routing. Subagents do not talk directly to each other — all messages route through the coordinator.
- **Subagents operate with isolated context**: A subagent does not automatically inherit or see the coordinator's conversation history. Each subagent invocation starts with only the context explicitly provided in its prompt.
- **Coordinator responsibilities**: Task decomposition (breaking a complex query into manageable pieces), delegation (selecting and invoking the right subagents), result aggregation (combining subagent outputs into a coherent whole), and adaptive routing (deciding which subagents are needed for a given query vs always running the full pipeline).
- **Risk of overly narrow task decomposition**: If the coordinator assigns each subagent too small a slice of the topic, gaps appear in coverage. For example, assigning "only search for papers published after 2020" to a search subagent may miss foundational work that is essential for context.
- The coordinator is itself an agentic loop — it calls subagents via tool calls (Task/Agent tool), receives results, and reasons about next steps.
- **Subagents vs agent teams**: The exam primarily tests the coordinator-subagent pattern (multiple agents within a single orchestrated session). "Agent teams" is a distinct concept in official docs — multiple independent sessions communicating via shared tasks. Know the distinction but expect exam questions to focus on subagent patterns.
- **Dynamic routing rule**: The coordinator should analyze the query's requirements — recency needs, attached documents, desired output format — **before** invoking any subagents. Routing decisions made after subagent invocations have already started waste resources and cannot be undone.

---

### What You Need to Be Able to Do

- Design coordinator agents that **dynamically select** which subagents to invoke based on query analysis, rather than always routing through the entire pipeline.
- **Partition research scope** across subagents to minimize duplication (e.g., assigning distinct subtopics, date ranges, or source types to each subagent).
- Implement **iterative refinement loops** where the coordinator evaluates synthesis output for gaps, re-delegates to search/analysis subagents with targeted follow-up queries, and re-invokes synthesis until coverage is sufficient.
- Route **all subagent communication through the coordinator** — never design direct subagent-to-subagent communication channels.

---

### Deep Dive

#### Hub-and-Spoke as the Canonical Architecture

In the multi-agent research system scenario, a coordinator agent receives a research topic and manages a team: a web search subagent, a document analysis subagent, a synthesis subagent, and a report generation subagent. The coordinator decides which agents to call, in what order, and with what inputs. No subagent ever directly invokes another.

This architecture provides three key benefits: **observability** (all data flows through one place, so you can log everything), **consistent error handling** (the coordinator applies uniform retry and escalation logic regardless of which subagent failed), and **controlled information flow** (sensitive data does not leak between subagents that should not share it).

```
User Query
    |
    v
[Coordinator Agent] <----loops back until complete---+
    |                                                 |
    |-- [Web Search Subagent] ---- results ---------> |
    |-- [Document Analysis Subagent] --- results ---> |
    |-- [Synthesis Subagent] --------- draft -------> |
    |-- [Report Generation Subagent] -- report -----> |
```

#### Dynamic vs Fixed Pipeline Routing

A naive implementation always invokes all four subagents for every query. This is wasteful and slow. A well-designed coordinator analyzes the query first and routes selectively. The coordinator's system prompt should specify:

- **When to invoke each subagent type**: for example, invoke `web_search_agent` for queries requiring current data; invoke `document_analysis_agent` only when documents are attached; invoke `report_generation_agent` only after synthesis is complete.
- **When NOT to invoke**: explicitly instruct the coordinator not to call agents whose outputs are not needed for the current query.
- **Routing based on query analysis**: the coordinator inspects query characteristics (recency requirements, presence of attachments, output format needed) to determine the minimal required subagent set.

The coordinator's tool set includes a Task tool (or Agent tool) for each subagent type. The coordinator calls only the subset that the query requires.

#### Iterative Refinement Loop

The most sophisticated coordinator pattern is the iterative refinement loop. After the synthesis subagent produces a draft, the coordinator evaluates it for coverage gaps and re-delegates targeted queries if gaps exist. This continues until the synthesis meets quality criteria:

1. **Initial collection**: invoke the web search and document analysis subagents with the original query to gather a first round of evidence.
2. **Initial synthesis**: pass all collected findings to the synthesis subagent to produce a first-draft synthesis.
3. **Gap evaluation loop**: the coordinator examines the synthesis against the original query. If coverage gaps exist, it issues targeted follow-up searches for each gap and re-invokes the synthesis subagent with both the prior synthesis and new data. This loop repeats up to a maximum number of refinement rounds, stopping early when no gaps remain.
4. **Report generation**: once synthesis quality is sufficient, invoke the report generation subagent with the final synthesis to produce the deliverable.

#### Scoping Subagents to Minimize Duplication

When multiple search subagents are invoked, topic overlap wastes API calls and produces redundant findings that the coordinator must deduplicate. The coordinator should partition scope:

| Approach | Assignment | Result |
|---|---|---|
| Bad: same query to all agents | Both agents get "climate change impacts" | Redundant findings, wasted API calls, deduplication burden |
| Good: distinct scope per agent | Agent 1 gets "economic impacts / academic papers"; Agent 2 gets "policy responses / news / last 2 years" | Complementary, non-overlapping coverage |

---

### Exam Tip

Questions on this task statement frequently present a multi-agent architecture and ask what is wrong with it or which design change would most improve reliability. Watch for:
- An answer option that has subagents communicate directly with each other — this is almost always wrong.
- Scenarios where the coordinator always runs all subagents regardless of the query — look for an option that adds dynamic routing.
- Questions about information gaps in research output — the correct fix usually involves iterative refinement with targeted re-delegation, not just adding more subagents.

---

### Anti-Pattern Alert

| Anti-Pattern | Why It Fails |
|---|---|
| Direct subagent-to-subagent communication | Breaks observability and consistent error handling. Hard to debug when data flows out of the coordinator's visibility. |
| Static pipeline that always runs all subagents | Inefficient; slow for simple queries; does not scale. Coordinator should dynamically select agents based on query needs. |
| Passing the coordinator's full conversation history to a subagent | Subagents receive the coordinator's entire context, including prior subagent outputs and system prompt. This may leak sensitive data, bloat the subagent's context, or cause confusion. Explicitly provide only relevant information. |
| One subagent assigned the entire research scope | No parallelism benefit; single point of failure; no specialization advantage. |
| Coordinator that never evaluates synthesis quality | Accepts first-pass synthesis regardless of coverage gaps. Always build in at least one evaluation checkpoint before final output. |

---

### Cross-Domain Connection

- **Task 1.3**: The mechanism for the coordinator to invoke subagents is the Task/Agent tool — context must be explicitly passed, not assumed.
- **Task 1.4**: When a subagent fails (e.g., web search times out), the coordinator is the right place to implement escalation and handoff logic.
- **Domain 2 (Task 2.3)**: Scoped tool access — each subagent should only have the tools relevant to its role. The coordinator does not give the synthesis subagent access to web search tools.
- **Domain 5 (Task 5.x)**: Aggregating outputs from multiple subagents can consume substantial context. Summarization strategies apply when passing combined results back to the coordinator.

---

## Task Statement 1.3

# Task Statement 1.3: Configure Subagent Invocation, Context Passing, and Spawning

---

### What You Need to Know

- The **Task tool** (also called the Agent tool — renamed in v2.1.63; both names remain valid as aliases; the exam guide uses "Task tool" as the primary name) is the mechanism for spawning subagents. For a coordinator to invoke subagents, `"Task"` (or `"Agent"`) must be included in `allowedTools` for the coordinator's configuration.
- **Subagent context is not inherited automatically**. Each subagent invocation starts fresh. Any findings, instructions, or constraints that the subagent needs must be explicitly included in its prompt. There is no shared memory between invocations.
- **AgentDefinition / subagent frontmatter** defines each subagent type. Key configuration fields:

  | Field | Values / Notes |
  |---|---|
  | `name` | Required. Unique identifier for the subagent |
  | `description` | Required. What the coordinator reads to decide when to invoke it |
  | `tools` | Allowlist — only these tools are available to the subagent |
  | `disallowedTools` | Denylist — these tools are blocked even if globally available |
  | `model` | `sonnet`, `opus`, `haiku`, `inherit`, or a full model ID |
  | `permissionMode` | `default`, `acceptEdits`, `dontAsk`, `bypassPermissions`, `plan` |
  | `maxTurns` | Integer cap on agentic turns for this subagent |
  | `skills` | Preload named skill content into the subagent's context |
  | `mcpServers` | Scope specific MCP servers to this subagent only |
  | `memory` | `user`, `project`, or `local` for persistent cross-session memory |
  | `background` | `true` = runs concurrently (non-blocking); `false` = foreground (blocking) |
  | `isolation` | `worktree` = isolated git worktree for the subagent |
  | `effort` | `low`, `medium`, `high`, `max` |

- **Exam-critical AgentDefinition fields** (the four fields most tested):

  | Field | Purpose | Exam relevance |
  |---|---|---|
  | `description` | Coordinator reads this to decide when to invoke the subagent | Must clearly state the conditions under which this agent should be used |
  | `system_prompt` | Instructions governing subagent behavior | Specify goals and quality criteria — not procedural steps |
  | `allowed_tools` | Restricts the subagent to role-relevant tools only | 4–5 tools per agent; prevents cross-role misuse |
  | `model` | Which model the subagent runs on | Use cheaper models (e.g., Haiku) for simple or read-only tasks |

- **Built-in subagents in Claude Code**: **Explore** (uses Haiku model, read-only tools — optimized for fast codebase search); **Plan** (inherits the main session's model, read-only tools — used during plan mode to map tasks before execution); **general-purpose** (inherits model, all tools — for complex multi-step tasks). These are pre-configured by Anthropic and available without additional `AgentDefinition` authoring.
- **Subagents CANNOT spawn other subagents.** Nesting is not supported. Only the main thread (invoked with `--agent`) can spawn subagents. If a subagent tries to use the Agent tool, the call will fail.
- **Foreground vs background execution**:
  - Foreground (default): blocking call; permission prompts are passed through to the user; coordinator waits for result.
  - Background (`background: true`): non-blocking; runs concurrently; unapproved permission requests are auto-denied (no user prompt).
- **`@-mention` vs natural language invocation**: Using `@subagent-name` in a prompt guarantees that specific subagent runs. Natural language routing relies on Claude interpreting the request and selecting a subagent — the coordinator may choose a different one. Use `@-mention` when the exact subagent must be deterministic.
- **`fork_session`** creates an independent branch from a shared analysis baseline, allowing divergent approaches to be explored in parallel from the same starting point without contaminating each other's state.
- Spawning **parallel subagents** is accomplished by emitting multiple Agent tool calls in a single coordinator response — not by making separate coordinator turns.

---

### What You Need to Be Able to Do

- Include **complete findings** from prior agents directly in the subagent's prompt — for example, passing the full web search results and document analysis outputs to the synthesis subagent so it has everything it needs.
- Use **structured data formats** (JSON, delimited sections) to separate content from metadata when passing context between agents, preserving attribution (source URLs, document names, page numbers).
- Spawn **parallel subagents** by emitting multiple Task tool calls within a single coordinator response.
- Design coordinator prompts that specify **goals and quality criteria** rather than step-by-step procedural instructions, enabling subagent adaptability.

---

### Deep Dive

#### The Agent Tool: Coordinator Configuration

For a coordinator to spawn subagents, the Agent tool (formerly Task tool) must be in its `allowedTools`. The coordinator's `AgentDefinition` shows the five key fields:

```json
{
  "name": "research_coordinator",
  "description": "Coordinates specialized research subagents to produce comprehensive reports.",
  "system_prompt": "You are a research coordinator. Analyze queries, delegate to subagents...",
  "allowed_tools": ["Agent"],
  "model": "claude-opus-4-5"
}
```

Subagent definitions are configured separately. For example, a `web_search_agent` would have `"tools": ["web_search", "fetch_url"]`, a focused description telling the coordinator exactly when to invoke it (i.e., when current data or recent developments are required), and a lighter model such as `claude-haiku-4-5`. The `description` field is particularly important: it is what the coordinator reads to decide whether to invoke that agent for a given query. The `disallowedTools` field can be used to block specific tools without maintaining a full allowlist.

#### Explicit Context Passing

The most common configuration mistake is assuming the subagent will "know" what was found earlier. It will not. The anti-pattern is a vague prompt with no context, such as "Synthesize the research findings into a comprehensive report" — the subagent receives this with no findings attached and cannot proceed meaningfully.

The correct approach embeds all prior findings directly in the subagent's prompt: the full web search results (with titles, URLs, dates, and key excerpts), the full document analysis findings (with document names, page numbers, and relevant passages), and explicit quality criteria (minimum source count, dimensions to address, how to handle contradictions). The subagent should receive everything it needs to complete its task without calling back to the coordinator for clarification.

#### Structured Data for Attribution Preservation

When passing findings between agents, use structured formats that keep content separate from metadata:

```json
{ "source_url": "https://research.org/paper42",
  "key_excerpt": "Intermittency remains the primary barrier...",
  "relevance_score": 0.87 }
```

Each finding carries its origin alongside its content, so the synthesis agent can cite specific sources without conflating content with its provenance.

#### Parallel Subagent Spawning

To spawn multiple subagents in parallel, the coordinator must emit all Agent tool calls in a **single response**, not across multiple turns. A single coordinator response containing three Agent tool_use blocks looks like:

```
[text block: "I'll search multiple dimensions in parallel."]
[tool_use: Task → web_search_agent, "Search for economic impacts..."]
[tool_use: Task → web_search_agent, "Search for policy frameworks..."]
[tool_use: Task → document_analysis_agent, "Analyze attached IPCC report..."]
```

All three subagents execute concurrently. Emitting them across three separate turns forces sequential execution, tripling latency.

#### Goals and Criteria vs Procedures

Coordinator prompts should specify what success looks like, not enumerate every step:

| Approach | Example | Problem |
|---|---|---|
| Procedural | "Step 1: Read web results. Step 2: Read doc analysis. Step 3: Identify themes. Step 4: Write summary. Step 5: List citations." | Constrains adaptability; if a step is irrelevant, the subagent may hallucinate to satisfy the instruction |
| Goals-and-criteria | "Synthesize findings into a comprehensive analysis. Success criteria: all major dimensions covered, contradictions resolved with reasoning, every claim attributed, knowledge gaps flagged." | Gives the subagent flexibility to adapt its approach to the available evidence |

---

### Exam Tip

This task statement tests both configuration knowledge (what must be in `allowedTools`?) and design judgment (how should context be passed?). Common question patterns:
- A scenario where a synthesis subagent produces incomplete output — the correct fix is usually to include more complete findings in the subagent's prompt, not to give the subagent more tools or a different model.
- A question about maximizing parallelism — the answer involves emitting multiple Task tool calls in a single response, not across multiple turns.
- A question about attribution loss — the fix is structured data formats that keep metadata attached to content.

---

### Anti-Pattern Alert

| Anti-Pattern | Why It Fails |
|---|---|
| Invoking Task tool without `"Task"` in `allowedTools` | The coordinator cannot call subagents. The tool simply won't be available. |
| Assuming subagent inherits parent context | Subagents start with only what is in their prompt. Prior findings are invisible unless explicitly provided. |
| Passing raw concatenated text without structure | Attribution is lost. The synthesis agent cannot tell which finding came from which source. |
| Emitting one Task tool call per turn for parallel work | Forces sequential execution. A 3-agent parallel task that takes 5s each becomes 15s instead of 5s. |
| Procedural step-by-step coordinator prompts | Prevents adaptive behavior. If step 3 is irrelevant for a given query, the subagent may hallucinate data to satisfy the instruction. |

---

### Cross-Domain Connection

- **Task 1.2**: This task statement is the implementation layer for the architecture described in 1.2 — how the coordinator actually invokes the subagents it manages.
- **Task 1.7**: `fork_session` is the session management mechanism for exploring divergent approaches — covered in 1.7 with practical usage patterns.
- **Domain 2 (Task 2.3)**: `AgentDefinition` tool restrictions tie directly to the principle of scoped tool access — subagents only get the tools listed in their `allowed_tools`.
- **Domain 4 (Prompt Engineering)**: Coordinator prompt design (goals vs procedures) is an application of the prompt engineering principles in Domain 4.

---

## Task Statement 1.4

# Task Statement 1.4: Implement Multi-Step Workflows with Enforcement and Handoff Patterns

---

### What You Need to Know

- **Programmatic enforcement** (hooks, prerequisite gates in code) vs **prompt-based guidance** (instructing Claude in the system prompt to follow a sequence): These are fundamentally different reliability guarantees.
- When **deterministic compliance is required** — such as verifying a customer's identity before executing a financial transaction — prompt instructions alone have a **non-zero failure rate**. Claude may follow the instruction most of the time, but "most of the time" is not acceptable for financial operations, security checks, or regulatory compliance.
- **Programmatic prerequisites** are gates in the orchestration code that check whether a precondition is satisfied before allowing the next tool call to proceed. They cannot be bypassed by model reasoning.
- **Structured handoff protocols** for escalation must include enough information for the receiving human agent to act without access to the conversation transcript: customer details, root cause analysis, steps already taken, and recommended next actions.

---

### What You Need to Be Able to Do

- Implement **programmatic prerequisites** that block downstream tool calls until prerequisite steps have completed (e.g., blocking `process_refund` until `get_customer` has returned a verified customer ID).
- **Decompose multi-concern requests** into distinct items, investigate each in parallel using shared context, then synthesize a unified resolution.
- Compile **structured handoff summaries** when escalating to human agents, including: customer ID, root cause, amounts involved, steps taken, and recommended action.

---

### Deep Dive

#### Programmatic Enforcement vs Prompt Guidance

This distinction is one of the most exam-critical concepts in Domain 1. Consider a customer support agent that should verify identity before issuing a refund.

**Prompt-only approach (unreliable):**
```python
system_prompt = """
Always call get_customer and verify the customer ID before calling process_refund.
Do not process any refund without first confirming the customer's identity.
"""
```

This works most of the time. But Claude is a probabilistic system. Under unusual phrasing, extreme context length, or adversarial prompting, it may occasionally call `process_refund` without the prerequisite. In a financial system, that is unacceptable.

**Programmatic enforcement approach (deterministic):**

A `WorkflowEnforcer` class maintains session state (e.g., `verified_customer_id = None`) and registers a `pre_tool_call_hook` that intercepts every outgoing tool call before it executes. When Claude attempts to call `process_refund`, the hook checks whether `verified_customer_id` is set. If it is not, the hook returns a structured error (`PREREQUISITE_VIOLATION`) and the tool call is blocked — the refund never executes. The hook also validates that the customer ID in the refund request matches the verified ID. A `post_tool_call_hook` records the verified ID after a successful `get_customer` call.

The critical difference: the programmatic enforcer **cannot be bypassed**. No matter what Claude reasons, no matter how the user phrases the request, `process_refund` will not execute until `get_customer` has returned a verified ID.

#### Decomposing Multi-Concern Requests

When a customer submits a request with multiple issues ("my order arrived damaged AND I was charged twice"), the agent should:

1. **Parse concerns**: identify all distinct issues in the message (e.g., damaged item, duplicate charge).
2. **Investigate in parallel**: emit tool calls for each concern simultaneously within a single response, using shared customer context.
3. **Synthesize**: combine all investigation results into one coherent resolution response, addressing each concern.

#### Structured Handoff Summaries

When escalating to a human agent, the summary must be self-contained. Human agents typically do not have access to the conversation transcript and cannot re-run tool calls.

```json
{
  "handoff_reason": "Refund of $649 exceeds automated authorization limit ($500)",
  "customer": { "id": "CUST-78234", "tier": "premium" },
  "issue_root_cause": "Order #ORD-445521 delivered damaged; photos confirm damage.",
  "steps_completed": ["Identity verified", "Damage confirmed", "Refund eligibility confirmed"],
  "recommended_action": "Approve full refund of $649.00 — premium customer, confirmed damage."
}
```

This handoff includes everything the human agent needs: who the customer is, what the problem is, what was already done, and what action is recommended — without requiring them to scroll through a chat transcript.

---

### Exam Tip

This task statement consistently tests the distinction between prompt-based and programmatic enforcement. When a scenario describes a compliance or security requirement, the correct answer will almost always involve a programmatic gate or hook — not a better-worded system prompt instruction. Look for answer options that use words like "hook," "gate," "prerequisite check," or "intercept" — these signal the correct enforcement approach.

Questions about handoff summaries test whether you know what information a human agent needs when they lack transcript access. The key fields are: customer identity, root cause, steps completed, and recommended action.

---

### Anti-Pattern Alert

| Anti-Pattern | Why It Fails |
|---|---|
| "Instruct Claude to always verify identity first" | Prompt instructions have non-zero failure rate. Not acceptable for financial/security-critical operations. |
| Escalating with just the conversation transcript | Human agents cannot efficiently parse a long transcript to find the relevant details. Structured summaries are faster and less error-prone. |
| Handling multi-concern requests sequentially in a single pass | Each concern gets less attention; the model may conflate issues; longer resolution time than parallel investigation. |
| Blocking all subsequent tool calls when one prerequisite fails | Too restrictive — other investigation tools (e.g., `lookup_order`) may be legitimate while `process_refund` is blocked. Gate only the specific downstream tools that require the prerequisite. |

---

### Cross-Domain Connection

- **Task 1.5**: Programmatic prerequisites are one application of hooks. Task 1.5 covers the Agent SDK hook system that makes these patterns implementable.
- **Task 1.2**: The coordinator in a multi-agent system is the natural location for enforcement logic when multiple subagents are involved.
- **Domain 5 (Reliability)**: Human-in-the-loop patterns and escalation thresholds are reliability mechanisms that complement the enforcement patterns in this task statement.

---

## Task Statement 1.5

# Task Statement 1.5: Apply Agent SDK Hooks for Tool Call Interception and Data Normalization

---

### What You Need to Know

- **PostToolUse hooks** intercept tool results **after** a tool executes and **before** the model processes the result. This enables transformation, normalization, enrichment, or filtering of tool outputs.
- **PreToolUse hooks** (interception hooks) intercept outgoing tool calls **before** they execute, enabling blocking, policy enforcement, redirection, or logging of tool invocations.
- **Hooks provide deterministic guarantees**; prompt instructions provide probabilistic compliance. This is the fundamental design choice: if a business rule must never be violated, use a hook. If guidance is acceptable, use a prompt instruction.
- Hooks operate at the orchestration layer, outside Claude's reasoning. Claude cannot reason around a hook — it either executes and gets the (potentially modified) result, or it gets a blocked response.
- Common use cases: normalizing heterogeneous data formats from multiple MCP tools, enforcing authorization thresholds, logging all tool calls for audit purposes, and injecting metadata.

---

### What You Need to Be Able to Do

- Implement **PostToolUse hooks** to normalize heterogeneous data formats (Unix timestamps, ISO 8601, numeric status codes) from different MCP tools before the agent processes them.
- Implement **tool call interception hooks** (PreToolUse) that block policy-violating actions (e.g., refunds exceeding $500) and redirect to alternative workflows (e.g., human escalation).
- Correctly **choose hooks over prompt-based enforcement** when business rules require guaranteed compliance.

---

### Deep Dive

#### The Hook System Architecture

Hooks sit between the tool execution layer and Claude's message processing. In the Agent SDK, you register hooks when constructing the agent session:

```python
session = AgentSession(
    model="claude-opus-4-5", tools=[...],
    hooks={"pre_tool_use": authorization_and_policy_hook,
           "post_tool_use": data_normalization_hook})
```

**Hook mechanics (exam-tested details):**

| Aspect | PreToolUse | PostToolUse |
|---|---|---|
| When it fires | Before the tool executes | After the tool succeeds |
| Input received (via stdin) | JSON with `tool_name`, `tool_input`, `tool_use_id` | Same fields plus the tool's output |
| How to block | Return `permissionDecision: "deny"` or exit with code 2 | N/A — tool already ran |
| How to allow | Return `permissionDecision: "allow"` or exit 0 | Return unchanged or modified result |
| How to ask user | Return `permissionDecision: "ask"` | N/A |
| Modifying output | N/A | Return `updatedMCPToolOutput` to transform result |
| Injecting context | N/A | Return `additionalContext` for Claude to see |

Exit code 2 signals a blocking error — the tool call is stopped and Claude receives the error message. Exit code 0 with no output allows the call to proceed unchanged.

#### PostToolUse Hook: Data Normalization

Different MCP tools in a heterogeneous system return data in different formats. Rather than writing normalization logic into every prompt or asking Claude to handle it, a PostToolUse hook normalizes once, centrally.

For example, a `data_normalization_hook` inspects the tool name on every result. For `get_customer` results, it converts Unix timestamp integers to ISO 8601 strings (`created_at: 1700000000` becomes `"2023-11-14T22:13:20Z"`) and maps numeric account status codes to human-readable strings (`1` becomes `"active"`). For `lookup_order` results, it normalizes timezone-offset timestamps to UTC and maps numeric fulfillment status codes to descriptive strings. Claude receives the normalized result — never the raw tool output. No prompt instruction is needed, and there is no risk of Claude misinterpreting `1` as an account status.

#### PreToolUse Hook: Authorization Enforcement

PreToolUse hooks enforce business rules deterministically. The hook function inspects the tool name and input before the call executes. If the rules are violated, it returns a structured blocking response instead of allowing the call through; returning `None` allows the call to proceed.

For example, an `authorization_and_policy_hook` intercepts calls to `process_refund`. If the refund amount exceeds $500, the hook returns a structured error with `reason: "EXCEEDS_AUTOMATED_LIMIT"` and `required_action: "escalate_to_human"` — the refund tool never executes. If the reason code is not in the permitted set, the hook returns `reason: "INVALID_REASON_CODE"` with the valid options. Claude receives this blocking response as the `tool_result` and must take an alternative path (e.g., calling `escalate_to_human`). Claude cannot retry the blocked call or reason around the enforcement.

#### Choosing Between Hooks and Prompt Instructions

This decision matrix is tested directly on the exam:

| Need | Use Hook | Use Prompt |
|---|---|---|
| "Never process refunds over $500" | PreToolUse hook (deterministic) | Non-zero failure rate |
| "Always normalize timestamps to ISO 8601" | PostToolUse hook (consistent) | Claude may apply inconsistently |
| "Prefer academic sources when available" | Overkill for a preference | Prompt instruction (preference OK) |
| "Log all tool calls for audit" | PreToolUse hook (must happen every call) | Cannot rely on model discretion |
| "Be empathetic in responses" | Overkill for tone guidance | Prompt instruction (tone guidance) |

Hooks are configured in `settings.json` at user, project, or local scope — not inside Claude's conversation. Claude receives only the hook's output (modified data or a blocking error), never the hook logic itself.

---

### Exam Tip

This task statement is heavily tested with scenario questions like: "A business rule requires that no refund over $500 be processed automatically. Which approach provides the strongest guarantee?" The answer will be a PreToolUse hook, not a system prompt instruction. Watch for distractor options that describe "adding a clear instruction to the system prompt" — these are the wrong answer when deterministic compliance is required.

Also watch for questions about data normalization: when tool outputs from different sources need to be in a consistent format before Claude processes them, a PostToolUse hook is the correct mechanism.

---

### Anti-Pattern Alert

| Anti-Pattern | Why It Fails |
|---|---|
| Using system prompt to enforce $500 refund limit | Non-zero failure rate. Will work 99% of the time but will eventually fail, especially under adversarial prompting or unusual phrasing. |
| Running normalization logic inside tool implementations | Couples normalization to each individual tool. Adding a new tool requires writing new normalization code. A PostToolUse hook centralizes this. |
| PostToolUse hook that modifies results silently without logging | Silent data transformation is a debugging and audit nightmare. Hooks that transform data should log what was changed and why. |
| Writing PostToolUse hooks that call Claude | Hooks execute synchronously in the hot path. Calling Claude from a hook creates recursive API calls and latency spikes. Keep hooks fast and deterministic. |

---

### Cross-Domain Connection

- **Task 1.4**: The programmatic prerequisites discussed in 1.4 are implemented using the PreToolUse hook pattern described here.
- **Task 1.2**: In multi-agent systems, hooks on the coordinator can enforce policies before subagent invocations are allowed to proceed.
- **Domain 2 (Task 2.2)**: MCP tool error responses are what PostToolUse hooks receive when tools fail — normalization hooks should handle error cases as well as success cases.

---

## Task Statement 1.6

# Task Statement 1.6: Design Task Decomposition Strategies for Complex Workflows

---

### What You Need to Know

- **Fixed sequential pipelines (prompt chaining)** are appropriate when the workflow structure is known in advance, the steps are predictable, and the outputs of one step feed consistently into the next.
- **Dynamic adaptive decomposition** is appropriate when the workflow structure depends on what is discovered at each step — when you cannot know in advance what sub-tasks will be needed.
- **Prompt chaining** breaks a complex task into a defined sequence of steps, where each step's output becomes the next step's input. A large code review broken into per-file analysis passes plus a cross-file integration pass is a prompt chaining pattern.
- **Adaptive investigation plans** generate sub-tasks dynamically based on intermediate findings. A legacy codebase test-coverage task where the structure is unknown in advance requires adaptive decomposition.
- **Attention dilution** is the risk of asking a model to review too many files in a single context — quality degrades as the model must distribute attention across more material than it can handle deeply.

---

### What You Need to Be Able to Do

- Select the appropriate decomposition pattern: **prompt chaining** for predictable multi-aspect reviews, **dynamic decomposition** for open-ended investigation tasks.
- Split large code reviews into **per-file local analysis passes** plus a separate **cross-file integration pass** to avoid attention dilution.
- Decompose open-ended tasks (e.g., "add comprehensive tests to a legacy codebase") by first **mapping structure**, then **identifying high-impact areas**, then **creating a prioritized plan** that adapts as dependencies are discovered.

---

### Deep Dive

#### Prompt Chaining: When the Pipeline Is Known

For a code review workflow, the steps are predictable: review each file, then assess cross-file concerns, then generate a summary. This is a prompt chaining pattern with three phases:

1. **Per-file local analysis** (can be parallelized): each file is reviewed independently for logic errors, performance, security vulnerabilities, style, and missing error handling. Each call returns structured findings with severity levels and line numbers. Because files are independent, these passes can be distributed to parallel subagents.
2. **Cross-file integration pass** (sequential — requires all per-file results): a single call receives all per-file findings and identifies concerns that only appear across files, such as interface mismatches between modules, inconsistent error handling patterns, duplicate logic eligible for extraction, and dependency cycles.
3. **Summary synthesis**: a final call combines per-file and cross-file findings into a prioritized pull request comment, ordered by severity, with a top-5 action items list.

Each phase has a defined structure. The pipeline does not need to adapt based on what it finds.

#### Dynamic Adaptive Decomposition: When the Structure Is Unknown

For "add comprehensive tests to a legacy codebase," the correct approach is not to immediately start writing tests — it is to first understand the codebase, then plan, then adapt as constraints emerge. Three phases:

1. **Map structure**: explore the repository with read and search tools to list all modules and their responsibilities, identify which files already have tests and which do not, note existing testing frameworks, and flag critical-path or high-risk files. No tests are written at this stage.
2. **Prioritize targets**: given the structure map, identify highest-impact targets: files with zero coverage on the critical path, functions with high cyclomatic complexity and no tests, code that calls external systems (databases, APIs, file I/O), and utility functions used widely across the codebase. Produce a prioritized work list with justification and flag known ordering constraints.
3. **Adaptive execution**: work through the prioritized list, implementing tests one item at a time and accumulating prior results as context. When a new dependency or constraint is discovered mid-implementation (e.g., testing `payment_processor.py` reveals that `config_loader.py` must be tested first to enable mocking), the remaining work list is updated before proceeding.

The key difference from prompt chaining: the plan can change. Unknown dependencies discovered during execution feed back into the work queue.

#### Why Per-File Passes Prevent Attention Dilution

Sending 50 files to Claude in a single review request is a well-documented failure mode. Attention is distributed across all 50 files, and subtle issues in any individual file receive less scrutiny. Per-file passes give each file full attention. The cross-file pass then focuses exclusively on inter-file concerns, which it can address with full attention because it is not simultaneously doing per-file analysis.

---

### Exam Tip

Questions in this task statement often present a scenario (code review, research, test generation) and ask which decomposition strategy is appropriate. The signal in the question is whether the workflow structure is known in advance:
- "Review each file for security issues, then produce a summary" — known structure, prompt chaining.
- "Investigate this unfamiliar legacy codebase and add coverage where it matters most" — unknown structure, adaptive decomposition.

Watch for answer options that suggest reviewing all files in one pass — these are distractor options exploiting the attention dilution anti-pattern.

---

### Anti-Pattern Alert

| Anti-Pattern | Why It Fails |
|---|---|
| Sending all files in a single review prompt | Attention dilution — subtle issues in individual files are missed as the model distributes attention across too much content. |
| Using adaptive decomposition for a well-defined pipeline | Unnecessary complexity. If the steps are known, prompt chaining is simpler, more predictable, and easier to debug. |
| Skipping the structure-mapping phase for open-ended tasks | The agent starts generating tests/code without understanding what already exists, leading to duplicate work, wrong priorities, and missing dependencies. |
| Creating too many sub-tasks in adaptive decomposition | Each sub-task invocation has API latency and token overhead. Overly granular decomposition degrades performance without improving quality. |
| Fixed pipeline with no mechanism to handle step failures | If step 2 fails in a 5-step chain, the entire pipeline fails. Build in error handling and partial result handling at each stage. |

---

### Cross-Domain Connection

- **Task 1.2**: Multi-agent systems use decomposition strategies to allocate work across subagents. Task 1.6 covers the decomposition logic; Task 1.2 covers the routing and delegation.
- **Task 1.3**: Parallel subagent spawning from Task 1.3 is a natural complement to prompt chaining — per-file analysis passes can be distributed across parallel subagents.
- **Domain 3 (Claude Code)**: Plan mode in Claude Code is an explicit tool for adaptive decomposition — the agent maps the task before executing. This connection is directly testable.
- **Domain 5 (Context Management)**: Long pipelines accumulate intermediate results. Context window strategies (summarization, selective inclusion) apply between pipeline stages.

---

## Task Statement 1.7

# Task Statement 1.7: Manage Session State, Resumption, and Forking

---

### What You Need to Know

- **Named session resumption** allows continuing a specific prior conversation using `--resume <session-name>` (accepts a session name or session ID). This restores the prior conversation history, including all prior tool results and reasoning, enabling continuation from where the session left off.
- **`--continue`** resumes the most recent conversation without needing a session name — useful for multi-step CI workflows or picking up immediately where a previous run ended. Use `--resume` when targeting a specific named session; use `--continue` when "the last session" is always the right one.
- **`fork_session`** creates an **independent branch** from a shared baseline conversation state. Both branches start with identical history and can diverge independently — changes in one branch do not affect the other.
- When resuming a session after code or file modifications, the agent must be **explicitly informed** about what changed. It does not automatically detect file system changes. Without this notification, it will reason based on its cached understanding, which may now be stale.
- **Starting fresh with a structured summary** is more reliable than resuming when prior tool results are stale (e.g., data has changed, files have been modified, external systems have updated). A fresh session with injected context avoids the risk of the agent reasoning from outdated information it believes is current.

---

### What You Need to Be Able to Do

- Use `--resume <session-name>` (or `--resume <session-id>`) to continue named investigation sessions across work sessions. Use `--continue` to resume the most recent conversation when no specific session name is needed.
- Use `fork_session` to create parallel exploration branches from a shared codebase analysis (e.g., comparing two testing strategies or two refactoring approaches).
- Choose between **session resumption** (appropriate when prior context is mostly valid and up-to-date) and **fresh start with injected summary** (appropriate when prior tool results are stale or significantly outdated).
- When resuming a session, inform the agent about specific file changes for targeted re-analysis rather than requiring full re-exploration.

---

### Deep Dive

#### Named Session Resumption

In Claude Code, sessions can be named for later resumption. The `--resume` flag continues from the exact state where the prior session ended:

```bash
claude --session-name "payment-investigation" "Map all database query patterns."
claude --resume "payment-investigation" "Which queries lack index coverage?"
```

For CI pipelines or scripts where the target is always the most recent session, `--continue` is the simpler alternative — no session name needed:

```bash
claude -p "Run the next phase of the migration analysis." --continue
```

`--resume <id>` accepts either a human-readable session name or the internal session ID. `--continue` is equivalent to `--resume` on the most-recently-modified session.

**Critical caveat**: if files have changed since the last session, the agent does not know this. When resuming, explicitly inform it which files were modified and ask it to re-analyze them before answering questions about them. Asking questions about changed files without this notification will produce answers based on stale cached understanding.

#### fork_session: Parallel Exploration

`fork_session` is ideal when you have completed an analysis phase and want to explore multiple implementation approaches independently from that shared baseline:

```python
baseline = AgentSession(session_name="codebase-baseline")
baseline.run("Analyze the authentication module: map flows, coverage, dependencies.")
fork_a = baseline.fork_session("approach-jwt", prompt="Design a JWT migration plan...")
fork_b = baseline.fork_session("approach-oauth", prompt="Design an OAuth 2.0/PKCE plan...")
```

Both forks start from identical baseline knowledge and diverge independently. Running the baseline analysis once instead of twice eliminates duplicate API cost and time.

#### Resumption vs Fresh Start: The Decision

The choice between resuming a prior session and starting fresh with injected context depends on the freshness of the prior tool results:

**Resume when:**
- The codebase or data has not changed significantly since the last session.
- The investigation is ongoing and prior findings are still valid.
- You are continuing a multi-session task that was interrupted.
- The agent's prior reasoning chain is valuable context for the next question.

**Start fresh with structured summary when:**
- Significant time has passed and external data (APIs, databases) has changed.
- Files analyzed in the prior session have been modified.
- The prior session contained errors or misunderstandings you want to correct.
- The prior session's tool results are stale and would mislead the agent if relied upon.

A fresh session prompt with injected context should include:

- A labeled context summary section (clearly marked as from a prior investigation)
- Key prior findings that are still likely valid (file counts, architecture notes, critical bugs found)
- Explicit dates and modification flags for any findings that may be stale
- The current task statement
- An instruction to re-verify critical findings and re-read any files modified since the prior session rather than relying on the summary

The fresh start explicitly marks what is known vs what needs verification, protecting against stale data.

---

### Exam Tip

This task statement tests practical knowledge of Claude Code session management. Expect questions like:
- "After modifying three files during a refactoring session, you want to continue your Claude Code investigation. What should you do?" — The answer: resume the session AND explicitly tell the agent which files changed.
- "You want to compare two architectural approaches using a shared codebase analysis baseline. Which mechanism is most efficient?" — The answer: `fork_session`.
- "Your investigation session from three weeks ago explored a codebase that has since had major changes. Should you resume or start fresh?" — The answer: start fresh with a structured summary of still-valid findings.

---

### Anti-Pattern Alert

| Anti-Pattern | Why It Fails |
|---|---|
| Resuming after file changes without notification | Agent reasons from stale cached understanding of modified files. May give incorrect answers about code that no longer exists or has changed behavior. |
| Running full baseline analysis twice for two-approach comparison | Wasteful — identical work done twice. `fork_session` solves this by sharing the analysis and branching only at the point of divergence. |
| Always resuming regardless of data freshness | Stale tool results look identical to fresh ones in the conversation history. The agent cannot tell the difference and will reason from outdated data as if it were current. |
| Starting fresh for every session of a long investigation | Loses the accumulated context and reasoning from prior sessions. May require re-running expensive tool calls that were already done. |
| Not naming sessions when starting long investigations | Cannot resume later — unnamed sessions expire or are harder to retrieve. Name sessions at the start of any investigation expected to span multiple work sessions. |

---

### Cross-Domain Connection

- **Task 1.3**: `fork_session` is introduced in Task 1.3 as a subagent spawning mechanism for divergent approaches. Task 1.7 covers the operational pattern for using it.
- **Domain 3 (Claude Code)**: Session management via `--resume` is a Claude Code-specific feature. Domain 3 covers additional Claude Code workflow configuration.
- **Domain 5 (Context Management)**: Long-running sessions accumulate conversation history. When resuming sessions with extensive history, context window limits may require summarization before resumption.

---

## Key Terminology Quick Reference

| Term | Definition | Task Statement |
|---|---|---|
| **Agentic loop** | The repeating cycle of: send request to Claude, inspect `stop_reason`, execute tools if `stop_reason == "tool_use"`, append results, repeat | 1.1 |
| **stop_reason** | API response field that signals why Claude stopped generating. Values: `"end_turn"` (done), `"tool_use"` (wants to call a tool), `"max_tokens"`, `"stop_sequence"` | 1.1 |
| **tool_use block** | Content block in Claude's response indicating a tool call request. Contains `id`, `name`, and `input` fields | 1.1 |
| **tool_result block** | Content block in the user message returning a tool's output. Must reference the `tool_use_id` from the corresponding tool_use block | 1.1 |
| **Hub-and-spoke architecture** | Multi-agent topology where a coordinator manages all inter-subagent communication; subagents never communicate directly | 1.2 |
| **Isolated context** | Each subagent invocation starts with only explicitly provided context — no automatic inheritance of coordinator history | 1.2, 1.3 |
| **Task tool / Agent tool** | The mechanism for spawning subagents (renamed to Agent tool in v2.1.63; exam guide uses "Task tool"). Must be in `allowedTools` for a coordinator to invoke subagents | 1.3 |
| **AgentDefinition** | Configuration object defining a subagent type: description, system prompt, allowed tools, model | 1.3 |
| **fork_session** | Creates an independent branch from a shared session baseline, enabling parallel divergent exploration | 1.3, 1.7 |
| **Parallel spawning** | Emitting multiple Task tool calls in a single coordinator response to execute subagents concurrently | 1.3 |
| **Programmatic enforcement** | Using hooks or prerequisite gates in code to enforce workflow rules — provides deterministic guarantees | 1.4, 1.5 |
| **Prompt-based guidance** | Using system prompt instructions to guide workflow behavior — probabilistic, non-zero failure rate | 1.4, 1.5 |
| **Prerequisite gate** | A programmatic check that blocks a downstream tool call until a required prior step has completed | 1.4 |
| **Structured handoff** | A self-contained summary provided to a receiving human agent during escalation, including customer details, root cause, steps taken, and recommended action | 1.4 |
| **PostToolUse hook** | Intercepts tool results after execution and before model processing — enables normalization, transformation, enrichment | 1.5 |
| **PreToolUse hook** | Intercepts outgoing tool calls before execution — enables authorization checks, policy enforcement, blocking | 1.5 |
| **Prompt chaining** | Fixed sequential pipeline where each step's output feeds the next step's input; appropriate when workflow structure is known in advance | 1.6 |
| **Adaptive decomposition** | Dynamic task breakdown where sub-tasks are generated based on intermediate findings; appropriate for open-ended investigation | 1.6 |
| **Attention dilution** | Quality degradation when too much content is included in a single model request, forcing attention distribution across too much material | 1.6 |
| **Named session** | A Claude Code session given an identifier for later resumption via `--resume <session-name>` or `--resume <session-id>` | 1.7 |
| **Session resumption** | Continuing a prior named session with full conversation history restored via `--resume`; `--continue` resumes the most recent session without specifying a name | 1.7 |
| **Model-driven decision-making** | Claude reasons autonomously about which tool to call next based on context, rather than following a pre-configured sequence | 1.1 |
| **Iterative refinement** | Coordinator pattern where synthesis output is evaluated for gaps and re-delegation occurs until quality criteria are met | 1.2 |

---

## Decision Matrix

### When to Use Hooks vs Prompt Instructions

| Scenario | Mechanism | Reason |
|---|---|---|
| Refunds over $500 must NEVER be processed automatically | PreToolUse hook | Deterministic — cannot be bypassed by model reasoning |
| Timestamps from multiple tools must be consistently ISO 8601 | PostToolUse hook | Centralized, guaranteed normalization on every call |
| Agent should prefer academic sources when available | System prompt instruction | Preference, flexibility acceptable |
| All tool calls must be logged for regulatory audit | PreToolUse + PostToolUse hooks | Must happen for every call, cannot rely on model discretion |
| Identity must be verified before financial operations | PreToolUse prerequisite gate | Non-zero failure rate is unacceptable for security |
| Agent should be professional in tone | System prompt instruction | Style guidance, probabilistic acceptable |
| Block access to competitor product data | PreToolUse hook | Business rule, compliance, deterministic required |
| Summarize findings at end of investigation | System prompt instruction | Behavioral preference, probabilistic acceptable |

---

### When to Use Prompt Chaining vs Adaptive Decomposition

| Signal | Pattern | Example |
|---|---|---|
| Workflow steps are known in advance | Prompt chaining | "Review each file, then produce summary" |
| Output of step N always feeds step N+1 in the same way | Prompt chaining | Code review pipeline: per-file → cross-file → summary |
| Task structure depends on what is found | Adaptive decomposition | "Add tests to this unfamiliar legacy codebase" |
| Sub-tasks are discovered during investigation | Adaptive decomposition | "Investigate why production latency increased" |
| Parallel execution of identical operations | Prompt chaining with parallelism | Per-file review with parallel subagent spawning |
| Unknown number of sub-tasks | Adaptive decomposition | "Research this topic comprehensively" |

---

### When to Resume a Session vs Start Fresh

| Condition | Decision | Reason |
|---|---|---|
| Files analyzed in prior session have NOT changed | Resume | Prior analysis is still valid, saving re-analysis cost |
| Files analyzed in prior session HAVE been modified | Resume + notify agent of changes | Context is valuable but specific files need re-analysis |
| Prior session had significant errors or misunderstandings | Start fresh with summary | Don't carry forward flawed reasoning |
| External data (APIs, DB) may have changed significantly | Start fresh with summary | Stale tool results look identical to fresh ones — risky |
| Continuing a multi-session investigation interrupted mid-task | Resume | All prior reasoning is relevant and valid |
| Prior session is weeks/months old with major changes | Start fresh with summary | More efficient to inject valid findings selectively |
| Quick follow-up question on yesterday's stable analysis | Resume | Context directly relevant, no significant changes |

---

### When to Use Each Coordinator Pattern

| Pattern | Use When | Avoid When |
|---|---|---|
| Fixed full pipeline (always all subagents) | Every query needs every agent's contribution | Many queries only need a subset of agents (wastes resources) |
| Dynamic selective routing | Queries vary significantly in what they require | All queries genuinely need all agents (adds complexity without benefit) |
| Parallel spawning (multiple Task calls in one response) | Sub-tasks are independent and can run concurrently | Tasks have strict dependencies (B requires A's output) |
| Iterative refinement loop | Quality criteria require evaluation and gap-filling | Simple one-shot tasks with low quality requirements |
| fork_session | Need to compare divergent approaches from a shared baseline | Single approach — no divergence needed |

---

## If You're Short on Time

These are the 5 most critical facts for Domain 1. If you absorb nothing else, know these cold.

---

**Fact 1: The agentic loop terminates on `stop_reason == "end_turn"`, not on any text signal.**

The loop continues when `stop_reason == "tool_use"`. It terminates when `stop_reason == "end_turn"`. No other signal — not the presence of text content, not a counter, not parsing Claude's output — is the correct primary termination mechanism. This is the most frequently tested concept in Task 1.1.

---

**Fact 2: Subagents have zero automatic context inheritance — all context must be explicitly passed.**

A subagent receives only what is in its prompt. It does not see the coordinator's conversation history, prior subagent outputs, or any shared memory. If the synthesis agent needs the web search results, those results must be embedded verbatim in the synthesis agent's prompt. This is the most frequently tested concept in Task 1.3.

---

**Fact 3: Hooks provide deterministic guarantees; prompt instructions have a non-zero failure rate.**

When a business rule must never be violated (identity verification before financial operations, refund thresholds, audit logging), the correct mechanism is a hook — PreToolUse for blocking, PostToolUse for transformation. Prompt instructions are appropriate for behavioral guidance and preferences, not for compliance requirements. This is the most frequently tested concept in Tasks 1.4 and 1.5.

---

**Fact 4: All subagent communication routes through the coordinator — never subagent-to-subagent.**

The hub-and-spoke architecture is non-negotiable in the exam context. Subagents do not call each other directly. The coordinator receives all subagent outputs and decides what to do with them. This is the foundational architectural principle of Task 1.2.

---

**Fact 5: Parallel subagent spawning requires multiple Task tool calls in a single coordinator response.**

To execute three subagents in parallel, the coordinator must emit all three Task tool calls in one response. Emitting them across three separate turns forces sequential execution. This timing and parallelism distinction is directly tested in Task 1.3.

---

*Study notes compiled for the Claude Certified Architect – Foundations exam, Domain 1: Agentic Architecture & Orchestration (27% of scored content).*
*Current as of: March 2026*
