# Domain 4: Prompt Engineering & Structured Output
**Exam Weight: 20% | Task Statements: 4.1 – 4.6**

---

## Task Statement 4.1: Design prompts with explicit criteria to improve precision and reduce false positives

### What You Need to Know
- **Explicit criteria** over vague instructions: "flag comments only when claimed behavior contradicts actual code behavior" vs. "check that comments are accurate"
- General hedging instructions ("be conservative", "only report high-confidence findings") do NOT improve precision
- **False positive rates**: high false positive categories undermine trust in accurate categories — one bad category can cause developers to ignore all findings

### What You Need to Be Able to Do
- Write specific review criteria defining which issues to report (bugs, security) vs. skip (minor style, local patterns)
- Temporarily disable high false-positive categories to restore developer trust while improving prompts for those categories
- Define explicit severity criteria with concrete code examples for each severity level for consistent classification

### Deep Dive

**From Vague to Explicit Criteria**

The core insight: vague instructions make the model apply its own interpretation of what counts as an issue. This creates inconsistency. Explicit criteria leave no room for interpretation.

Vague examples that fail: `"flag any issues you find with high confidence"`, `"be conservative"`, `"check comments for accuracy"`.

```markdown
# Explicit (consistent, auditable):
"Flag a comment as inaccurate ONLY when:
  - The comment claims the function does X, but reading the code shows it does Y
  - The comment claims a variable holds data in format A, but the code assigns format B
  - The comment says a call is synchronous, but the code shows it is async

Do NOT flag:
  - Comments that are incomplete but not contradictory
  - Comments that use informal language or abbreviations
  - Missing comments on self-documenting code
  - Outdated comments that are merely imprecise (not contradictory)"
```

**Severity Classification with Code Examples**

```markdown
Classify findings into exactly three severity levels:

CRITICAL — report immediately, blocks merge:
  - `query = f"SELECT * FROM users WHERE id = {user_input}"`
    (SQL injection; unsanitized user input in query string)

HIGH — report, should fix before merge:
  - API call without try/except where network failure crashes app

MEDIUM — report as advisory, can fix later:
  - N+1 query in loop (performance degradation at scale)
```

SKIP (not worth reporting): minor style issues covered by a linter, short-but-clear variable names, style choices not in team standards. These categories should be omitted from the prompt entirely rather than listed as low-priority, to avoid the model producing low-value findings that erode trust.

**Managing False Positive Trust Erosion**

When one category (e.g., "performance") produces 40% false positives, developers begin ignoring ALL findings — including legitimate security issues. Trust erosion spreads from the noisy category to accurate ones.

The three-step recovery strategy: (1) temporarily disable the noisy category entirely so developers regain trust in the accurate categories; (2) refine the disabled category's criteria with concrete examples and test on a subset; (3) re-enable only once the false positive rate is acceptable (target < 10% before restoring to CI).

> **Exam Tip:** Questions about reducing false positives always have two candidates: (A) add explicit categorical criteria, and (B) add hedging language like "be conservative" or "only report high-confidence findings." The answer is always (A). Hedging language doesn't work — it shifts interpretation to the model without improving its calibration.

> **Anti-Pattern Alert:** "Report only findings you're 90%+ confident about" sounds reasonable but fails because the model's self-assessed confidence does not correlate with actual correctness. Confidence is poorly calibrated (also relevant to Task 5.2 on escalation).

> **Cross-Domain Connection:** Task 4.2 covers few-shot examples — when explicit categorical criteria aren't sufficient alone, adding 2–3 few-shot examples of ambiguous cases with explanations is the next-level refinement.

---

## Task Statement 4.2: Apply few-shot prompting to improve output consistency and quality

### What You Need to Know
- Few-shot examples are the **most effective technique** for consistently formatted output when detailed instructions alone produce inconsistency
- Few-shot examples demonstrate **ambiguous-case handling** (e.g., which tool to select for ambiguous queries, what counts as a branch-level coverage gap)
- Few-shot examples enable the model to **generalize judgment** to novel patterns — not just match pre-specified cases
- Few-shot examples are effective for **reducing hallucination** in extraction tasks (informal measurements, varied document structures)

### What You Need to Be Able to Do
- Create 2–4 targeted few-shot examples for ambiguous scenarios that show reasoning for why one action was chosen over plausible alternatives
- Include few-shot examples demonstrating specific desired output format (location, issue, severity, suggested fix)
- Provide examples distinguishing acceptable code patterns from genuine issues (reduces false positives while enabling generalization)
- Use few-shot examples to demonstrate correct handling of varied document structures (inline citations vs. bibliographies, methodology sections vs. embedded details)
- Add few-shot examples for correct extraction from documents with varied formats (empty/null extraction of required fields)

### Deep Dive

**Structure of Effective Few-Shot Examples**

```markdown
# Few-shot examples for code review findings

## Output format:
[SEVERITY] file:line — Issue description. Suggested fix.

## Example 1 — Security finding (report):
Code: `cursor.execute("SELECT * FROM orders WHERE user_id = " + user_id)`
Finding: [CRITICAL] payment.py:47 — SQL injection via string concatenation.
  Use parameterized query: cursor.execute("SELECT * FROM orders WHERE user_id = %s", (user_id,))

## Example 2 — Local pattern (skip):
Code: `x = get_user(id)  # get user by id`
Finding: SKIP — Redundant comment, but not contradictory or misleading.
  Self-documenting code doesn't require comment; omitting doesn't improve anything.
```

A third ambiguous-case example is valuable when false positives cluster around a specific pattern (e.g., async functions without try/catch that are HIGH not MEDIUM because of how the framework handles unhandled rejections). Add it only when you have a real ambiguous case to resolve.

**Few-Shot for Ambiguous Tool Selection**

```markdown
# System prompt few-shot examples for agent tool selection

## Example 1 — No order ID:
Query: "I placed an order last Tuesday"
Tool: get_customer
Reasoning: No order ID available; must retrieve customer first, then search
  their orders by date. Start with get_customer.

## Example 2 — Order ID present but customer unverified:
Query: "Can you check order 12345 for John Smith?"
Tool: get_customer
Reasoning: Despite the order ID being present, customer identity is unverified.
  Always verify customer identity via get_customer before accessing order data.
```

The key insight these two examples teach: even when an order ID is available, the correct first step is identity verification via `get_customer`. The examples together rule out the plausible-but-wrong shortcut of jumping straight to `lookup_order`.

**Few-Shot for Extraction from Varied Document Structures**

Provide one example for each of the four key document conditions the model will encounter:

- **(a) Dedicated section** — data is clearly labelled and present: extract all fields normally.
- **(b) Embedded data** — data is present but not in a labelled section (e.g., buried in the introduction): extract what is stated; leave unprovided fields as null.
- **(c) Absent data** — the document does not mention the field at all: return null. Do NOT infer or fabricate from context.
- **(d) Imprecise data** — field is mentioned but cannot be quantified (e.g., "a handful of interviews", "several years"): return null for the numeric field. "Handful" is not a number; fabricating one is hallucination.

The overriding principle demonstrated by all four examples: **never fabricate**. If the document does not clearly state the value, null is always the correct answer.

> **Exam Tip:** When a question describes "inconsistent output format" or "incorrect tool selection on ambiguous cases" and asks for the most effective fix — the answer is few-shot examples. Not more detailed instructions, not confidence thresholds, not routing classifiers.

> **Anti-Pattern Alert:** Using 8–10 few-shot examples for simple, unambiguous cases is unnecessary overhead. 2–4 targeted examples for the ambiguous cases is sufficient. More examples don't always help and add token cost.

> **Cross-Domain Connection:** Task 3.5 (input/output examples for iterative refinement) uses the same example-based technique — concrete examples to constrain model behavior where prose descriptions fail.

---

## Task Statement 4.3: Enforce structured output using tool use and JSON schemas

### What You Need to Know
- **`tool_use` with JSON schemas** is the most reliable approach for guaranteed schema-compliant structured output — eliminates JSON syntax errors
- `tool_choice: "auto"` — model may return text instead of calling a tool
- `tool_choice: "any"` — model must call a tool but can choose which
- Forced tool selection — model must call a specific named tool
- Strict JSON schemas via tool use eliminate **syntax errors** but do NOT prevent **semantic errors** (e.g., line items don't sum to total, values in wrong fields)
- Schema design: required vs. optional fields; `enum` with `"other"` + detail string for extensible categories; nullable fields to prevent hallucination

### What You Need to Be Able to Do
- Define extraction tools with JSON schemas and extract structured data from `tool_use` responses
- Set `tool_choice: "any"` to guarantee structured output when multiple extraction schemas exist
- Force a specific tool with `tool_choice: {"type": "tool", "name": "..."}` to ensure a particular extraction runs before enrichment
- Design schema fields as optional (nullable) when source documents may not contain the information
- Add `"unclear"` enum values for ambiguous cases and `"other"` + detail fields for extensible categorization
- Include format normalization rules in prompts alongside strict output schemas

### Deep Dive

**tool_choice Decision Table**

| Scenario | `tool_choice` | Why |
|---|---|---|
| Agent may need to respond conversationally | `"auto"` | Allows text fallback; model may skip the tool entirely |
| Multiple extraction schemas, unknown document type | `"any"` | Forces a tool call; model picks which schema fits |
| Metadata must be extracted before enrichment | `{"type": "tool", "name": "extract_metadata"}` | Guarantees ordering; specific tool runs first |
| Single schema, guaranteed structured output | `"any"` or forced | Either works; forced is marginally more explicit |

The distinction between `"auto"` and `"any"` is the most common exam trap. `"auto"` means the model may return plain text — use it only when a conversational fallback is acceptable. `"any"` guarantees a tool is called but leaves the choice of tool to the model — use it when you have multiple schemas and do not know which applies.

**Tool Use for Structured Extraction**

The schema below illustrates the three key design patterns in a single tool definition: a required string field, a nullable field (data may be absent), and an enum-with-other field (extensible category). In production the `required` list and `tool_choice` setting enforce that the model always calls the tool and always returns a schema-compliant object.

```python
extract_invoice_tool = {
    "name": "extract_invoice",
    "description": "Extract structured data from an invoice document",
    "input_schema": {
        "type": "object",
        "properties": {
            "invoice_number": {"type": "string"},                          # required
            "invoice_date":   {"type": ["string", "null"],                 # nullable
                               "description": "ISO 8601 or null if absent"},
            "total_amount":   {"type": ["number", "null"]},                # nullable
            "payment_terms":  {"type": "string",                           # enum with other
                               "enum": ["net_30", "net_60", "net_90", "immediate", "other"]},
            "payment_terms_detail": {"type": ["string", "null"],
                               "description": "Original text when payment_terms='other'"},
            "confidence":     {"type": "string", "enum": ["high", "medium", "low"]}
        },
        "required": ["invoice_number", "payment_terms", "confidence"]
    }
}

# Forced tool use — model cannot return plain text
tool_choice = {"type": "tool", "name": "extract_invoice"}
```

**Nullable Fields to Prevent Hallucination**

When a JSON schema field is `required` but the source document may not contain that information, the model is forced to fabricate a value — it has no other valid option. Making the field nullable with `["string", "null"]` gives the model an honest "not found" option. This is the primary anti-hallucination technique for structured extraction.

A field typed `"type": "string"` forces the model to provide a value — so when the field is absent from the source document, the model fabricates a plausible-sounding one. Typing it `"type": ["string", "null"]` lets the model honestly report absence with `null` instead of inventing data. Apply this pattern to every field that may not appear in every document.

**Enum with "other" Pattern**

Without `"other"`, a closed enum forces the model to pick the closest category for unanticipated values — producing systematic misclassifications.

```python
# Before: "R&D" or "Customer Success" → forced wrong pick
"enum": ["engineering", "marketing", "sales", "hr", "finance"]

# After: captures original value without misclassifying
"enum": ["engineering", "marketing", "sales", "hr", "finance", "other"]
"department_detail": {"type": ["string", "null"]}  # "R&D"; null if not "other"
```

**Syntax vs. Semantic Errors**

`tool_use` eliminates JSON syntax errors — bad braces, missing quotes, wrong types, invalid JSON structure. The model cannot return a response that violates the schema.

`tool_use` does NOT eliminate semantic errors — line items that do not sum to the stated total, dates placed in wrong fields, a discount rate exceeding 100%. Semantic errors require separate validation logic applied after extraction. These two categories are tested as distinct concepts: schema enforces structure; your code enforces meaning.

Examples of semantic errors that require post-extraction validation:

- Line items do not sum to the stated total
- Invoice date is implausibly far in the future
- A discount rate exceeds 100%
- A required reference number is structurally valid but does not match the expected format for this vendor
- The same field contains contradictory values sourced from different parts of the document

> **Exam Tip:** "What prevents JSON syntax errors?" → `tool_use` with JSON schema. "What prevents semantic errors (totals don't add up)?" → additional validation logic (Task 4.4). These are distinct questions with distinct answers.

> **Anti-Pattern Alert:** Setting required fields for data that may not be in the source document forces the model to fabricate values. Always make fields nullable when the source may not contain them.

> **Cross-Domain Connection:** Task 4.4 covers validation and retry loops — tool use eliminates syntax errors but semantic errors caught by validation trigger the retry-with-feedback loop described there.

---

## Task Statement 4.4: Implement validation, retry, and feedback loops for extraction quality

### What You Need to Know
- **Retry-with-error-feedback**: append specific validation errors to the prompt on retry to guide correction
- Retries are ineffective when the required information is **simply absent** from the source document (vs. format or structural errors)
- **Feedback loop design**: `detected_pattern` field tracks which code constructs trigger findings — enables systematic dismissal pattern analysis
- **Semantic vs. syntax**: tool use eliminates syntax errors; semantic errors (values don't sum, wrong field placement) require validation logic

### What You Need to Be Able to Do
- Implement follow-up requests including original document, failed extraction, and specific validation errors for model self-correction
- Identify when retries will be ineffective (information exists only in an external document not provided) vs. when they will succeed (format mismatches, structural output errors)
- Add `detected_pattern` fields to structured findings for false positive pattern analysis
- Design self-correction validation flows: extract `calculated_total` alongside `stated_total` to flag discrepancies; add `conflict_detected` booleans for inconsistent source data

### Deep Dive

**Retry-with-Error-Feedback Pattern**

The retry-with-feedback loop follows five steps:

1. **Extract** — send document with forced tool use; receive structured output.
2. **Validate** — run semantic validation checks against the extracted data.
3. **Return on success** — if no errors, return the result immediately.
4. **Build retry message on failure** — append to the conversation: the assistant's previous tool call, then a user message containing (a) the specific validation errors, (b) the original document, and (c) the previous extraction. Do NOT simply say "try again."
5. **Repeat up to max_retries** — after exhausting retries, return the best available result with errors flagged for human review.

**Retry Decision Logic**

The core question before retrying: is the required data actually present in the document?

**When retry WILL help** — the data exists but the extraction went wrong:
- Format mismatches (date present but in wrong format; model just needs format guidance)
- Arithmetic errors (line items are in the document; model miscalculated the sum)
- Field placement errors (value is in the document but placed in wrong field)

**When retry is WASTEFUL** — the data does not exist and cannot be produced by re-reading:
- Information simply absent from the document (field not mentioned anywhere)
- Document type mismatch (not the expected document type; retrying the same document won't change what it contains)
- Validation requires external data not provided (purchase order not included in the request)

The retry message must include: specific validation errors describing exactly what failed + the original document + the previous failed extraction. "Please try again" or "be more careful" does not help — it gives the model no new information to act on.

**When to Retry vs. When to Stop**

| Error type | Retry? | Reason |
|---|---|---|
| Date in wrong format | Yes | Data is present; format guidance will fix it |
| Line items don't sum to stated total | Yes | Arithmetic/placement error; model can recalculate |
| Required field absent from document | No | Data does not exist; retry cannot invent it |
| Document is wrong type (not an invoice) | No | Type mismatch; retrying same document won't help |
| Validation requires an external document not provided | No | Missing context cannot be supplied via retry |
| Generic / unknown error | Yes | Retry may help; include specific error text |

**detected_pattern for Feedback Loop Analysis**

The `detected_pattern` field in a code review tool schema captures the specific code construct that triggered each finding (e.g., `"string concatenation in SQL"`, `"bare except clause"`, `"mutable default argument"`). It is required on every finding.

The feedback loop works as follows: when a developer dismisses a finding, the system logs the `detected_pattern` alongside the dismissal reason. Over time, aggregating dismissals by pattern reveals which constructs have high false positive rates. Patterns with consistently high dismissal rates are candidates for prompt refinement — either by adding an explicit SKIP rule for that pattern, or by adding a few-shot example showing why it is acceptable code. This turns individual developer dismissals into systematic prompt improvements.

**Self-Correction Validation Fields**

The schema can include fields that make the model perform validation work at extraction time, reducing the complexity of post-extraction checks:

- `stated_total` — the total amount as literally written in the document (nullable)
- `calculated_total` — the model's own sum of line items; computed by the model, not copied from the document (nullable)
- `totals_match` — boolean; `true` if `stated_total == calculated_total` within $0.01; the discrepancy is surfaced before your code runs any checks
- `conflict_detected` — boolean; `true` if the document contains contradictory values for the same field (e.g., different totals on different pages)
- `conflict_description` — string describing the conflict when `conflict_detected` is `true`; null otherwise

When `conflict_detected` is true, route directly to human review rather than retrying — the source document is ambiguous, not the extraction.

> **Exam Tip:** Retry questions come in two flavors: (1) "What to include in a retry prompt?" — the answer is specific validation errors + original document + failed extraction. (2) "When is retry ineffective?" — the answer is when the required data does not exist in the provided document.

> **Anti-Pattern Alert:** Retrying with "please try again" or "be more careful" does not improve outcomes. The retry must include specific, actionable error messages describing exactly what failed and why.

> **Cross-Domain Connection:** Task 4.3 covers schema design — semantic validation errors caught here feed back into schema improvements (adding `conflict_detected`, `calculated_total` fields to make validation easier).

---

## Task Statement 4.5: Design efficient batch processing strategies

### What You Need to Know
- **Message Batches API**: 50% cost savings, up to 24-hour processing window, no guaranteed latency SLA
- Appropriate for: non-blocking, latency-tolerant workloads (overnight reports, weekly audits, nightly test generation)
- Inappropriate for: blocking workflows (pre-merge checks that must complete before merge is allowed)
- **Batch API does NOT support multi-turn tool calling** within a single request
- `custom_id` fields for correlating batch request/response pairs

### What You Need to Be Able to Do
- Match API approach to workflow latency requirements: synchronous for blocking pre-merge checks, batch for overnight/weekly analysis
- Calculate batch submission frequency based on SLA constraints (e.g., 4-hour windows to guarantee 30-hour SLA with 24-hour batch processing)
- Handle batch failures: resubmit only failed documents (identified by `custom_id`) with appropriate modifications
- Use prompt refinement on a sample set before batch-processing large volumes

### Deep Dive

**Batch API Use Case Decision**

The key question is not "is this a large workload?" — it is: does this workflow block something waiting for the result?

| Workflow | Blocks? | Correct API | Why |
|---|---|---|---|
| Pre-merge code review | YES — developer waits | Synchronous | 24h window is unacceptable |
| Real-time customer support | YES — customer waits | Synchronous | Response must be immediate |
| Interactive extraction during user session | YES — user waits | Synchronous | Latency directly visible |
| Nightly test generation | NO — runs overnight | Batch | Non-blocking; 50% savings |
| Weekly compliance audit | NO — report due next day | Batch | Non-blocking; tolerates hours |
| Monthly re-extraction of historical records | NO — no urgency | Batch | Non-blocking; 50% savings |

Key facts to memorize: **50% cost savings**, **up to 24-hour processing**, **no guaranteed latency SLA**, **no multi-turn tool calling** within a single batch request, **`custom_id`** is the only way to correlate responses to source documents (responses are not returned in submission order).

**Batch API Request Structure**

Each request in the batch is a standard Messages API params object plus a `custom_id` string you supply. The `custom_id` is the only way to correlate a response back to the source document, since batch responses are not guaranteed to arrive in submission order.

Submit the batch as a single API call (`batches.create`). Poll `batches.retrieve` until `processing_status == "ended"` — up to 24 hours. When ended, iterate `batches.results` and match each result back to its source document via `custom_id`. Results with `result.type == "succeeded"` contain the full message; failures contain an error type used to decide resubmission strategy.

**Handling Batch Failures**

When a batch completes, resubmit only the failed documents (identified by `custom_id`) — never the whole batch. The action depends on the error type:

| Error type | Action |
|---|---|
| `overloaded_error` | Resubmit same request unchanged; transient capacity issue |
| Context window exceeded | Chunk the document and resubmit as multiple smaller requests |
| `succeeded` | Extract tool result and store; no resubmission needed |
| Other / unknown | Log for human review; resubmission unlikely to help without prompt changes |

**SLA Calculation for Batch Submission Frequency**

```
Scenario: Must guarantee all documents processed within 30-hour SLA
  Batch API processes in up to 24 hours

Calculation:
  Available window before SLA breach: 30 hours
  Maximum batch processing time: 24 hours
  Latest submission time = 30 - 24 = 6 hours after document arrives

  If documents arrive continuously:
  Submit batches every 4-6 hours (allowing buffer)
  Any document in a batch completes within 24h
  Submitted within 6h of arrival → completes within 30h

  Submission cadence: every 4 hours (conservative buffer for SLA safety)
```

**Prompt Refinement Before Bulk Processing**

Always test a new prompt on a stratified sample of 30–50 documents (covering diverse document types and quality levels) before submitting a large batch. A 50-document synchronous sample costs 0.5% of the full 10,000-document run and catches prompt failures before they propagate at scale. Iterate until extraction quality is acceptable; only then submit the bulk batch.

> **Exam Tip:** The critical batch API facts for the exam: (1) 50% cost savings, (2) up to 24-hour window, (3) no guaranteed latency SLA, (4) no multi-turn tool calling support. Questions about batch use case fit always turn on whether the workflow blocks something (synchronous needed) or runs offline (batch appropriate).

> **Anti-Pattern Alert:** Using batch API for pre-merge code reviews is wrong — the developer is blocked waiting for the result. The 24-hour window is unacceptable for anything blocking a human workflow.

> **Cross-Domain Connection:** Task 4.4 handles batch failures via resubmission — the `custom_id` field is what enables targeted resubmission of only the failed documents rather than the full batch.

---

## Task Statement 4.6: Design multi-instance and multi-pass review architectures

### What You Need to Know
- **Self-review limitations**: the model retains reasoning context from generation, making it less likely to question its own decisions in the same session
- **Independent review instances** (without prior reasoning context) are more effective than self-review instructions or extended thinking
- **Multi-pass review**: per-file local analysis passes + cross-file integration passes avoids attention dilution and contradictory findings

### What You Need to Be Able to Do
- Use a second independent Claude instance to review generated code without the generator's reasoning context
- Split large multi-file reviews into focused per-file passes for local issues + separate integration passes for cross-file data flow analysis
- Run verification passes where the model self-reports confidence alongside each finding for calibrated review routing

### Deep Dive

**Why Self-Review Fails**

The generating session retains its reasoning context — it already decided its code was correct, made specific design choices, and resolved specific ambiguities in a particular way. Asking it to self-review in the same session is asking it to question decisions it just made with full confidence. It cannot approach its own output as a stranger. An independent review instance starts fresh with no generation bias: it sees only the code, has no memory of why choices were made, and reviews as if encountering the code for the first time.

This is the precise reason "add a self-review instruction" is always the wrong answer on the exam when an independent instance is an option.

**Independent Review Instance Pattern**

The pattern uses two completely separate API calls — two distinct sessions with no shared message history:

Session A (generator) receives the specification and produces code. Its system prompt frames it as a software engineer writing production code. When this call completes, only the generated code text is extracted and passed forward — not the conversation, not the spec, not any reasoning.

Session B (reviewer) receives only the generated code, with a system prompt framing it as a senior reviewer with no knowledge of how the code was written. Because it has no access to the generation reasoning, it cannot unconsciously confirm the generator's assumptions. It reviews the code as a stranger would.

The key implementation detail: Session B's `messages` array contains only the code-under-review, never the original generation prompt or the generator's message history. This is what makes the review genuinely independent.

**Multi-Pass Review Architecture**

See Domain 1 Task 1.6 for the full CI/CD integration context. Key architectural points:

- **Pass 1 — per-file local analysis**: each file is reviewed in its own API call, scoped explicitly to local issues (logic errors, null checks, security within the file). The prompt instructs the model NOT to flag cross-file concerns. This prevents attention dilution across large PRs.
- **Pass 2 — cross-file integration analysis**: a single subsequent call receives the per-file finding summaries plus the full diff. It focuses exclusively on integration concerns: data flow across module boundaries, inconsistent error handling across services, breaking changes to shared interfaces, circular dependencies.
- The two-pass split is essential when reviewing 20+ files. Reviewing all files in one call causes attention dilution — findings in middle files are less reliable than findings in the first and last files.

**Confidence-Based Review Routing**

Each finding includes `severity` (`critical/high/medium/low`), `confidence` (`high/medium/low`, where high = clearly wrong, low = may be wrong), and an optional `requires_domain_knowledge` boolean. Routing decisions:

| Severity | Confidence | requires_domain_knowledge | Action |
|---|---|---|---|
| critical | any | any | block_merge |
| high | high | false | auto_comment |
| high | medium | false | auto_comment |
| any | low | any | human_review |
| any | any | true | human_review |
| medium/low | medium/high | false | auto_comment |

> **Exam Tip:** "Why is self-review less effective than independent review?" — The generating session retains reasoning context that biases toward confirming its own decisions. The exam will present self-review ("add a self-review instruction") as a wrong answer when independent instance review is an option.

> **Anti-Pattern Alert:** Reviewing 20+ files in one pass causes attention dilution — findings in the middle files are less reliable. Always split large reviews into per-file passes first, then a cross-file integration pass.

> **Cross-Domain Connection:** Task 3.6 covers CI/CD integration — independent review instances are implemented as separate Claude Code invocations in the CI pipeline, not within the same session.

---

## Key Terminology Quick Reference

| Term | Definition | Exam Relevance |
|------|-----------|----------------|
| Explicit criteria | Specific, categorical rules for what to flag/skip | Reduces false positives; "be conservative" doesn't work |
| False positive rate | % of flagged issues that are actually acceptable | High rates erode trust in accurate categories |
| Few-shot examples | Demonstrated input/output pairs in prompt | Most effective for output consistency on ambiguous cases |
| `tool_use` | Structured output via tool schema | Eliminates syntax errors; most reliable structured output |
| `tool_choice: "auto"` | Model can call tool or return text | Default behavior; allows conversational fallback |
| `tool_choice: "any"` | Model must call some tool | Guarantees tool is called; model picks which one |
| Forced tool selection | `{"type": "tool", "name": "X"}` | Model must call this specific tool |
| Nullable field | `"type": ["string", "null"]` | Prevents hallucination when field may be absent |
| Enum with "other" | Category enum + detail string field | Handles extensible/unanticipated categories |
| Semantic error | Value logically wrong (totals don't sum) | NOT eliminated by tool use; requires validation logic |
| Retry-with-feedback | Retry including original doc + specific error | More effective than "try again"; targeted correction |
| `detected_pattern` | Field tracking which code construct triggered finding | Enables false positive pattern analysis |
| Message Batches API | Async bulk processing, 50% savings, up to 24h | Non-blocking workloads only |
| `custom_id` | Batch request correlation field | Maps responses back to source documents |
| No multi-turn in batch | Batch API cannot execute tool calls mid-request | Batch = single-turn requests only |
| Self-review limitation | Generator session retains reasoning context | Use independent instance for review |
| Multi-pass review | Per-file pass + cross-file integration pass | Prevents attention dilution on large reviews |
| Attention dilution | Quality drops when reviewing too many files at once | Mitigated by multi-pass architecture |

---

## Decision Matrix

### Which tool_choice setting?

| Scenario | Setting | Key Reason |
|----------|---------|------------|
| Model may need to answer conversationally | `"auto"` | Text fallback is acceptable |
| Multiple extraction schemas, unknown document type | `"any"` | Forces a tool call; model picks which schema fits |
| Specific extraction must run first (ordered pipeline) | `{"type": "tool", "name": "X"}` | Guarantees ordering |
| One extraction schema, must use it | `{"type": "tool", "name": "X"}` | Explicit; `"any"` also works |

### Synchronous API vs. Batch API?

| Criterion | Synchronous | Batch |
|-----------|-------------|-------|
| Latency requirement | Immediate/real-time | Tolerates hours |
| Blocks human workflow | Yes | Never |
| Cost priority | Normal rate | 50% savings |
| Multi-turn tool calling | Supported | NOT supported |
| Examples | Pre-merge reviews, real-time chat | Nightly audits, weekly reports |

### What fixes which quality problem?

| Problem | Fix |
|---------|-----|
| Inconsistent output format | Few-shot examples showing correct format |
| Wrong tool selection for ambiguous queries | Few-shot examples showing tool selection reasoning |
| Hallucinated field values when data is absent | Nullable schema fields |
| Misclassified niche category | Enum "other" + detail field |
| JSON syntax errors | `tool_use` with JSON schema |
| Totals don't sum to line items | Semantic validation + retry-with-feedback |
| High false positive rate | Explicit categorical criteria (not "be conservative") |
| Inconsistent severity classification | Severity criteria with concrete code examples |
| Reviewer misses issues in own generated code | Independent review instance |

---

## If You're Short on Time

1. **Explicit categorical criteria beat hedging language** — "flag only when X" outperforms "be conservative." Hedging shifts interpretation to the model without improving calibration.
2. **`tool_use` with JSON schema eliminates syntax errors, not semantic errors** — totals that don't sum require validation logic, not just a schema.
3. **Nullable fields (`["string", "null"]`) prevent hallucination** — when data may be absent from the source, the field must be nullable or the model will fabricate a value.
4. **Batch API = 50% savings, up to 24h, no multi-turn tool calling** — use only for non-blocking workflows; pre-merge checks require synchronous API.
5. **Independent review instances beat self-review** — the generating session retains reasoning context that biases toward confirming its own decisions.
