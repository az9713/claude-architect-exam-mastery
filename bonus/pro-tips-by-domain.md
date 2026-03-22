# Pro Tips by Domain
**Claude Certified Architect – Foundations**

*Insider knowledge distilled from source documentation and sample question analysis*

---

## How to Use This Guide

This guide goes beyond what study notes cover. Each tip represents a concept the exam tests in a specific, non-obvious way — either as a correct answer that requires precise knowledge, or as a trap that catches candidates who know the concept but miss a detail.

Read through once before your mock exams. Then revisit the domain sections for any area where you scored below 70% on domain quizzes.

---

## Domain 1: Agentic Architecture & Orchestration (27%)

**Tip 1.1 — The Money/Security Hook Rule**
> "When in doubt about enforcement: if money or security is involved → programmatic hooks; if style or preference → prompts."

This single rule answers at least 2 questions on every exam. If a question involves: financial transactions, identity verification before operations, refund limits, access control, or irreversible actions — the answer involves a hook or prerequisite gate, not a system prompt instruction. Prompt instructions have a non-zero failure rate. Business rules with financial consequences require zero failure rate.

**Tip 1.2 — Hub-and-Spoke is Non-Negotiable**
> "If an answer has subagents talking directly to each other, skip it — it's wrong."

The coordinator agent routes ALL communication. Subagent-to-subagent direct calls reduce observability, break consistent error handling, and bypass the coordinator's ability to make routing decisions. Any answer suggesting "the search agent passes results directly to the synthesis agent" is wrong. Everything goes through the coordinator.

**Tip 1.3 — Subagents Are Isolated by Default**
> "Subagents do NOT inherit the coordinator's conversation history — tested repeatedly."

This appears in questions about subagent context, why a subagent "doesn't know" earlier findings, or how to pass web search results to a synthesis agent. The answer always involves explicitly including prior findings in the subagent's prompt. There is no automatic sharing. There is no shared memory between invocations.

**Tip 1.4 — Parallel Spawning = One Response**
> "Multiple Task calls in ONE coordinator response = parallel. Task calls across separate turns = sequential."

This is the mechanism for parallel subagents. If the coordinator emits two Task tool calls in a single response message, both subagents run in parallel. If it emits them in separate turns (first one, wait for result, then the other), they run sequentially. The exam tests this distinction in latency-optimization questions.

**Tip 1.5 — `stop_reason` Is the Only Loop Control**
> "Loop termination: `stop_reason == 'end_turn'`. Loop continuation: `stop_reason == 'tool_use'`. Nothing else."

Anti-patterns that appear as wrong answers: checking for assistant text content as a completion indicator, setting arbitrary iteration caps as the primary stop condition, parsing natural language signals from the model. All wrong. Only `stop_reason` is authoritative.

**Tip 1.6 — Prompt Chaining = Predictable; Dynamic Decomposition = Open-Ended**
> "Code review (per-file then cross-file integration) = prompt chaining. 'Add tests to legacy codebase' = dynamic decomposition."

Prompt chaining is for tasks with known, enumerable steps. Dynamic decomposition is for tasks where what needs to be done depends on what you find. The distinction appears in task decomposition questions.

**Tip 1.7 — Coordinator Decomposition is the Root Cause of Coverage Gaps**
> "If all subagents worked correctly but coverage is incomplete — blame the coordinator's scope assignment."

This is exactly what Q7 tests. Each subagent did its job. The coordinator gave them all visual arts subtopics. The root cause is narrow coordinator decomposition, not subagent execution failures.

---

## Domain 2: Tool Design & MCP Integration (18%)

**Tip 2.1 — Tool Description is the #1 Lever**
> "Tool descriptions are the primary mechanism LLMs use for tool selection — directly stated in the exam guide. This is tested in Q2."

When tool selection is unreliable, improve descriptions before anything else. Not routing layers, not few-shot examples (those add overhead without fixing the root cause), not consolidation (valid but complex, not a first step). Descriptions include: what the tool does, input formats, example queries, what it DOESN'T handle, and when to use it vs. similar tools.

**Tip 2.2 — Four Error Categories, Two Booleans**
> "transient (retryable=true), validation (retryable=false), permission (retryable=false), business (retryable=false)"

Most error types are not retryable. Only transient errors (timeout, service unavailability) benefit from retry. Know the four categories and their typical `isRetryable` values cold. The exam tests this when asking what error structure enables intelligent agent recovery.

**Tip 2.3 — Empty Results Are NOT Errors**
> "A search returning [] is a success. Only mark `isError: true` when the tool operation itself failed."

This distinction is tested. If a database query completes and finds no matching records, that is a successful empty result. If the database is unavailable, that is a failure. Returning an empty result as `isError: true` causes the coordinator to retry a perfectly valid (if empty) query. Use explicit `isEmptyResult: true` to make the distinction clear.

**Tip 2.4 — 4-5 Tools Per Agent is the Sweet Spot**
> "18 tools for one agent degrades selection reliability. Each agent should have only what it needs for its role."

The synthesis agent doesn't need web_search. The search agent doesn't need report compilation tools. Scope tool access to each agent's role. Cross-role tools (like `verify_fact` for the synthesis agent) are acceptable for high-frequency needs, but should be constrained alternatives, not the full toolkit.

**Tip 2.5 — Project `.mcp.json` vs. User `~/.claude.json`**
> "`.mcp.json` is committed to git — team uses it. `~/.claude.json` is personal — never committed. Credentials always via `${ENV_VAR}`."

This scoping rule appears in configuration questions. Two developers cloning the same repo should get the same MCP servers (project-level). Experimental personal tools that shouldn't affect teammates go in user-level config. Never hardcode tokens — always `${GITHUB_TOKEN}` style expansion.

**Tip 2.6 — MCP Resources vs. Tools**
> "Resources expose data catalogs (what exists). Tools perform operations (fetch, search, mutate). Resources reduce exploratory tool calls."

When you want the agent to know what data is available before deciding what to query, use MCP resources to expose a catalog. This is more efficient than having the agent make multiple exploratory tool calls to discover available data.

---

## Domain 3: Claude Code Configuration & Workflows (20%)

**Tip 3.1 — The Scoping Rule Applies Everywhere**
> "Project-level = version-controlled = shared with team. User-level = personal = NOT shared. This rule applies to CLAUDE.md, commands, skills, AND MCP servers."

`.claude/commands/` = team commands. `~/.claude/commands/` = personal.
`.claude/CLAUDE.md` = team standards. `~/.claude/CLAUDE.md` = personal.
`.mcp.json` = team tools. `~/.claude.json` = personal tools.

One rule, four applications. The exam tests it in all four contexts.

**Tip 3.2 — `context: fork` Prevents Context Pollution**
> "`context: fork` = isolated sub-agent context; no `context:` = runs inline in main conversation"

Skills that produce verbose output (codebase analysis, research, brainstorming) should use `context: fork` so their output doesn't pollute the main conversation. The skill runs in an isolated context and returns a summary. Without `context: fork`, verbose skill output becomes part of the main conversation history and consumes context tokens.

**Tip 3.3 — Path-Scoped Rules Win Cross-Directory Conventions**
> "One `.claude/rules/testing.md` with `paths: ["**/*.test.tsx"]` beats N directory CLAUDE.md files."

When a convention applies to a file type that exists in many directories (test files, migration files, config files), a single path-scoped rule is the right answer. Directory CLAUDE.md files are directory-bound — they can't follow files to new locations. Path-scoped rules follow the file pattern everywhere.

**Tip 3.4 — Plan Mode Has 3 Triggers**
> "Multiple files (5+), architectural decisions, or multiple valid approaches → use plan mode. Anything else → direct execution."

The exam presents tasks with explicit scope signals: "dozens of files", "decisions about service boundaries", "multiple valid approaches" → plan mode. "Fix the bug on line 47", "add a single validation", "update the timeout value" → direct execution.

**Tip 3.5 — `-p` Flag is the ONLY CI Answer**
> "The only way to run Claude Code non-interactively in CI is `claude -p`. There is no `CLAUDE_HEADLESS`, no `--batch` flag, no `/dev/null` workaround."

This is tested directly in Q10. The exam includes non-existent options (`CLAUDE_HEADLESS` environment variable, `--batch` flag) as distractors. `-p` (or `--print`) is the documented, correct answer.

**Tip 3.6 — CLAUDE.md Provides CI Context**
> "Project CLAUDE.md is read by CI-invoked Claude Code sessions — put testing standards, fixture info, and review criteria there."

When running automated code review or test generation in CI, Claude Code reads the project CLAUDE.md. Include: test framework conventions, available fixtures, review criteria with explicit severity definitions, what NOT to flag (linting, formatting). This improves CI output quality without changing the CLI invocation.

**Tip 3.7 — Independent Review Instance > Self-Review**
> "A Claude session that generated code is less effective at reviewing it — retains reasoning context from generation."

For CI code review, use a separate, fresh Claude Code invocation for the review step. The generator retains its decision-making context which biases review toward confirming its own choices. An independent instance with no generation history reviews more objectively.

---

## Domain 4: Prompt Engineering & Structured Output (20%)

**Tip 4.1 — Explicit Categories Beat Confidence Hedging**
> "'Be conservative' and 'only report high-confidence findings' do not reduce false positives. Explicit categorical criteria do."

Hedging language shifts interpretation to the model — the model decides what "conservative" means. Explicit categories define exactly what to flag and skip with concrete examples. Q3's distractor (self-reported confidence threshold) fails for the same reason: confidence is poorly calibrated.

**Tip 4.2 — Few-Shot for Consistency; Examples for Transformation**
> "Inconsistent output → few-shot examples. Inconsistent transformation → concrete input/output examples."

Few-shot examples are the most effective technique for consistently formatted output AND for demonstrating how to handle ambiguous cases. When detailed instructions produce inconsistency, examples fix it. 2–4 targeted examples is the sweet spot; more is not always better.

**Tip 4.3 — `tool_use` Eliminates Syntax, Not Semantics**
> "JSON schema + tool_use = no syntax errors. But line items not summing to total = semantic error = not eliminated."

Know this distinction cold. "What prevents JSON syntax errors?" → `tool_use` + JSON schema. "What prevents a total field that doesn't match line items?" → validation logic + retry-with-feedback. Two different problems, two different solutions.

**Tip 4.4 — Nullable Fields Prevent Hallucination**
> "If the source document might not contain this field, make it `['string', 'null']`. Required non-nullable fields force the model to fabricate values."

This is one of the most practically important schema design decisions. Invoices without dates, research papers without sample sizes, tickets without reference numbers — if you require these fields, the model will make something up. Nullable fields allow honest `null` reporting.

**Tip 4.5 — Enum + "other" + detail = Extensible Categories**
> "Add `'other'` to every enum that might encounter unanticipated values. Pair it with a `detail` string field."

Without "other": model forced to pick closest category → misclassification. With "other" + detail: model reports the actual value, downstream system can route it. This is the exam's pattern for handling extensible categorical fields.

**Tip 4.6 — Batch API: Three No's**
> "Batch API: NOT for blocking workflows. NOT for multi-turn tool calling. NOT guaranteed latency."

The 50% savings is the lure; these three limitations define when NOT to use it. Pre-merge checks (blocking), agentic loops (multi-turn tool calling), and anything with a tight SLA (no guaranteed latency) should all use the synchronous API.

**Tip 4.7 — Retry Only When Data Is Present**
> "Retry with specific error feedback works for format/structural errors. Retry is useless when the required data simply isn't in the document."

This is tested in validation-retry loop questions. Distinguish: "wrong format but data exists" (retry will fix it) vs. "data exists only in an external document not provided" (retry won't help — wrong source).

---

## Domain 5: Context Management & Reliability (15%)

**Tip 5.1 — Case Facts Block = Never Summarize Numbers**
> "Exact amounts, dates, order numbers, and customer-stated expectations must be preserved verbatim outside summarized history."

Progressive summarization is a feature that destroys precision. "$127.99" becomes "over $100". "Event tomorrow" becomes "urgent". "Order ORD-20240892 placed November 15" becomes "recent order". Extract these facts into a structured JSON block that lives outside of summarized conversation history and is included verbatim in every prompt.

**Tip 5.2 — Three Escalation Triggers Only**
> "Explicit human request = escalate immediately. Policy gap = escalate. Cannot progress = escalate. Everything else = try to resolve."

Two reliable wrong answers always present: sentiment-based escalation and self-reported confidence scores. Sentiment measures emotional tone, not case complexity. Confidence scores are poorly calibrated (the model is already confidently wrong on hard cases). Neither is an appropriate escalation trigger.

**Tip 5.3 — Explicit Human Request = Immediate Escalation**
> "Customer says 'I want to talk to a human' → escalate without investigating first. No exceptions."

This is specifically tested. The wrong answer in this pattern is: "investigate the issue first to see if you can resolve it, then offer escalation." When a customer explicitly requests a human, honoring that request immediately builds trust. Investigating first before offering escalation is paternalistic and wrong.

**Tip 5.4 — Empty Result vs. Access Failure**
> "`results: []` from a successful query is NOT an error. A timeout or connection failure IS an error. Always distinguish these explicitly."

Coordinators make different recovery decisions based on this distinction. An empty result means "this customer has no orders" (true, actionable information). An access failure means "we couldn't check" (may need retry or alternative). Conflating them causes either unnecessary retries or missed recovery opportunities.

**Tip 5.5 — Scratchpad + Subagents for Long Sessions**
> "Context degradation: agent references 'typical patterns' instead of specific classes found 50 tool calls ago → use scratchpad files."

The signal for context degradation is the agent becoming less specific. When you see an agent that was precise about specific code structures early in a session giving generic "typical pattern" responses later — it has lost earlier findings. Scratchpad files (written to disk) and subagent delegation (returns only summaries) both counter this.

**Tip 5.6 — Conflicting Sources = Annotate Both, Not Pick One**
> "Two credible sources with different statistics: always annotate both with source attribution. Check if publication dates explain the difference. Never silently select one value."

The synthesis agent's job is to preserve information provenance, not to resolve contradictions. If Source A says 30% and Source B says 12%, the synthesis output should include both values with attribution and a temporal check (are they measuring different years?). Letting the synthesis agent silently choose one value removes the coordinator's/human's ability to make an informed decision.

**Tip 5.7 — Aggregate Accuracy Masks Segment Failures**
> "97% overall accuracy does NOT mean ready to automate. Always segment by document type and field before reducing human review."

A system that is 99.5% accurate on standard documents and 68% accurate on non-English documents reports 97% overall if non-English is 15% of volume. Those 32% errors on a specific segment are unacceptable. Use stratified sampling to check error rates in each segment separately.

---

## Cross-Domain Tips

**Tip X.1 — "Proportionate First Response" Is Always Right**
> "The exam rewards proportionate responses. The correct answer addresses the root cause directly, without introducing more infrastructure than the problem requires."

This applies across all domains. When multiple options solve the stated problem, the correct answer is the one that:
1. Addresses the root cause (not a symptom)
2. Does so without unnecessary infrastructure
3. Has not been superseded by an even simpler approach

**Tip X.2 — Root Cause vs. Symptom**
> "Most wrong answers address a symptom; correct answers address the root cause."

Q2 example: agent calls wrong tool (symptom) because descriptions are minimal (root cause). Adding more few-shot examples addresses the symptom (adds examples of correct behavior) but doesn't fix the root cause (descriptions still don't differentiate the tools).

**Tip X.3 — "Most Effective FIRST Step" Matters**
> "When the question says 'most effective first step', the answer is the simplest proportionate approach — not the most complete or robust solution."

This wording appears in several questions. It rules out over-engineered solutions that would be valuable eventually but are not the first step.
