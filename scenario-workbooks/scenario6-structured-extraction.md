# Scenario 6: Structured Data Extraction

## Scenario Context

You are building a structured data extraction system using Claude. The system:

- **Extracts information** from unstructured documents (contracts, invoices, research papers, support tickets)
- **Validates output** using JSON schemas and cross-field semantic checks
- **Handles edge cases** gracefully — missing information, ambiguous values, conflicting data
- **Integrates with downstream systems** that consume the structured output programmatically
- **Processes at scale** using batch processing for cost-efficient high-volume extraction

**Primary Domains:** D4 (Prompt Engineering & Structured Output), D5 (Context Management & Reliability)

---

## Key Concepts for This Scenario

### Relevant Task Statements and Why They Matter

**Task Statement 4.2 — Few-Shot Prompting for Extraction**
Few-shot examples are particularly effective for extraction tasks because they demonstrate how to handle ambiguous document structures (inline citations vs bibliographies, narrative descriptions vs structured tables). Examples show correct behavior for edge cases that prose instructions often miss. When adding examples that show null/empty extraction for missing fields, the model generalizes to returning null rather than fabricating values for other missing fields too.

**Task Statement 4.3 — Structured Output via Tool Use**
Tool use with JSON schemas is the most reliable approach for guaranteed schema-compliant output — it eliminates JSON syntax errors. The `tool_choice` configuration determines when tools are called: `"auto"` allows text responses, `"any"` guarantees a tool is called (but the model chooses which), and forced selection (`{"type": "tool", "name": "..."}`) requires a specific tool. Strict schemas eliminate syntax errors but do NOT prevent semantic errors (line items that don't sum to the stated total). Optional (nullable) fields prevent fabrication of missing values.

**Task Statement 4.4 — Validation, Retry, and Feedback Loops**
Retry-with-error-feedback appends specific validation errors to the prompt, guiding the model toward correction. Retries are effective for format mismatches and structural errors but ineffective when the required information is simply absent from the source document. Semantic validation errors (values don't sum, wrong field placement) require programmatic post-processing — they are not caught by schema syntax validation.

**Task Statement 4.5 — Batch Processing**
The Message Batches API provides 50% cost savings for non-blocking workloads. It does not support multi-turn tool calling within a single batch request. The `custom_id` field is essential for correlating requests and responses in batch processing. Failed documents should be resubmitted individually with modifications (e.g., chunking oversized documents) rather than resubmitting the entire batch.

**Task Statement 5.5 — Human Review and Confidence Calibration**
Aggregate accuracy metrics (97% overall) can mask poor performance on specific document types or fields. Stratified random sampling of high-confidence extractions detects novel error patterns even after human review thresholds are reduced. Field-level confidence scores calibrated using labeled validation sets route review attention efficiently. Always validate accuracy by document type and field segment before automating high-confidence extractions.

---

## Practice Questions

### Question 1

Your extraction system is fabricating values for the `contract_value` field when contracts don't explicitly state a dollar amount (e.g., consulting agreements that bill hourly). How should you modify the JSON schema to prevent this?

**A)** Define `contract_value` as optional and nullable in the schema, so the model can return `null` when the information is not present in the document.

**B)** Add a prompt instruction: "Do not guess contract values — only extract explicitly stated dollar amounts."

**C)** Remove `contract_value` from the schema entirely if it cannot always be extracted.

**D)** Add `contract_value` to a separate second extraction pass that only runs on contracts that mention dollar amounts.

**Correct Answer: A**

**Explanation:** Making fields optional and nullable directly signals to the model that returning `null` is a valid, expected response when information is absent — this is a schema-level constraint that prevents fabrication. When a field is required (non-nullable), the model experiences pressure to provide a value and may fabricate one to satisfy the schema. Option B (prompt instruction) is probabilistic — the model may still fabricate occasionally. Option C (removing the field) eliminates the data for all documents, including those that do have contract values. Option D (conditional second pass) is over-engineered for what is a simple schema design fix, and the "mentions dollar amounts" detection is its own accuracy problem.

---

### Question 2

You are extracting contract types from legal documents. Most contracts are clearly one of five types (Service, Employment, NDA, License, Partnership), but some unusual contracts don't fit any category. How should you design the `contract_type` schema field to handle both standard and unusual cases?

**A)** Use an enum field with values `["Service", "Employment", "NDA", "License", "Partnership", "other"]` combined with a separate `contract_type_detail` string field that is required when `contract_type` is `"other"`.

**B)** Use a free-text string field so the model can describe any contract type without constraints.

**C)** Use an enum field with only the five standard values and instruct the model to select the closest match for unusual contracts.

**D)** Use an enum field with only the five standard values and a `confidence` float field indicating how certain the classification is.

**Correct Answer: A**

**Explanation:** The `enum` with `"other"` plus a detail string pattern handles both cases cleanly: standard contracts get a reliable, consistent categorical value (enabling filtering and aggregation); unusual contracts get classified as `"other"` with a human-readable explanation in the detail field. This is the explicit schema pattern described in Task Statement 4.3. Option B (free-text string) eliminates the reliability of categorical values — downstream systems cannot filter by type if values are free text. Option C (closest match forcing) produces unreliable classifications and hides the fact that unusual cases exist. Option D (confidence float with forced classification) still forces a potentially wrong category for unusual contracts; `"other"` is more honest and more useful.

---

### Question 3

Your invoice extraction system produces semantic validation errors: the line item amounts don't sum to the stated total. The JSON schema is strict and there are no syntax errors. How should you implement correction?

**A)** Implement a follow-up request that includes the original invoice, the failed extraction, and the specific validation error ("line items sum to $847.50 but stated_total is $892.00") to enable model self-correction.

**B)** Post-process all extraction outputs programmatically to recalculate totals and overwrite the `stated_total` field with the calculated sum.

**C)** Add a JSON schema constraint requiring line items to sum to total before the schema accepts the output.

**D)** Retry the extraction with the same prompt and document — semantic errors often resolve on second attempt.

**Correct Answer: A**

**Explanation:** Retry-with-error-feedback provides specific validation context that guides the model toward correction: "your extraction has this specific discrepancy." The model can then re-examine the invoice document and find whether the error is in the line items or in the total — perhaps a line item was misread, or the stated total includes fees not listed as line items. Option B (overwriting the stated total) corrupts the data: the invoice may legitimately have a discrepancy that should be flagged rather than silently corrected. Option C (JSON schema arithmetic constraints) is not a feature of JSON Schema — schemas validate structure and types, not cross-field arithmetic relationships. Option D (retrying without context) gives the model no new information; the same error may recur.

---

### Question 4

After 3 months of operation, your extraction system reports 97% accuracy overall. You propose reducing human review to only the 3% of extractions the system flags as low-confidence. A teammate argues this is premature. What is the most important validation step before reducing human review?

**A)** Analyze accuracy broken down by document type and field to verify consistent performance across all segments before reducing review.

**B)** Increase the sample size of the accuracy measurement from 100 to 1,000 documents to improve statistical confidence.

**C)** Run a 30-day pilot with reduced review and monitor for any increase in customer complaints about data errors.

**D)** Have the system generate its own accuracy estimate by comparing current extractions against previous extractions of the same documents.

**Correct Answer: A**

**Explanation:** Task Statement 5.5 is explicit: aggregate accuracy metrics can mask poor performance on specific document types or fields. A 97% overall accuracy might be 99.9% on standard invoices but only 72% on multi-page contracts with complex structures. Reducing human review for all high-confidence extractions while performance on specific segments is poor would miss the errors in those segments. Before reducing review, validate that accuracy is consistently high across all document types and all fields that matter. Option B (larger sample) is useful but misses the segmentation point — a 1,000-document sample with the same uneven distribution still hides segment-level failures. Option C (live pilot) risks real errors reaching downstream systems. Option D is circular — self-comparison against prior extractions does not provide ground truth for accuracy.

---

### Question 5

You need to process 500 insurance claim documents overnight and extract structured claim data from each. The results feed a morning report reviewed by claims analysts. A few documents regularly exceed the context window. How should you design the batch processing approach?

**A)** Submit all 500 documents using the Message Batches API with unique `custom_id` values; for documents exceeding context limits, resubmit them individually with chunking modifications.

**B)** Use the real-time Anthropic API for all 500 documents to guarantee results are available in time for the morning report.

**C)** Submit all 500 documents as a single batch; if any documents fail, resubmit the entire batch with stricter chunking for all documents.

**D)** Process all documents with real-time API calls in parallel threads, then use the Message Batches API for the subset that fail initially.

**Correct Answer: A**

**Explanation:** The Message Batches API is ideal for this workload: overnight processing, non-blocking, cost-sensitive (50% savings), with results needed for the next morning — well within the 24-hour processing window. Each document gets a unique `custom_id` for correlation. When documents fail (context limit exceeded), resubmit only those specific documents (identified by `custom_id`) with modifications (e.g., split into two chunks). Option B (real-time API for 500 documents) incurs full API cost and provides no benefit for an overnight workload. Option C (resubmitting the entire batch on any failure) wastes API calls reprocessing 490+ documents that succeeded; selective resubmission via `custom_id` is the correct pattern. Option D is overly complex — it adds a hybrid architecture when the batch API already handles the use case cleanly.

---

## Guided Walkthrough

### Problem: Building a Contract Extraction Tool with Validation

Your legal team needs structured data extracted from contract documents: party names, effective date, contract value, payment terms, and termination clauses.

**Step 1: Define the Extraction Tool**

```python
extraction_tool = {
    "name": "extract_contract_data",
    "description": "Extract structured data from a legal contract document",
    "input_schema": {
        "type": "object",
        "properties": {
            "party_1": {
                "type": "string",
                "description": "First party (typically the service provider)"
            },
            "party_2": {
                "type": "string",
                "description": "Second party (typically the client or customer)"
            },
            "effective_date": {
                "type": ["string", "null"],
                "description": "Contract effective date in ISO 8601 format (YYYY-MM-DD), or null if not stated"
            },
            "contract_value": {
                "type": ["number", "null"],
                "description": "Total contract value in USD, or null if not a fixed-price contract"
            },
            "contract_type": {
                "type": "string",
                "enum": ["Service", "Employment", "NDA", "License", "Partnership", "other"]
            },
            "contract_type_detail": {
                "type": ["string", "null"],
                "description": "Required when contract_type is 'other' — describe the contract type"
            },
            "payment_terms": {
                "type": ["string", "null"],
                "description": "Payment schedule or terms, or null if not specified"
            },
            "extraction_confidence": {
                "type": "object",
                "properties": {
                    "effective_date": {"type": "number", "minimum": 0, "maximum": 1},
                    "contract_value": {"type": "number", "minimum": 0, "maximum": 1},
                    "contract_type": {"type": "number", "minimum": 0, "maximum": 1}
                }
            }
        },
        "required": ["party_1", "party_2", "contract_type", "extraction_confidence"]
    }
}
```

**Step 2: Configure tool_choice**

For contract extraction, use forced tool selection to ensure the extraction tool is always called:

```python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=2048,
    tools=[extraction_tool],
    tool_choice={"type": "tool", "name": "extract_contract_data"},
    system="You are a contract analysis specialist. Extract information from the contract exactly as stated. Do not infer values not present in the document.",
    messages=[{"role": "user", "content": f"Extract data from this contract:\n\n{contract_text}"}]
)
```

**Step 3: Add Few-Shot Examples**

Add examples in the system prompt showing varied document structures:

```
Example 1 — Hourly consulting agreement (contract_value is null):
Document: "...Services shall be billed at $250 per hour..."
Extraction: {"contract_value": null, "payment_terms": "$250/hour"}

Example 2 — NDA with no effective date (effective_date is null):
Document: "...This Agreement shall become effective upon execution..."
Extraction: {"effective_date": null} (no specific date stated)

Example 3 — Fixed-price contract:
Document: "...total compensation of $45,000 payable in three equal installments..."
Extraction: {"contract_value": 45000, "payment_terms": "Three equal installments of $15,000"}
```

**Step 4: Implement Validation-Retry Loop**

```python
def extract_with_validation(contract_text, max_retries=2):
    response = call_extraction_api(contract_text)
    extracted = parse_tool_use_response(response)

    # Schema validation (caught by tool_use, rarely fails)
    validation_errors = validate_schema(extracted)

    # Semantic validation
    if extracted.get("contract_type") == "other" and not extracted.get("contract_type_detail"):
        validation_errors.append("contract_type_detail is required when contract_type is 'other'")

    if validation_errors and max_retries > 0:
        # Retry with specific error context
        retry_prompt = f"""
The previous extraction had these validation errors:
{chr(10).join(validation_errors)}

Original extraction:
{json.dumps(extracted, indent=2)}

Please review the document again and correct these specific issues.
"""
        return extract_with_validation(
            contract_text + "\n\n" + retry_prompt,
            max_retries - 1
        )

    return extracted
```

**Step 5: Route Low-Confidence Fields to Human Review**

```python
CONFIDENCE_THRESHOLD = 0.8

def route_for_review(extracted):
    low_confidence_fields = [
        field for field, score in extracted["extraction_confidence"].items()
        if score < CONFIDENCE_THRESHOLD
    ]

    if low_confidence_fields:
        queue_for_human_review(extracted, fields_needing_review=low_confidence_fields)
    else:
        queue_for_automated_processing(extracted)
```

---

## Try It Yourself

### Exercise: Design the Full Extraction Pipeline

You are building an extraction pipeline for research grant applications (50,000 per year). Each application contains: applicant name, institution, requested grant amount, research area, project duration, and abstract.

**Your tasks:**

1. **Design the JSON schema** for the extraction tool. Which fields should be required vs optional? Which fields need enum types? Design the `research_area` field to handle both standard categories and unusual interdisciplinary research using the `"other"` + detail pattern.

2. **Write the few-shot examples.** Design 3 examples: one with all fields present, one where grant amount is not stated (fellowship rather than grant), and one where project duration is expressed as a date range rather than months. What extraction should each example produce?

3. **Design the retry-with-feedback prompt.** A batch run produces extractions where `grant_amount` and `requested_budget` don't match (semantic error). The values differ by exactly $10,000 — likely an overhead calculation error. Write the follow-up prompt that includes the document, the failed extraction, and the specific validation error to guide self-correction.

4. **Design the stratified sampling strategy.** Your system processes applications across 8 research areas. Overall accuracy is 96%, but you suspect performance varies by area. Design a sampling plan to validate accuracy by segment before reducing human review. How many samples per segment? What document types get sampled separately?

**Exam Connection:** This exercise targets Task Statements 4.2, 4.3, 4.4, 4.5, and 5.5. Key concepts: nullable fields prevent fabrication, `"other"` + detail handles extensible enumerations, retry-with-feedback corrects format errors (not absent information), batch `custom_id` enables selective resubmission, and segment-level accuracy validation is required before automating high-confidence extractions.
