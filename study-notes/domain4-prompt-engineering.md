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

```markdown
# Vague (produces false positives and inconsistency):
"Review the code and flag any issues you find with high confidence."
"Be conservative — only report things that are definitely problems."
"Check comments for accuracy."

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
  - Code example: `query = f"SELECT * FROM users WHERE id = {user_input}"`
    (SQL injection; unsanitized user input in query string)
  - Code example: `password = "admin123"` (hardcoded credential)

HIGH — report, should fix before merge:
  - Code example: API call without try/except where network failure crashes app
  - Code example: Integer overflow possible in payment calculation

MEDIUM — report as advisory, can fix later:
  - Code example: N+1 query in loop (performance degradation at scale)
  - Code example: Missing input validation (not security-critical path)

SKIP entirely (not worth reporting):
  - Inconsistent spacing within files that use a linter
  - Variable names that are shorter than preferred but clear
  - Minor style choices not covered by team standards
```

**Managing False Positive Trust Erosion**

```
Scenario: Code review tool reports 40% false positives in "performance" category
  → Developers start ignoring ALL findings including legitimate security issues
  → Trust erosion spreads from bad category to accurate categories

Strategy:
  Step 1: Temporarily disable the performance category entirely
          → Developers regain trust in security findings
  Step 2: Refine performance criteria with concrete examples
          → Test on subset before re-enabling
  Step 3: Re-enable with improved criteria and validate false positive rate
          → Target < 10% false positives before restoring to CI
```

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

## Output format to use:
[SEVERITY] file:line — Issue description. Suggested fix.

## Examples:

### Example 1 — Security finding (report):
Code: `cursor.execute("SELECT * FROM orders WHERE user_id = " + user_id)`
Finding: [CRITICAL] payment.py:47 — SQL injection via string concatenation.
  Use parameterized query: cursor.execute("SELECT * FROM orders WHERE user_id = %s", (user_id,))

### Example 2 — Local pattern (skip):
Code: `x = get_user(id)  # get user by id`
Finding: SKIP — Redundant comment, but not contradictory or misleading.
  Self-documenting code doesn't require comment; omitting doesn't improve anything.

### Example 3 — Ambiguous case (reasoning shown):
Code:
  async function fetchUser(id) {
    const user = await db.query(...)
    return user  // note: no try/catch
  }
Finding: [HIGH] userService.js:23 — Unhandled promise rejection. DB failure propagates uncaught.
  Reasoning: async function without try/catch is HIGH not MEDIUM because unhandled rejections
  in this framework silently crash the process, not just fail the request.
  Fix: Wrap in try/catch and return structured error response.
```

**Few-Shot for Ambiguous Tool Selection**

```markdown
# System prompt few-shot examples for agent tool selection

## Examples:

Query: "What happened to order ORD-12345?"
Tool: lookup_order
Reasoning: The query contains a specific order identifier — use the tool
  designed for order lookups, not customer account lookup.

Query: "I need help with my account"
Tool: get_customer
Reasoning: No order ID mentioned; start with customer lookup to understand
  account status before proceeding.

Query: "I placed an order last Tuesday"
Tool: get_customer
Reasoning: No order ID available; must retrieve customer first, then
  search their orders by date. Start with get_customer.

Query: "Can you check order 12345 for John Smith?"
Tool: get_customer
Reasoning: Despite the order ID being present, customer identity is unverified.
  Always verify customer identity via get_customer before accessing order data.
```

**Few-Shot for Extraction from Varied Document Structures**

```markdown
# Few-shot examples for extracting methodology from research papers

## Example 1 — Methodology in dedicated section:
Document excerpt: "## Methodology\nWe surveyed 500 participants over 6 months..."
Extraction: {"methodology": "Survey-based", "sample_size": 500, "duration_months": 6}

## Example 2 — Methodology embedded in introduction:
Document excerpt: "...this analysis draws on 3 years of transaction logs from..."
Extraction: {"methodology": "Longitudinal data analysis", "sample_size": null, "duration_months": 36}

## Example 3 — No methodology mentioned:
Document excerpt: "This report summarizes the quarterly findings from our team..."
Extraction: {"methodology": null, "sample_size": null, "duration_months": null}
Note: Return null, do NOT infer or fabricate methodology from context.

## Example 4 — Informal measurement:
Document excerpt: "...based on a handful of customer interviews..."
Extraction: {"methodology": "Qualitative interviews", "sample_size": null, "duration_months": null}
Note: "handful" is imprecise; do not fabricate a number. Return null for sample_size.
```

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

**Tool Use for Structured Extraction**

```python
import anthropic
import json

client = anthropic.Anthropic()

# Define extraction tool with JSON schema
extract_invoice_tool = {
    "name": "extract_invoice",
    "description": "Extract structured data from an invoice document",
    "input_schema": {
        "type": "object",
        "properties": {
            "invoice_number": {
                "type": "string",
                "description": "Invoice identifier (e.g., INV-2026-0042)"
            },
            "vendor_name": {
                "type": "string"
            },
            "invoice_date": {
                "type": ["string", "null"],    # nullable — may be absent
                "description": "ISO 8601 date (YYYY-MM-DD) or null if not found"
            },
            "total_amount": {
                "type": ["number", "null"],
                "description": "Total invoice amount in USD, or null if ambiguous"
            },
            "line_items": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "description": {"type": "string"},
                        "quantity": {"type": ["number", "null"]},
                        "unit_price": {"type": ["number", "null"]},
                        "total": {"type": "number"}
                    },
                    "required": ["description", "total"]
                }
            },
            "payment_terms": {
                "type": "string",
                "enum": ["net_30", "net_60", "net_90", "immediate", "other"],
                "description": "Payment terms; use 'other' if non-standard"
            },
            "payment_terms_detail": {
                "type": ["string", "null"],
                "description": "Detail when payment_terms is 'other'; null otherwise"
            },
            "confidence": {
                "type": "string",
                "enum": ["high", "medium", "low"],
                "description": "Extraction confidence based on document clarity"
            }
        },
        "required": ["invoice_number", "vendor_name", "line_items", "payment_terms", "confidence"]
    }
}

# Force tool use to guarantee structured output
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=2048,
    tools=[extract_invoice_tool],
    tool_choice={"type": "tool", "name": "extract_invoice"},  # forced
    messages=[{
        "role": "user",
        "content": f"Extract invoice data. Normalize all dates to ISO 8601 (YYYY-MM-DD).\n\n{document_text}"
    }]
)

# Extract tool result
for block in response.content:
    if block.type == "tool_use":
        invoice_data = block.input
        break
```

**Nullable Fields to Prevent Hallucination**

```python
# Without nullable fields: model MUST provide a value
# → Model fabricates plausible-sounding data when field is absent
"invoice_date": {"type": "string"}  # BAD: forces hallucination

# With nullable fields: model can honestly report absence
"invoice_date": {
    "type": ["string", "null"],
    "description": "ISO 8601 date or null if not clearly stated in document"
}

# Same pattern for any field that may be absent:
"serial_number": {"type": ["string", "null"]}
"discount_rate": {"type": ["number", "null"]}
"purchase_order_reference": {"type": ["string", "null"]}
```

**Enum with "other" Pattern**

```python
# Without "other": forces the model to pick the closest category
# → Misclassification of genuinely different cases
"department": {
    "type": "string",
    "enum": ["engineering", "marketing", "sales", "hr", "finance"]
    # What if the document says "R&D" or "Customer Success"? → forced wrong pick
}

# With "other" + detail: handles extensibility gracefully
"department": {
    "type": "string",
    "enum": ["engineering", "marketing", "sales", "hr", "finance", "other"]
},
"department_detail": {
    "type": ["string", "null"],
    "description": "Original department name when 'other'; null otherwise"
}
# "R&D" → department: "other", department_detail: "R&D"
```

**Syntax vs. Semantic Errors**

```python
# Tool use eliminates: syntax errors
# - Unclosed braces
# - Missing quotes
# - Invalid JSON
# - Wrong field types

# Tool use does NOT eliminate: semantic errors
# These require additional validation logic

def validate_invoice(data):
    errors = []

    # Semantic check 1: line items must sum to total
    if data.get("total_amount") and data.get("line_items"):
        calculated = sum(item["total"] for item in data["line_items"])
        if abs(calculated - data["total_amount"]) > 0.01:
            errors.append(f"Line items sum {calculated} != stated total {data['total_amount']}")

    # Semantic check 2: date must be in the past or near-present
    if data.get("invoice_date"):
        invoice_date = datetime.fromisoformat(data["invoice_date"])
        if invoice_date > datetime.now() + timedelta(days=30):
            errors.append(f"Invoice date {data['invoice_date']} is implausibly far in the future")

    return errors
```

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

```python
import anthropic
import json

client = anthropic.Anthropic()

def extract_with_retry(document_text, tool, max_retries=2):
    messages = [{
        "role": "user",
        "content": f"Extract invoice data from this document:\n\n{document_text}"
    }]

    for attempt in range(max_retries + 1):
        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=2048,
            tools=[tool],
            tool_choice={"type": "tool", "name": tool["name"]},
            messages=messages
        )

        # Extract result
        extracted = None
        for block in response.content:
            if block.type == "tool_use":
                extracted = block.input
                break

        if not extracted:
            break

        # Validate
        errors = validate_invoice(extracted)
        if not errors:
            return {"success": True, "data": extracted, "attempts": attempt + 1}

        if attempt < max_retries:
            # Retry with specific error feedback — NOT just "try again"
            messages.append({"role": "assistant", "content": response.content})
            messages.append({
                "role": "user",
                "content": (
                    f"The extraction has validation errors. Please correct:\n\n"
                    f"Errors:\n" + "\n".join(f"- {e}" for e in errors) + "\n\n"
                    f"Original document:\n{document_text}\n\n"
                    f"Previous extraction:\n{json.dumps(extracted, indent=2)}\n\n"
                    f"Resubmit corrected extraction."
                )
            })

    return {"success": False, "data": extracted, "errors": errors, "attempts": attempt + 1}
```

**When to Retry vs. When to Stop**

```python
def should_retry(validation_error, document_text):
    """
    Retry is useful: format/structural errors where data IS present but malformed
    Retry is wasteful: data simply doesn't exist in the document

    Returns: (should_retry: bool, reason: str)
    """

    # Retry will help: data present but wrong format
    if "date format" in validation_error:
        # Data is there, format is wrong — retry with format guidance
        return True, "Format error; data likely present in different format"

    if "line items don't sum to total" in validation_error:
        # Arithmetic error or wrong field placement — retry to recalculate
        return True, "Semantic arithmetic error; retry to recalculate"

    # Retry won't help: data is genuinely absent
    if "required field" in validation_error and field_mentioned_in_doc(validation_error, document_text):
        return False, "Field mentioned in document exists but cannot be parsed"

    if "vendor_name" in validation_error and "invoice" not in document_text.lower():
        # Not actually an invoice
        return False, "Document type mismatch; this is not an invoice"

    if validation_error.startswith("external_reference"):
        # Error requires data from another document
        return False, "Validation requires external document not provided"

    return True, "Generic error; retry may help"
```

**detected_pattern for Feedback Loop Analysis**

```python
# Code review tool structure with detected_pattern field
code_review_tool = {
    "name": "report_finding",
    "input_schema": {
        "type": "object",
        "properties": {
            "file": {"type": "string"},
            "line": {"type": "integer"},
            "severity": {"type": "string", "enum": ["critical", "high", "medium", "low"]},
            "category": {"type": "string"},
            "description": {"type": "string"},
            "detected_pattern": {
                "type": "string",
                "description": (
                    "The specific code construct that triggered this finding. "
                    "Example: 'string concatenation in SQL', 'bare except clause', "
                    "'mutable default argument'. Used for false positive analysis."
                )
            },
            "suggested_fix": {"type": "string"}
        },
        "required": ["file", "line", "severity", "category", "description", "detected_pattern"]
    }
}

# Developer dismisses finding → log detected_pattern
def on_developer_dismiss(finding_id, reason):
    finding = get_finding(finding_id)
    log_dismissal({
        "detected_pattern": finding["detected_pattern"],
        "category": finding["category"],
        "reason": reason,
        "timestamp": datetime.now()
    })

# Analysis: which patterns have high dismissal rates?
def analyze_false_positive_patterns():
    dismissals = load_dismissal_log()
    pattern_dismissal_rates = {}
    for d in dismissals:
        p = d["detected_pattern"]
        pattern_dismissal_rates[p] = pattern_dismissal_rates.get(p, 0) + 1

    # High dismissal rate → update prompt to skip this pattern
    # or add few-shot examples showing why it's acceptable
    return sorted(pattern_dismissal_rates.items(), key=lambda x: -x[1])
```

**Self-Correction Validation Fields**

```python
# Adding explicit validation-helper fields to schema
extraction_schema = {
    "properties": {
        "stated_total": {
            "type": ["number", "null"],
            "description": "Total amount as explicitly stated in document"
        },
        "calculated_total": {
            "type": ["number", "null"],
            "description": "Sum of all line items (model calculates this)"
        },
        "totals_match": {
            "type": "boolean",
            "description": "True if stated_total == calculated_total (within $0.01)"
        },
        "conflict_detected": {
            "type": "boolean",
            "description": "True if document contains contradictory values for same field"
        },
        "conflict_description": {
            "type": ["string", "null"],
            "description": "Describe conflict if conflict_detected is true"
        }
    }
}

# Validation using these self-reported fields
def validate_with_self_reported(data):
    if data.get("conflict_detected"):
        route_to_human_review(data, reason=data["conflict_description"])

    if not data.get("totals_match") and data.get("stated_total") is not None:
        # Retry with specific arithmetic feedback
        retry_with_feedback(data, "Line items sum does not match stated total")
```

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

```
Question: Does this workflow BLOCK something waiting for the result?

YES → Use synchronous API
  - Pre-merge code reviews (developer waiting for merge permission)
  - Real-time customer support responses
  - Interactive extraction during user sessions

NO → Consider batch API (50% savings, up to 24h)
  - Nightly test generation for new code committed that day
  - Weekly compliance audit of documentation
  - Monthly batch re-extraction of historical records
  - Overnight bulk document processing
```

**Batch API Request Structure**

```python
import anthropic

client = anthropic.Anthropic()

# Build batch requests
def build_batch_requests(documents):
    return [
        {
            "custom_id": f"doc-{doc['id']}",  # For correlation on response
            "params": {
                "model": "claude-opus-4-6",
                "max_tokens": 2048,
                "tools": [extract_invoice_tool],
                "tool_choice": {"type": "tool", "name": "extract_invoice"},
                "messages": [{
                    "role": "user",
                    "content": f"Extract invoice data:\n\n{doc['text']}"
                }]
            }
        }
        for doc in documents
    ]

# Submit batch
batch = client.beta.messages.batches.create(
    requests=build_batch_requests(documents)
)
batch_id = batch.id

# Poll for completion (batch takes up to 24 hours)
import time
while True:
    status = client.beta.messages.batches.retrieve(batch_id)
    if status.processing_status == "ended":
        break
    time.sleep(3600)  # Check hourly; no SLA guarantee

# Process results — correlate via custom_id
results = {}
for result in client.beta.messages.batches.results(batch_id):
    doc_id = result.custom_id  # Matches our "doc-{id}"
    if result.result.type == "succeeded":
        results[doc_id] = result.result.message
    else:
        # Track failures for resubmission
        handle_failure(doc_id, result.result)
```

**Handling Batch Failures**

```python
def process_batch_results(batch_id):
    successes = []
    failures = []

    for result in client.beta.messages.batches.results(batch_id):
        if result.result.type == "succeeded":
            successes.append({
                "custom_id": result.custom_id,
                "data": extract_tool_result(result.result.message)
            })
        else:
            failures.append({
                "custom_id": result.custom_id,
                "error_type": result.result.type,
                "error": result.result.error if hasattr(result.result, 'error') else None
            })

    # Resubmit only failures — not the whole batch
    if failures:
        retry_requests = []
        for failure in failures:
            doc = get_document_by_custom_id(failure["custom_id"])

            if failure.get("error_type") == "overloaded_error":
                # Transient: resubmit same request
                retry_requests.append(build_single_request(doc))

            elif "context_window" in str(failure.get("error", "")):
                # Too long: chunk the document
                chunks = chunk_document(doc, max_tokens=50000)
                retry_requests.extend(build_chunked_requests(doc["id"], chunks))

        if retry_requests:
            retry_batch = client.beta.messages.batches.create(requests=retry_requests)

    return successes, failures
```

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

```python
# Never send 10,000 documents through a new prompt without testing
def validate_prompt_before_bulk(sample_documents, prompt_template, n_samples=50):
    """
    Test on a stratified sample before bulk processing:
    1. Diverse document types and quality levels
    2. Manual review of results
    3. Iteration until acceptable quality

    Reason: Iterative resubmission of 10k documents is expensive.
    A 50-document sample catches prompt issues at 0.5% of full cost.
    """
    sample = random.sample(sample_documents, min(n_samples, len(sample_documents)))
    results = []
    for doc in sample:
        result = extract_synchronously(doc, prompt_template)
        results.append(result)

    # Manual review
    accuracy = manual_review(results)
    if accuracy < 0.90:
        raise ValueError(f"Prompt accuracy {accuracy:.1%} insufficient for bulk; refine first")

    return results
```

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

**Independent Review Instance Pattern**

```python
import anthropic

client = anthropic.Anthropic()

def generate_and_review_independently(spec: str):
    # Step 1: Generate code (Session A)
    generation_response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=4096,
        system="You are an expert software engineer. Generate clean, production-ready code.",
        messages=[{"role": "user", "content": f"Implement: {spec}"}]
    )
    generated_code = generation_response.content[0].text

    # Step 2: Independent review (Session B — fresh context, no generation history)
    # Key: the reviewer does NOT see the generation prompt or reasoning
    review_response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=2048,
        system=(
            "You are a senior code reviewer. You have no context about how this code was written. "
            "Review it objectively for correctness, security, and edge cases."
        ),
        messages=[{
            "role": "user",
            "content": (
                f"Review this code for bugs, security issues, and missing edge cases.\n"
                f"For each finding, include your confidence level (high/medium/low).\n\n"
                f"Code:\n{generated_code}"
            )
        }]
    )

    return generated_code, review_response.content[0].text
```

**Multi-Pass Review Architecture**

```python
def multi_pass_code_review(pr_files: list[dict]):
    """
    Pass 1: Per-file local analysis
    Pass 2: Cross-file integration analysis

    Avoids attention dilution when reviewing 20+ files simultaneously.
    """

    # Pass 1: Per-file reviews — local issues only
    per_file_findings = []
    for file in pr_files:
        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=1024,
            messages=[{
                "role": "user",
                "content": (
                    f"Review {file['name']} for LOCAL issues only:\n"
                    f"- Logic errors within this file\n"
                    f"- Missing null checks\n"
                    f"- Security issues in this file\n"
                    f"Do NOT flag cross-file integration issues.\n\n"
                    f"Code:\n{file['content']}"
                )
            }]
        )
        per_file_findings.append({
            "file": file["name"],
            "findings": response.content[0].text
        })

    # Pass 2: Cross-file integration analysis
    # Provide per-file summaries + full changed code for integration review
    integration_context = "\n\n".join([
        f"File: {f['file']}\n{f['findings']}" for f in per_file_findings
    ])

    integration_response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": (
                f"Review these cross-file integration concerns:\n"
                f"- Data flow across module boundaries\n"
                f"- Inconsistent error handling across services\n"
                f"- Breaking changes to shared interfaces\n"
                f"- Circular dependencies introduced\n\n"
                f"Per-file findings summary:\n{integration_context}\n\n"
                f"Full diff:\n{get_full_diff(pr_files)}"
            )
        }]
    )

    return {
        "per_file": per_file_findings,
        "integration": integration_response.content[0].text
    }
```

**Confidence-Based Review Routing**

```python
review_with_confidence_tool = {
    "name": "report_finding",
    "input_schema": {
        "type": "object",
        "properties": {
            "description": {"type": "string"},
            "severity": {"type": "string", "enum": ["critical", "high", "medium", "low"]},
            "confidence": {
                "type": "string",
                "enum": ["high", "medium", "low"],
                "description": (
                    "High: clearly wrong. Medium: likely wrong, context dependent. "
                    "Low: may be wrong, needs human judgment."
                )
            },
            "requires_domain_knowledge": {"type": "boolean"}
        },
        "required": ["description", "severity", "confidence"]
    }
}

def route_finding(finding):
    """Route findings based on confidence and severity"""
    if finding["severity"] == "critical":
        return "block_merge"  # Always block regardless of confidence

    if finding["confidence"] == "high" and finding["severity"] in ["critical", "high"]:
        return "auto_comment"  # High confidence high/critical: post automatically

    if finding["confidence"] == "low" or finding.get("requires_domain_knowledge"):
        return "human_review"  # Uncertain: send to human reviewer

    return "auto_comment"  # Medium/high confidence medium/low: auto-comment
```

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

| Scenario | Setting |
|----------|---------|
| Model may need to answer conversationally | `"auto"` |
| Multiple extraction schemas, unknown document type | `"any"` |
| Specific extraction must run first (ordered pipeline) | `{"type": "tool", "name": "X"}` |
| One extraction schema, must use it | `{"type": "tool", "name": "X"}` |

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
