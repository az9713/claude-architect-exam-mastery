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

```python
# Dangerous summarization — loses precision
original_info = """
Customer: Sarah Chen
Order: ORD-20240892
Ordered: 2024-11-15
Issue: Received wrong color (ordered "Midnight Blue" but received "Royal Blue")
Refund requested: $127.99
Customer stated: "I need this resolved today, I have an event tomorrow"
"""

# Bad summary (typical progressive summarization):
summary = "Customer had order issue and requested refund"
# LOST: customer name, order ID, exact amounts, exact dates, customer urgency

# Better: extract critical facts separately
case_facts = {
    "customer_id": "CHEN-4892",
    "customer_name": "Sarah Chen",
    "order_id": "ORD-20240892",
    "order_date": "2024-11-15",
    "issue": "Wrong color received: ordered Midnight Blue, received Royal Blue",
    "refund_amount": 127.99,
    "customer_urgency": "Needs resolution today (event tomorrow)",
    "session_start": "2024-03-22T14:30:00Z"
}
# case_facts is included verbatim in every prompt as structured JSON
# Never summarized — preserved exactly as stated
```

**Case Facts Block Pattern**

```python
def build_prompt_with_case_facts(case_facts, conversation_summary, latest_message):
    """
    Three-layer context structure:
    1. Case facts: exact numerical/date data (never summarized)
    2. Conversation summary: prose summary of what happened
    3. Current message: the latest user input
    """
    return f"""
## Case Facts (do not alter these)
{json.dumps(case_facts, indent=2)}

## Conversation Summary
{conversation_summary}

## Current Message
{latest_message}
"""

# When updating case facts — update, never summarize
def update_case_facts(case_facts, new_info):
    # New refund amount confirmed
    if "confirmed_refund" in new_info:
        case_facts["refund_confirmed"] = new_info["confirmed_refund"]
        case_facts["confirmation_timestamp"] = new_info["timestamp"]
    return case_facts
```

**Trimming Verbose Tool Outputs**

```python
# Full order lookup response: 40+ fields
raw_order = {
    "order_id": "ORD-20240892",
    "customer_id": "CHEN-4892",
    "status": "delivered",
    "created_at": "2024-11-15T10:23:45Z",
    "updated_at": "2024-11-20T16:00:00Z",
    "delivery_date": "2024-11-20",
    "items": [...],
    "shipping_address": {...},
    "billing_address": {...},
    "payment_method": {...},
    "warehouse_id": "WH-WEST-42",
    "carrier": "FedEx",
    "tracking_number": "793815649042",
    "shipping_cost": 0.00,
    "tax_amount": 10.72,
    "discount_applied": 0.00,
    "subtotal": 117.27,
    "total": 127.99,
    "fulfillment_center": "Phoenix",
    "packing_timestamp": "2024-11-17T08:15:00Z",
    # ... 20+ more warehouse/logistics fields
}

# Trim to return-relevant fields only
def trim_order_for_context(raw_order):
    return {
        "order_id": raw_order["order_id"],
        "status": raw_order["status"],
        "delivery_date": raw_order["delivery_date"],
        "total": raw_order["total"],
        "items": [{
            "sku": item["sku"],
            "description": item["description"],
            "color": item.get("color"),
            "quantity": item["quantity"],
            "price": item["price"]
        } for item in raw_order["items"]],
        "carrier": raw_order["carrier"],
        "tracking_number": raw_order["tracking_number"]
    }
    # 7 fields instead of 40+; same information for return processing
```

**Mitigating the "Lost in the Middle" Effect**

```python
def build_aggregated_research_prompt(agent_findings):
    """
    When aggregating multiple agent results, structure matters:
    - Key summaries at the BEGINNING (reliably attended)
    - Detailed results with section headers
    - Summary of gaps at the END (reliably attended)
    """
    key_findings = extract_key_findings(agent_findings)

    return f"""
## KEY FINDINGS SUMMARY (read this section first)
{key_findings}

## DETAILED RESULTS BY SOURCE

### Source 1: Web Research Agent
{agent_findings['web_search']}

### Source 2: Document Analysis Agent
{agent_findings['document_analysis']}

### Source 3: Database Query Results
{agent_findings['database']}

## COVERAGE GAPS AND LIMITATIONS
{summarize_gaps(agent_findings)}
"""
```

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

**The Three Escalation Triggers (with examples)**

```markdown
## Escalation Decision Guide

### Trigger 1: Explicit human request → ESCALATE IMMEDIATELY
Customer says: "I want to speak to a real person"
Customer says: "Let me talk to your supervisor"
Customer says: "Stop trying to help me with AI, I want a human"
Action: Escalate WITHOUT attempting investigation or resolution first
Reasoning: Honoring explicit requests builds trust; investigating first is paternalistic

### Trigger 2: Policy gap/exception → ESCALATE
Customer requests: "Price match to competitor's price" (policy only covers own-site)
Customer requests: "Retroactive discount for an order from 6 months ago" (policy: 30 days)
Customer requests: "Custom invoice format for tax filing" (not in standard options)
Action: Escalate with context (what was requested, why policy doesn't cover it)
Reasoning: Agents should not make policy exceptions; escalate to human with authority

### Trigger 3: Cannot make progress → ESCALATE
After 3+ attempts: Information system returns no records despite customer certainty
After attempts: Tool errors prevent access to required account data
Pattern: Each attempt produces the same unhelpful result
Action: Escalate with summary of what was tried and what information is missing
Reasoning: Continued failed attempts frustrate customers; escalation unblocks them

### NOT an escalation trigger:
Customer is frustrated (frustration ≠ complexity; offer resolution first)
Case seems complex (model's assessment of complexity is unreliable)
Agent is unsure (uncertainty ≠ escalation; attempt with stated uncertainty)
```

**Few-Shot Examples for Escalation Calibration**

```markdown
## System Prompt: Escalation Examples

Example 1 — Straightforward case with frustrated customer:
Customer: "I've been waiting 3 weeks for my order! This is ridiculous! I want this fixed!"
Action: RESOLVE (do not escalate based on frustration alone)
Reasoning: Order delay is within the agent's capability to investigate and resolve.
Response: "I understand this has been frustrating. Let me look into your order right now
and get this resolved for you."

Example 2 — Explicit escalation request:
Customer: "I've talked to two chatbots already. I need a real human."
Action: ESCALATE IMMEDIATELY
Reasoning: Explicit request for human agent; do not attempt resolution first.
Response: "I'll connect you with a team member right away."

Example 3 — Policy gap:
Customer: "Can you price-match the item I bought to the price at BestBuy?"
Action: ESCALATE
Reasoning: Our policy covers only our own price adjustments, not competitor price matching.
Response: "Price matching to other retailers requires supervisor approval. I'm connecting
you with a team member who can review this."

Example 4 — Multiple customer matches:
Tool returns: [Sarah Chen (CHEN-4892), Sarah Chen (CHEN-1127)]
Action: ASK FOR CLARIFICATION (not heuristic selection)
Response: "I found two accounts matching that name. Could you provide your email address
or account number so I can access the correct account?"
```

**Ambiguity Resolution: Multiple Matches**

```python
def handle_customer_lookup(results):
    if len(results) == 0:
        return {
            "action": "ask_for_more_info",
            "message": "I wasn't able to find an account with that name. "
                       "Could you provide your email address or order number?"
        }

    if len(results) == 1:
        return {
            "action": "proceed",
            "customer": results[0]
        }

    if len(results) > 1:
        # Request additional identifiers — NEVER select by heuristic
        # (e.g., "most recent account" or "first result")
        return {
            "action": "request_clarification",
            "message": (
                f"I found {len(results)} accounts matching that name. "
                "To access the correct account, could you please provide "
                "your email address, phone number, or order number?"
            )
        }
```

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

```python
# Bad error propagation — coordinator cannot recover intelligently
def bad_search_agent_error():
    return {
        "status": "error",
        "message": "Search unavailable"
    }
    # Coordinator has no idea: transient? permanent? retry? skip? escalate?

# Good error propagation — enables intelligent recovery
def good_search_agent_error(query, error_type, partial_results=None):
    error_info = {
        "success": False,
        "errorType": error_type,
        "attemptedQuery": query,
        "partialResults": partial_results or [],
        "partialResultsCount": len(partial_results) if partial_results else 0
    }

    if error_type == "timeout":
        error_info.update({
            "isRetryable": True,
            "retryAfterSeconds": 30,
            "coordinatorOptions": [
                "RETRY: Wait 30s and retry same query",
                "NARROW: Retry with date range narrowed to last 90 days",
                "SKIP: Use partial results (limited coverage)"
            ]
        })

    elif error_type == "rate_limit":
        error_info.update({
            "isRetryable": True,
            "retryAfterSeconds": 60,
            "coordinatorOptions": [
                "RETRY: Wait 60s and retry",
                "BATCH: Combine with other queries to reduce request count"
            ]
        })

    elif error_type == "source_unavailable":
        error_info.update({
            "isRetryable": False,
            "alternativeSources": ["database_B", "archive_search"],
            "coordinatorOptions": [
                "SUBSTITUTE: Query database_B for similar data",
                "PROCEED: Note gap in synthesis output"
            ]
        })

    return error_info
```

**Access Failure vs. Empty Result Distinction**

```python
def execute_search(query):
    try:
        results = search_engine.query(query)

        if results is None:
            # None from the API = access failure (shouldn't happen)
            return build_error("unexpected_null", query)

        # Empty list = valid result; NO matches found
        # This is NOT an error — the coordinator should know search worked
        return {
            "success": True,
            "results": results,                # Could be []
            "resultsFound": len(results),
            "queryExecuted": query,
            "isEmptyResult": len(results) == 0,  # Explicit flag for clarity
            "note": "Query executed successfully; no matching records found" if len(results) == 0 else None
        }

    except TimeoutError as e:
        return build_error("timeout", query, str(e))

    except ConnectionError as e:
        return build_error("connection_failure", query, str(e))

def build_error(error_type, query, detail=None):
    return {
        "success": False,
        "errorType": error_type,
        "attemptedQuery": query,
        "detail": detail,
        "isRetryable": error_type in ["timeout", "connection_failure", "rate_limit"]
    }
```

**Coverage Annotations in Synthesis Output**

```python
def synthesize_with_coverage(research_results):
    synthesis = {
        "findings": [],
        "coverageAnnotations": {}
    }

    for topic, data in research_results.items():
        if data["success"] and data["resultsFound"] > 0:
            synthesis["coverageAnnotations"][topic] = {
                "status": "well_supported",
                "sourceCount": len(data["sources"]),
                "confidence": "high"
            }
        elif data["success"] and data["resultsFound"] == 0:
            synthesis["coverageAnnotations"][topic] = {
                "status": "no_data_found",
                "note": "Query executed successfully; topic may not have coverage in this source",
                "confidence": "n/a"
            }
        else:
            synthesis["coverageAnnotations"][topic] = {
                "status": "access_failure",
                "errorType": data["errorType"],
                "note": f"Could not retrieve data: {data.get('detail', 'unknown error')}",
                "confidence": "unknown",
                "suggestion": "Retry or substitute alternative source"
            }

    return synthesis
```

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

```python
# Agent writes findings to scratchpad as exploration progresses
scratchpad_path = ".agent_workspace/exploration_notes.md"

def update_scratchpad(section, content):
    """Append a finding to the scratchpad"""
    with open(scratchpad_path, "a") as f:
        f.write(f"\n## {section}\n{content}\n")

# During exploration phase:
update_scratchpad(
    "Auth Module Dependencies",
    """
    - auth/middleware.ts: imports SessionStore, TokenValidator
    - auth/routes.ts: imports UserRepository, PasswordHasher
    - 12 service files import from auth/middleware.ts
    - Key concern: SessionStore is shared with WebSocket service
    """
)

update_scratchpad(
    "Refund Flow",
    """
    - payment/refund.ts → calls process_refund MCP tool
    - Requires: customer_id (verified), order_id, amount
    - Guard: RefundPolicy.check() before tool call
    - Maximum: $500 without supervisor approval (line 47)
    """
)

# On subsequent questions, reference the scratchpad:
def answer_question_with_scratchpad(question):
    notes = open(scratchpad_path).read()
    prompt = f"""
    Reference your exploration notes below before answering.
    Do not re-explore; use these notes as your source of truth.

    EXPLORATION NOTES:
    {notes}

    QUESTION: {question}
    """
    return call_claude(prompt)
```

**Subagent Delegation for Verbose Discovery**

```python
def explore_codebase_with_subagents(codebase_goal):
    """
    Main agent: coordinates, asks questions, builds understanding
    Subagents: do verbose file reading, searching, tracing — return summaries
    """

    # Phase 1: Spawn subagent for initial structure mapping
    structure_summary = spawn_explore_subagent(
        task="Map the overall codebase structure. Find entry points, main modules, "
             "and their responsibilities. Return a concise summary — do not include "
             "full file contents.",
        allowed_tools=["Read", "Grep", "Glob"]
    )
    # Verbose: reads 50+ files. Summary returned to main agent: ~500 tokens

    # Write summary to scratchpad before next phase
    update_scratchpad("Codebase Structure", structure_summary)

    # Phase 2: Spawn targeted subagent for specific investigation
    auth_summary = spawn_explore_subagent(
        task=f"Investigate the authentication module. Based on structure: {structure_summary}. "
             "Focus on: what does auth flow look like, what are the dependencies, "
             "where are the security boundaries?",
        allowed_tools=["Read", "Grep"]
    )

    update_scratchpad("Auth Module Detail", auth_summary)

    # Main agent context remains clean — saw only summaries, not 50+ file reads
```

**Crash Recovery with Manifests**

```python
import json
import os
from datetime import datetime

class AgentStateManifest:
    def __init__(self, manifest_path):
        self.path = manifest_path

    def save(self, agent_id, state):
        """Each agent exports its state before potential crash points"""
        manifest = self.load()
        manifest[agent_id] = {
            "state": state,
            "timestamp": datetime.now().isoformat(),
            "status": "in_progress"
        }
        with open(self.path, "w") as f:
            json.dump(manifest, f, indent=2)

    def mark_complete(self, agent_id, result_path):
        manifest = self.load()
        if agent_id in manifest:
            manifest[agent_id]["status"] = "complete"
            manifest[agent_id]["result_path"] = result_path
        with open(self.path, "w") as f:
            json.dump(manifest, f, indent=2)

    def load(self):
        if os.path.exists(self.path):
            with open(self.path) as f:
                return json.load(f)
        return {}

    def get_pending_agents(self):
        """On resume: which agents did not complete?"""
        manifest = self.load()
        return {
            agent_id: data
            for agent_id, data in manifest.items()
            if data.get("status") != "complete"
        }

# Usage in coordinator
manifest = AgentStateManifest(".agent_workspace/manifest.json")

# Agent saves state periodically
manifest.save("search_agent_1", {
    "completedQueries": ["AI trends 2024", "ML frameworks comparison"],
    "pendingQueries": ["GPT-5 analysis", "Claude architecture"],
    "partialResults": [...]
})

# On crash recovery:
def resume_coordinator():
    manifest = AgentStateManifest(".agent_workspace/manifest.json")
    pending = manifest.get_pending_agents()

    if pending:
        print(f"Resuming: {len(pending)} agents incomplete")
        for agent_id, state in pending.items():
            # Inject saved state into resumed agent
            resume_agent(agent_id, prior_state=state["state"])
    else:
        print("All agents complete; proceeding to synthesis")
```

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

```
Scenario: Invoice extraction system, 97% overall accuracy

Aggregate metric says: READY TO AUTOMATE

Reality by segment:
  Standard invoices: 99.5% accuracy ✓
  Multi-currency invoices: 94% accuracy — acceptable?
  Handwritten invoices: 71% accuracy ✗
  Non-English invoices: 68% accuracy ✗

Decision: The 97% aggregate masks that two document types
need human review or prompt improvement before automation.

Rule: Never reduce human review based on aggregate accuracy alone.
      Segment by: document type, field, complexity level.
```

**Stratified Random Sampling**

```python
import random

def setup_ongoing_sampling(extraction_results, sample_rate=0.05):
    """
    Stratified sampling: sample proportionally from each confidence stratum
    Purpose: catch errors in high-confidence extractions (most dangerous)
    """
    strata = {
        "high_confidence": [r for r in extraction_results if r["confidence"] == "high"],
        "medium_confidence": [r for r in extraction_results if r["confidence"] == "medium"],
        "low_confidence": [r for r in extraction_results if r["confidence"] == "low"]
    }

    samples = {}
    for stratum_name, stratum_results in strata.items():
        # Sample each stratum at the same rate
        sample_size = max(1, int(len(stratum_results) * sample_rate))
        samples[stratum_name] = random.sample(stratum_results, min(sample_size, len(stratum_results)))

    return samples

def analyze_error_rates(reviewed_samples):
    """
    After human reviewers check the sample:
    Calculate error rate per stratum, per field, per document type
    """
    analysis = {}
    for stratum, reviewed in reviewed_samples.items():
        errors = [r for r in reviewed if r["human_judgment"] != r["model_output"]]
        analysis[stratum] = {
            "error_rate": len(errors) / len(reviewed) if reviewed else 0,
            "sample_size": len(reviewed),
            "novel_patterns": [e for e in errors if e.get("is_novel_pattern")]
        }

    return analysis
```

**Field-Level Confidence Calibration**

```python
# Step 1: Model outputs field-level confidence scores
extraction_schema_with_confidence = {
    "type": "object",
    "properties": {
        "invoice_number": {"type": "string"},
        "invoice_number_confidence": {
            "type": "string",
            "enum": ["high", "medium", "low"],
            "description": "high=clearly stated, medium=inferred, low=ambiguous/imprecise"
        },
        "total_amount": {"type": ["number", "null"]},
        "total_amount_confidence": {
            "type": "string",
            "enum": ["high", "medium", "low"]
        },
        "payment_terms": {"type": "string"},
        "payment_terms_confidence": {
            "type": "string",
            "enum": ["high", "medium", "low"]
        }
    }
}

# Step 2: Calibrate thresholds using labeled validation set
def calibrate_confidence_thresholds(validation_set):
    """
    validation_set: [{model_confidence, model_output, human_correct_output}]
    Find confidence threshold where model_output == human_correct_output >= target%
    """
    results = {"high": [], "medium": [], "low": []}

    for item in validation_set:
        correct = item["model_output"] == item["human_correct_output"]
        results[item["model_confidence"]].append(correct)

    accuracy_by_confidence = {
        level: sum(correct_list) / len(correct_list)
        for level, correct_list in results.items()
        if correct_list
    }

    # Determine routing threshold
    # e.g., if high confidence = 98% accurate → automate
    #        if medium confidence = 87% accurate → review
    #        if low confidence = 61% accurate → always review
    return accuracy_by_confidence

# Step 3: Route based on calibrated confidence
def route_extraction(extraction, thresholds):
    for field, value in extraction.items():
        if field.endswith("_confidence"):
            base_field = field.replace("_confidence", "")
            confidence = value
            if confidence == "low" or thresholds.get(confidence, 0) < 0.95:
                return "human_review", f"Low confidence on {base_field}"

    # Also route ambiguous source documents
    if extraction.get("conflict_detected"):
        return "human_review", "Conflicting values in source document"

    return "automate", None
```

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

```python
# Subagent output — claim-source mappings preserved
def search_agent_output():
    return {
        "agentId": "web_search_agent",
        "queryDate": "2026-03-22",
        "claims": [
            {
                "claim": "Global AI market size reached $500B in 2025",
                "sources": [
                    {
                        "url": "https://example-research.com/ai-market-2025",
                        "title": "AI Market Report 2025",
                        "publishDate": "2025-11-01",
                        "excerpt": "The global AI market reached $500 billion in revenue during 2025...",
                        "credibilityScore": 0.92
                    }
                ],
                "confidence": "high",
                "isContested": False
            },
            {
                "claim": "AI will replace 30% of jobs by 2030",
                "sources": [
                    {
                        "url": "https://source-A.com/ai-jobs",
                        "publishDate": "2024-03-15",
                        "excerpt": "AI automation will displace approximately 30% of current jobs...",
                        "credibilityScore": 0.85
                    },
                    {
                        "url": "https://source-B.com/ai-jobs-analysis",
                        "publishDate": "2025-01-10",
                        "excerpt": "Contrary to previous estimates, AI job displacement will be closer to 12%...",
                        "credibilityScore": 0.88
                    }
                ],
                "confidence": "low",
                "isContested": True,
                "contestDescription": "Source A (2024) says 30%; Source B (2025) says 12%"
            }
        ]
    }
```

**Synthesis Agent: Preserving Provenance Through Merging**

```python
def synthesize_with_provenance(agent_outputs):
    """
    Synthesis must NOT compress claims into bare statements.
    Each claim in output must trace to its sources.
    """
    synthesized_claims = []

    for agent_output in agent_outputs:
        for claim in agent_output["claims"]:
            # Check if this claim conflicts with existing claims
            existing = find_similar_claim(synthesized_claims, claim["claim"])

            if existing and existing["sources"][0]["publishDate"] != claim["sources"][0]["publishDate"]:
                # Different dates — this may be temporal, not contradiction
                mark_as_temporal_difference(existing, claim)
            elif existing and existing.get("isContested"):
                # Merge additional conflicting source
                existing["sources"].extend(claim["sources"])
                existing["contestDescription"] += f"; Also: {claim['contestDescription']}"
            else:
                synthesized_claims.append(claim)

    return synthesized_claims

def build_report_with_sections(claims):
    """
    Structure report: well-established vs. contested findings
    Different content types: different rendering
    """

    well_established = [c for c in claims if not c.get("isContested") and c["confidence"] == "high"]
    contested = [c for c in claims if c.get("isContested")]
    uncertain = [c for c in claims if c["confidence"] in ["medium", "low"] and not c.get("isContested")]

    report_sections = {
        "wellEstablishedFindings": format_as_table_or_list(well_established),
        "contestedFindings": format_contested_section(contested),
        "uncertainFindings": format_uncertain_section(uncertain),
        "methodologicalNotes": extract_methodological_context(claims)
    }

    return report_sections
```

**Handling Conflicting Statistics**

```python
def handle_conflicting_statistics(claim, source_a, source_b):
    """
    Rule: Never arbitrarily select one value over another.
    Always: annotate with both values and source attribution.
    """

    # Check if conflict is temporal (different measurement dates)
    date_a = source_a.get("dataCollectionDate") or source_a.get("publishDate")
    date_b = source_b.get("dataCollectionDate") or source_b.get("publishDate")

    if date_a and date_b and date_a != date_b:
        return {
            "claim": claim,
            "resolution": "temporal_difference",
            "valueA": {"value": source_a["value"], "date": date_a, "source": source_a["title"]},
            "valueB": {"value": source_b["value"], "date": date_b, "source": source_b["title"]},
            "note": "Values differ by measurement date, not contradiction"
        }

    # Genuine conflict — annotate both, let coordinator/human decide
    return {
        "claim": claim,
        "resolution": "genuine_conflict",
        "valueA": {"value": source_a["value"], "source": source_a["title"]},
        "valueB": {"value": source_b["value"], "source": source_b["title"]},
        "note": "Credible sources disagree; present both values with attribution",
        "requiresCoordinatorDecision": True
    }
```

**Temporal Data Requirements**

```python
# Require subagents to always include dates
subagent_output_schema = {
    "properties": {
        "statistics": {
            "type": "array",
            "items": {
                "properties": {
                    "value": {"type": "number"},
                    "unit": {"type": "string"},
                    "dataCollectionDate": {
                        "type": ["string", "null"],
                        "description": "Date data was collected (not published). ISO 8601. REQUIRED for statistics."
                    },
                    "publicationDate": {
                        "type": "string",
                        "description": "When the source was published. ISO 8601. REQUIRED."
                    },
                    "source": {"type": "string"}
                },
                "required": ["value", "publicationDate", "source"]
            }
        }
    }
}

# Without dates: "AI market was $200B (Source A) vs $500B (Source B) — contradiction?"
# With dates: "$200B in 2022 (Source A, published 2022) vs $500B in 2025 (Source B, published 2025)"
# → Not a contradiction; temporal difference. Market grew from $200B to $500B.
```

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
