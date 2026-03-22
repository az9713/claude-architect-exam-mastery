# Domain 5 Cheat Sheet: Context Management & Reliability
**Weight: 15% | Task Statements: 5.1 – 5.6**

---

## Key Concepts

| Concept | Definition | Exam Relevance |
|---------|-----------|----------------|
| Progressive summarization risk | Condensing numerical values into vague prose | Destroys precision; use case facts block instead |
| Case facts block | Structured JSON of exact transactional data | Preserved verbatim across turns; NEVER summarized |
| Lost in the middle | Model attends less to middle-of-input content | Place key findings at beginning; use section headers |
| Token accumulation | Tool outputs consuming context tokens | Trim to relevant fields only before adding to context |
| Escalation trigger | Condition requiring human handoff | Explicit request / policy gap / inability to progress |
| Sentiment-based escalation | Escalating frustrated customers | Unreliable; frustration does NOT equal complexity |
| Self-reported confidence | Model's stated certainty level | Poorly calibrated; do NOT use for escalation routing |
| Structured error context | Error with type, query, partial results, options | Enables intelligent coordinator recovery |
| Access failure | Tool unavailable (timeout, connection error) | Retryable; DISTINCT from empty results |
| Empty result | Query succeeded; zero matching records | NOT an error; explicitly flag `isEmptyResult: true` |
| Scratchpad file | Persistent file recording agent findings | Counteracts context degradation across long sessions |
| Context degradation | Model references "typical patterns" not specifics | Key signal: findings from early session are lost |
| State manifest | JSON with each agent's exported state | Enables crash recovery; coordinator loads on resume |
| `/compact` | Claude Code command to reduce context | Use during extended exploration sessions |
| Stratified random sampling | Sample proportionally from each confidence stratum | Catches errors in high-confidence extractions |
| Aggregate accuracy masking | Overall metric hiding segment-level failures | Always segment before automating |
| Field-level confidence | Per-field certainty score | Calibrated with labeled validation set; routes to review |
| Claim-source mapping | Structured link between claim and its source | Must be preserved through synthesis, not summarized away |
| Temporal difference | Values differ due to measurement date | Distinguish from genuine contradiction; always require dates |
| Conflict annotation | Flag contradictory values with both sources | Never silently select; pass conflict to coordinator |

---

## Decision Matrix

### When to escalate?

| Trigger | Action |
|---------|--------|
| Customer explicitly requests human | Escalate IMMEDIATELY — no investigation first |
| Policy gap / exception needed | Escalate with context |
| Cannot make progress after attempts | Escalate with summary of what was tried |
| Customer is frustrated / angry | Offer resolution first; escalate only if reiterated |
| Case seems complex | Attempt resolution (complexity estimate is unreliable) |
| Low model confidence | Attempt with caveats (confidence ≠ escalation) |

### Which context management approach?

| Problem | Solution |
|---------|----------|
| Exact amounts/dates lost across long sessions | Case facts block (preserved verbatim) |
| Tool output consuming too many tokens | Trim to relevant fields before adding to context |
| Agent referencing "typical patterns" vs. specifics | Scratchpad file or subagent delegation |
| Verbose discovery filling main context | Explore subagent (returns summary only) |
| Context fills during extended session | `/compact` command |
| Crash mid-workflow | State manifest; coordinator loads on resume |
| Findings in middle of long input are missed | Key summaries at beginning; section headers |

### How to handle conflicting sources?

| Situation | Approach |
|-----------|----------|
| Source A: X (2022), Source B: Y (2025) | Temporal difference; not contradiction |
| Source A: X, Source B: Y (same date) | Annotate both; flag for coordinator decision |
| Contradictory values in same document | `conflict_detected: true` + description |
| Missing date makes conflict ambiguous | Require dates in subagent output schema |

---

## Code Reference

```python
# Case facts block — preserved verbatim, NEVER summarized
case_facts = {
    "customer_id": "CHEN-4892",
    "order_id": "ORD-20240892",
    "refund_amount": 127.99,          # Never summarize this to "over $100"
    "customer_urgency": "Needs resolution today (event tomorrow)"
}

# Prompt structure: case facts + summary + current message
prompt = f"""
## Case Facts (preserve exactly)
{json.dumps(case_facts)}

## Conversation Summary
{summary}

## Current Message
{latest_message}
"""

# Empty result is NOT an error
return {
    "success": True,
    "results": [],           # [] = successful search with no matches
    "isEmptyResult": True,   # Explicit flag for coordinator
    "note": "Query executed; no matching records found"
}
# vs. access failure:
return {"success": False, "errorType": "timeout", "isRetryable": True}

# Claim-source mapping — preserved through synthesis
{
    "claim": "AI market reached $500B in 2025",
    "sources": [{"url": "...", "publishDate": "2025-11-01", "excerpt": "..."}],
    "isContested": False
}

# Conflict annotation — never silently select
{
    "claim": "AI job displacement rate",
    "isContested": True,
    "valueA": {"value": "30%", "source": "Study A (2024)"},
    "valueB": {"value": "12%", "source": "Study B (2025)"},
    "requiresCoordinatorDecision": True
}
```

---

## Anti-Patterns (Common Wrong Answers)

- **Summarizing numerical values**: progressive summarization destroys "$127.99" → "over $100"
- **Sentiment-based escalation**: frustrated customer ≠ complex case; offer resolution first
- **Self-reported confidence for escalation**: poorly calibrated; model is wrong on hard cases with high confidence
- **Reporting empty results as errors**: `[]` from successful search is valid; don't mark it `isError: true`
- **Generic `"search unavailable"` errors**: hides failure type from coordinator; always include `errorType` + options
- **Silently suppressing subagent errors**: returning empty results instead of failure context hides critical information
- **Terminating entire workflow on single subagent failure**: propagate structured error; other agents may succeed
- **Automating based on aggregate accuracy (97%)**: may mask 68% accuracy on specific document types
- **Synthesis agent choosing between conflicting values**: pass conflict up; don't resolve unilaterally
- **No dates in subagent output**: temporal differences mistaken for contradictions

---

## If You Remember Nothing Else

1. **Case facts block**: exact amounts, dates, order IDs preserved verbatim outside summarized history — progressive summarization destroys numerical precision
2. **Three escalation triggers**: explicit human request / policy gap / inability to progress — sentiment and self-reported confidence are unreliable
3. **Structured error propagation**: `errorType` + `attemptedQuery` + `partialResults` + recovery options — generic status hides recovery information
4. **Scratchpad files + subagent delegation** combat context degradation — verbose discovery in main session pushes early findings out of attention
5. **Conflicting sources**: annotate both with attribution and dates; check if dates explain the difference; never silently select one value
