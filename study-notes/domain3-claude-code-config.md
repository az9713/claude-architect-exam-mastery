# Domain 3: Claude Code Configuration & Workflows
**Exam Weight: 20% | Task Statements: 3.1 – 3.6**

---

## Task Statement 3.1: Configure CLAUDE.md files with appropriate hierarchy, scoping, and modular organization

### What You Need to Know
- **CLAUDE.md hierarchy** (four levels, applied in order):
  1. **Managed policy**: `/Library/Application Support/ClaudeCode/CLAUDE.md` (macOS) or `/etc/claude-code/CLAUDE.md` (Linux) — organization-wide, managed by IT, CANNOT be excluded by users
  2. **User-level**: `~/.claude/CLAUDE.md` — applies to that user only, NOT shared via version control
  3. **Project-level**: `.claude/CLAUDE.md` or root `CLAUDE.md` — committed to repo, shared with team
  4. **Directory-level**: subdirectory `CLAUDE.md` files — load on demand when Claude reads files in those subdirectories
- `@import` syntax for referencing external files; supports relative paths (resolved from importing file), absolute paths, home directory paths (`@~/.claude/my-instructions.md`), and recursive imports (max 5 hops)
- `.claude/rules/` directory for organizing topic-specific rule files as alternative to monolithic CLAUDE.md
- CLAUDE.md files are loaded as **user messages**, not as system prompt — they are guidance Claude can follow, not enforced config. Use `--append-system-prompt` for system-prompt-level instructions
- Target **under 200 lines** per CLAUDE.md — longer files reduce adherence
- `claudeMdExcludes` setting skips irrelevant CLAUDE.md files in monorepos
- `/init` command auto-generates a starting CLAUDE.md; set `CLAUDE_CODE_NEW_INIT=true` for interactive multi-phase generation
- CLAUDE.md fully survives `/compact` — it is re-read from disk after compaction, not lost with conversation history

### What You Need to Be Able to Do
- Diagnose configuration hierarchy issues (e.g., new team member not receiving instructions because they're in user-level, not project-level config)
- Use `@import` to selectively include relevant standards files in each package's CLAUDE.md
- Split large CLAUDE.md files into focused topic-specific files in `.claude/rules/` (e.g., `testing.md`, `api-conventions.md`, `deployment.md`)
- Use `/memory` command to verify which memory files are loaded, toggle auto memory, and diagnose inconsistent behavior
- Recognize when to use `--append-system-prompt` (CI, enforced instructions) vs. CLAUDE.md (team guidance)

### Deep Dive

**The Four-Level Hierarchy**

```
/etc/claude-code/CLAUDE.md             # Managed policy (org-wide, IT-managed, cannot exclude)
~/.claude/CLAUDE.md                    # User-level (NOT in repo; personal only)
./CLAUDE.md  or  ./.claude/CLAUDE.md   # Project-level (committed to git; whole team)
packages/backend/CLAUDE.md             # Directory-level (on demand when files read there)
packages/frontend/CLAUDE.md            # Directory-level (on demand when files read there)
```

Key loading behavior:
- Managed policy always applies; users cannot skip it
- Directory-level files load **on demand** (when Claude reads a file in that directory), not at session start
- All levels merge additively; higher levels take precedence on conflicts

**Common Hierarchy Diagnostic**

- **Problem:** "New developer joined; Claude doesn't follow our code review standards"
  - Diagnosis: Standards are in `~/.claude/CLAUDE.md` (user-level); new developer doesn't have it
  - Fix: Move standards to project-level `.claude/CLAUDE.md` or root `CLAUDE.md`

- **Problem:** "Claude behavior differs between team members"
  - Diagnosis: Some instructions are in user-level config
  - Use `/memory` to see which files are loaded for the current session

- **Problem:** "Backend and frontend have different conventions; CLAUDE.md is huge and adherence is slipping"
  - Fix: Split into directory-level CLAUDE.md files per package with `@import`; keep each under 200 lines

- **Problem:** "In a monorepo, a package's CLAUDE.md is loaded even when working in an unrelated package"
  - Fix: Add that path to `claudeMdExcludes` in settings

- **Problem:** "Instructions in CLAUDE.md are ignored after I use /compact"
  - Clarification: This is a misconception — CLAUDE.md is re-read from disk after compaction; it is not lost

**Modular Organization with `.claude/rules/`**

```
.claude/
├── CLAUDE.md          # Root configuration (imports or references rules)
└── rules/
    ├── testing.md            # Test framework, coverage requirements
    ├── api-conventions.md    # REST conventions, versioning, auth patterns
    ├── deployment.md         # CI/CD procedures, environment handling
    └── security.md           # Auth, secrets handling, OWASP reminders
```

```markdown
@import .claude/rules/testing.md
@import .claude/rules/api-conventions.md
```

> **Exam Tip:** Hierarchy diagnosis questions always have one correct answer tied to scope. "New team member doesn't get the setting" = the setting is at user-level. "Rules apply everywhere but shouldn't" = needs directory-level CLAUDE.md. The `/memory` command confirms what's loaded.

> **Anti-Pattern Alert:** Putting all conventions in one massive CLAUDE.md is a maintainability anti-pattern. It also loads all context every session even when irrelevant. Use `.claude/rules/` with topic-specific files.

> **Cross-Domain Connection:** Task 3.6 uses CLAUDE.md to provide project context (testing standards, fixture conventions) to CI-invoked Claude Code — same project-level configuration mechanism.

---

## Task Statement 3.2: Create and configure custom slash commands and skills

### What You Need to Know
- **Project-scoped commands**: `.claude/commands/` — shared via version control, available to all developers
- **User-scoped commands**: `~/.claude/commands/` — personal only, not shared
- **Skills**: `.claude/skills/` with `SKILL.md` files supporting YAML frontmatter: `context: fork`, `allowed-tools`, `argument-hint`
- `context: fork` runs the skill in an isolated sub-agent context, preventing skill output from polluting the main conversation
- Personal skill customization: create personal variants in `~/.claude/skills/` with different names to avoid affecting teammates

### What You Need to Be Able to Do
- Create project-scoped slash commands in `.claude/commands/` for team-wide availability
- Use `context: fork` to isolate skills that produce verbose output or exploratory context
- Configure `allowed-tools` in skill frontmatter to restrict tool access during skill execution
- Use `argument-hint` frontmatter to prompt developers for required parameters
- Choose between skills (on-demand, task-specific) and CLAUDE.md (always-loaded, universal standards)

### Deep Dive

**Slash Command Structure**

```
.claude/
└── commands/
    ├── review.md         # /review command
    ├── test.md           # /test command
    └── deploy-check.md   # /deploy-check command
```

```markdown
<!-- .claude/commands/review.md -->
Run our standard code review checklist on the current changes:

1. Check for security issues (SQL injection, XSS, insecure dependencies)
2. Verify error handling on all external calls
```

**Skills with YAML Frontmatter**

```markdown
<!-- .claude/skills/analyze-codebase.md -->
---
context: fork
allowed-tools: ["Read", "Grep", "Glob"]
argument-hint: "Component or module to analyze (e.g., 'auth module', 'payment service')"
---

Analyze the architecture of {{argument}}. Produce:
1. Key files and their responsibilities
2. Dependency graph (what depends on what)
3. Data flow overview
4. Potential architectural concerns

Summarize findings concisely for the main session.
```

**Frontmatter Options Explained**

| Key | Purpose | When to Use |
|---|---|---|
| `context: fork` | Isolated sub-agent | Verbose output would pollute main conversation |
| `allowed-tools` | Restricts tool access | Read-only analysis shouldn't allow Write/Bash |
| `argument-hint` | Prompts for parameter | Parameterized tasks needing user input |
| `disable-model-invocation` | No AI processing | Pure template expansion |

**When Skills vs. CLAUDE.md**

| | CLAUDE.md | Skills |
|---|---|---|
| **Loading** | Always loaded automatically | Invoked on demand |
| **Best for** | Universal standards, conventions, security requirements | Task-specific workflows, verbose analysis, parameterized tasks |
| **Context impact** | Always in context | Isolated when `context: fork` |
| **Examples** | Coding style, framework preferences | `/review`, `/analyze`, `/document` |

**Skill Invocation Control**

The frontmatter fields `disable-model-invocation` and `user-invocable` control who can trigger a skill and when its description is loaded:

| Frontmatter | User invokes? | Claude invokes? | Description loading |
|---|---|---|---|
| (default) | Yes | Yes | Description always loaded; full content on invocation |
| `disable-model-invocation: true` | Yes | No | Description NOT loaded; full content only when user invokes |
| `user-invocable: false` | No | Yes | Description always loaded; full content when Claude invokes |

Key exam pattern: "Skill available to all devs on clone" → `.claude/commands/` or `.claude/skills/`. "Personal only" → `~/.claude/commands/` or `~/.claude/skills/`.

**Commands and Skills Unified**

Commands have been merged into skills — `.claude/commands/deploy.md` and `.claude/skills/deploy/SKILL.md` both create the `/deploy` command. Existing command files keep working without changes; skills are the recommended approach going forward because they support additional features (frontmatter fields, invocation control, `context: fork`).

**Personal Skill Customization**

Create a personal variant in `~/.claude/skills/` using a different name — this ensures teammates are not affected by personal preference changes to team skills.

> **Exam Tip:** "Command available to all developers when they clone the repo" = `.claude/commands/` (project-scoped). "Personal command only this developer uses" = `~/.claude/commands/` (user-scoped). Same scoping logic as CLAUDE.md and MCP servers.

> **Anti-Pattern Alert:** Putting verbose exploratory skills without `context: fork` fills the main conversation with discovery noise. Use `context: fork` for any skill that generates output longer than a few paragraphs.

> **Cross-Domain Connection:** Task 3.4 covers the Explore subagent — `context: fork` in skills is the Claude Code equivalent of delegating verbose exploration to a subagent in the Agent SDK.

---

## Task Statement 3.3: Apply path-specific rules for conditional convention loading

### What You Need to Know
- `.claude/rules/` files support **YAML frontmatter `paths` fields** with glob patterns for conditional rule activation
- Path-scoped rules load **only when editing matching files**, reducing irrelevant context and token usage
- Glob-pattern rules are better than directory-level CLAUDE.md files for conventions that span multiple directories (e.g., test files spread throughout a codebase)

### What You Need to Be Able to Do
- Create `.claude/rules/` files with YAML frontmatter path scoping (e.g., `paths: ["terraform/**/*"]`)
- Use glob patterns to apply conventions to files by type regardless of directory location (e.g., `**/*.test.tsx`)
- Choose path-specific rules over subdirectory CLAUDE.md files when conventions must apply to files spread across the codebase

### Deep Dive

**Path-Specific Rule File Structure**

```markdown
<!-- .claude/rules/testing.md -->
---
paths:
  - "**/*.test.ts"
  - "**/*.test.tsx"
  - "**/*.spec.ts"
  - "tests/**/*"
---

## Testing Conventions

Use Jest with React Testing Library for all frontend tests.
- Use `describe` blocks to group related tests
- Test IDs: use `data-testid` not CSS selectors
- Mock external APIs using `msw`
- Snapshot tests are discouraged; prefer explicit assertions
- Minimum coverage: 80% for new code paths
```

The same pattern applies for Terraform, API routes, migration files, and any other file-type convention that spans multiple directories.

**Unconditional vs. Path-Scoped Loading**

Rules files in `.claude/rules/` without a `paths` field load unconditionally at session launch — they behave identically to content in `.claude/CLAUDE.md`. Only rules files with a `paths` frontmatter field are conditional. Path-scoped rules trigger when Claude reads files matching the glob pattern, not on every tool use — so a rule scoped to `**/*.test.tsx` activates the first time Claude opens a test file, not when it runs Bash or edits a source file.

**Path-Specific vs. Directory CLAUDE.md**

Scenario: Test conventions that apply to `*.test.tsx` files spread across `tests/`, `src/components/__tests__/`, `src/api/__tests__/`, and `packages/backend/tests/`.

- **Option A — Directory CLAUDE.md in each test directory:** requires 4 separate files, each kept in sync manually; adding a new test directory needs another CLAUDE.md.
- **Option B — `.claude/rules/testing.md` with `paths: ["**/*.test.tsx"]`:** single file, one glob pattern, automatically applies to any test file wherever located. Preferred approach for cross-directory conventions.

> **Exam Tip:** When the question describes a convention that applies to a file type across multiple directories (test files, config files, migration files), the answer is always a `.claude/rules/` file with a glob pattern — not multiple directory-level CLAUDE.md files.

> **Anti-Pattern Alert:** Creating a directory-level CLAUDE.md in every test directory is duplication. One path-scoped rule handles all test files in one place, maintainable from a single source.

> **Cross-Domain Connection:** Task 3.1 covers the overall CLAUDE.md hierarchy — path-specific rules in `.claude/rules/` are a layer within that hierarchy, more granular than directory-level files.

---

## Task Statement 3.4: Determine when to use plan mode vs direct execution

### What You Need to Know
- **Plan mode**: designed for complex tasks — large-scale changes, multiple valid approaches, architectural decisions, multi-file modifications
- **Direct execution**: appropriate for simple, well-scoped changes (single file, clear scope, no architectural decisions)
- Plan mode enables safe codebase exploration and design before committing to changes
- **Explore subagent**: isolates verbose discovery output and returns summaries to preserve main conversation context

### What You Need to Be Able to Do
- Select plan mode for tasks with architectural implications (microservice restructuring, library migrations affecting 45+ files)
- Select direct execution for well-understood changes (single-file bug fix with clear stack trace, adding a date validation conditional)
- Use the Explore subagent for verbose discovery phases to prevent context window exhaustion
- Combine plan mode for investigation with direct execution for implementation

### Deep Dive

**Plan Mode Decision Criteria**

```
Use PLAN MODE when ANY of these are true:
  ✓ Changes span multiple files (5+)
  ✓ There are multiple valid architectural approaches
  ✓ The task requires understanding existing dependencies first
  ✓ Getting it wrong would require significant rework
  ✓ Architectural decisions must be made (not just coding decisions)

Examples:
  → "Migrate the app from Express to Fastify" — plan mode
  → "Restructure the codebase into microservices" — plan mode
  → "Add comprehensive tests to a legacy codebase" — plan mode
  → "Migrate from library v1 to v3 across 45+ files" — plan mode

Use DIRECT EXECUTION when ALL of these are true:
  ✓ Scope is clear and bounded (1-3 files)
  ✓ No architectural decisions required
  ✓ The approach is obvious from the problem statement
  ✓ Mistakes are easily reversible

Examples:
  → "Fix the null pointer error on line 47 in payment.ts" — direct execution
  → "Add email format validation to the registration form" — direct execution
  → "Update the API timeout from 5s to 10s" — direct execution
```

**Built-in Subagent Types**

| Subagent | Model | Tools Available | Primary Use |
|---|---|---|---|
| Explore | Haiku (fast, lightweight) | Read-only (Read, Grep, Glob) | File discovery and codebase search |
| Plan | Inherits main model | Read-only | Research and investigation during plan mode |
| general-purpose | Inherits main model | All tools | Complex multi-step tasks requiring writes or Bash |

The Explore subagent's Haiku model and read-only constraint make it safe and cheap for high-volume discovery. The general-purpose subagent gets the full tool set but should only be spawned when write operations or shell commands are actually needed.

**The Explore Subagent Pattern**

```
Problem: Multi-phase tasks where phase 1 is verbose discovery
  → Reading 50+ files fills the context window
  → Main session loses coherence after discovery phase

Solution: Use Explore subagent for discovery phase
  1. Spawn Explore subagent with investigation goal
  2. Explore reads, searches, traces — verbose output stays isolated
  3. Explore returns a concise summary to the main agent
  4. Main agent retains clean context for planning and implementation

Pattern:
  PLAN MODE: investigate with Explore subagent
    → Explore: "Find all files that import the Auth module, trace dependencies"
    → Explore returns: "Auth module imported in 12 files across 3 services.
       Key dependencies: SessionStore, UserRepository, TokenValidator..."
  DIRECT EXECUTION: implement based on the plan
    → Now execute the planned changes with clean context
```

**Combined Plan + Execute Pattern**

Session 1 uses plan mode: the Explore subagent investigates the codebase, produces a migration plan with affected files, approach, step sequence, and risks. Session 2 uses direct execution: each step is implemented against a clean context with no verbose discovery history. Splitting sessions prevents the discovery phase from filling the context window before implementation begins.

> **Exam Tip:** Questions presenting large-scale restructuring (microservices, library migrations affecting 45+ files, architectural decisions with multiple approaches) → plan mode. Questions with "fix the bug on line X" or "add a single validation check" → direct execution. Architectural implications = plan mode trigger.

> **Anti-Pattern Alert:** Switching to plan mode for a single-file bug fix is overhead without value. Starting with direct execution on a 45-file architectural migration leads to costly rework.

> **Cross-Domain Connection:** Task 1.6 covers task decomposition strategies in the Agent SDK — the plan mode / direct execution decision mirrors prompt chaining (fixed sequential) vs. dynamic adaptive decomposition.

---

## Task Statement 3.5: Apply iterative refinement techniques for progressive improvement

### What You Need to Know
- **Concrete input/output examples** are the most effective way to communicate expected transformations when prose descriptions produce inconsistent results
- **Test-driven iteration**: write test suites first, then iterate by sharing test failures to guide improvement
- **Interview pattern**: have Claude ask questions to surface considerations you may not have anticipated before implementing
- When to provide all issues in a single message (interacting problems) vs. fix them sequentially (independent problems)

### What You Need to Be Able to Do
- Provide 2–3 concrete input/output examples to clarify transformation requirements when natural language produces inconsistent results
- Write test suites covering expected behavior, edge cases, and performance before implementation
- Use the interview pattern to surface design considerations before implementing in unfamiliar domains
- Provide specific test cases with example input and expected output to fix edge case handling (null values, etc.)
- Address multiple interacting issues in a single detailed message when fixes interact; use sequential iteration for independent issues

### Deep Dive

**When Prose Descriptions Fail: Use Examples**

Vague instructions ("convert the date format to a standardized format") produce inconsistent results. Specific descriptions ("convert to ISO 8601 YYYY-MM-DD") are better. The most reliable technique is concrete input/output examples:

```
Transform these date strings to ISO 8601 format. Examples:

Input: "March 5, 2026"  → Output: "2026-03-05"
Input: "05/03/2026"     → Output: "2026-03-05"
Input: null or "TBD"    → Output: null
```

Examples constrain ambiguous edge cases (e.g., whether `05/03` means May 3 or March 5) that prose cannot reliably resolve.

**Test-Driven Iteration**

Write the test suite first — covering expected behavior, edge cases, and error conditions. Run the tests before asking Claude to implement. Share specific test failures (test name, actual vs. expected output) with Claude rather than vague feedback like "it doesn't work." Claude iterates against precise failure signals, which is far more effective than prose descriptions of the desired behavior.

**The Interview Pattern**

Instruct Claude to ask questions before implementing ("Before building the caching layer, ask me 5-10 questions about requirements I might not have considered"). Claude surfaces considerations the developer may not have anticipated:

1. What cache invalidation strategy fits the update frequency?
2. Should the cache be in-process (memory) or distributed (Redis)?
3. What's the acceptable staleness window for different data types?
4. What happens if the cache becomes unavailable — fallback to DB or fail?

Developer answers ground the implementation in actual requirements. Without the interview, these questions surface only after costly rework.

**Single Message vs. Sequential Iteration**

- **Multiple interacting issues** — send in a single detailed message. Example: sort logic, null handling, and output format tests all touch the same code path; fixing them separately causes each fix to break the previous one.
- **Independent issues** — fix sequentially. Example: an off-by-one error in pagination and missing error handling in file upload can each be reviewed and validated independently. Batching unrelated issues creates review noise without benefit.

> **Exam Tip:** Questions about getting consistent output after repeated prompt attempts → the answer is concrete input/output examples. Questions about surfacing unstated requirements → interview pattern. The exam distinguishes "inconsistent output" (examples fix it) from "missing requirements" (interview pattern surfaces them).

> **Anti-Pattern Alert:** Trying to fix interacting issues sequentially causes the second fix to break the first. Identify when issues interact (they touch the same code path or have dependent behavior) and address them together.

> **Cross-Domain Connection:** Task 4.2 covers few-shot examples for output consistency — same principle as input/output examples for transformation tasks. Both use concrete examples to constrain ambiguous model behavior.

---

## Task Statement 3.6: Integrate Claude Code into CI/CD pipelines

### What You Need to Know
- `-p` (or `--print`) flag: runs Claude Code in **non-interactive mode** — essential for automated pipelines (prevents hanging waiting for input)
- `--output-format json`: structured JSON response with result, session ID, and metadata; combine with `--json-schema` to enforce a schema (result appears in `structured_output` field)
- `--bare`: skips hooks, skills, plugins, MCP servers, auto memory, and CLAUDE.md loading — produces identical results on every machine regardless of local configuration; use for reproducible CI baselines
- `--allowedTools`: auto-approves specific tools in CI without interactive prompts (uses permission rule syntax, e.g., `--allowedTools Bash(git:*)`)
- `--append-system-prompt`: adds CI-specific instructions at the system prompt level — unlike CLAUDE.md (user message), this cannot be overridden by model behavior
- **CLAUDE.md** provides project context (testing standards, fixture conventions, review criteria) to CI-invoked Claude Code
- **Session context isolation**: same Claude session that generated code is less effective at reviewing its own changes — use independent review instances

### What You Need to Be Able to Do
- Run Claude Code in CI with `-p` flag to prevent interactive input hangs
- Use `--output-format json` with `--json-schema` to produce machine-parseable structured findings for inline PR comments
- Include prior review findings in context when re-running reviews after new commits (report only new/still-unaddressed issues)
- Provide existing test files in context so test generation avoids duplicating already-covered scenarios
- Document testing standards, valuable test criteria, and available fixtures in CLAUDE.md to improve test generation quality

### Deep Dive

**Non-Interactive CI Mode**

```bash
# -p flag: non-interactive (print) mode — Claude answers and exits; no input wait
claude -p "Review this PR diff for security issues" \
  --output-format json \
  --json-schema ./schemas/review-findings.json \
  > review_output.json
```

**JSON Schema for Structured Review Output**

```json
{
  "findings": [{
    "file": "string",
    "line": "integer",
    "severity": "critical | high | medium | low",
    "description": "string"
  }]
}
```

**Avoiding Duplicate Review Comments**

When a developer pushes fixes after an initial review, include the prior review findings in the context of the follow-up invocation. Instruct Claude to report only issues not addressed in the prior review and new issues introduced by the latest changes. Without prior findings in context, Claude re-reports already-resolved issues as new findings.

**Session Context Isolation for Review**

Each review should use an independent Claude Code invocation with no shared session context from the code generation step. A session that generated the code retains its own reasoning, making it less likely to question its own decisions. Independent review instances produce more objective findings.

The correct CI pattern for generation + review is two separate `claude -p` invocations. The reviewer receives only the generated code as input — it has no access to the generator's intermediate reasoning, chain-of-thought, or design decisions. Use `--append-system-prompt "You have no context about how this code was written. Review it purely on its merits."` to reinforce isolation at the system prompt level.

**CLAUDE.md for CI Context**

The project-level CLAUDE.md provides context to CI-invoked Claude Code. Key sections for CI:

- Test framework, available fixtures, and files not to duplicate
- Explicit review criteria: what to flag (SQL without parameterization, missing try/except, hardcoded credentials) and what to skip (style/formatting handled by linter)
- Severity definitions so output is consistent across pipeline runs
- Priority areas for test generation (edge cases, null inputs, concurrent access)

> **Exam Tip:** `-p` flag is the only way to run Claude Code non-interactively in CI. Without it, Claude waits for interactive input and the pipeline hangs. This is a factual question with a single correct answer.

> **Anti-Pattern Alert:** Using the same Claude Code session that generated the code to review it is less effective than an independent instance. The generating session retains reasoning context that biases review toward confirmation of its own decisions.

> **Cross-Domain Connection:** Task 4.1 covers designing prompts with explicit criteria — the same explicit review criteria principles apply here when writing CLAUDE.md review instructions for CI.

---

## Key Terminology Quick Reference

| Term | Definition | Exam Relevance |
|------|-----------|----------------|
| `~/.claude/CLAUDE.md` | User-level configuration | Not shared via version control; personal only |
| `.claude/CLAUDE.md` | Project-level configuration | Shared via git; all developers receive it |
| Directory CLAUDE.md | Subdirectory-level rules | Applies only within that directory subtree |
| `@import` | Syntax to include external file in CLAUDE.md | Modular organization; import by context |
| `.claude/rules/` | Directory for topic-specific rule files | Alternative to monolithic CLAUDE.md |
| `paths:` frontmatter | YAML field in rules files for glob-based scoping | Loads rule only when editing matching files |
| `.claude/commands/` | Project-scoped slash commands | Version-controlled; available to whole team |
| `~/.claude/commands/` | User-scoped slash commands | Personal; not shared |
| `.claude/skills/` | Skill definition files with SKILL.md | Frontmatter: context, allowed-tools, argument-hint |
| `context: fork` | Frontmatter: runs skill in isolated sub-agent | Prevents verbose skill output from polluting main session |
| `allowed-tools` | Frontmatter: restricts tools during skill execution | Prevents destructive operations in read-only skills |
| `argument-hint` | Frontmatter: prompts for parameter when invoked without args | User-facing guidance for parameterized skills |
| Plan mode | Exploration before execution | Use for architectural/multi-file changes |
| Direct execution | Immediate implementation | Use for clear, bounded, single-file changes |
| Explore subagent | Isolates verbose discovery output | Returns summary; preserves main context |
| `-p` flag | Non-interactive (print) mode | Required for CI/CD pipeline use |
| `--output-format json` | Structured JSON output mode | Machine-parseable; combine with `--json-schema` for schema-enforced output |
| `--bare` | Skips hooks, skills, MCP, CLAUDE.md | Reproducible CI baselines; identical output on any machine |
| `--allowedTools` | Auto-approves named tools in CI | Prevents interactive permission prompts in pipelines |
| `--append-system-prompt` | Adds CI instructions at system prompt level | Stronger than CLAUDE.md (user message); cannot be overridden |
| `/memory` command | Shows which memory files are currently loaded | Diagnoses inconsistent configuration behavior |
| Interview pattern | Claude asks questions before implementing | Surfaces unstated requirements in unfamiliar domains |
| Test-driven iteration | Write tests first; iterate by sharing failures | Most effective for precise behavioral specification |

---

## Decision Matrix

### CLAUDE.md level for a given requirement

| Requirement | Level | File Location |
|-------------|-------|---------------|
| Universal team coding standards | Project-level | `.claude/CLAUDE.md` or `CLAUDE.md` |
| Personal response style preference | User-level | `~/.claude/CLAUDE.md` |
| Backend-specific Python conventions | Directory-level | `packages/backend/CLAUDE.md` |
| Test conventions for all test files everywhere | Path-scoped rule | `.claude/rules/testing.md` with `paths: ["**/*.test.ts"]` |
| Terraform conventions for infra files | Path-scoped rule | `.claude/rules/terraform.md` with `paths: ["terraform/**/*"]` |

### Skills vs. CLAUDE.md vs. Slash Commands

| Use Case | Mechanism | Reason |
|----------|-----------|--------|
| Code review checklist, always applied | CLAUDE.md | Always-loaded, universal |
| On-demand PR review workflow | Skill or command | Invoked when needed, not every session |
| Verbose codebase analysis that pollutes context | Skill with `context: fork` | Output isolated in sub-agent |
| Personal customization of a team skill | `~/.claude/skills/` variant | Different name; doesn't affect teammates |
| Team-wide /deploy-check command | `.claude/commands/deploy-check.md` | Version-controlled, shared |

### Plan mode vs. Direct execution

| Signals → Plan Mode | Signals → Direct Execution |
|--------------------|---------------------------|
| Multiple files (5+) | Single file |
| Architectural decisions | Implementation decisions only |
| Multiple valid approaches | Approach is obvious |
| Dependencies to map first | Clear stack trace / scope |
| "Restructure", "migrate", "refactor across" | "Fix", "add validation", "update timeout" |
| High rework cost if wrong | Easy to reverse |

---

## If You're Short on Time

1. **CLAUDE.md hierarchy**: user-level (`~/.claude/CLAUDE.md`) is personal/not-shared; project-level (`.claude/CLAUDE.md`) is team-wide via version control.
2. **`context: fork` in skills** isolates verbose skill output — use it for any skill generating more output than needed in the main session.
3. **Path-scoped rules** in `.claude/rules/` with `paths:` YAML frontmatter are better than directory CLAUDE.md for file-type conventions spanning multiple directories. Rules without a `paths` field load unconditionally at launch.
4. **`-p` flag** is required for non-interactive CI use. `--bare` skips all local config for reproducible baselines. `--allowedTools` prevents permission prompts.
5. **Plan mode** = architectural/multi-file complexity; **direct execution** = clear, bounded, single-file tasks. Explore subagent uses Haiku + read-only tools for cheap discovery.
6. **Two separate `claude -p` invocations** for generate + review — the reviewer must have no access to the generator's reasoning to produce objective findings.
