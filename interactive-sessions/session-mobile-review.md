# Interactive Session: Full-Domain Review (Mobile)

**Surface:** Mobile app (Claude mobile)
**Duration:** 30–45 minutes (shorter format for mobile)
**Task Statements:** All domains — D1, D2, D3, D4, D5

---

## Learning Objectives

This session consolidates all five exam domains into a conversational review format. Use it as a final preparation exercise — go through questions in short bursts during commute or breaks.

---

## Prerequisites

- Claude mobile app installed
- Familiarity with all five exam domains from the study notes
- No coding required — this is a conversational concept review

---

## Session Format

This session uses a question-and-answer format. For each prompt, answer mentally first, then ask Claude to explain the concept. Compare your answer to Claude's explanation.

---

## Domain 1 Review: Agentic Architecture

### Conversation 1 — Stop Reason Loop Control

**Prompt:**
```
Explain the agentic loop to me like I'm preparing for an architecture exam.
Focus on: what stop_reason values matter, what happens for each, and what the most common anti-patterns are.
Keep it concise — I'm on mobile.
```

**Before reading Claude's answer:** What are the two stop_reason values? When does the loop continue vs terminate?

**Checkpoint:** Did Claude mention these anti-patterns?
- Parsing natural language to determine loop termination
- Setting arbitrary iteration caps as the primary stopping mechanism
- Checking for assistant text content as a completion indicator

---

### Conversation 2 — Coordinator vs Subagent

**Prompt:**
```
In a multi-agent research system, the coordinator decomposes the topic "impact of automation on employment" into subtasks: "automation impact on manufacturing jobs" and "automation impact on service sector jobs."

The final report completely misses the impact on knowledge workers and creative professionals.

What is the root cause and how would you fix it?
```

**Before reading:** Is this a coordinator problem, a subagent problem, or a synthesis problem?

**Checkpoint:** The root cause is coordinator task decomposition — too narrow. Subagents executed correctly within their assigned scope.

---

### Conversation 3 — Hooks vs Prompt Rules

**Prompt:**
```
My customer support agent should never process refunds above $500 without manager approval.
I've added this to the system prompt: "Never process refunds above $500."
But in production, 8% of large refunds are still being processed automatically.

What's the architectural fix and why does the prompt approach fail?
```

**Checkpoint:** Pre-call hook intercepts `process_refund` before execution, checks amount, blocks if over $500. Prompt instructions are probabilistic (non-zero failure rate). For financial operations requiring deterministic compliance, programmatic enforcement is required.

---

## Domain 2 Review: Tool Design & MCP

### Conversation 4 — Tool Description Quality

**Prompt:**
```
I have two MCP tools:
- get_customer: "Retrieves customer information"
- lookup_order: "Retrieves order details"

Both accept similar identifier formats. The agent frequently calls get_customer when it should call lookup_order.

What is the root cause and what is the most effective first fix?
```

**Checkpoint:** Root cause is minimal tool descriptions. The model lacks context to differentiate. Fix: expand descriptions to include input formats, example queries, edge cases, and explicit boundaries explaining when to use each tool versus the other.

---

### Conversation 5 — Tool Distribution Principle

**Prompt:**
```
In my multi-agent research system, I give all 18 research tools to every subagent.
The synthesis agent (whose job is to combine findings) keeps attempting to do web searches.

What is the principle I'm violating and how do I fix it?
```

**Checkpoint:** Principle of least privilege / scoped tool access. Each agent should have only the tools needed for its role. The synthesis agent needs synthesis tools (read-only access to findings) plus possibly a scoped verify_fact tool for simple fact checks — not full web search access. Too many tools increases decision complexity and causes misuse.

---

### Conversation 6 — MCP Configuration Scope

**Prompt:**
```
Quick recall: what is the difference between .mcp.json and ~/.claude.json for MCP server configuration?

When would you use each? Give me a concrete example of a server that belongs in each location.
```

**Checkpoint:**
- `.mcp.json`: project-level, version-controlled, available to all team members. Example: team GitHub/Jira server
- `~/.claude.json`: user-level, not version-controlled, personal. Example: experimental local database server, personal productivity tools

---

## Domain 3 Review: Claude Code Configuration

### Conversation 7 — CLAUDE.md Hierarchy Diagnostic

**Prompt:**
```
A new developer joins the team. Claude Code on their machine is not following the TypeScript strict mode conventions.

You wrote those conventions in ~/.claude/CLAUDE.md on your own machine.

What is the bug and what is the fix?
```

**Checkpoint:** User-level (`~/.claude/CLAUDE.md`) is not shared via version control. The new developer's machine doesn't have your personal config. Fix: move team standards to the project-level CLAUDE.md (committed to the repo).

---

### Conversation 8 — Path Rules vs Subdirectory CLAUDE.md

**Prompt:**
```
I have TypeScript test files spread throughout the codebase (Button.test.tsx next to Button.tsx, auth.test.ts next to auth.ts, etc.).

I want all test files to follow the same testing conventions.

Should I use directory-level CLAUDE.md files or .claude/rules/ with glob patterns? Why?
```

**Checkpoint:** `.claude/rules/` with `paths: ["**/*.test.tsx", "**/*.test.ts"]`. Directory-level CLAUDE.md can't handle files spread across multiple directories. Path-specific rules load based on the file being edited, regardless of its location in the directory tree.

---

### Conversation 9 — Plan Mode Decision

**Prompt:**
```
For each of these tasks, should I use plan mode or direct execution?

A) Fix a bug in validateEmail that accepts emails without a TLD
B) Migrate the entire authentication system from JWTs to session tokens (affects 15+ files)
C) Add a console.log statement to debug a specific function
D) Restructure the project from monolith to microservices

Give me the criteria you're using to decide.
```

**Checkpoint:**
- A) Direct execution — single file, clear change
- B) Plan mode — multi-file, architectural implications
- C) Direct execution — trivial, well-scoped
- D) Plan mode — large-scale, multiple valid approaches, architectural decisions

---

## Domain 4 Review: Prompt Engineering

### Conversation 10 — Structured Output Mechanics

**Prompt:**
```
I need guaranteed schema-compliant JSON output from Claude.

I have three options:
A) Instruct Claude in the prompt to output JSON
B) Use tool_use with a JSON schema
C) Set tool_choice: "auto" with a tool defined

Which option gives the strongest guarantee? What's the difference between "auto", "any", and forced tool selection?
```

**Checkpoint:**
- Option B with forced tool selection: strongest guarantee
- "auto": model may return text instead of calling a tool
- "any": model must call a tool but can choose which (useful when multiple extraction schemas exist and document type is unknown)
- Forced `{"type": "tool", "name": "..."}`: model must call that specific tool

---

### Conversation 11 — Validation Retry When Retries Are Futile

**Prompt:**
```
My contract extraction system keeps retrying when it can't extract contract_value.

The contract references "Exhibit A for pricing details" but Exhibit A is not provided.

Will retrying help? What should the system do instead?
```

**Checkpoint:** Retries are futile — the information is not in the document. Retrying with the same document won't produce the missing information. The system should return `null` for `contract_value` (nullable field) and document why the field is empty, rather than retrying.

---

### Conversation 12 — Batch API Constraints

**Prompt:**
```
My team wants to use the Message Batches API for cost savings. Give me the three most important constraints to evaluate before deciding.
```

**Checkpoint:**
1. Up to 24-hour processing window — no guaranteed latency SLA (rules out blocking workflows)
2. Does not support multi-turn tool calling within a single batch request
3. 50% cost savings — significant for latency-tolerant workloads

Use case fit: overnight reports, weekly audits, nightly test generation. Does not fit: blocking pre-merge checks.

---

## Domain 5 Review: Context & Reliability

### Conversation 13 — Context Degradation Patterns

**Prompt:**
```
What are the two main ways context degrades in long Claude Code sessions?

For each one, what is the recommended mitigation?
```

**Checkpoint:**
1. **Context fills with verbose tool output** → /compact to compress; Explore subagent to isolate verbose phases; trim tool outputs to relevant fields
2. **"Lost in the middle" effect** → place key findings at the beginning of inputs, use explicit section headers, extract critical facts into a persistent "case facts" block

---

### Conversation 14 — Provenance in Multi-Agent Systems

**Prompt:**
```
My synthesis agent combines findings from web search and document analysis agents.

After synthesis, the final report has no source citations — it just states facts without attribution.

Where in the pipeline is attribution being lost? How do you fix it?
```

**Checkpoint:** Attribution is lost during the summarization/synthesis step when findings are combined without preserving claim-source mappings. Fix: require subagents to return structured outputs with claim-source pairs (source URL, document name, page number, publication date). The synthesis agent must preserve these mappings when combining — not merge them into undifferentiated prose.

---

### Conversation 15 — Aggregate Accuracy Trap

**Prompt:**
```
My extraction system reports 97% overall accuracy. I propose reducing human review to only the 3% of extractions the system flags as low-confidence.

My teammate says this is premature. What is their argument and are they right?
```

**Checkpoint:** They are right. 97% overall accuracy can mask poor performance on specific document types or fields. The system might be 99.9% accurate on standard invoices but 72% accurate on complex multi-page contracts. Without segmented accuracy analysis (by document type, by field), reducing review across the board would miss errors in the worst-performing segments. Solution: validate accuracy by document type and field before reducing review.

---

## Session Debrief

**Exam-Ready Checklist:**

After completing this session, you should be able to quickly answer:

- [ ] What are the two stop_reason values and what do they trigger?
- [ ] Where does the root cause of coverage gaps usually live in a multi-agent system?
- [ ] Why are hooks stronger than prompt instructions for business rule compliance?
- [ ] What is the most effective first fix for tool selection misrouting?
- [ ] What is the difference between `.mcp.json` and `~/.claude.json`?
- [ ] When is plan mode vs direct execution appropriate?
- [ ] What does `context: fork` do in a SKILL.md?
- [ ] What is the difference between "any" and forced tool selection in tool_choice?
- [ ] When is retry-with-feedback effective? When is it futile?
- [ ] What does the Message Batches API's "no guaranteed latency SLA" mean in practice?
- [ ] What is the "lost in the middle" effect and how do you mitigate it?
- [ ] Why does aggregate accuracy mask poor performance on specific segments?

**If any checkbox is unclear:** Go back to the relevant domain study notes or scenario workbook for that topic.
