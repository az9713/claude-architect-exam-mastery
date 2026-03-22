# Scenario 5: Claude Code for Continuous Integration

## Scenario Context

You are integrating Claude Code into your Continuous Integration/Continuous Deployment (CI/CD) pipeline. The system:

- **Runs automated code reviews** on pull requests, providing inline feedback on issues
- **Generates test cases** for new code changes with coverage of edge cases
- **Provides feedback on pull requests** with actionable, structured findings
- **Must minimize false positives** to maintain developer trust in the automated system

You need to design prompts that provide actionable feedback, structure output for automated posting as PR comments, and decide when to use batch processing vs real-time API calls.

**Primary Domains:** D3 (Claude Code Config), D4 (Prompt Engineering & Structured Output)

---

## Key Concepts for This Scenario

### Relevant Task Statements and Why They Matter

**Task Statement 3.6 — CI/CD Integration**
The `-p` (or `--print`) flag runs Claude Code in non-interactive mode: it processes the prompt and exits without waiting for user input. Without this flag, Claude Code hangs waiting for interactive input in CI pipelines. `--output-format json` combined with `--json-schema` produces machine-parseable structured findings for automated processing. CLAUDE.md provides project context to CI-invoked Claude Code instances. Independent review instances (a fresh Claude session reviewing code) are more effective than self-review because the generator session retains its reasoning context.

**Task Statement 4.1 — Prompt Design for Precision**
General instructions ("be conservative," "only report high-confidence findings") fail to reduce false positives. Specific categorical criteria define which issues to report versus skip. False positive categories undermine developer trust in accurate categories — even if security findings are reliable, a noisy style checker erodes trust in the whole system. The fix for a high-false-positive category is either improving its specific criteria or temporarily disabling it while improving.

**Task Statement 4.2 — Few-Shot Prompting**
Few-shot examples are the most effective technique for achieving consistently formatted, actionable output when instructions alone produce inconsistent results. Examples demonstrate ambiguous-case handling (what counts as a real bug vs an acceptable pattern) and enable generalization to novel patterns beyond pre-specified cases.

**Task Statement 4.5 — Batch Processing**
The Message Batches API saves 50% on API costs but has up to 24-hour processing time with no guaranteed latency SLA. It does not support multi-turn tool calling within a single request. Pre-merge checks that block developer merges require real-time API calls. Overnight reports, weekly audits, and nightly test generation are ideal batch workloads. The `custom_id` field correlates batch request/response pairs.

**Task Statement 4.6 — Multi-Instance Review Architecture**
A model retains reasoning context from code generation, making self-review less effective at catching subtle issues. Independent review instances (without prior generation context) catch issues the generator would rationalize away. Splitting large reviews into per-file local passes plus separate cross-file integration passes avoids attention dilution and contradictory findings.

---

## Practice Questions

### Question 1

Your pipeline script runs `claude "Analyze this pull request for security issues"` but the job hangs indefinitely. Logs indicate Claude Code is waiting for interactive input. What is the correct approach to run Claude Code in an automated pipeline?

**A)** Add the `-p` flag: `claude -p "Analyze this pull request for security issues"`

**B)** Set the environment variable `CLAUDE_HEADLESS=true` before running the command

**C)** Redirect stdin from `/dev/null`: `claude "Analyze this pull request for security issues" < /dev/null`

**D)** Add the `--batch` flag: `claude --batch "Analyze this pull request for security issues"`

**Correct Answer: A**

**Explanation:** The `-p` (or `--print`) flag is the documented way to run Claude Code in non-interactive mode. It processes the prompt, outputs the result to stdout, and exits without waiting for user input — exactly what CI/CD pipelines require. Option B (`CLAUDE_HEADLESS=true`) is a non-existent environment variable; there is no such configuration in Claude Code. Option C (stdin redirection) is a Unix workaround that doesn't properly address Claude Code's command syntax and is not the intended mechanism. Option D (`--batch`) is a non-existent flag. The correct, documented answer is the `-p` flag.

---

### Question 2

Your team wants to reduce API costs for automated analysis. Currently real-time Claude calls power two workflows: (1) a blocking pre-merge check that must complete before developers can merge, and (2) a technical debt report generated overnight for review the next morning. Your manager proposes switching both to the Message Batches API for its 50% cost savings. How should you evaluate this proposal?

**A)** Use batch processing for the technical debt reports only; keep real-time calls for pre-merge checks.

**B)** Switch both workflows to batch processing with status polling to check for completion.

**C)** Keep real-time calls for both workflows to avoid batch result ordering issues.

**D)** Switch both to batch processing with a timeout fallback to real-time if batches take too long.

**Correct Answer: A**

**Explanation:** The Message Batches API offers 50% cost savings but has processing times up to 24 hours with no guaranteed latency SLA. This makes it unsuitable for blocking pre-merge checks where developers wait for results before merging — a 24-hour wait is unacceptable. For overnight technical debt reports, a 24-hour processing window is fine: the report is generated overnight and reviewed the next morning. Option B is wrong because "often faster" batch completion is not acceptable for blocking workflows — there is no SLA guarantee. Option C reflects a misconception: batch results can be correlated using `custom_id` fields, so ordering issues are not a problem. Option D adds unnecessary complexity (a fallback system) when the simpler solution is matching each API to its appropriate use case based on latency requirements.

---

### Question 3

A pull request modifies 14 files across the stock tracking module. Your single-pass review analyzing all files together produces inconsistent results: detailed feedback for some files but superficial comments for others, obvious bugs missed, and contradictory feedback — flagging a pattern as problematic in one file while approving identical code elsewhere. How should you restructure the review?

**A)** Split into focused passes: analyze each file individually for local issues, then run a separate integration-focused pass examining cross-file data flow.

**B)** Require developers to split large PRs into smaller submissions of 3–4 files before the automated review runs.

**C)** Switch to a higher-tier model with a larger context window to give all 14 files adequate attention in one pass.

**D)** Run three independent review passes on the full PR and only flag issues that appear in at least two of the three runs.

**Correct Answer: A**

**Explanation:** Splitting reviews into focused passes directly addresses the root cause: attention dilution when processing many files at once. File-by-file analysis ensures consistent depth for local issues, while a separate integration pass catches cross-file data flow issues (e.g., a value written in file A being misread in file B). Option B shifts burden to developers without improving the system — it also enforces an arbitrary limit that doesn't match natural PR scope. Option C misunderstands that larger context windows don't solve attention quality issues; the model can hold more tokens but still distributes attention inconsistently across many files. Option D would suppress detection of real bugs by requiring consensus across three independent runs — issues that are real but caught intermittently (perhaps only in 1 of 3 runs) would be falsely dismissed.

---

### Question 4

Your automated code review system flags 35% of valid code patterns as issues in the "naming conventions" category, but its security findings are accurate. Developers are starting to dismiss all automated feedback because of the naming convention noise. What is the most effective immediate response?

**A)** Temporarily disable the naming conventions review category and restore developer trust while improving the specific criteria for that category.

**B)** Add "be conservative" to the overall review prompt to reduce false positives across all categories.

**C)** Increase the confidence threshold for all findings so only very high-confidence issues are reported.

**D)** Replace the current review with a multi-pass approach where naming conventions are checked in a separate pass from security.

**Correct Answer: A**

**Explanation:** Task Statement 4.1 is explicit: high false positive rates in one category undermine developer trust in accurate categories. The most effective immediate response is temporarily disabling the noisy category while improving it. This preserves the reliable security feedback (which is valuable) while stopping the trust erosion from the naming conventions noise. Option B ("be conservative") is an example of a vague instruction that fails to reduce false positives — Task Statement 4.1 explicitly states this does not work. Option C (confidence threshold) is similar: without specific categorical criteria, "high-confidence" is not well-calibrated. Option D (separate passes) does not address the false positive rate — it just separates when the false positives appear, not whether they appear.

---

### Question 5

You want the CI review to produce structured output that can be automatically parsed and posted as inline PR comments on GitHub. Each finding should include: the file path, line number, severity, issue description, and suggested fix. Which CLI approach produces this output?

**A)** Use `claude -p "Review this PR" --output-format json --json-schema review_schema.json` with a schema defining the expected output structure.

**B)** Use `claude -p "Review this PR. Respond in JSON format with fields: file, line, severity, issue, fix."` with no additional flags.

**C)** Use `claude -p "Review this PR" --output-format markdown` and parse the markdown output with a script.

**D)** Use `claude -p "Review this PR" --stream-json` to receive incremental findings as they are generated.

**Correct Answer: A**

**Explanation:** `--output-format json` combined with `--json-schema` is the documented mechanism for enforcing structured output in CI contexts. The schema file defines the exact structure (array of findings, each with file, line, severity, issue, fix), and Claude Code enforces this structure in its output. This produces machine-parseable JSON that can be directly processed to create inline PR comments. Option B (instructing JSON format via prompt alone) is less reliable — the model may deviate from the requested format, add prose before the JSON, or produce schema variations. Option C (markdown output) requires brittle parsing of a human-readable format. Option D (`--stream-json`) is a non-existent flag in Claude Code's documented CLI options.

---

## Guided Walkthrough

### Problem: Designing the PR Review Prompt for Actionable Findings

Your team has been running automated reviews for two weeks. Developers report: findings are too vague ("this could be improved"), inconsistently formatted, and include too many style complaints that obscure the real bugs.

**Step 1: Define Specific Review Categories**

Replace vague review criteria with explicit categorical criteria:

```markdown
## Code Review Criteria

### REPORT these issues:
- Bugs: Code that will cause incorrect behavior or runtime errors
- Security: Authentication bypass, SQL injection, unvalidated user input, exposed secrets
- Performance: O(n²) or worse algorithms in hot paths, N+1 query patterns

### SKIP these issues:
- Minor style preferences (prefer const over let when both work)
- Local patterns specific to this team's conventions (e.g., unusual but consistent naming)
- Comments explaining what code does (only flag if comment contradicts actual behavior)

### Severity criteria:
- CRITICAL: Exploitable security vulnerability or data loss bug
- HIGH: Bug that will affect multiple users or corrupt data
- MEDIUM: Bug that affects specific edge cases or degrades performance measurably
- LOW: Best practice violation that could become a bug under future changes
```

**Step 2: Add Few-Shot Examples**

Add examples demonstrating the exact output format and ambiguous-case handling:

```markdown
## Review Output Examples

Example 1 — Real bug (should report):
```json
{
  "file": "src/auth/tokenValidator.js",
  "line": 47,
  "severity": "CRITICAL",
  "issue": "Token expiration check uses local time instead of UTC, allowing replay attacks from different timezones",
  "suggested_fix": "Replace new Date().getTime() with Date.now() and compare against exp claim directly"
}
```

Example 2 — Acceptable pattern (should NOT report):
```
Code: validateUser = async (id) => { ... }
Decision: Arrow function for async is a valid TypeScript pattern. Skip — this is not a bug.
```

Example 3 — Ambiguous case (boundary illustration):
```json
{
  "file": "src/payment/refund.js",
  "line": 23,
  "severity": "MEDIUM",
  "issue": "Missing null check on order.customerId before calling lookup — will throw TypeError if order has no customer association",
  "suggested_fix": "Add null guard: if (!order.customerId) { throw new ValidationError('Order has no customer') }"
}
```
```

**Step 3: Use `-p` with `--output-format json --json-schema`**

CI pipeline step:

```yaml
- name: Claude Code Review
  run: |
    claude -p "$(cat pr_review_prompt.md)\n\nFiles to review:\n$(cat changed_files.txt)" \
      --output-format json \
      --json-schema review_schema.json \
      > review_output.json

    # Post findings as inline PR comments
    python post_review_comments.py review_output.json
```

**Step 4: Independent Review Instance**

The code generator and the code reviewer should be separate Claude sessions. In the CI pipeline, the review step uses a fresh session with no knowledge of how the code was generated. This is more effective than asking the same session that generated code to review it.

**Step 5: Handling Duplicate Comments on Re-Runs**

When a new commit is pushed to a PR that already has review comments:

```markdown
## Re-run Context
The following issues were flagged in the previous review run:
[list of previous findings]

For this review, report ONLY:
1. New issues not in the previous list
2. Previous issues that are still present and unaddressed

Do NOT re-report issues that have been fixed.
```

---

## Try It Yourself

### Exercise: Design the Batch vs Real-Time Decision Framework

Your engineering organization runs Claude Code for four automated workflows:
1. Pre-merge security check (blocks PR merging until complete)
2. Weekly codebase technical debt analysis (CTO reviews Monday morning)
3. Nightly test coverage gap analysis (developers review next day)
4. Real-time "explain this code" feature in the developer IDE

**Your tasks:**

1. **Classify each workflow.** For each of the four workflows, decide: real-time Anthropic API, Message Batches API, or Claude Code CLI with `-p` flag. Justify each decision using the specific criteria from Task Statement 4.5 (latency requirements, blocking vs non-blocking, cost sensitivity).

2. **Design the security check prompt.** Write explicit categorical criteria for the pre-merge security check. What categories should be reported? What should be explicitly skipped? Include severity criteria with concrete examples at each level.

3. **Structure the batch job.** For the nightly test coverage analysis (100 files per night), design the Message Batches API request structure. How do you use `custom_id` to correlate results? How do you handle failures for individual files (resubmission strategy)?

4. **Reduce duplicate findings.** On a PR with 3 commits, the review runs three times. Design the prompt modification that prevents the third review run from re-reporting issues already flagged in the first two runs. What context do you include, and what instruction do you give?

**Exam Connection:** This exercise targets Task Statements 3.6, 4.1, 4.5, and 4.6. Key concepts: `-p` flag for non-interactive CI, batch vs real-time decision criteria, explicit categorical criteria to reduce false positives, and independent review instances for higher quality feedback.
