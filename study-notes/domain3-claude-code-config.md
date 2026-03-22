# Domain 3: Claude Code Configuration & Workflows
**Exam Weight: 20% | Task Statements: 3.1 – 3.6**

---

## Task Statement 3.1: Configure CLAUDE.md files with appropriate hierarchy, scoping, and modular organization

### What You Need to Know
- **CLAUDE.md hierarchy** (three levels, applied in order):
  1. **User-level**: `~/.claude/CLAUDE.md` — applies to that user only, NOT shared via version control
  2. **Project-level**: `.claude/CLAUDE.md` or root `CLAUDE.md` — committed to repo, shared with team
  3. **Directory-level**: subdirectory `CLAUDE.md` files — apply only within that directory subtree
- `@import` syntax for referencing external files to keep CLAUDE.md modular
- `.claude/rules/` directory for organizing topic-specific rule files as alternative to monolithic CLAUDE.md

### What You Need to Be Able to Do
- Diagnose configuration hierarchy issues (e.g., new team member not receiving instructions because they're in user-level, not project-level config)
- Use `@import` to selectively include relevant standards files in each package's CLAUDE.md
- Split large CLAUDE.md files into focused topic-specific files in `.claude/rules/` (e.g., `testing.md`, `api-conventions.md`, `deployment.md`)
- Use `/memory` command to verify which memory files are loaded and diagnose inconsistent behavior

### Deep Dive

**The Three-Level Hierarchy**

```
Repository structure:
├── CLAUDE.md                          # Project-level (or .claude/CLAUDE.md)
│                                      # Committed to git; all developers get this
├── ~/.claude/CLAUDE.md                # User-level (NOT in repo)
│                                      # Personal settings; NOT shared with teammates
├── packages/
│   ├── backend/
│   │   └── CLAUDE.md                  # Directory-level: backend-specific rules
│   │       @import ../../standards/python-style.md
│   │       @import ../../standards/api-contracts.md
│   └── frontend/
│       └── CLAUDE.md                  # Directory-level: frontend-specific rules
│           @import ../../standards/react-patterns.md
│           @import ../../standards/accessibility.md
└── standards/
    ├── python-style.md
    ├── api-contracts.md
    ├── react-patterns.md
    └── accessibility.md
```

**Common Hierarchy Diagnostic**

```
Problem: "New developer joined; Claude doesn't follow our code review standards"
Diagnosis: Check WHERE the standards are defined
  - If in ~/.claude/CLAUDE.md → user-level; new developer doesn't have it
  - Fix: Move standards to project-level .claude/CLAUDE.md or root CLAUDE.md

Problem: "Claude behavior differs between team members"
Diagnosis: Some instructions are in user-level config
  - Use /memory to see which files are loaded for the current session
  - User-level additions override or supplement project-level

Problem: "Backend and frontend have different conventions; CLAUDE.md is huge"
Fix: Use directory-level CLAUDE.md files per package with @import
```

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
# .claude/CLAUDE.md
This project uses Python 3.11 with FastAPI. See rules for specific conventions.

@import .claude/rules/testing.md
@import .claude/rules/api-conventions.md
@import .claude/rules/security.md
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
3. Confirm test coverage for new code paths
4. Validate API contracts match the openapi spec
5. Check for hardcoded credentials or secrets

Report findings as: [SEVERITY] file:line — description
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

```yaml
# context: fork — runs skill in isolated sub-agent
# Use when: skill generates verbose output that would pollute main conversation
# Example: codebase exploration, research, brainstorming alternatives
context: fork

# allowed-tools — restricts which tools the skill can use
# Use when: you want to prevent destructive operations during skill execution
# Example: read-only analysis should not allow Write or Bash
allowed-tools: ["Read", "Grep", "Glob"]

# argument-hint — shown when skill is invoked without arguments
# Prompts developer to provide required parameter
argument-hint: "Module name or file path to analyze"

# disable-model-invocation — prevents the skill from calling back to Claude
# Use for: pure template expansion without AI processing
disable-model-invocation: true
```

**When Skills vs. CLAUDE.md**

```
CLAUDE.md (always loaded):
  - Universal coding standards
  - Repository conventions
  - Security requirements
  - Language/framework preferences
  → Applied to EVERY interaction automatically

Skills (on-demand invocation):
  - Complex task-specific workflows (/review, /analyze, /document)
  - Verbose operations that would bloat main context (context: fork)
  - Parameterized tasks that need arguments
  → Invoked explicitly; output isolated when context: fork
```

**Personal Skill Customization**

```
~/.claude/skills/
└── analyze-codebase-verbose.md    # Personal variant: full verbose output (no fork)

.claude/skills/
└── analyze-codebase.md            # Team variant: forked, returns summary
```

Creating a personal variant with a different name ensures teammates are not affected. Never modify team skills for personal preferences.

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

```markdown
<!-- .claude/rules/terraform.md -->
---
paths:
  - "terraform/**/*"
  - "infrastructure/**/*.tf"
---

## Terraform Conventions

- All resources must have `environment` and `team` tags
- Use `for_each` over `count` for resource iteration
- Store state in S3 with DynamoDB locking
- Run `terraform fmt` before committing
- Module versions must be pinned (no `latest`)
```

```markdown
<!-- .claude/rules/api-routes.md -->
---
paths:
  - "src/routes/**/*.ts"
  - "src/api/**/*.ts"
---

## API Route Conventions

- All routes must validate input with Zod schemas
- Error responses: { error: string, code: string, status: number }
- Authentication: use `requireAuth` middleware
- Rate limiting: apply `rateLimiter` to all public endpoints
- Version prefix: /api/v1/...
```

**Path-Specific vs. Directory CLAUDE.md**

```
Scenario: Test conventions that apply to *.test.tsx files everywhere
  tests/ (in root)
  src/components/__tests__/
  src/api/__tests__/
  packages/backend/tests/

Option A: Directory CLAUDE.md in each test directory
  → Requires 4 separate files
  → Each file must be kept in sync manually
  → Adding a new test directory needs another CLAUDE.md

Option B: .claude/rules/testing.md with paths: ["**/*.test.tsx"]
  → Single file with one glob pattern
  → Automatically applies to ANY test file wherever located
  → Preferred approach for cross-directory conventions
```

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

```markdown
Session 1 (Plan Mode):
  User: "I need to migrate from Passport.js to our custom JWT auth system"

  Claude (plan mode):
  1. Analyze current Passport.js usage (Explore subagent)
  2. Identify all affected files
  3. Design migration approach
  4. Identify risks and dependencies

  Output: Migration plan with file list, approach, step sequence, risks

Session 2 (Direct Execution):
  User: "Execute the migration plan for step 1: update auth middleware"

  Claude (direct execution):
  → Implements specific planned changes
  → Clean context (no verbose discovery history)
```

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

```python
# Vague instruction (produces inconsistent results):
"Convert the date format to a standardized format"

# Better: specific transformation description
"Convert all dates to ISO 8601 format (YYYY-MM-DD)"

# Best: concrete input/output examples (most reliable)
"""
Transform these date strings to ISO 8601 format. Examples:

Input: "March 5, 2026"     → Output: "2026-03-05"
Input: "05/03/2026"        → Output: "2026-03-05"
Input: "5-Mar-26"          → Output: "2026-03-05"
Input: null or missing     → Output: null
Input: "TBD"               → Output: null

Handle: European format (DD/MM/YYYY), US format (MM/DD/YYYY),
natural language (Month D, YYYY), and ISO already-formatted.
"""
```

**Test-Driven Iteration**

```python
# Step 1: Write the test suite FIRST
# tests/test_date_converter.py
def test_iso_format():
    assert convert_date("March 5, 2026") == "2026-03-05"

def test_us_format():
    assert convert_date("05/03/2026") == "2026-03-05"

def test_null_input():
    assert convert_date(None) is None

def test_tbd_string():
    assert convert_date("TBD") is None

def test_already_iso():
    assert convert_date("2026-03-05") == "2026-03-05"

# Step 2: Run tests, share failures
# "These tests are failing: test_us_format, test_tbd_string.
#  Error: 05/03/2026 interpreted as May 3 (US format), expecting March 5"

# Step 3: Claude iterates based on specific failure, not vague feedback
```

**The Interview Pattern**

```markdown
# Instruction to Claude
Before implementing the caching layer, ask me 5-10 questions about
requirements and constraints I might not have considered.

# Claude asks:
1. What cache invalidation strategy is appropriate given the update frequency?
2. Should the cache be in-process (memory) or distributed (Redis)?
3. What's the acceptable staleness window for different data types?
4. How should concurrent writes be handled (write-through, write-behind)?
5. What happens if the cache becomes unavailable? Fallback to DB or fail?
6. Are there regulatory requirements for certain data not being cached?
7. What's the expected cache hit ratio, and how will you monitor it?

# Developer answers → implementation is grounded in actual requirements
# Without interview: developer discovers these questions after rework
```

**Single Message vs. Sequential Iteration**

```
Multiple INTERACTING issues → single detailed message
  Example:
  "The following issues all interact with each other:
   1. The sort function has incorrect comparison logic
   2. The null handling changes the sort order expectation
   3. The output format test depends on sorted null-handling behavior
   Fix all three together in a single pass to avoid conflicts."

Independent issues → fix sequentially
  Example:
  Message 1: "Fix the off-by-one error in the pagination function"
  (Review fix)
  Message 2: "Fix the missing error handling in the file upload function"
  (Review fix)
  Reason: Each fix can be validated independently; batching creates review noise
```

> **Exam Tip:** Questions about getting consistent output after repeated prompt attempts → the answer is concrete input/output examples. Questions about surfacing unstated requirements → interview pattern. The exam distinguishes "inconsistent output" (examples fix it) from "missing requirements" (interview pattern surfaces them).

> **Anti-Pattern Alert:** Trying to fix interacting issues sequentially causes the second fix to break the first. Identify when issues interact (they touch the same code path or have dependent behavior) and address them together.

> **Cross-Domain Connection:** Task 4.2 covers few-shot examples for output consistency — same principle as input/output examples for transformation tasks. Both use concrete examples to constrain ambiguous model behavior.

---

## Task Statement 3.6: Integrate Claude Code into CI/CD pipelines

### What You Need to Know
- `-p` (or `--print`) flag: runs Claude Code in **non-interactive mode** — essential for automated pipelines (prevents hanging waiting for input)
- `--output-format json` with `--json-schema`: enforces structured output for machine-parseable CI findings
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
# -p flag: print mode (non-interactive)
# Claude Code answers the prompt and exits — no interactive session
claude -p "Review the changes in this PR for security issues" \
  --output-format json \
  --json-schema ./schemas/review-findings.json \
  < diff_output.txt

# Full CI pipeline example
#!/bin/bash
PR_DIFF=$(git diff origin/main...HEAD)

claude -p "$(cat <<EOF
Review the following code changes for:
1. Security vulnerabilities (SQL injection, XSS, secrets exposure)
2. Logic errors and null pointer risks
3. Missing error handling on external calls

Produce findings only for definite issues, not style preferences.
Context: $(cat CLAUDE.md)

Diff:
$PR_DIFF
EOF
)" \
  --output-format json \
  --json-schema ./schemas/review-findings.json \
  > review_output.json
```

**JSON Schema for Structured Review Output**

```json
// schemas/review-findings.json
{
  "type": "object",
  "properties": {
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "file": {"type": "string"},
          "line": {"type": "integer"},
          "severity": {"type": "string", "enum": ["critical", "high", "medium", "low"]},
          "category": {"type": "string", "enum": ["security", "logic", "error_handling", "performance"]},
          "description": {"type": "string"},
          "suggested_fix": {"type": "string"}
        },
        "required": ["file", "line", "severity", "category", "description"]
      }
    },
    "summary": {"type": "string"},
    "total_findings": {"type": "integer"}
  }
}
```

**Avoiding Duplicate Review Comments**

```bash
# First review — save findings
claude -p "Review PR #$PR_NUMBER" \
  --output-format json \
  > review_first_pass.json

# Post comments to PR
post_review_comments review_first_pass.json

# After developer pushes fixes — second review
# Include prior findings to avoid duplicate comments
PRIOR_FINDINGS=$(cat review_first_pass.json)

claude -p "$(cat <<EOF
Review the updated code changes. Prior review findings:
$PRIOR_FINDINGS

Report ONLY:
1. Issues that were NOT addressed in the prior review
2. New issues introduced by the latest changes

Do NOT re-report issues already listed in prior findings.
EOF
)" \
  --output-format json \
  > review_second_pass.json
```

**Session Context Isolation for Review**

```python
# Anti-pattern: same session reviews its own generated code
# The model retains its reasoning from generation, making it
# less likely to question its own decisions

# Better: two independent Claude Code invocations
# Step 1: Generate code
claude -p "Generate a payment processing service" \
  --output-format text \
  > payment_service.py

# Step 2: Independent review instance
# Note: fresh session, no knowledge of generation reasoning
claude -p "$(cat <<EOF
Review this code for correctness, security, and edge cases.
You have no context about how this code was written.
$(cat payment_service.py)
EOF
)" \
  --output-format json \
  > review_output.json
```

**CLAUDE.md for CI Context**

```markdown
<!-- CLAUDE.md — provides context to CI-invoked Claude Code -->
# CI Review Standards

## Test Generation
- Framework: pytest with fixtures in tests/conftest.py
- Available fixtures: `test_db`, `mock_payment_gateway`, `sample_order`
- Do not generate tests that duplicate tests/test_payment.py
- Priority areas: edge cases, null inputs, concurrent access

## Code Review Criteria
- Flag: SQL queries without parameterization
- Flag: missing try/except on external API calls
- Flag: hardcoded credentials or environment-specific values
- Skip: minor style issues handled by linter
- Skip: formatting (handled by black/prettier)

## Severity Definitions
- Critical: security vulnerability, data corruption risk
- High: production failure risk under normal load
- Medium: failure risk under edge conditions
- Low: reliability improvements
```

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
| `--output-format json` | Structured JSON output mode | Machine-parseable; combine with `--json-schema` |
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
3. **Path-scoped rules** in `.claude/rules/` with `paths:` YAML frontmatter are better than directory CLAUDE.md for file-type conventions spanning multiple directories.
4. **`-p` flag** is required for non-interactive CI use — without it, Claude Code hangs waiting for input.
5. **Plan mode** = architectural/multi-file complexity; **direct execution** = clear, bounded, single-file tasks. The Explore subagent prevents verbose discovery from filling the main context.
