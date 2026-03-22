# Scenario 1: Customer Support Resolution Agent

## Scenario Context

You are building a customer support resolution agent using the Claude Agent SDK. The agent handles high-ambiguity requests like returns, billing disputes, and account issues. It has access to your backend systems through custom Model Context Protocol (MCP) tools:

- `get_customer` — retrieves verified customer record by ID or email
- `lookup_order` — retrieves order details by order number
- `process_refund` — initiates a refund for a given order
- `escalate_to_human` — transfers the case to a human agent with a structured handoff summary

Your target is **80%+ first-contact resolution** while knowing when to escalate appropriately.

**Primary Domains:** D1 (Agentic Architecture), D2 (Tool Design & MCP), D5 (Context Management & Reliability)

---

## Key Concepts for This Scenario

### Relevant Task Statements and Why They Matter

**Task Statement 1.1 — Agentic Loop Design**
The customer support agent runs an agentic loop: after each turn, Claude inspects `stop_reason`. When it is `"tool_use"`, the loop executes the requested tools and returns results. When it is `"end_turn"`, the loop presents the final response to the customer. Without correct stop_reason handling, the agent either gets stuck or terminates too early.

**Task Statement 1.4 — Multi-Step Workflows with Enforcement**
Customer interactions often involve prerequisite dependencies: you must verify a customer's identity before processing a refund. Prompt instructions alone have a non-zero failure rate. Programmatic enforcement (blocking `process_refund` until `get_customer` has returned a verified ID) provides deterministic compliance for financial operations.

**Task Statement 1.5 — Hooks for Tool Call Interception**
The Agent SDK's `PostToolUse` hooks can intercept tool results and normalize them (e.g., converting Unix timestamps to human-readable dates) before the model processes them. Pre-call hooks can enforce business rules (e.g., blocking refunds above $500 and redirecting to human escalation) with guaranteed compliance.

**Task Statement 2.1 — Tool Interface Design**
With four tools that accept similar identifier formats, tool descriptions are the primary mechanism Claude uses to select the right tool. Minimal descriptions like "Retrieves customer information" cause misrouting. Descriptions should include: what the tool does, what inputs it expects, when to use it versus alternatives, and example queries.

**Task Statement 2.2 — Structured Error Responses**
Each MCP tool must return structured errors — not generic "Operation failed" messages. The agent needs to know: is this a transient timeout (retry?), a validation error (bad input), a business rule violation (non-retryable), or a permissions error? Without structured error metadata, the agent cannot make appropriate recovery decisions.

**Task Statement 5.1 — Context Management**
Customer support sessions accumulate verbose tool output (order records with 40+ fields, transaction logs). Only a few fields per tool response are relevant to the current issue. Trimming outputs before they accumulate preserves context budget. Transactional facts (amounts, dates, order numbers, statuses) should be extracted into a persistent "case facts" block included in every prompt.

**Task Statement 5.2 — Escalation and Ambiguity Resolution**
Agents without clear escalation criteria either over-escalate (wasting human capacity) or under-escalate (attempting cases beyond their capability). Explicit escalation criteria with few-shot examples defining when to escalate versus resolve autonomously improve calibration. Sentiment-based escalation is an unreliable proxy for case complexity.

---

## Practice Questions

### Question 1

Production data shows that in 12% of cases, your agent skips `get_customer` entirely and calls `lookup_order` using only the customer's stated name, occasionally leading to misidentified accounts and incorrect refunds. What change would most effectively address this reliability issue?

**A)** Add a programmatic prerequisite that blocks `lookup_order` and `process_refund` calls until `get_customer` has returned a verified customer ID.

**B)** Enhance the system prompt to state that customer verification via `get_customer` is mandatory before any order operations.

**C)** Add few-shot examples showing the agent always calling `get_customer` first, even when customers volunteer order details.

**D)** Implement a routing classifier that analyzes each request and enables only the subset of tools appropriate for that request type.

**Correct Answer: A**

**Explanation:** When a specific tool sequence is required for critical business logic — verifying customer identity before processing refunds — programmatic enforcement provides deterministic guarantees that prompt-based approaches cannot. The agent has a 12% failure rate on this rule even without prompt guidance; adding more prompt guidance (Option B) or examples (Option C) still leaves a non-zero failure rate, which is unacceptable when errors have direct financial consequences (wrong refunds). Option D addresses tool availability, not tool ordering, which is the actual problem — the agent can call `get_customer`, it just sometimes skips it. A programmatic prerequisite gate that checks for a verified customer ID before allowing downstream tool calls is the only approach that guarantees compliance 100% of the time.

---

### Question 2

Production logs show the agent frequently calls `get_customer` when users ask about orders (e.g., "check my order #12345"), instead of calling `lookup_order`. Both tools have minimal descriptions ("Retrieves customer information" / "Retrieves order details") and accept similar identifier formats. What is the most effective first step to improve tool selection reliability?

**A)** Add few-shot examples to the system prompt demonstrating correct tool selection patterns, with 5–8 examples showing order-related queries routing to `lookup_order`.

**B)** Expand each tool's description to include input formats it handles, example queries, edge cases, and boundaries explaining when to use it versus similar tools.

**C)** Implement a routing layer that parses user input before each turn and pre-selects the appropriate tool based on detected keywords and identifier patterns.

**D)** Consolidate both tools into a single `lookup_entity` tool that accepts any identifier and internally determines which backend to query.

**Correct Answer: B**

**Explanation:** Tool descriptions are the primary mechanism LLMs use for tool selection. When descriptions are minimal, the model lacks the context to differentiate between tools with similar purposes. Option B directly addresses this root cause with a low-effort, high-leverage fix — adding input formats, example queries, and clear boundaries (e.g., "use this when the customer references an order number; use `get_customer` when they reference their account or identity"). Few-shot examples (Option A) add token overhead and are the right next step *after* descriptions are fixed, but they don't address the root cause. Option C is an over-engineered bypass of the LLM's natural language understanding. Option D is a valid architectural choice but requires significant backend refactoring — it is not proportionate as a "first step" when the immediate problem is inadequate descriptions.

---

### Question 3

Your agent achieves 55% first-contact resolution, well below the 80% target. Logs show it escalates straightforward cases (standard damage replacements with clear photo evidence) while attempting to autonomously handle complex situations requiring policy exceptions. What is the most effective way to improve escalation calibration?

**A)** Add explicit escalation criteria to your system prompt with few-shot examples demonstrating when to escalate versus resolve autonomously.

**B)** Have the agent self-report a confidence score (1–10) before each response and automatically route requests to humans when confidence falls below a threshold.

**C)** Deploy a separate classifier model trained on historical tickets to predict which requests need escalation before the main agent begins processing.

**D)** Implement sentiment analysis to detect customer frustration levels and automatically escalate when negative sentiment exceeds a threshold.

**Correct Answer: A**

**Explanation:** The root cause is unclear decision boundaries — the agent doesn't know what it is authorized to handle. Explicit escalation criteria with few-shot examples directly fix this: examples show which cases should be resolved autonomously (damage replacement with evidence, standard refunds within policy) versus escalated (policy exceptions, cases requiring manager discretion). Option B fails because LLM self-reported confidence is poorly calibrated — the agent is already incorrectly confident on hard cases and incorrectly uncertain on easy ones. Option C is over-engineered: it requires labeled historical data and ML infrastructure when prompt optimization has not yet been tried. Option D solves a different problem: sentiment measures customer frustration, not case complexity. A very frustrated customer may have a simple, solvable problem; a calm customer may have an unsolvable one.

---

### Question 4

A customer session involves three separate issues: a missing refund, a damaged item replacement, and an account billing discrepancy. The agent processes each issue sequentially, which means it calls `get_customer` three times and takes nearly 90 seconds end-to-end. How should the architecture be improved?

**A)** Process all three issues in parallel using a single `get_customer` call to retrieve shared context, then decompose into parallel investigation threads before synthesizing a unified resolution.

**B)** Use a routing classifier to route the customer to three separate specialized agents, each handling one issue type independently.

**C)** Process the issues sequentially but cache the `get_customer` result in a session variable so it is only called once per session.

**D)** Increase the max_tokens limit so the agent can reason about all three issues in a single model call without tool use.

**Correct Answer: A**

**Explanation:** Task Statement 1.4 describes exactly this pattern: decompose multi-concern requests into distinct items, investigate each in parallel using shared context, then synthesize a unified resolution. A single `get_customer` call establishes the verified customer context, then parallel investigation threads (using the Agent SDK's parallel tool execution) work on all three issues simultaneously. This reduces latency from sequential tool chains to the duration of the longest single thread. Option B introduces unnecessary complexity: creating three separate agents for what is one customer session loses the shared context needed for a coherent unified response. Option C addresses only the repeated `get_customer` calls but keeps the sequential structure — the fundamental bottleneck. Option D misunderstands the problem: the latency comes from actual tool execution time, not model reasoning time.

---

### Question 5

When your agent calls `process_refund` for a refund of $750, your company policy requires human manager approval for any refund over $500. Currently the agent sometimes processes these refunds autonomously. What is the correct architectural solution?

**A)** Implement a pre-call hook that intercepts all `process_refund` tool calls, checks the refund amount, and if it exceeds $500, blocks the call and redirects to the `escalate_to_human` workflow instead.

**B)** Add a rule to the system prompt: "Never call `process_refund` for amounts over $500. Always use `escalate_to_human` for large refunds."

**C)** Add validation logic inside the `process_refund` MCP tool itself to reject refunds above $500 with an error response.

**D)** Add few-shot examples showing the agent choosing `escalate_to_human` when refund amounts exceed $500.

**Correct Answer: A**

**Explanation:** Task Statement 1.5 establishes that hooks provide deterministic guarantees for business rules requiring guaranteed compliance, while prompt instructions are probabilistic. A pre-call hook intercepts the tool call *before* execution, inspects the amount, and redirects to the escalation workflow — the `process_refund` tool is never called for amounts above the threshold. This is guaranteed compliance. Option B (system prompt rule) and Option D (few-shot examples) both leave a non-zero failure rate — the agent already violates this policy sometimes, and neither approach changes the fundamental mechanism. Option C puts the validation inside the MCP tool, but this means the tool returns an error *after* being called. The hook approach prevents the call entirely and actively routes to the correct workflow, giving the agent (and the customer) a clear path forward rather than just an error response.

---

## Guided Walkthrough

### Problem: Designing the Refund Resolution Workflow

A customer sends: "I returned my order three weeks ago but still haven't received my refund. My email is sarah@example.com and the order number is #ORD-4892."

**Step 1: Agentic Loop Initialization**

The system prompt establishes the agent's role and provides explicit escalation criteria. The agent receives the customer message and enters the agentic loop.

**Step 2: First Model Turn — Tool Selection**

Claude processes the message and identifies the intent: verify customer identity, look up the order, investigate the return/refund status. Because `get_customer` has a detailed description ("Use to verify customer identity by email or account ID before any order operations — required first step for all account-related requests"), Claude selects it first.

The model returns `stop_reason: "tool_use"` with a call to `get_customer(email="sarah@example.com")`.

**Step 3: Programmatic Gate**

Before executing the tool call, the programmatic prerequisite layer allows `get_customer` — it has no dependencies. The tool executes and returns:

```json
{
  "customer_id": "CUST-1247",
  "name": "Sarah Chen",
  "account_status": "active",
  "verified": true
}
```

The `PostToolUse` hook normalizes the response (no timestamp conversion needed here) and stores `customer_id: "CUST-1247"` in the "verified context" layer.

**Step 4: Second Model Turn**

Tool result is appended to conversation history. Claude now calls `lookup_order(order_id="ORD-4892", customer_id="CUST-1247")`. The programmatic gate allows this because `get_customer` has already returned a verified customer ID.

The tool returns a verbose record with 38 fields. The `PostToolUse` hook trims this to only return-relevant fields:

```json
{
  "order_id": "ORD-4892",
  "order_date": "2026-01-15",
  "return_initiated": "2026-02-28",
  "return_status": "received",
  "refund_status": "pending",
  "refund_amount": 89.99,
  "refund_due_date": "2026-03-14"
}
```

**Step 5: Third Model Turn**

Claude analyzes the order context. The refund was due March 14 — it is now overdue. The refund_amount ($89.99) is well below the $500 hook threshold. Claude calls `process_refund(order_id="ORD-4892", customer_id="CUST-1247", amount=89.99)`.

The pre-call hook checks: amount ($89.99) < threshold ($500) → allow. The tool executes and returns:

```json
{
  "success": true,
  "refund_id": "REF-7731",
  "processing_time": "3-5 business days"
}
```

**Step 6: Final Turn**

`stop_reason: "end_turn"`. Claude synthesizes a response:

> "I've confirmed your return was received on February 28th, and I've now processed your refund of $89.99 (refund ID: REF-7731). You should see it within 3–5 business days. I apologize for the delay beyond the expected processing window."

**What Made This Work:**
- Programmatic prerequisites ensured `get_customer` ran first
- The pre-call hook checked the refund amount (no intervention needed this time)
- The `PostToolUse` hook trimmed verbose tool output, preserving context budget
- The case was resolved in one contact (first-contact resolution achieved)

---

## Try It Yourself

### Exercise: Design the Escalation Path

A customer says: "I've been trying to resolve this for two weeks and I'm absolutely furious. I want to speak to a manager RIGHT NOW about my duplicate charge."

**Your task:**

1. **Write the escalation criteria** for this scenario. Consider: should the agent escalate immediately because of the explicit request? Or attempt to resolve the duplicate charge first? What does the exam guide say about explicit customer requests for human agents?

2. **Design the handoff summary** that `escalate_to_human` should receive. What structured fields does a human agent need who has no access to the conversation transcript? (Hint: Task Statement 1.4 specifies what a structured handoff should include.)

3. **Write the system prompt instruction** for handling this case using few-shot examples. Include both the "immediate escalation" example and a contrast example where the agent acknowledges frustration and offers resolution.

4. **Identify which hook** (pre-call or `PostToolUse`) you would use to ensure that when a customer explicitly requests a human, the agent cannot call any further tools and must call `escalate_to_human`. Explain why a hook provides stronger guarantees than a prompt instruction here.

**Exam Connection:** This exercise targets Task Statements 1.4, 1.5, and 5.2. The key insight is that explicit customer requests for human escalation must be honored immediately (TS 5.2), that structured handoff summaries must include specific fields for human agents without transcript access (TS 1.4), and that deterministic enforcement via hooks is stronger than probabilistic prompt compliance (TS 1.5).
