# Domain 4 Cheat Sheet: Prompt Engineering & Structured Output
**Weight: 20% | Task Statements: 4.1 – 4.6**

---

## Key Concepts

| Concept | Definition | Exam Relevance |
|---------|-----------|----------------|
| Explicit criteria | Specific categorical rules for what to flag/skip | Reduces false positives; hedging language doesn't work |
| False positive erosion | High FP rate in one category undermines trust in all | Temporarily disable bad category while fixing |
| Few-shot examples | Demonstrated input/output pairs in prompt | Most effective for output consistency on ambiguous cases |
| `tool_use` + JSON schema | Structured output via schema enforcement | Eliminates syntax errors; most reliable structured output |
| `tool_choice: "auto"` | Model can call tool or return text | Default; allows conversational fallback |
| `tool_choice: "any"` | Model must call some tool | Guarantees tool is called; model picks which |
| Forced tool selection | `{"type": "tool", "name": "X"}` | Model must call this specific tool |
| Nullable field | `"type": ["string", "null"]` | Prevents hallucination when field may be absent |
| Enum with `"other"` | Category + detail string field | Handles unanticipated categories without misclassification |
| Syntax error | Malformed JSON structure | Eliminated by `tool_use` with schema |
| Semantic error | Values logically wrong (totals don't sum) | Requires validation logic; NOT eliminated by schema |
| Retry-with-feedback | Retry including specific error message + doc | More effective than "try again"; targeted correction |
| `detected_pattern` | Field tracking which code construct triggered finding | Enables false positive pattern analysis from dismissals |
| Batch API | Async bulk processing, 50% savings, up to 24h | Non-blocking workloads only |
| `custom_id` | Batch request correlation field | Maps responses back to source documents |
| No multi-turn in batch | Batch cannot execute tool calls mid-request | Batch = single-turn requests only |
| Self-review limitation | Generator retains reasoning context | Less likely to catch own errors; use independent instance |
| Multi-pass review | Per-file pass + cross-file integration pass | Prevents attention dilution on large reviews |
| Attention dilution | Quality drops when reviewing too many files at once | Split large reviews: per-file first, then cross-file |

---

## Decision Matrix

### What fixes which quality problem?

| Problem | Fix |
|---------|-----|
| Inconsistent output format | Few-shot examples showing correct format |
| Wrong tool selection for ambiguous queries | Few-shot examples with reasoning |
| High false positive rate | Explicit categorical criteria (not "be conservative") |
| JSON syntax errors | `tool_use` with JSON schema |
| Totals don't sum to line items | Semantic validation + retry-with-feedback |
| Hallucinated field values | Nullable schema fields (`["string", "null"]`) |
| Niche category misclassified | Enum `"other"` + detail string field |
| Reviewer misses issues in own code | Independent review instance |

### Synchronous API vs. Batch API?

| Criterion | Synchronous | Batch |
|-----------|-------------|-------|
| Blocks human workflow | Yes | Never (24h window) |
| Cost | Normal | 50% savings |
| Multi-turn tool calling | Supported | NOT supported |
| Pre-merge checks | ✓ | ✗ |
| Nightly audits, weekly reports | ✗ | ✓ |

### Which `tool_choice` setting?

| Scenario | Setting |
|----------|---------|
| May need conversational response | `"auto"` |
| Multiple schemas, unknown doc type | `"any"` |
| Specific extraction must run first | `{"type": "tool", "name": "X"}` |

---

## Code Reference

```python
# Explicit criteria — correct pattern
"Flag a comment ONLY when:
  - Comment claims function does X, but code shows it does Y
  - Not: comments that are incomplete but not contradictory"

# Never use:
"Be conservative" / "only report high-confidence findings"
# These shift interpretation to model without improving calibration

# Nullable fields to prevent hallucination
"invoice_date": {
    "type": ["string", "null"],
    "description": "ISO 8601 date or null if not found in document"
}

# Enum with "other" for extensibility
"department": {
    "type": "string",
    "enum": ["engineering", "marketing", "sales", "other"]
},
"department_detail": {"type": ["string", "null"]}

# Retry-with-feedback — include specific errors
messages.append({
    "role": "user",
    "content": (
        f"Validation errors:\n{specific_errors}\n\n"
        f"Original document:\n{document_text}\n\n"
        f"Previous extraction:\n{json.dumps(prior_extraction)}\n\n"
        f"Resubmit corrected extraction."
    )
})

# Batch API custom_id for correlation
{"custom_id": f"doc-{doc['id']}", "params": {...}}
```

---

## Anti-Patterns (Common Wrong Answers)

- **"Be conservative" to reduce false positives**: shifts interpretation without improving calibration
- **Self-reported confidence for routing**: poorly calibrated; wrong on hard cases with high stated confidence
- **Required schema fields for potentially-absent data**: forces hallucination; use nullable
- **Batch API for pre-merge checks**: 24-hour window unacceptable for blocking workflows
- **Retry with "please try again"**: no information for model to act on; include specific errors
- **Retrying when data is absent from document**: retry is ineffective; data simply isn't there
- **Reviewing all 20+ files in one pass**: attention dilution; split per-file + cross-file

---

## If You Remember Nothing Else

1. **Explicit categorical criteria beat hedging language** — "flag only when X" outperforms "be conservative"
2. **`tool_use` + JSON schema eliminates syntax errors, not semantic errors** — totals not summing requires validation
3. **Nullable fields prevent hallucination** — non-nullable required fields force fabrication when data is absent
4. **Batch API = 50% savings, up to 24h, no multi-turn tool calling** — non-blocking workloads only
5. **Independent review instances beat self-review** — the generating session retains reasoning context
