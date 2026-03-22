# Quick Wins: Last-Minute Power Review
**Claude Certified Architect – Foundations**

*Read this in the 30 minutes before your exam. Everything on this page is testable.*

---

## The 10 Most Important Configuration Paths

| Path | What It Is | Shared? |
|------|-----------|---------|
| `~/.claude/CLAUDE.md` | User-level instructions | No — personal only |
| `.claude/CLAUDE.md` or `CLAUDE.md` | Project-level instructions | Yes — git/version control |
| `subdirectory/CLAUDE.md` | Directory-scoped instructions | Yes — applies to that dir only |
| `.claude/rules/topic.md` | Path-scoped rule files (YAML frontmatter) | Yes — git |
| `.claude/commands/cmd.md` | Project slash commands | Yes — git, all developers |
| `~/.claude/commands/cmd.md` | Personal slash commands | No — personal only |
| `.claude/skills/skill.md` | Skill definition files | Yes — git |
| `~/.claude/skills/variant.md` | Personal skill variants | No — personal only |
| `.mcp.json` | Project MCP server config | Yes — git, all developers |
| `~/.claude.json` | User MCP server config | No — personal/experimental |

**The Rule**: Everything in `~/.claude/` or `~/` = personal, not shared. Everything in `.claude/` within the repo = shared via git.

---

## The 5 `tool_choice` Values

| Value | Behavior | Use When |
|-------|----------|----------|
| `{"type": "auto"}` | Model may call a tool OR return text | Default; conversational fallback OK |
| `{"type": "any"}` | Model MUST call some tool | Need guaranteed structured output; multiple schemas |
| `{"type": "tool", "name": "X"}` | Model MUST call this specific tool | Specific extraction must run first |
| *(not set)* | Defaults to `"auto"` | Same as `"auto"` |

**Memory trick**: `"any"` = call *a* tool (your choice); forced = call *this* tool (my choice).

---

## The 4 MCP Error Categories

| Category | Meaning | `isRetryable` |
|----------|---------|---------------|
| `transient` | Timeout, service unavailable | `true` — retry after delay |
| `validation` | Invalid input, wrong format | `false` — fix input first |
| `permission` | Access denied, unauthorized | `false` — escalate or change credentials |
| `business` | Policy violation, limit exceeded | `false` — escalate to human or inform user |

**Memory trick**: Only transient errors are retryable. All others require a different action.

**Required error response fields**: `errorCategory`, `isRetryable`, `description` + `isError: true`

---

## The 3 Hook Patterns That Provide Deterministic Guarantees

| Hook | When It Fires | Use For |
|------|--------------|---------|
| `PreToolUse` (tool call interception) | Before outgoing tool call executes | Block policy-violating actions (e.g., refund > $500) |
| `PostToolUse` | After tool returns, before model sees result | Normalize heterogeneous data formats (timestamps, status codes) |
| Prerequisite Gate | Blocks downstream calls until prerequisite completes | Identity verification before financial operations |

**The key principle**: Hooks = deterministic. Prompts = probabilistic. Financial/security rules always need hooks.

---

## The 2 API Modes

| Mode | When to Use | Key Limitation |
|------|------------|----------------|
| **Synchronous** (real-time) | Blocking workflows, human waiting, pre-merge checks | Full cost |
| **Message Batches API** | Overnight, non-blocking, latency-tolerant | Up to 24h, no latency SLA, **no multi-turn tool calling** |

**Batch API memory trick**: "50% savings, 24 hours, no multi-turn." Three numbers, pass the batch API questions.

**Never use batch for**: pre-merge checks, real-time responses, anything with a < 24h SLA.

---

## The 1 Golden Rule

> **"Programmatic enforcement (hooks, prerequisite gates) for anything with financial, security, or irreversible consequences. Prompt instructions for preferences and style."**

This single rule answers at least 2–3 questions per exam. When you see:
- "in X% of cases the agent skips..." → programmatic prerequisite
- "refunds above $500 must be escalated" → PreToolUse hook
- "customer verification before any financial operation" → programmatic gate

Prompt instructions have a non-zero failure rate. Zero-failure requirements need code-level enforcement.

---

## CLI Flags Reference

| Flag | Meaning | Without It |
|------|---------|------------|
| `-p` or `--print` | Non-interactive (print) mode | Hangs waiting for user input in CI |
| `--output-format json` | Output as JSON | Returns plain text |
| `--json-schema ./schema.json` | Enforce JSON schema on output | No schema enforcement |
| `--resume <session-name>` | Resume named prior session | Starts fresh session |

**The CI command**: `claude -p "Your prompt" --output-format json --json-schema ./schema.json`

---

## Key Distinctions to Get Right

| These are DIFFERENT | Don't Confuse Them |
|--------------------|--------------------|
| `stop_reason == "tool_use"` (continue loop) | `stop_reason == "end_turn"` (terminate loop) |
| Empty result `[]` (success) | Access failure / timeout (error) |
| Access failure (retryable) | Business error (not retryable) |
| Project-level config (shared) | User-level config (personal) |
| `context: fork` (isolated skill) | No `context:` (inline in main session) |
| Syntax error (eliminated by tool_use) | Semantic error (requires validation logic) |
| Batch API (no multi-turn tool use) | Synchronous API (supports tool use loops) |
| Tool descriptions (for selection) | Tool schemas (for input structure) |
| Scratchpad files (persist findings) | Progressive summarization (destroys precision) |
| Explicit escalation request (immediate) | Frustrated customer (offer resolution first) |

---

## The 5 Distractor Patterns (Instant Elimination)

Seeing these signals → eliminate the option:

1. **"deploy a separate classifier / train a model"** → Over-Engineering; simpler first step exists
2. **"sentiment analysis to detect frustration → escalate"** → Wrong Problem; sentiment ≠ complexity
3. **"enhance the system prompt to state it's mandatory"** → Insufficient Guarantee; needs programmatic enforcement
4. **"~/.claude/commands/"** (when team-wide sharing is required) → Wrong Scope; use `.claude/commands/`
5. **"proactively cache / accumulate and batch at end"** (when items have dependencies) → Premature Optimization; fails when later steps depend on earlier verified facts

---

## Escalation: Quick Decision Tree

```
Customer says "I want a human"?
  YES → Escalate immediately. No investigation first.
  NO  ↓

Policy gap or exception needed?
  YES → Escalate with context (what was requested, why policy doesn't cover it)
  NO  ↓

Unable to make progress after attempts?
  YES → Escalate with summary of what was tried
  NO  ↓

Customer is frustrated/angry?
  → Acknowledge frustration, OFFER resolution first
  → Only escalate if they reiterate the request for a human
```

**Never escalate based on**: sentiment level, self-reported confidence score, perceived complexity.

---

## Schema Design Quick Rules

| Situation | Schema Rule |
|-----------|------------|
| Field might be absent from source doc | `"type": ["string", "null"]` |
| Category with unanticipated values | `"enum": [..., "other"]` + `"detail": {"type": ["string", "null"]}` |
| Ambiguous input | Add `"unclear"` to enum options |
| Arithmetic cross-validation needed | Include `calculated_total` + `stated_total` + `totals_match` fields |
| Conflicting source values | Include `conflict_detected` + `conflict_description` fields |

---

## Subagent Context: Always Explicit

**What subagents DO NOT get automatically:**
- Coordinator's conversation history
- Prior subagent findings
- Parent agent's context
- Shared memory between invocations

**What you must do:**
- Include relevant findings directly in the subagent's prompt
- Pass structured data (claim + source URL + date + excerpt) for provenance
- Specify research goals and quality criteria (not step-by-step instructions)

---

## Parallel vs. Sequential Subagents

```
PARALLEL:  Coordinator emits TWO Task tool calls in ONE response message
SEQUENTIAL: Coordinator emits Task calls across separate turns (one at a time)
```

For parallel execution: the coordinator must emit multiple Task calls in a single response. Separate turns = sequential execution.

---

## Built-In Tool Selection (30-Second Reminder)

| Goal | Tool |
|------|------|
| Find files by name/extension pattern | `Glob("**/*.test.tsx")` |
| Find files by content | `Grep(pattern="function processRefund")` |
| Read a file | `Read` |
| Make targeted edit (unique anchor) | `Edit` |
| Edit fails (non-unique anchor) | `Read` → modify → `Write` |
| Shell command | `Bash` |

---

## You're Ready

- 720/1000 = ~72% correct — you have a meaningful buffer
- Read the question stem before the scenario
- Eliminate distractors using the 5 patterns above
- No penalty for guessing — answer every question
- Trust your first instinct when 70%+ confident
