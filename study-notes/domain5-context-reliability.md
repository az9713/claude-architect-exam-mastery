# Domain 5: Context Management & Reliability
**Exam Weight: 15% | Task Statements: 5.1 – 5.6**

---

## Task Statement 5.1: Manage conversation context to preserve critical information across long interactions

### What You Need to Know
- **Progressive summarization risks**: condensing numerical values, percentages, dates, and customer-stated expectations into vague summaries loses precision
- **"Lost in the middle" effect**: models reliably process information at beginning and end of long inputs; findings from middle sections may be omitted
- Tool results accumulate in context and consume tokens disproportionately to their relevance (e.g., 40+ fields per order lookup when only 5 are relevant)
- Complete conversation history must be passed in subsequent API requests to maintain conversational coherence

### What You Need to Be Able to Do
- Extract transactional facts (amounts, dates, order numbers, statuses) into a persistent "case facts" block included in each prompt, outside summarized history
- Extract and persist structured issue data into a separate context layer for multi-issue sessions
- Trim verbose tool outputs to only relevant fields before they accumulate in context
- Place key findings summaries at the beginning of aggregated inputs with explicit section headers to mitigate position effects
- Require subagents to include metadata (dates, source locations, context) in structured outputs for accurate downstream synthesis

### Deep Dive

**Progressive Summarization Risk**

Bad summary (typical progressive summarization):

```
summary = "Customer had order issue and requested refund"
# LOST: customer name, order ID, exact amounts, exact dates, customer urgency
```

Better: extract critical facts into a separate structured block:

```python
case_facts = {
    "customer_id": "CHEN-4892",
    "order_id": "ORD-20240892",
    "order_date": "2024-11-15",
    "issue": "Wrong color: ordered Midnight Blue, received Royal Blue",
    "refund_amount": 127.99,
    "customer_urgency": "Needs resolution today (event tomorrow)"
}
# Included verbatim in every prompt — never summarized
```

**Case Facts Block Pattern**

**Three-Layer Context Structure:**
1. **Case facts** (exact data — never summarized): customer ID, order ID, amounts, dates, customer-stated expectations
2. **Conversation summary** (prose — can be summarized): what happened so far
3. **Current message**: the latest user input

Case facts are included verbatim in every prompt. When new facts emerge, update the facts block — never merge into the summary.

**Trimming Verbose Tool Outputs**

A full order lookup returns 40+ fields (warehouse IDs, packing timestamps, fulfillment center details). For return-processing, trim to ~7 relevant fields: `order_id`, `status`, `delivery_date`, `total`, `items`, `carrier`, `tracking_number`. This prevents token waste from irrelevant data accumulating in context.

**Mitigating the "Lost in the Middle" Effect**

When aggregating multiple agent results, structure the prompt to exploit model attention patterns: place key findings at the **BEGINNING** (reliably attended), put detailed results with explicit section headers in the **MIDDLE**, and place coverage gaps and limitations at the **END** (reliably attended). Content buried without headers in the middle of a long input is the most likely to be missed.

> **Exam Tip:** When a question describes the agent "forgetting" exact amounts, dates, or customer-stated requirements after several turns — the fix is extracting those facts into a separate persistent "case facts" block, not summarizing more carefully.

> **Anti-Pattern Alert:** Using progressive summarization for numerical data is the key anti-pattern here. Summaries are for conversational context. Exact numbers, dates, and stated expectations must be preserved verbatim in a structured facts layer.

> **Cross-Domain Connection:** Task 5.4 covers scratchpad files for long exploration sessions — the same principle of preserving exact facts outside of summarized context, but persisted to disk for crash recovery.

---

## Task Statement 5.2: Design effective escalation and ambiguity resolution patterns

### What You Need to Know
- **Appropriate escalation triggers**:
  1. Customer explicitly requests a human agent
  2. Policy exceptions or gaps (policy doesn't cover this case)
  3. Inability to make meaningful progress after attempts
- Explicit customer request → **immediate escalation** (do not attempt to resolve first)
- **Sentiment-based escalation** (frustrated customer = escalate) is unreliable — frustration doesn't correlate with case complexity
- **Self-reported confidence scores** are unreliable proxies for case complexity — the model is already wrong on hard cases with high confidence
- Multiple customer matches → request additional identifiers, not heuristic selection

### What You Need to Be Able to Do
- Add explicit escalation criteria with few-shot examples demonstrating when to escalate vs. resolve autonomously
- Honor explicit customer requests for human agents immediately without first attempting investigation
- Acknowledge frustration while offering resolution when issue is within capability; escalate only if customer reiterates
- Escalate when policy is ambiguous or silent on the customer's specific request
- Instruct agent to ask for additional identifiers when tool results return multiple matches

### Deep Dive

**Escalation Trigger Decision Table**

| Situation | Action | Reasoning |
|---|---|---|
| Customer explicitly says "I want a human" | Escalate IMMEDIATELY | Do NOT investigate first; honoring the request builds trust |
| Policy gap (request not covered by policy) | Escalate with context | Agent cannot make policy exceptions |
| Cannot make progress after 3+ attempts | Escalate with summary of attempts | Continued failure frustrates; escalation unblocks |
| Customer is frustrated or angry | Offer resolution first | Frustration does not equal complexity; resolve if within capability |
| Agent is uncertain | Attempt with stated uncertainty | Uncertainty is not an escalation trigger |
| Self-reported confidence is low | Attempt resolution | Confidence is poorly calibrated; already wrong on hard cases with high confidence |

**Three WRONG answers on exam:**
1. Sentiment-based escalation — an angry customer with a simple problem should be resolved by the agent.
2. Confidence-based escalation — model confidence does not indicate case complexity.
3. "Investigate first, then offer escalation" when the customer has already explicitly asked for a human — this is the most frequently tested wrong answer. Explicit request = escalate immediately, no investigation.

**The Three Escalation Triggers (with examples)**

#### Trigger 1: Explicit human request → ESCALATE IMMEDIATELY
Customer says: "I want to speak to a real person" / "Let me talk to your supervisor" / "Stop trying to help me with AI, I want a human"

**Action:** Escalate WITHOUT attempting investigation or resolution first.
**Reasoning:** Honoring explicit requests builds trust; investigating first is paternalistic.

#### Trigger 2: Policy gap/exception → ESCALATE
Customer requests: "Price match to competitor's price" (policy only covers own-site) / "Retroactive discount for an order from 6 months ago" (policy: 30 days) / "Custom invoice format for tax filing" (not in standard options)

**Action:** Escalate with context (what was requested, why policy doesn't cover it).
**Reasoning:** Agents should not make policy exceptions; escalate to human with authority.

#### Trigger 3: Cannot make progress → ESCALATE
After 3+ attempts: information system returns no records despite customer certainty; tool errors prevent access to required account data; each attempt produces the same unhelpful result.

**Action:** Escalate with summary of what was tried and what information is missing.
**Reasoning:** Continued failed attempts frustrate customers; escalation unblocks them.

#### NOT an escalation trigger:
- Customer is frustrated (frustration ≠ complexity; offer resolution first)
- Case seems complex (model's assessment of complexity is unreliable)
- Agent is unsure (uncertainty ≠ escalation; attempt with stated uncertainty)

**Few-Shot Examples for Escalation Calibration**

**Example 1 — Straightforward case with frustrated customer:**
> Customer: "I've been waiting 3 weeks for my order! This is ridiculous! I want this fixed!"
>
> Action: RESOLVE (do not escalate based on frustration alone)
> Reasoning: Order delay is within the agent's capability to investigate and resolve.
> Response: "I understand this has been frustrating. Let me look into your order right now and get this resolved for you."

**Example 2 — Explicit escalation request:**
> Customer: "I've talked to two chatbots already. I need a real human."
>
> Action: ESCALATE IMMEDIATELY
> Reasoning: Explicit request for human agent; do not attempt resolution first.
> Response: "I'll connect you with a team member right away."

**Ambiguity Resolution: Multiple Matches**

When a customer lookup returns results, the correct action depends on the match count:

- **0 matches:** Ask for more information (email address or order number)
- **1 match:** Proceed with that record
- **Multiple matches:** Request additional identifiers (email, phone, order number). NEVER select by heuristic (e.g., "most recent account" or "first result")

> **Exam Tip:** Three wrong answers always appear: (1) sentiment-based escalation, (2) self-reported confidence escalation, (3) "investigate first then offer escalation" when customer explicitly requested human. Remember: explicit request = immediate escalation.

> **Anti-Pattern Alert:** Building a sentiment classifier to detect frustrated customers and route them to humans solves the wrong problem. Frustration is a communication style, not an indicator of case complexity. An angry customer with a simple case should be resolved by the agent.

> **Cross-Domain Connection:** Task 1.4 covers structured handoff protocols when escalation occurs — the case facts and handoff summary structure built in Task 5.1 feeds directly into the escalation handoff.

---

## Task Statement 5.3: Implement error propagation strategies across multi-agent systems

### What You Need to Know
- **Structured error context**: failure type + attempted query + partial results + alternative approaches enables intelligent coordinator recovery
- Distinction between **access failures** (timeouts needing retry decisions) and **valid empty results** (successful query with no matches)
- Generic error statuses (`"search unavailable"`) hide valuable context from the coordinator
- Both **silently suppressing errors** (returning empty results as success) and **terminating entire workflows on single failures** are anti-patterns

### What You Need to Be Able to Do
- Return structured error context including failure type, what was attempted, partial results, and potential alternatives
- Distinguish access failures from valid empty results so the coordinator makes appropriate decisions
- Have subagents implement local recovery for transient failures; propagate only unresolvable errors
- Structure synthesis output with coverage annotations indicating well-supported vs. gap findings

### Deep Dive

**Structured Error Context Pattern**

A generic error response (`"status": "error", "message": "Search unavailable"`) tells the coordinator nothing useful: is this transient? permanent? should it retry, skip, or escalate?

A structured error response enables intelligent coordinator recovery. Key fields to include:

- `success`: false
- `errorType`: specific category (e.g., `timeout`, `rate_limit`, `source_unavailable`)
- `attemptedQuery`: what was tried
- `partialResults`: any results retrieved before failure
- `isRetryable`: boolean — whether a retry is worth attempting
- `retryAfterSeconds`: how long to wait before retry (for transient errors)
- `coordinatorOptions`: list of specific recovery actions (RETRY, NARROW, SUBSTITUTE, SKIP)
- `alternativeSources`: fallback sources to query (for permanent source failures)

**Access Failure vs. Empty Result Distinction**

These two outcomes must be explicitly distinguished in every tool response. Conflating them causes the coordinator to retry successful empty searches (waste) or silently drop real access failures (missed errors).

| Response | Meaning | Correct coordinator action |
|---|---|---|
| `[]` with `success: true` | Query executed; no records found | Tell customer "no orders found." NOT an error. Do NOT retry. |
| `None` / exception with `success: false` | Access failure — tool could not execute | Flag `isRetryable: true`; consider retry or escalate |

The fix: every tool response must explicitly set `isEmptyResult: true` when the query completed successfully but returned no records. This single boolean is what separates a correctly-handled "nothing found" from an access failure requiring retry or escalation.

See Task 2.2 for the same principle applied at the individual tool level.

**Coverage Annotations in Synthesis Output**

Each finding in a synthesis output should carry a coverage annotation so downstream consumers know how much to trust it:

| Data Status | `status` value | Confidence | Notes |
|---|---|---|---|
| Query succeeded, results found | `well_supported` | high | Include source count |
| Query succeeded, zero results | `no_data_found` | n/a | Not an error; topic has no coverage in this source |
| Query failed (access error) | `access_failure` | unknown | Include errorType; suggest retry or substitute source |

> **Exam Tip:** Questions about error propagation always contrast two bad patterns: (1) silent suppression (returning empty results as if successful) and (2) total failure (crashing the entire workflow on one subagent failure). The correct answer is structured error propagation with partial results and coordinator options.

> **Anti-Pattern Alert:** A search returning zero results is NOT an error. Reporting it as an error causes the coordinator to retry an operation that completed successfully. Explicitly flag `isEmptyResult: true` to distinguish from access failures.

> **Cross-Domain Connection:** Task 2.2 covers MCP tool-level structured error responses — the same `errorCategory`/`isRetryable` pattern applied at the individual tool level. Task 5.3 applies the same principle at the subagent-to-coordinator level.

---

## Task Statement 5.4: Manage context effectively in large codebase exploration

### What You Need to Know
- **Context degradation**: in extended sessions, models start giving inconsistent answers and referencing "typical patterns" rather than specific classes discovered earlier
- **Scratchpad files**: persist key findings across context boundaries; agents reference them for subsequent questions
- **Subagent delegation**: isolates verbose exploration output while main agent coordinates high-level understanding
- **Structured state persistence for crash recovery**: each agent exports state to a known location; coordinator loads a manifest on resume

### What You Need to Be Able to Do
- Spawn subagents to investigate specific questions while main agent preserves high-level coordination
- Have agents maintain scratchpad files recording key findings; reference them to counteract context degradation
- Summarize key findings from one exploration phase before spawning sub-agents for the next phase
- Design crash recovery using structured agent state exports (manifests) that coordinator loads on resume
- Use `/compact` to reduce context usage during extended exploration when context fills with verbose discovery output

### Deep Dive

**Scratchpad File Pattern**

A scratchpad file is a persistent on-disk document (e.g., `.agent_workspace/exploration_notes.md`) that the agent appends to as it makes discoveries during codebase exploration. Each significant finding — a module's dependencies, a data flow, a security boundary — is written to the scratchpad with a section header immediately upon discovery.

This counteracts context degradation: as the context window fills with hundreds of tool call results, earlier findings get pushed outside the model's effective attention range. By writing findings to disk and referencing the scratchpad file at the start of subsequent prompts ("use these notes as your source of truth; do not re-explore"), the agent can answer questions accurately even after extensive exploration — drawing on the file rather than relying on context memory.

**Subagent Delegation for Verbose Discovery**

The main agent spawns an Explore subagent for each investigation task (e.g., "map the codebase structure" or "trace the authentication flow"). The subagent performs the verbose work — reading 50+ files, running searches, tracing dependencies — using tools like Read, Grep, and Glob. It then returns a concise summary (~500 tokens) to the main agent, writes that summary to the scratchpad, and terminates.

The main agent's context stays clean: it only sees summaries, not the 50+ file reads the subagent performed. This allows the main agent to coordinate across many investigation phases without its context filling with raw tool output.

**The `/compact` Command and Instruction Persistence**

The `/compact` command reduces context usage during extended sessions by condensing accumulated tool output and conversation history. Two important rules about what survives compaction:

- **CLAUDE.md survives compaction** — it is re-read from disk after compaction. Instructions, project context, and agent rules placed in CLAUDE.md are always available regardless of how many times `/compact` is run.
- **Conversation-only instructions do NOT survive compaction** — instructions given only in the chat or system prompt during the session are gone after compaction. Any instruction that must persist across compaction events must be written to CLAUDE.md, not left only in the conversation.

This distinction matters for agent design: behavioral rules and persistent context belong in CLAUDE.md; ephemeral task instructions can remain in the conversation.

**Crash Recovery with Manifests**

Each agent periodically exports its current state — completed work, pending work, and partial results — to a known location (e.g., `.agent_workspace/manifest.json`). When an agent finishes, it updates its manifest entry to `"status": "complete"`.

On crash recovery, the coordinator loads the manifest and inspects which agents have `"status": "in_progress"` (i.e., did not complete). It re-spawns only those agents, injecting their previously saved state so they resume from where they left off rather than starting over. Agents that completed are skipped; their results are already available on disk.

> **Exam Tip:** When a question describes the agent "forgetting" classes it found 50 tool calls ago or giving inconsistent answers after extensive exploration — the fix is scratchpad files or subagent delegation. Both prevent context degradation.

> **Anti-Pattern Alert:** Running all file exploration in the main agent context fills it with verbose tool output that pushes earlier findings out of effective attention range. Delegate verbose discovery to subagents that return summaries.

> **Cross-Domain Connection:** Task 1.3 covers subagent spawning in the Agent SDK — the Explore subagent referenced in Task 3.4 is the same pattern applied in Claude Code. Both isolate verbose discovery from the main coordination context.

---

## Task Statement 5.5: Design human review workflows and confidence calibration

### What You Need to Know
- Aggregate accuracy metrics (e.g., "97% overall accuracy") may mask poor performance on specific document types or fields
- **Stratified random sampling**: measure error rates in high-confidence extractions and detect novel error patterns
- **Field-level confidence scores** calibrated using labeled validation sets for routing review attention
- Validate accuracy by document type and field segment before automating high-confidence extractions

### What You Need to Be Able to Do
- Implement stratified random sampling of high-confidence extractions for ongoing error rate measurement
- Analyze accuracy by document type and field to verify consistent performance across all segments
- Have models output field-level confidence scores; calibrate review thresholds using labeled validation sets
- Route extractions with low model confidence or ambiguous/contradictory source documents to human review

### Deep Dive

**Why Aggregate Metrics Are Insufficient**

A system with 97% overall accuracy may have 99.5% accuracy on standard invoices but 68% accuracy on non-English invoices. The 97% aggregate masks that one segment has a 32% error rate. Reducing human review based on the aggregate number means accepting a 1-in-3 failure rate on an entire document class.

```
Scenario: Invoice extraction system, 97% overall accuracy

Aggregate metric says: READY TO AUTOMATE

Reality by segment:
  Standard invoices: 99.5% accuracy ✓
  Multi-currency invoices: 94% accuracy — acceptable?
  Handwritten invoices: 71% accuracy ✗
  Non-English invoices: 68% accuracy ✗  ← 32% error rate hidden by aggregate

Decision: The 97% aggregate masks that two document types
need human review or prompt improvement before automation.

Rule: Never reduce human review based on aggregate accuracy alone.
      Segment by: document type AND field before making automation decisions.
```

**Stratified Random Sampling**

Stratified random sampling draws a proportional sample from each confidence stratum (high, medium, low) at the same rate (e.g., 5% of each). The critical purpose is catching errors in **high-confidence extractions** — these are the most dangerous failures because they bypass review. After human reviewers check the samples, error rates are calculated per stratum, per field, and per document type. Novel error patterns discovered in the sample trigger prompt improvements or threshold adjustments before they affect production volume.

**Field-Level Confidence Calibration**

**4-Step Workflow:**

1. **Model outputs field-level confidence scores** alongside each extracted value (e.g., `invoice_number_confidence: "high"`, `payment_terms_confidence: "low"`). Scores use three levels: high = clearly stated, medium = inferred, low = ambiguous or imprecise.

2. **Calibrate thresholds using a labeled validation set.** For each confidence level, measure what percentage of the model's extractions match human-correct outputs. Example result: high confidence = 98% accurate (automate), medium = 87% (review), low = 61% (always review).

3. **Route each extraction based on calibrated thresholds.** Any field with low confidence, or any field below the accuracy threshold, routes to human review. Documents with internal conflicts (`conflict_detected: true`) also route to human review.

4. **Apply stratified sampling continuously** to catch drift — error rates in high-confidence extractions shift over time as document formats change.

> **Exam Tip:** "The system has 97% overall accuracy. Should we automate?" — the correct answer is no, not without segmenting by document type and field first. Aggregate accuracy masking segment-level failures is the core concept.

> **Anti-Pattern Alert:** Using overall accuracy to make automation decisions ignores segment-level variation. A 97% overall with 68% on non-English documents means 32% error rate for that segment — unacceptable.

> **Cross-Domain Connection:** Task 4.4 covers confidence scores in code review findings for routing decisions — same calibrated confidence routing concept applied to code review rather than document extraction.

---

## Task Statement 5.6: Preserve information provenance and handle uncertainty in multi-source synthesis

### What You Need to Know
- Source attribution is lost during summarization when findings are compressed without preserving claim-source mappings
- **Structured claim-source mappings** must be preserved and merged by the synthesis agent when combining findings
- **Conflicting statistics from credible sources**: annotate conflicts with source attribution rather than arbitrarily selecting one value
- **Temporal data**: require publication/collection dates in structured outputs to prevent temporal differences from being misinterpreted as contradictions

### What You Need to Be Able to Do
- Require subagents to output structured claim-source mappings (source URLs, document names, relevant excerpts) that downstream agents preserve through synthesis
- Structure reports with sections distinguishing well-established findings from contested ones
- Complete document analysis with conflicting values included and explicitly annotated; let coordinator decide how to reconcile
- Require subagents to include publication or data collection dates in structured outputs
- Render different content types appropriately (financial data as tables, news as prose, technical findings as lists)

### Deep Dive

**Claim-Source Mapping Structure**

Each claim a subagent returns must carry its source attribution — not as a footnote but as a structured field the synthesis agent can read and preserve:

```python
{
    "claim": "Global AI market size reached $500B in 2025",
    "sources": [{"url": "...", "title": "AI Market Report 2025",
                 "publishDate": "2025-11-01", "excerpt": "..."}],
    "confidence": "high",
    "isContested": False
}
```

For a contested claim, the structure additionally carries both conflicting sources and a `contestDescription`:

```python
{
    "claim": "AI will replace X% of jobs by 2030",
    "confidence": "low",
    "isContested": True,
    "contestDescription": "Source A (2024) says 30%; Source B (2025) says 12%"
}
```

**Synthesis Agent: Preserving Provenance Through Merging**

The synthesis agent must not compress claims into bare statements — each claim in the output must trace to its originating sources. When merging findings from multiple subagents, the synthesis agent checks whether a similar claim already exists. If the publication dates differ, it marks the pair as a temporal difference rather than a contradiction. If both sources genuinely conflict, it merges them into a contested claim with `isContested: true` and the combined attribution.

The final report is structured in sections: well-established findings (high confidence, uncontested), contested findings (annotated with both sources), and uncertain findings (medium/low confidence). Financial data is rendered as tables, news as prose, and technical findings as lists — matching the rendering format to the content type.

**Handling Conflicting Statistics**

| Situation | Action | NEVER do |
|---|---|---|
| Source A: $200B, Source B: $500B (different dates) | Flag as temporal difference; present both values with their dates | Silently pick one value |
| Source A: 30%, Source B: 12% (same date, credible sources) | Annotate both with full source attribution; flag `requiresCoordinatorDecision: true` | Let synthesis agent choose |
| Either source is missing a publication date | Require subagents to include dates before synthesis; flag ambiguity | Assume it is a contradiction |

The synthesis agent's role is to surface conflicts, not resolve them. Resolving a conflict requires judgment about source credibility — that judgment belongs to the coordinator or a human reviewer, not the synthesis step.

**Temporal Data Requirements**

Subagent output schemas must require `publicationDate` (when the source was published) and `dataCollectionDate` (when the data was actually measured) as ISO 8601 strings on every statistic. Without dates, apparent contradictions cannot be distinguished from temporal evolution.

Example: "AI market was $200B (Source A) vs $500B (Source B) — contradiction?" With dates: "$200B in 2022 vs $500B in 2025" — not a contradiction; the market grew over three years.

> **Exam Tip:** "Source A says X, Source B says Y — how should synthesis handle this?" The correct answer is always to annotate both with source attribution, check if dates explain the difference, and let the coordinator (or human) decide. Never silently pick one value.

> **Anti-Pattern Alert:** Having the synthesis agent decide which conflicting source to trust removes provenance. The synthesis agent should flag conflicts and pass them up, not resolve them unilaterally.

> **Cross-Domain Connection:** Task 5.1 covers the general context management principle — the claim-source mapping is a specific instance of preserving exact, precise data (like case facts) rather than summarizing it away.

---

## Key Terminology Quick Reference

| Term | Definition | Exam Relevance |
|------|-----------|----------------|
| Progressive summarization | Condensing context into prose summaries | Destroys numerical precision; use case facts block instead |
| Case facts block | Structured JSON of exact transactional data | Preserved verbatim across turns; never summarized |
| Lost in the middle | Model attends less to middle-of-input content | Place key findings at beginning; use section headers |
| Token accumulation | Tool results consuming context tokens | Trim to relevant fields only before adding to context |
| Escalation trigger | Condition requiring human handoff | Explicit request, policy gap, inability to progress |
| Sentiment-based escalation | Escalating frustrated customers | Unreliable; frustration ≠ complexity |
| Self-reported confidence | Model's stated certainty level | Poorly calibrated; already wrong on hard cases |
| Structured error context | Error with type, query, partial results, options | Enables coordinator recovery decisions |
| Access failure | Tool unavailable (timeout, connection error) | Retryable; distinct from empty results |
| Empty result | Query succeeded; no matching records | NOT an error; explicitly flag `isEmptyResult` |
| Scratchpad file | Persistent file recording agent findings | Counteracts context degradation across long sessions |
| Context degradation | Model references "typical patterns" vs. specifics | Indicator that key findings have been lost from context |
| State manifest | JSON file with each agent's exported state | Enables crash recovery; coordinator loads on resume |
| `/compact` | Claude Code command to reduce context usage | Use during extended exploration sessions |
| Stratified random sampling | Sampling proportionally from each confidence stratum | Catches errors in high-confidence extractions |
| Aggregate accuracy masking | Overall metric hiding segment-level failures | Always segment by document type and field before automating |
| Field-level confidence | Per-field certainty score (high/medium/low) | Calibrated with labeled set; used to route to human review |
| Claim-source mapping | Structured link between claim and its source | Must be preserved through synthesis, not summarized away |
| Temporal difference | Values differ due to measurement date | Distinguish from genuine contradiction; require dates in output |
| Conflict annotation | Flagging contradictory values with both sources | Never silently select; pass conflict up to coordinator |

---

## Decision Matrix

### When to escalate?

| Trigger | Action | Notes |
|---------|--------|-------|
| Customer explicitly requests human | Escalate immediately | Do NOT investigate first |
| Policy gap/exception needed | Escalate with context | Policy defines scope; agent cannot extend it |
| Cannot make progress after attempts | Escalate with summary | Include what was tried and what was missing |
| Customer is frustrated/angry | Offer resolution first | Only escalate if customer reiterates after offer |
| Case seems complex | Attempt resolution | Complexity estimate is unreliable |
| Low model confidence | Attempt with caveats | Confidence ≠ escalation trigger |

### Which context management approach?

| Problem | Solution |
|---------|----------|
| Exact amounts/dates lost in long sessions | Case facts block (preserved verbatim) |
| Tool output consuming too many tokens | Trim to relevant fields before adding to context |
| Agent referencing "typical patterns" vs. specifics | Scratchpad file or subagent delegation |
| Verbose discovery filling main context | Explore subagent (returns summaries) |
| Context fills during extended session | `/compact` command |
| Crash mid-workflow | State manifest; coordinator loads on resume |
| Findings in middle of long input are missed | Key summaries at beginning; section headers |

### How to handle conflicting sources?

| Situation | Approach |
|-----------|----------|
| Source A says X, Source B says Y (same date) | Annotate both; flag `requiresCoordinatorDecision` |
| Source A says X (2022), Source B says Y (2025) | Flag as temporal difference; not contradiction |
| Contradictory values in same document | `conflict_detected: true` + description |
| Missing publication date makes conflict ambiguous | Require dates in subagent output schema |

---

## If You're Short on Time

1. **Case facts block**: exact transactional data (amounts, dates, order IDs) must be preserved verbatim outside of summarized conversation history — progressive summarization destroys numerical precision.
2. **Three escalation triggers**: (1) customer explicitly requests human, (2) policy gap/exception, (3) inability to progress. Sentiment and self-reported confidence are NOT reliable escalation signals.
3. **Structured error propagation**: `errorType` + `attemptedQuery` + `partialResults` + `coordinatorOptions` — generic `"search unavailable"` hides recovery information from the coordinator.
4. **Scratchpad files + subagent delegation** combat context degradation in long exploration sessions — verbose discovery output in the main session pushes earlier findings out of effective attention.
5. **Conflicting sources**: always annotate both values with source attribution and check if publication dates explain the difference — never silently pick one value; let the coordinator or human decide.
