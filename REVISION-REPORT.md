# Study Notes Revision Report

**Date:** 2026-03-23
**Purpose:** Remove excessive code blocks from study notes to better align with the exam format
**Exam:** Claude Certified Architect — Foundations (multiple-choice conceptual exam, NOT a coding exam)

---

## Why This Revision Was Made

The 5 study-notes files contained 45–70% code blocks (3,319 lines of code vs 1,617 lines of prose). This was disproportionate for a multiple-choice exam that tests architectural judgment and decision-making — not implementation skills. Students were spending reading time on full Python functions, complete JSON schemas, and lengthy bash scripts instead of the concepts, decision logic, and anti-patterns that actually appear on the exam.

The rest of the study package (cheat-sheets, scenario-workbooks, labs, interactive-sessions, bonus materials) already had well-calibrated code density and was **not modified**.

---

## Summary of Changes

| File | Before (lines) | After (lines) | Before Code % | After Code % | Code Blocks: Before → After |
|------|:-:|:-:|:-:|:-:|:-:|
| domain1-agentic-architecture.md | 1,398 | ~850 | 44.8% | ~15% | 25 → ~9 |
| domain2-tool-design-mcp.md | 760 | ~340 | 62.8% | ~17% | 18 → ~7 |
| domain3-claude-code-config.md | 758 | ~490 | 57.7% | ~16% | 26 → ~9 |
| domain4-prompt-engineering.md | 997 | ~500 | 68.8% | ~16% | 22 → ~7 |
| domain5-context-reliability.md | 1,028 | ~580 | 70.0% | ~15% | 20 → ~6 |
| **Total** | **4,941** | **~2,760** | **~60%** | **~16%** | **111 → ~38** |

---

## What Was Removed (by pattern)

### Full function implementations → Conceptual prose
- `run_agentic_loop()` (70 lines → 13-line skeleton)
- `coordinate_research()`, `WorkflowEnforcer`, `data_normalization_hook`, `authorization_and_policy_hook`
- `run_code_review_pipeline()`, `adaptive_test_coverage_workflow()`
- `handle_tool_error()`, `search_orders()`, `SearchSubagent` class
- `extract_with_retry()`, `should_retry()`, `validate_invoice()`
- `process_batch_results()`, `generate_and_review_independently()`, `multi_pass_code_review()`
- `build_prompt_with_case_facts()`, `trim_order_for_context()`, `build_aggregated_research_prompt()`
- `handle_customer_lookup()`, `execute_search()`, `synthesize_with_coverage()`
- `AgentStateManifest` class, `synthesize_with_provenance()`, `calibrate_confidence_thresholds()`

### Complete JSON/YAML schemas → Compact excerpts (5-15 lines)
- Tool definition JSON blocks (73 lines → 5-element structure description)
- `extract_invoice_tool` schema (70+ lines → 15-line excerpt showing key patterns)
- MCP resource JSON, review findings schema, subagent output schemas
- `.mcp.json` multi-server config → 1-server example with `${ENV_VAR}` pattern

### Multi-variant examples → Comparison tables
- Three `tool_choice` API calls → 3-row table
- Four error category code blocks → 4-row error category table
- Goals vs procedures prompts → 2-row comparison table
- Hooks vs prompt instructions → kept existing table (already good)

### Lengthy bash scripts → Essential flags only
- Full CI pipeline script → 3-4 lines showing `-p`, `--output-format json`, `--json-schema`
- Duplicate-avoidance review script → prose description

---

## What Was Replaced With

| Replacement Format | Count | Purpose |
|---|:-:|---|
| Decision/comparison tables | ~15 | Quick-scan format for exam-relevant tradeoffs |
| Numbered pattern descriptions | ~10 | "The pattern works in N phases" walkthroughs |
| Bullet-point principles | ~12 | Bold key terms with concise explanations |
| Conceptual prose paragraphs | ~20 | Describing what code does and why, not how |
| Cross-references | ~5 | Avoiding duplication between domains |
| Compact pseudocode (≤15 lines) | ~8 | Only when the structure itself is the tested concept |

---

## What Was Preserved (unchanged)

- All **Exam Tip** blockquotes (30 total across 5 files)
- All **Anti-Pattern Alert** tables/blockquotes (30 total)
- All **Cross-Domain Connection** blockquotes (30 total)
- All **Key Terminology Quick Reference** tables (5 tables)
- All **Decision Matrix** tables (15+ tables)
- All **"If You're Short on Time"** sections (5 sections)
- All section headers and task statement structure
- All "What You Need to Know" and "What You Need to Be Able to Do" sections

---

## Files NOT Modified

The following were reviewed and found to have appropriate code density:

- `cheat-sheets/*.md` — 13 code blocks across 5 files (~265 lines). Sparse and correct for quick-reference.
- `scenario-workbooks/*.md` — 36 blocks across 6 files. Proportionate to scenario complexity.
- `labs/*.md` — 23 blocks across 2 files. Appropriate for hands-on exercises.
- `interactive-sessions/*.md` — 17 blocks across 6 files. Light density, conversation-focused.
- `bonus/*.md` — 1 code block across 4 files. Strategically code-free.
- All HTML files (quizzes, mock-exams, infographics, flashcards, concept-map)
- Root files (README.md, study-guide.md, preparation-process.md, index.html)
- Reference files (exam guide PDF/TXT, skills guide TXT)
