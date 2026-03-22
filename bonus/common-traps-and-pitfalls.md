# Common Traps & Pitfalls Guide
**Claude Certified Architect – Foundations**

*Decoding the distractor patterns from the 12 sample questions*

---

## Introduction

Every wrong answer on this exam is wrong for a reason. The distractors are not random — they follow repeating patterns that reflect real-world temptations. This guide names each pattern, explains the underlying logic, and shows you 2–3 concrete examples drawn directly from the exam guide's sample questions.

Once you can name the pattern, you can eliminate it in seconds.

---

## Pattern 1: The Over-Engineering Trap

**Definition**: The option introduces significant infrastructure, complexity, or a multi-step system for a problem that has a simpler, more direct first-step solution. The distractor sounds sophisticated and technically valid, but it's the wrong level of response for the stated situation.

**Why it's tempting**: Engineers naturally gravitate toward robust, scalable solutions. Over-engineered options often sound more "production-ready."

**How to spot it**: Look for: custom routing classifiers, training ML models on historical data, separate classifier services, caching layers, batching systems — when the question asks "what's the most effective first step" or "what's the most effective approach."

**The test**: "Has the simpler approach actually been tried first? If not, is this infrastructure really justified?"

---

### Example 1.1 — Q2 (Tool Design)

**Scenario**: Agent frequently calls `get_customer` instead of `lookup_order`. Both tools have minimal descriptions.

**The over-engineering distractor**:
> C) Implement a routing layer that parses user input before each turn and pre-selects the appropriate tool based on detected keywords and identifier patterns.

**Why it's wrong**: A routing layer adds a pre-processing component that parses user input, detects keywords, and pre-selects tools — bypassing the model's natural language understanding entirely. This is significant engineering overhead. The root cause is minimal tool descriptions. Fixing descriptions takes 10 minutes and addresses the actual problem.

**The tell**: "routing layer that parses user input" is a custom infrastructure component that wouldn't be needed if descriptions were adequate.

---

### Example 1.2 — Q3 (Escalation)

**Scenario**: Agent achieves 55% first-contact resolution (target: 80%). Escalating simple cases, handling complex ones autonomously.

**The over-engineering distractor**:
> C) Deploy a separate classifier model trained on historical tickets to predict which requests need escalation before the main agent begins processing.

**Why it's wrong**: This requires labeled historical data, a trained model, a deployment pipeline, and a pre-processing inference step — before you've even tried improving the system prompt. The exam guide explicitly notes this "requires labeled data and ML infrastructure when prompt optimization hasn't been tried." The proportionate first response is explicit escalation criteria with few-shot examples.

**The tell**: "Deploy a separate classifier model trained on historical tickets" — this is a significant ML system build for a problem solvable with prompt engineering.

---

### Example 1.3 — Q11 (Batch API)

**Scenario**: Manager proposes switching both blocking pre-merge checks AND overnight reports to batch API for 50% cost savings.

**The over-engineering distractor**:
> D) Switch both to batch processing with a timeout fallback to real-time if batches take too long.

**Why it's wrong**: A timeout fallback mechanism that automatically switches between two API modes adds architectural complexity ("often faster" batch completion with real-time fallback) that isn't needed. The simpler solution is matching each workflow to the right API: batch for reports, real-time for blocking checks.

**The tell**: "timeout fallback to real-time" — adding a hybrid switching system when the solution is just choosing the right API for each use case.

---

## Pattern 2: The Wrong Problem Trap

**Definition**: The option solves a related problem, but not the problem the question actually identified. The distractor often addresses a real concern (frustration, confidence, complexity) — but not the specific failure mode described in the scenario.

**Why it's tempting**: The distractor identifies something that sounds relevant to the scenario context. If you skim the question, you might not notice the distinction.

**How to spot it**: Ask yourself "does this option actually fix what failed in the scenario?" If the scenario says "skipping tool X" but the option addresses "too many tools," it's the wrong problem.

**The test**: "Does the root cause identified by the question match what this option fixes?"

---

### Example 2.1 — Q1 (Tool Ordering)

**Scenario**: Agent skips `get_customer` in 12% of cases, leading to misidentified accounts and incorrect refunds.

**The wrong problem distractor**:
> D) Implement a routing classifier that analyzes each request and enables only the subset of tools appropriate for that request type.

**Why it's wrong**: The problem is tool ORDERING — the agent calls tools in the wrong sequence. A routing classifier addresses tool AVAILABILITY (which tools are enabled for a given request type), not the order in which they're called. The agent already has the right tools; the issue is that `lookup_order` is being called without first calling `get_customer`.

**The tell**: "enables only the subset of tools" — tool availability vs. tool sequencing are different problems.

---

### Example 2.2 — Q3 (Escalation Calibration)

**Scenario**: Agent escalates simple cases and handles complex ones autonomously. Root cause: unclear escalation decision boundaries.

**The wrong problem distractor**:
> D) Implement sentiment analysis to detect customer frustration levels and automatically escalate when negative sentiment exceeds a threshold.

**Why it's wrong**: The scenario explicitly shows the agent is miscalibrated on CASE COMPLEXITY — it escalates simple cases and handles complex ones. Sentiment analysis measures emotional tone, not case complexity. A highly frustrated customer with a simple standard return should be resolved by the agent. The exam guide notes: "sentiment doesn't correlate with case complexity, which is the actual issue."

**The tell**: The question says "escalation calibration" but sentiment measures emotions, not complexity.

---

### Example 2.3 — Q7 (Narrow Decomposition)

**Scenario**: Research system covers only visual arts (digital art, graphic design, photography), completely missing music, writing, and film production.

**Wrong problem distractors**:
> C) The web search agent's queries are not comprehensive enough and need to be expanded.
> D) The document analysis agent is filtering out non-visual sources due to overly restrictive relevance criteria.

**Why they're wrong**: The coordinator's logs show exactly what happened: it assigned three visual arts subtopics. The subagents executed those assignments correctly. Blaming the web search agent (C) or document analysis agent (D) when their assigned topics were all visual arts is diagnosing the wrong agent. The root cause is the coordinator's narrow topic decomposition — not the downstream agents.

**The tell**: When the logs directly show the failure in the coordinator's assignments, the answer doesn't blame downstream agents.

---

## Pattern 3: The Insufficient Guarantee Trap

**Definition**: The option uses a probabilistic mechanism (prompt instructions, few-shot examples, guidance text) for a scenario that requires a deterministic guarantee. If the question describes a failure rate (even 1%) that has financial, security, or irreversible consequences, prompt-based approaches are insufficient.

**Why it's tempting**: "Enhance the system prompt" or "add few-shot examples" are the most natural first responses to agent misbehavior. For most problems, they work. This pattern specifically targets cases where 100% compliance is required.

**How to spot it**: Look for: financial consequences ("incorrect refunds"), security requirements, identity verification before operations, "in X% of cases" failure rates for critical business logic.

**The test**: "Can this failure happen even once if the model follows the instructions perfectly? If yes, the guarantee is insufficient."

---

### Example 3.1 — Q1 (Tool Ordering, Again)

**Scenario**: Agent skips `get_customer` in 12% of cases, leading to misidentified accounts and incorrect refunds.

**The insufficient guarantee distractor**:
> B) Enhance the system prompt to state that customer verification via get_customer is mandatory before any order operations.

**Why it's wrong**: A system prompt instruction saying "verification is mandatory" is probabilistic. The agent already skips it 12% of the time — meaning it fails even when instructions exist. An enhanced prompt might reduce this to 5%, but it cannot reduce it to 0%. When the consequence is incorrect refunds (financial), 0.1% failure rate is still unacceptable. The exam guide states: "prompt-based approaches cannot [provide deterministic guarantees]."

**The tell**: "Enhance the system prompt to state... is mandatory" — "mandatory" in a prompt is advisory, not enforced.

---

### Example 3.2 — Q1 (Few-Shot as Insufficient Guarantee)

**The same scenario, different distractor**:
> C) Add few-shot examples showing the agent always calling get_customer first.

**Why it's wrong**: Few-shot examples improve reliability but don't guarantee compliance. They improve the model's tendency to follow the pattern — potentially from 88% to 96% — but the failure rate remains non-zero. For financial operations requiring identity verification, the 4% failure rate is still unacceptable.

**The tell**: Few-shot examples are excellent for format consistency and ambiguous case handling. They are not substitutes for programmatic enforcement when 100% compliance is required.

---

### Example 3.3 — Domain 3 (Q5 related)

**Similar pattern in CI**: "Add instructions to CLAUDE.md to always run in non-interactive mode" vs. the `-p` flag.

**Why prompt-based is wrong**: Instructions in CLAUDE.md about running non-interactively are irrelevant — the `-p` flag is a runtime flag that sets the actual execution mode. You cannot instruct Claude Code via CLAUDE.md to change its own invocation mode.

**The tell**: The solution to a technical execution problem (non-interactive mode) is a CLI flag, not a configuration instruction.

---

## Pattern 4: The Premature Optimization Trap

**Definition**: The option jumps to an optimization strategy (caching, batching, pre-computation, scaling) before the basic approach has been validated. The distractor is technically sophisticated but applies optimization thinking before the fundamentals are proven.

**Why it's tempting**: Optimization options often solve real efficiency concerns. The trap is applying them as the first or primary response to a reliability or quality problem.

**How to spot it**: Look for: caching "to anticipate needs", batching to reduce round-trips when the batching creates blocking dependencies, pre-computing results speculatively.

**The test**: "Is the basic approach working correctly before this optimization is applied? If not, optimize later."

---

### Example 4.1 — Q9 (Scoped Tools for Synthesis)

**Scenario**: Synthesis agent takes 2–3 extra round trips to coordinator for fact verification. 85% are simple checks, 15% require deeper investigation.

**The premature optimization distractor**:
> D) Have the web search agent proactively cache extra context around each source during initial research, anticipating what the synthesis agent might need to verify.

**Why it's wrong**: Proactive caching requires the web search agent to predict what the synthesis agent will need to verify — speculative computation based on anticipating future needs. This is fragile: if the synthesis agent needs to verify something outside the cached context, the optimization fails. The simpler, more reliable solution is giving the synthesis agent a scoped `verify_fact` tool for the 85% common case.

**The tell**: "Proactively cache... anticipating what the synthesis agent might need" — speculative optimization for a problem with a direct solution.

---

### Example 4.2 — Q9 (Batching as Wrong Optimization)

**Same scenario**, another distractor:
> B) Have the synthesis agent accumulate all verification needs and return them as a batch to the coordinator at the end of its pass, which then sends them all to the web search agent at once.

**Why it's wrong**: The synthesis agent may need a verified fact to inform subsequent synthesis decisions in the same pass. Batching all verifications to the end creates blocking dependencies within the synthesis pass itself — the agent can't complete synthesis without the verifications, so it can't batch them at the end anyway.

**The tell**: "Accumulate all verification needs... at the end" — batching works when items are independent; fails when later steps depend on earlier results.

---

## Pattern 5: The Correct But Wrong Scope Trap

**Definition**: The option applies the right technique, but at the wrong level of scope (user vs. project, subagent vs. coordinator, local vs. global). The technique itself is valid — the problem is where it's applied.

**Why it's tempting**: The concept in the distractor is genuinely correct. The trap is in the scope detail, which is easy to miss when reading quickly.

**How to spot it**: Look for: user-level vs. project-level config, subagent doing coordinator work, coordinator micromanaging subagents, global rules where path-specific rules are needed.

**The test**: "Is the right concept being applied at the right scope for the described requirement?"

---

### Example 5.1 — Q4 (Command Scoping)

**Scenario**: Create a `/review` slash command available to every developer when they clone the repository.

**The wrong scope distractor**:
> B) In `~/.claude/commands/` in each developer's home directory.

**Why it's wrong**: `~/.claude/commands/` is user-scoped — each developer would need to manually create it in their home directory. It is not version-controlled and is not shared when developers clone the repo. The requirement is "available to every developer when they clone" — that requires version control via `.claude/commands/`.

**The tell**: "~/.claude/commands/" is user-level personal commands; the scenario requires team-level shared commands.

---

### Example 5.2 — Q6 (Path-Scoped Rules vs. CLAUDE.md)

**Scenario**: Test files spread throughout codebase need same conventions regardless of location.

**The wrong scope distractor**:
> D) Place a separate CLAUDE.md file in each subdirectory containing that area's specific conventions.

**Why it's wrong**: Directory-level CLAUDE.md files are scope-bound to their directory. Test files at `src/components/__tests__/`, `src/api/__tests__/`, and `packages/backend/tests/` would each need their own CLAUDE.md. When a new test directory is added, another file must be created. Path-scoped `.claude/rules/` with `paths: ["**/*.test.tsx"]` covers all test files with a single file, regardless of where they are.

**The tell**: "a separate CLAUDE.md file in each subdirectory" — this is the wrong scope for cross-directory file-type conventions.

---

### Example 5.3 — Q6 (Skills vs. Rules)

**Same scenario**, another distractor:
> C) Create skills in `.claude/skills/` for each code type that include the relevant conventions.

**Why it's wrong**: Skills require manual invocation (`/skill-name`). The requirement is that conventions are "automatically applied" when editing matching files. Skills are on-demand; path-scoped rules are automatic. Using skills for "always-apply" conventions is the wrong scope (on-demand vs. automatic).

**The tell**: "Create skills... that include conventions" — skills are invoked; rules are automatic.

---

## Quick Reference: Spotting Traps at a Glance

| Trap Pattern | Signal Words/Phrases | The Counter-Test |
|-------------|---------------------|------------------|
| Over-Engineering | "deploy a classifier", "train a model", "routing layer", "caching layer" | Has the simpler approach been tried first? |
| Wrong Problem | Option solves related but different issue | Does this fix what the question said failed? |
| Insufficient Guarantee | "Enhance system prompt", "add instructions", "add few-shot" for financial/security | Can this still fail even if model follows instructions? |
| Premature Optimization | "proactively cache", "accumulate and batch", "pre-compute" | Is the basic approach working before optimizing? |
| Wrong Scope | User-level vs. project-level, subagent vs. coordinator | Is the right concept at the right scope? |

---

## Using This Guide During Study

The most effective use of this guide is **before taking mock exams**. Read through the pattern descriptions and examples once to build pattern awareness. Then, when you encounter wrong answers in mock exams or practice quizzes, ask yourself: "Which pattern is this?"

Over time, pattern recognition becomes automatic — you'll spot the over-engineered option in less than 5 seconds, leaving more time for the harder judgment calls.
