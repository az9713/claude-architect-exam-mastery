# Interactive Session: Agentic Loop and Orchestration

**Surface:** Claude Code CLI
**Duration:** 45–60 minutes
**Task Statements:** 1.1, 1.2, 1.3

---

## Learning Objectives

By the end of this session, you will be able to:
- Observe stop_reason-driven loop control in practice
- Understand how tool results are appended to conversation history
- Recognize the coordinator-subagent pattern and context isolation
- Design task decomposition for parallel subagent execution

---

## Prerequisites

- Claude Code CLI installed (`claude --version` should return a version)
- API key configured (`echo $ANTHROPIC_API_KEY` should return a value)
- A project directory with at least a few files

---

## Session Steps

### Step 1 — Single Tool Call Loop

Start Claude Code in your project directory:
```bash
cd /your/project
claude
```

**Prompt 1:**
```
What is the total number of TypeScript files in this project? Count them using tools.
```

**Observe:** Claude should use Glob or Bash to count files. Watch the interaction — Claude makes a tool call, receives the result, and produces a final answer.

**Checkpoint 1:** How many iterations did the loop take? What was the `stop_reason` that ended the loop? (It should be "end_turn" after the tool result was incorporated.)

---

### Step 2 — Multi-Step Tool Chain

**Prompt 2:**
```
Find the largest TypeScript file in this project (by line count), read it, and tell me what the file's primary purpose is.
```

**Observe:** This requires multiple tool calls:
1. Glob to find TypeScript files
2. Bash or multiple Reads to check line counts
3. Read to load the largest file
4. Analysis and response

**Observe the chain:** Each tool call result is processed, and the next tool call is determined based on what was learned. The loop continues as long as `stop_reason == "tool_use"`.

**Checkpoint 2:** Did Claude need to read each file to find the largest? Or did it find a smarter approach (Bash wc -l)?

---

### Step 3 — Observe Context Accumulation

**Prompt 3:**
```
Search for all files that import from 'axios' or 'fetch', read each one briefly, and give me a summary of how HTTP calls are made in this codebase.
```

**Observe:** If there are many files, the context accumulates tool results. Watch for:
- How many Grep results appear
- How many Read operations are performed
- Whether Claude reads all files or samples strategically

**Checkpoint 3:** If there are 10+ files that match, does Claude read all of them? Or does it sample and acknowledge the limitation? This is the "incremental understanding" pattern from Task Statement 2.5.

---

### Step 4 — Coordinator Pattern with Subagent-Style Delegation

Claude Code does not expose raw Task tool calls in the UI, but you can practice the coordinator decomposition pattern by giving Claude a task that requires it to plan subagent work:

**Prompt 4:**
```
I want to understand this codebase well enough to safely refactor the authentication module. Please:
1. First, map the overall structure (what modules exist and their relationships)
2. Then, specifically trace the authentication flow
3. Finally, identify the files that would be affected by authentication refactoring

Do this in phases — complete each phase before moving to the next.
```

**Observe:** Claude should naturally decompose this into phases, completing structural mapping before tracing auth flow. Each phase builds on the previous.

**Checkpoint 4:** Notice how each phase uses the knowledge gathered in previous phases. This is the coordinator pattern applied to a sequential investigation.

---

### Step 5 — Explicit Context Passing Experiment

**Prompt 5:**
```
I'm going to ask you a question that requires combining two separate pieces of research.

First: find and read the README.md or main entry point to understand what this project does.

Second: find the test files and read one of them to understand the testing approach.

Then answer: if I wanted to add a new feature to this project, what would be the complete workflow — from implementation to test?

The key: both pieces of research must inform your answer. Show how you combined the findings.
```

**Observe:** This exercises the pattern from Task Statement 1.3 — combining findings from separate research activities. The "answer" should cite specific things from both the README and the test file.

**Checkpoint 5:** Did Claude correctly integrate findings from both sources? Did it acknowledge what it found in each?

---

### Step 6 — Parallel vs Sequential Investigation

**Prompt 6:**
```
I need to understand three independent things about this codebase simultaneously:
1. How errors are handled (what error handling patterns exist)
2. How the database is accessed (what ORM or query patterns are used)
3. How configuration is loaded (env vars, config files, etc.)

These are independent questions — investigate them in parallel if possible, and report findings for all three.
```

**Observe:** In Claude Code's interactive mode, watch whether Claude investigates these sequentially or interleaves tool calls. Parallel tool execution would investigate all three simultaneously.

**Checkpoint 6:** Were the investigations sequential or parallel? How does this relate to the Task tool's parallel subagent spawning pattern described in the exam guide?

---

### Step 7 — Stop Reason Demonstration

**Prompt 7:**
```
Check if this project has a package.json and if so, what Node.js version is required.
```

If there is no package.json, Claude should answer directly without tool calls — `stop_reason: "end_turn"` without any `tool_use` blocks.

If there is a package.json, Claude will read it — one iteration of the loop.

**Prompt 7b:**
```
Find all dependencies that have security vulnerabilities. Check the package.json, the package-lock.json, and any audit reports.
```

**Observe:** Multiple tool calls, multiple loop iterations.

**Checkpoint 7:** In Prompt 7, how did the loop terminate differently when no tool calls were needed vs when they were?

---

### Step 8 — Session Fork Pattern

Start a new investigation session, build up some context, then practice the fork concept:

**Prompt 8:**
```
I've analyzed this codebase. Now I want to explore two different approaches to refactoring the main module:

Approach A: Extract business logic into services, keep controllers thin
Approach B: Keep current structure but add a middleware layer for cross-cutting concerns

What are the key trade-offs between these two approaches for this specific codebase? Don't implement either yet — just analyze.
```

**Observe:** This represents the scenario where you would want to `fork_session` — explore each approach from the shared analysis baseline without mixing their contexts.

**Checkpoint 8:** If you were to actually implement Approach A, would you want to do it in this session (which also has Approach B analysis in context) or start fresh? When would you use `fork_session` vs starting a new session?

---

### Step 9 — Iterative Refinement on a Real Task

**Prompt 9:**
```
I want to add input validation to the user registration endpoint. Before you write any code, ask me questions to surface any design decisions I haven't thought through yet.
```

**Observe:** The interview pattern — Claude asks about validation rules, error messages, whether to validate on the client or server, partial validation vs all-or-nothing, etc.

Answer Claude's questions, then let it implement. If the implementation has issues:
```
The email validation is too strict — it's rejecting plus-addressed emails like "user+tag@domain.com". Fix just the email regex without changing the other validations.
```

**Observe:** Sequential iteration for an independent issue (email regex doesn't interact with other validations).

**Checkpoint 9:** Did the interview pattern surface considerations you hadn't specified? Did sequential iteration work for the independent issue?

---

### Step 10 — Variation: Session Resumption

```bash
# Start and name a session
claude --session-name "refactor-investigation"

# Do some analysis...
# Exit

# Resume next time
claude --resume "refactor-investigation"
```

**Prompt 10:**
```
What did we determine in the previous session about the authentication module?
```

**Observe:** The resumed session should have access to the previous conversation context. If the codebase changed since the last session, Claude's prior tool results may be stale.

**Checkpoint 10:** When would you start a fresh session with an injected summary instead of resuming? What makes a prior session's context "stale"?

---

## Session Debrief

**Key Takeaways:**

1. **stop_reason is the termination signal** — the loop continues on "tool_use" and terminates on "end_turn"
2. **Tool results are appended to conversation history** — Claude sees its prior tool results when deciding the next action
3. **Subagents need explicit context** — they cannot inherit what the coordinator knows; it must be passed in the prompt
4. **Parallel Task calls** in a single response enable concurrent execution; sequential calls create serial dependencies
5. **fork_session** enables exploring multiple approaches from a shared baseline without context mixing

**Exam Connection:**
- Task Statement 1.1: stop_reason handling, tool result appending, loop termination
- Task Statement 1.2: coordinator hub-and-spoke, task decomposition
- Task Statement 1.3: explicit context passing, parallel vs sequential Task calls
