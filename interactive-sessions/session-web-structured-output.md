# Interactive Session: Structured Output and Extraction

**Surface:** Claude.ai (web)
**Duration:** 45–60 minutes
**Task Statements:** 4.3, 4.4, 4.5

---

## Learning Objectives

By the end of this session, you will be able to:
- Understand why tool_use with JSON schemas is more reliable than prompt-based JSON output
- Recognize the difference between syntax errors (eliminated by schema) and semantic errors (not caught by schema)
- Design nullable fields to prevent value fabrication
- Understand validation-retry loop design
- Apply the `"other"` + detail pattern for extensible categorization

---

## Prerequisites

- Claude.ai account
- Basic familiarity with JSON structure
- No coding required — this session uses prose prompts with structured output demonstrations

---

## Session Steps

### Step 1 — Demonstrate the Fragility of Prompt-Based JSON

**Prompt:**
```
Extract the following information from this invoice and return it as JSON:
- vendor_name
- invoice_date
- total_amount
- line_items (array)

INVOICE:
TechSupply Co.
Invoice #INV-2024-891
Date: March 15, 2024

Item: Cloud Storage License (12 months) ........... $1,200.00
Item: Support Package .............................. $300.00
Subtotal: $1,500.00
Tax (8%): $120.00
Total Due: $1,620.00

Payment terms: Net 30
```

**Observe:** Did Claude add extra fields beyond what was requested? Is the JSON syntactically valid? Did it include prose before or after the JSON? Is the date format consistent?

**Checkpoint 1:** How would a downstream system parsing this output need to handle the variability you observed?

---

### Step 2 — Schema Design: Required vs Optional Fields

**Prompt:**
```
I'm designing a JSON schema for invoice extraction. Help me think through which fields should be required versus optional.

For each field, tell me:
1. Should it be required (must appear in valid output) or optional/nullable (can be null)?
2. What happens if I make it required when it may be absent from some invoices?

Fields to evaluate:
- vendor_name (some invoices just say "From: [address]" with no company name)
- invoice_date (some invoices only have a "payment due" date)
- total_amount (all invoices have this)
- po_number (purchase order — often absent from vendor-generated invoices)
- tax_rate (some jurisdictions don't add tax)
- payment_terms (e.g., "Net 30" — not always present)
```

**Observe:** Claude should explain that required fields for information that may be absent pressure the model to fabricate values.

**Checkpoint 2:** What is the risk of making `po_number` required? What should happen when the model encounters an invoice without a PO number?

---

### Step 3 — The "other" + Detail Pattern

**Prompt:**
```
I'm building an invoice categorization system. Invoices belong to one of these categories:
Software License, Hardware, Professional Services, Maintenance, Travel, Other.

The problem: some invoices are unusual (e.g., "annual conference booth fee," "legal settlement payment").
If I force the model to pick from only my 5 standard categories, it will miscategorize unusual invoices.

Design a schema field for `invoice_category` that:
1. Gives me reliable categorical values for standard cases (enabling filtering and reporting)
2. Handles unusual cases honestly without forcing a wrong categorization
3. Captures the detail for unusual cases so humans can review them

Show me the JSON schema definition for this field.
```

**Observe:** Claude should propose the `enum` with `"other"` plus a `category_detail` field pattern.

**Checkpoint 3:** Why is a free-text field for `invoice_category` worse than the `"other"` + detail pattern? What downstream capabilities does categorical data enable that free-text doesn't?

---

### Step 4 — Syntax Errors vs Semantic Errors

**Prompt:**
```
I'm using strict JSON schema validation for invoice extraction. My schema requires:
- invoice_total: number
- line_items: array of objects with {description, amount}
- line_items_sum (calculated): number

My validation caught these two types of errors today. Explain which ones the JSON schema validation would catch, and which ones it would NOT catch:

Error Type A:
The extracted JSON was missing the closing brace:
{"invoice_total": 1620, "line_items": [{"description": "License", "amount": 1200}

Error Type B:
The extracted JSON was valid but:
- invoice_total: 1620
- line_items: [{amount: 1200}, {amount: 300}]
- line_items_sum: 1500  (but 1200 + 300 = 1500, while invoice_total is 1620 — the $120 tax is in the total but not in line items)
```

**Observe:** JSON schema validation catches Error Type A (syntax). Error Type B (semantic mismatch) is syntactically valid — schema validation passes, but the values are logically inconsistent.

**Checkpoint 4:** What additional validation logic would you need to catch Error Type B? Is a retry going to help with Error Type B?

---

### Step 5 — Retry-With-Error-Feedback Design

**Prompt:**
```
I extracted data from an invoice and the extraction has a semantic validation error. I need to send a retry request.

Original document excerpt:
"Line items: Support Package ($300.00), Cloud Storage License ($1,200.00). Total: $1,620.00 (includes $120.00 tax)"

Failed extraction:
{
  "vendor": "TechSupply Co",
  "line_items": [{"description": "Support Package", "amount": 300}, {"description": "Cloud Storage", "amount": 1200}],
  "subtotal": 1500,
  "total": 1620
}

Validation error: line_items sum ($1,500) does not match total ($1,620). Discrepancy: $120.

Write the follow-up prompt I should send to guide the model toward correcting this specific error.
Include: the original document, the failed extraction, and the specific validation error with the discrepancy amount.
```

**Observe:** Claude should write a prompt that includes all three elements: original document, failed extraction, and specific error message with the discrepancy.

**Checkpoint 5:** Why is the specific error message ($120 discrepancy) important in the retry prompt? What would a vague "please fix the extraction" achieve?

---

### Step 6 — Identifying When Retries Are Futile

**Prompt:**
```
I'm running a retry loop for invoice extraction. For each scenario, tell me whether a retry will likely succeed or fail, and why:

Scenario A:
The invoice is a PDF scan. The extraction produced dates in inconsistent formats (some ISO 8601, some US format "Jan 15, 2024"). Validation failed because my schema requires ISO 8601.

Scenario B:
The contract references "Exhibit A for pricing details" but Exhibit A is a separate document that was not provided. The extraction returned null for contract_value. I retried with the same document asking it to extract the contract value.

Scenario C:
The model extracted the subtotal as the invoice_total, missing the tax. My schema validates that total must be >= subtotal. I retry with the specific error: "extracted total ($1,500) is less than or equal to subtotal ($1,500) — the actual total should include tax."

Scenario D:
The model returned the vendor address in the vendor_name field. I retry with: "vendor_name should be the company name only, not the address."
```

**Observe:** Scenarios A, C, D should resolve on retry (format errors, structural misplacement). Scenario B will not resolve — the information is not in the document.

**Checkpoint 6:** What is the key distinction between a retryable extraction error and a non-retryable one?

---

### Step 7 — Batch Processing Trade-offs

**Prompt:**
```
I'm planning the processing architecture for invoice extraction. I have three workflows:

1. Real-time invoice approval: A purchasing manager submits an invoice for approval and waits for the extracted data to confirm the amounts match the PO before approving.

2. Monthly spend analysis: At the end of each month, process all 800 invoices received that month to generate a spend report for the CFO (reviewed the next morning).

3. Annual audit preparation: Process 10,000 historical invoices over a weekend to prepare structured data for auditors (results needed by Monday morning).

For each workflow, recommend: real-time Anthropic API or Message Batches API.
Explain the key factor in each decision. What does "no guaranteed latency SLA" mean for each use case?
```

**Observe:** Real-time for workflow 1 (blocking, user waits), Batches for workflows 2 and 3 (non-blocking, overnight/weekend).

**Checkpoint 7:** The Message Batches API has up to 24-hour processing time. For workflow 3 (weekend processing needed by Monday), is this acceptable? What is the maximum submission time to guarantee results by Monday 9 AM?

---

### Step 8 — Variation: Human Review Routing

**Prompt:**
```
My invoice extraction system processes 500 invoices per day. I want to implement a human review routing strategy.

The system outputs field-level confidence scores (0.0-1.0) for each extracted field.

Here is a sample extraction result:
{
  "vendor_name": {"value": "Acme Corp", "confidence": 0.98},
  "invoice_date": {"value": "2024-03-15", "confidence": 0.95},
  "total_amount": {"value": 1620.00, "confidence": 0.92},
  "po_number": {"value": "PO-2024-445", "confidence": 0.67},
  "invoice_category": {"value": "Software License", "confidence": 0.71}
}

Design the review routing logic:
1. What confidence threshold should route to human review? Justify your choice.
2. Should the threshold be the same for all fields? Which fields might need a lower threshold (stricter review)?
3. How would you validate that your confidence threshold is well-calibrated?
4. What is stratified sampling and why is it important even after you reduce human review?
```

**Observe:** Claude should address: threshold selection, field-specific thresholds (financial amounts need lower threshold = more review), calibration against labeled validation sets, and stratified sampling for ongoing error detection.

---

## Session Debrief

**Key Takeaways:**

1. **Tool use with JSON schemas eliminates syntax errors** — but semantic errors (values don't sum, wrong fields) require programmatic cross-validation
2. **Optional/nullable fields prevent fabrication** — required fields pressure the model to invent values for absent information
3. **The "other" + detail pattern** provides categorical reliability for standard cases and honest handling for edge cases
4. **Retry-with-error-feedback works for format/structure errors** — it is futile when the information is simply absent from the document
5. **Aggregate accuracy metrics mask segment-level failures** — validate by document type and field before reducing human review

**Exam Connection:**
- Task Statement 4.3: tool_choice options, nullable fields, enum + "other" + detail
- Task Statement 4.4: retry-with-feedback, when retries are ineffective, semantic vs syntax errors
- Task Statement 4.5: batch API characteristics, custom_id, latency-appropriate API selection
