# Interactive Session: CI/CD Code Review Design

**Surface:** Claude.ai (Cowork / collaborative)
**Duration:** 50–60 minutes
**Task Statements:** 3.6, 4.1, 4.6

---

## Learning Objectives

By the end of this session, you will be able to:
- Design prompts that produce actionable, structured code review output
- Apply explicit categorical criteria to reduce false positives
- Understand the independent review instance pattern
- Design multi-pass review for large pull requests

---

## Prerequisites

- Claude.ai account (Cowork access for collaborative use, or use individually)
- Familiarity with pull request review processes
- Sample code snippets to review (provided in the session)

---

## Session Steps

### Step 1 — Establish the Review Problem

**Prompt 1:**
```
I'm designing an automated code review system that runs on every pull request. The system currently produces too many false positives — developers are dismissing all feedback because there's too much noise.

Here's a sample of the feedback it produces today:
- "Variable name 'data' is too generic — consider more descriptive naming"
- "This function is 47 lines — consider splitting into smaller functions"
- "Missing JSDoc comment on this function"
- "Potential SQL injection in line 23"
- "Authentication check appears to be missing"

Which of these are real issues vs style preferences? How would you categorize them by importance?
```

**Observe:** Claude should identify SQL injection and missing auth check as high-priority real issues. Naming, function length, and JSDoc are style preferences that create noise without real safety benefit.

**Checkpoint 1:** Which findings would you disable first to restore developer trust? What is the risk of keeping noisy categories alongside accurate ones?

---

### Step 2 — Write Explicit Review Criteria

**Prompt 2:**
```
Based on our discussion, help me write explicit review criteria for an automated PR review system.

The system should REPORT:
- Security vulnerabilities
- Bugs that cause incorrect behavior
- Data integrity risks

The system should SKIP:
- Naming conventions (teams have valid local conventions)
- Code organization preferences (function length, file structure)
- Comment coverage (unless a comment directly contradicts code behavior)

For each "REPORT" category, write a specific definition that distinguishes a real issue from a false positive. For "SQL injection," for example, what's the exact pattern to report vs skip?
```

**Observe:** Claude should write specific criteria like "flag SQL injection ONLY when user-controlled input is concatenated into a query string without parameterization" — not just "flag any SQL query."

**Checkpoint 2:** How is "flag when user-controlled input is concatenated without parameterization" more precise than "flag SQL injection"? What does the extra precision prevent?

---

### Step 3 — Design Few-Shot Examples for Consistent Output

**Prompt 3:**
```
I want every finding from the review system to be formatted identically so it can be automatically parsed and posted as GitHub PR comments.

Here's the format I want:
FILE: [path]
LINE: [number]
SEVERITY: [CRITICAL/HIGH/MEDIUM/LOW]
ISSUE: [one sentence]
FIX: [one sentence with specific code change]

Using this format, create 4 few-shot examples I can include in my review prompt:
1. A CRITICAL SQL injection finding
2. A HIGH authentication bypass finding
3. A MEDIUM missing null check finding
4. An example of something that looks like an issue but should be SKIPPED (to calibrate the negative case)
```

**Observe:** The negative example (what to skip) is as important as the positive examples for calibrating the model.

**Checkpoint 3:** Why is including a "skip this" example important in the few-shot set? What happens if you only show examples of what to report?

---

### Step 4 — Multi-Pass Review Design

**Prompt 4:**
```
A pull request modifies 12 files across the payment processing module. When I run a single-pass review on all 12 files together, I get inconsistent results — detailed feedback on some files but superficial treatment of others.

How would you structure a multi-pass review architecture to address this? What should each pass look for, and in what order?

Consider: local issues within each file vs cross-file integration issues (e.g., a function in file A returns a type that file B handles incorrectly).
```

**Observe:** Claude should describe:
- Pass 1: Per-file local analysis (bugs, security, logic errors within each file)
- Pass 2: Cross-file integration pass (data flow across files, type mismatches, inconsistent assumptions)

**Checkpoint 4:** Why doesn't a single large-context pass solve the attention dilution problem even with a large context window?

---

### Step 5 — Independent Review Instance vs Self-Review

**Prompt 5:**
```
I have two options for reviewing generated code:

Option A: After Claude generates the code, I ask the same Claude session: "Now review this code you just wrote for bugs."

Option B: I use a completely separate Claude session with no knowledge of how the code was generated. I give it only the code to review.

Which produces better review quality? Why?

Explain the concept of "reasoning context retention" and how it affects self-review.
```

**Observe:** Claude should explain that a session that generated code has reasoning context — it knows why it made each decision and is less likely to question those decisions. An independent session approaches the code fresh.

**Checkpoint 5:** In a CI/CD pipeline, how would you ensure the review instance is always independent? What specific implementation pattern achieves this?

---

### Step 6 — Handling Duplicate Findings on Iterative PRs

**Prompt 6:**
```
A developer pushes 3 commits to a PR over the course of a day. My automated review runs on each push. By the third run, the developer has received the same comment about a SQL injection vulnerability 3 times — they already saw it the first time.

Design the prompt modification that makes the third review run:
1. Skip findings from the first two runs that have already been reported
2. Only flag: new issues in the new commit, and previous issues that are still present and unaddressed

What context do you need to include in the third run's prompt? How do you distinguish "fixed" from "still present"?
```

**Observe:** The prompt needs: original findings list, the updated code, instruction to compare and report only new/unresolved issues.

**Checkpoint 6:** Why can't you simply ask "don't repeat findings from previous runs" without providing the previous findings list?

---

### Step 7 — CI/CD Integration: The -p Flag

**Prompt 7:**
```
I'm integrating this review system into GitHub Actions. My current CI step is:

```yaml
- name: Code Review
  run: claude "Review the changed files for security issues"
```

The CI job hangs indefinitely. What is wrong and what is the exact fix?

Also: I want the review output to be machine-parseable JSON so I can automatically post findings as inline GitHub comments. What CLI flags do I need?
```

**Observe:** Missing `-p` flag causes interactive hang. `--output-format json --json-schema` enables structured output.

**Checkpoint 7:** What does the `-p` flag do? Why doesn't stdin redirection (`< /dev/null`) fully solve the problem?

---

### Step 8 — CLAUDE.md for CI Context

**Prompt 8:**
```
My CI review currently generates low-quality test suggestions because it doesn't know:
- What testing framework we use (Jest + Supertest)
- What test fixtures already exist (we have a `__mocks__` directory with common mocks)
- What we consider a valuable test (not just "add a test that it returns 200")

How should I provide this context to the CI-invoked Claude Code instance? Where should this information live?

Should this be in the system prompt sent via the API? Or somewhere else?
```

**Observe:** CLAUDE.md is the correct answer — it is loaded by CI-invoked Claude Code automatically. Testing standards, fixture conventions, and review criteria documented there improve output quality without engineering a custom context injection system.

**Checkpoint 8:** Why is CLAUDE.md better than a long system prompt passed via the `-p` command for providing CI context?

---

### Step 9 — Confidence Scoring for Review Routing

**Prompt 9:**
```
My review system sometimes flags issues with low confidence — uncertain whether something is actually a bug or an intentional pattern. I want to route these differently:

- HIGH confidence findings → auto-post as PR comments
- LOW confidence findings → queue for human architect review
- CRITICAL findings → block the PR regardless of confidence

Design the review output schema that captures confidence, and write the routing logic description.
```

**Observe:** The schema needs a `confidence` field (float 0-1 or enum), plus a `severity` field for the CRITICAL bypass logic.

**Checkpoint 9:** Why would you block CRITICAL findings even at low confidence? What is the risk profile difference between a false positive CRITICAL vs a missed CRITICAL?

---

### Step 10 — Variation: Batch vs Real-Time for CI

**Prompt 10:**
```
My engineering org runs code reviews in two scenarios:
1. Pre-merge check: blocks the PR until review completes (developers wait)
2. Weekly security audit: scans all code changed in the past week for patterns (reviewed Monday morning)

My manager wants to use the Message Batches API for scenario 2 to save 50% on costs.

Evaluate: is this appropriate? What constraints does the Batches API have that matter for each scenario?
```

**Observe:** Batch API appropriate for scenario 2 (non-blocking, overnight window fits within 24-hour SLA). Not appropriate for scenario 1 (blocking, developer waits).

**Checkpoint 10:** The Batches API has a 24-hour processing window. For the weekly audit reviewed Monday morning, what is the latest the batch can be submitted on Sunday?

---

## Session Debrief

**Key Design Principles:**

1. **Explicit criteria > vague instructions** — "be conservative" doesn't work; "skip style, report security" does
2. **Few-shot examples calibrate both format AND judgment** — negative examples (what to skip) are as important as positive ones
3. **Multi-pass prevents attention dilution** — per-file + integration passes beat single large-context passes
4. **Independent review instances find bugs the generator would rationalize** — same session = biased review
5. **Duplicate suppression requires previous findings in context** — you cannot skip what you cannot compare against
6. **`-p` flag enables CI non-interactive mode** — without it, pipelines hang

**CLAUDE.md in CI:**
- Provides testing framework, fixture conventions, review criteria
- Loaded automatically by CI-invoked Claude Code
- Replaces engineering custom context injection

**Exam Connection:**
- Task Statement 3.6: -p flag, --output-format json, CLAUDE.md in CI, independent review sessions
- Task Statement 4.1: explicit criteria to reduce false positives, disabling high-noise categories
- Task Statement 4.6: independent review instances, multi-pass review architecture
