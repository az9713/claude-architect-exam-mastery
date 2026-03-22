# Lab Guide: Claude Code CLI

10 hands-on terminal exercises covering CLAUDE.md hierarchy, path-specific rules, skills, MCP servers, plan mode, iterative refinement, CI/CD mode, session management, built-in tools, and the Explore subagent.

**Prerequisites:** Claude Code installed, project directory with sample files, terminal access.

---

## Lab 1: CLAUDE.md Hierarchy

**Task Statement:** 3.1
**Objective:** Create configuration at user, project, and directory levels and observe how they stack.

### Prerequisites
- Claude Code installed
- A sample project directory (any existing project or a new empty directory)

### Step-by-Step Instructions

**Step 1: Create user-level configuration**
```bash
mkdir -p ~/.claude
cat > ~/.claude/CLAUDE.md << 'EOF'
# User Preferences (Personal Only — NOT shared via version control)

Always use British English spelling (e.g., "colour" not "color").
My preferred variable naming: use full descriptive names, never single letters except loop indices.
EOF
```

**Step 2: Create project-level configuration**
Navigate to your project directory, then:
```bash
# Option A: Project root CLAUDE.md (simpler)
cat > CLAUDE.md << 'EOF'
# Project Standards (Shared via version control — all team members receive this)

This is a Node.js Express API project.

## Architecture
- src/controllers/ — HTTP route handlers
- src/services/ — Business logic
- src/models/ — Database models (Mongoose)

## Coding Standards
- Use async/await for all asynchronous operations
- All functions must have JSDoc comments
- Error handling: use try/catch and call next(error) in controllers
EOF
```

**Step 3: Create directory-level configuration**
```bash
mkdir -p src/services
cat > src/services/CLAUDE.md << 'EOF'
# Service Layer Conventions

Services must be stateless — no instance variables.
All public methods must be async.
Input validation goes here, not in controllers.
Return plain objects, not Mongoose documents.
EOF
```

**Step 4: Verify with /memory command**

Open Claude Code in the project root:
```bash
claude
```

Then type:
```
/memory
```

### Expected Output
The `/memory` command displays which CLAUDE.md files are currently loaded. You should see:
- `~/.claude/CLAUDE.md` (user-level)
- `./CLAUDE.md` (project-level)

Navigate into `src/services/` and run `/memory` again — you should also see `src/services/CLAUDE.md` loaded.

### What to Observe
- **User-level settings are not visible to teammates** — this file would not appear in `git status` if you only committed the project directory
- **Directory-level files load when Claude Code is in that directory** — test this by comparing `/memory` output at root vs inside `src/services/`
- **All three levels are loaded simultaneously** when in `src/services/`

### Exam Relevance
Question type: "A new team member is not receiving the TypeScript strict mode conventions. You wrote them in `~/.claude/CLAUDE.md` on your machine. What is the problem?" — Answer: user-level configuration is not shared via version control.

---

## Lab 2: Path-Specific Rules

**Task Statement:** 3.3
**Objective:** Create `.claude/rules/` files with glob patterns and verify conditional loading.

### Prerequisites
- Lab 1 completed (project directory exists)
- Sample files in multiple directories

### Step-by-Step Instructions

**Step 1: Create the rules directory**
```bash
mkdir -p .claude/rules
```

**Step 2: Create a frontend-specific rule**
```bash
cat > .claude/rules/react.md << 'EOF'
---
paths: ["src/components/**/*", "src/pages/**/*"]
---
# React Component Conventions

Use functional components with hooks (no class components).
TypeScript: all props must have explicit TypeScript interfaces.
CSS: use CSS modules (ComponentName.module.css), not inline styles.
Component files: PascalCase (e.g., UserProfile.tsx).
Export default the component; named exports for types.
EOF
```

**Step 3: Create a test-specific rule spanning all directories**
```bash
cat > .claude/rules/tests.md << 'EOF'
---
paths: ["**/*.test.ts", "**/*.test.tsx", "**/*.spec.ts", "**/*.spec.tsx"]
---
# Test Conventions

Testing library: Jest with React Testing Library for UI.
Test file naming: [ComponentName].test.tsx adjacent to the component.
Describe block: must match the file under test.
Test names: "should [expected behavior] when [condition]".
No snapshot tests — use explicit assertions.
Mock external modules at the top of each test file.
EOF
```

**Step 4: Create a Terraform-specific rule**
```bash
cat > .claude/rules/terraform.md << 'EOF'
---
paths: ["infra/**/*", "terraform/**/*"]
---
# Terraform Conventions

Always include a description for every variable and output.
Resource naming: [company]-[env]-[resource-type] (e.g., acme-prod-api-sg).
Use data sources instead of hardcoded IDs.
All resources must have standard tags: Environment, Project, ManagedBy.
EOF
```

**Step 5: Verify conditional loading**

Open Claude Code and run `/memory` while editing different files:
```bash
# While editing a React file — should load react.md
claude src/components/UserProfile.tsx

# While editing a test file — should load tests.md regardless of location
claude src/services/auth.test.ts

# While editing a Terraform file — should load terraform.md
claude infra/main.tf
```

### Expected Output
Each session should show only the relevant rule files in `/memory` output, plus the always-loaded CLAUDE.md files.

### What to Observe
- **Path rules load conditionally** — the react.md rule does NOT load when editing a service file
- **Glob patterns work across directories** — `**/*.test.tsx` matches test files anywhere in the tree
- **Token efficiency** — path-specific rules mean Claude Code doesn't load irrelevant conventions into context

### Exam Relevance
Question type: "Test files are spread throughout the codebase. What mechanism applies test conventions regardless of directory location?" — Answer: `.claude/rules/` with `paths: ["**/*.test.*"]` glob pattern.

---

## Lab 3: Custom Skills

**Task Statement:** 3.2
**Objective:** Create a skill with `context: fork` and `allowed-tools` and observe isolation behavior.

### Prerequisites
- Lab 1 or 2 completed (project directory exists)

### Step-by-Step Instructions

**Step 1: Create the skills directory**
```bash
mkdir -p .claude/skills/generate-component
```

**Step 2: Create the skill definition**
```bash
cat > .claude/skills/generate-component/SKILL.md << 'EOF'
---
name: generate-component
description: Generate a complete React component with TypeScript interface, styles, and test file
argument-hint: "ComponentName (e.g., UserProfile)"
context: fork
allowed-tools: ["Write", "Read", "Glob"]
---

# Generate Component Skill

Generate a complete React component based on the argument provided.

Create the following files:
1. `src/components/{ComponentName}/{ComponentName}.tsx` — The component with TypeScript props interface
2. `src/components/{ComponentName}/{ComponentName}.module.css` — CSS module with a container class
3. `src/components/{ComponentName}/{ComponentName}.test.tsx` — Test file with at least one render test
4. `src/components/{ComponentName}/index.ts` — Barrel export

Follow all conventions in the project CLAUDE.md and any active rules.
EOF
```

**Step 3: Create a skill with restricted tools**
```bash
mkdir -p .claude/skills/audit-imports
cat > .claude/skills/audit-imports/SKILL.md << 'EOF'
---
name: audit-imports
description: Audit all import statements in the project and report unused or circular imports
context: fork
allowed-tools: ["Grep", "Glob", "Read"]
---

# Import Audit Skill

Analyze all import statements in the TypeScript source files.
Report: unused imports, circular dependencies, and imports from deprecated modules.
Output a structured summary — do not modify any files.
EOF
```

**Step 4: Invoke and observe**

In Claude Code, invoke the skill:
```
/generate-component Button
```

### Expected Output
The skill runs in an isolated context. You should see the component files created. The main conversation context should receive a short summary, not the full verbose generation output.

### What to Observe
- **`context: fork`** — the skill's verbose exploration and file creation happen in a subagent; your main conversation only sees the results
- **`allowed-tools`** — the audit-imports skill cannot use `Write` or `Bash`, preventing accidental file modifications during a read-only audit
- **`argument-hint`** — if you run `/generate-component` without an argument, Claude Code prompts you for the component name

### Exam Relevance
Key distinction: skills with `context: fork` prevent verbose output from polluting the main conversation. This is different from CLAUDE.md (always-loaded) — skills are on-demand invocations.

---

## Lab 4: MCP Server Configuration

**Task Statement:** 2.4
**Objective:** Configure a project-level MCP server with environment variable expansion.

### Prerequisites
- A project directory
- Access to an MCP-compatible server (or use the mock configuration below)

### Step-by-Step Instructions

**Step 1: Create project-level MCP configuration**
```bash
cat > .mcp.json << 'EOF'
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "jira": {
      "command": "npx",
      "args": ["-y", "@your-org/mcp-jira"],
      "env": {
        "JIRA_HOST": "${JIRA_HOST}",
        "JIRA_TOKEN": "${JIRA_TOKEN}"
      }
    }
  }
}
EOF
```

**Step 2: Add personal experimental server to user-level config**
```bash
# This is personal — not committed to version control
cat >> ~/.claude.json << 'EOF'
{
  "mcpServers": {
    "my-experimental-db": {
      "command": "node",
      "args": ["/Users/me/tools/db-mcp-server/index.js"],
      "env": {
        "DB_CONNECTION_STRING": "${LOCAL_DB_URL}"
      }
    }
  }
}
EOF
```

**Step 3: Set environment variables**
```bash
export GITHUB_TOKEN="your_token_here"
export JIRA_HOST="yourcompany.atlassian.net"
export JIRA_TOKEN="your_jira_token"
```

**Step 4: Verify both servers are available**
```bash
claude
# In Claude Code, tools from both servers should be available
# Type: "What tools do you have available?" to see all discovered tools
```

### Expected Output
Both the project-level GitHub/Jira servers and your personal experimental server should appear in the available tools list.

### What to Observe
- **All configured MCP servers are discovered at connection time** — you see all tools simultaneously
- **Environment variable expansion** means no tokens are committed to version control
- **Scope separation** — `.mcp.json` is team-shared; `~/.claude.json` is personal

### Exam Relevance
Question type: "An MCP server that uses `${GITHUB_TOKEN}` in .mcp.json — where does this token come from?" — Answer: the shell environment variable, not the file itself.

---

## Lab 5: Plan Mode vs Direct Execution

**Task Statement:** 3.4
**Objective:** Run the same conceptual task in both modes and observe the difference.

### Prerequisites
- Any project with at least 5 source files

### Step-by-Step Instructions

**Task A (Direct Execution — appropriate):**
Single-file bug fix with a clear stack trace.
```
claude "The function validateEmail in src/utils/validation.js returns true for emails without a TLD (e.g., user@domain). Add a regex check that requires at least one dot after the @ symbol."
```

**Task B (Plan Mode — appropriate):**
Restructure a module affecting multiple files.
```
# In Claude Code interactive mode
claude
> I need to extract the authentication logic from UserController.js into a separate AuthService class. This will affect UserController.js, any files that import from it, and we'll need a new AuthService.js. Let's plan this first.
```
Enter plan mode before responding or type `/plan` if available.

**Observe the difference:**
- Task A (direct): Claude immediately makes the single change
- Task B (plan mode): Claude explores dependencies, maps the impact, proposes a plan, and waits for approval before making any changes

**Task C — Using Explore subagent:**
```
claude "Explore the codebase and find all places where authentication is currently handled, then summarize the findings without making any changes."
```

### Expected Output
- Task A: Single focused edit
- Task B: Exploration output + proposed plan + approval prompt
- Task C: Exploration happens in an isolated subagent; main context receives a concise summary

### What to Observe
- **Plan mode prevents premature changes** — you see the full impact before committing
- **The Explore subagent** isolates verbose discovery from the main conversation context
- **Direct execution** is fast and appropriate when the change is clear and scoped

### Exam Relevance
Criteria for choosing plan mode: "large-scale changes, multiple valid approaches, architectural decisions, multi-file modifications." Criteria for direct execution: "simple, well-scoped changes with clear stack traces."

---

## Lab 6: Iterative Refinement

**Task Statement:** 3.5
**Objective:** Practice the interview pattern, test-driven iteration, and input/output examples.

### Step-by-Step Instructions

**Technique 1 — Input/Output Examples:**

Instead of describing what you want in prose:
```
# Vague (produces inconsistent results):
"Convert the date format in user records"

# Better — concrete examples:
"Convert date formats in user records.
Input: {created_at: '2024-01-15T09:30:00Z', last_login: 1705312200}
Output: {created_at: '2024-01-15', last_login: '2024-01-15'}
Input: {created_at: null, last_login: 0}
Output: {created_at: null, last_login: null}"
```

Try both in Claude Code and compare the consistency.

**Technique 2 — Interview Pattern:**
```
"I want to implement a caching layer for the API. Before you write any code,
ask me questions to understand the requirements. Ask about: expiry strategy,
invalidation triggers, cache storage, and failure handling."
```

Observe that Claude asks about cache invalidation strategies and failure modes you may not have specified — surfacing considerations you hadn't thought through.

**Technique 3 — Test-Driven Iteration:**
```
# Step 1: Write tests first
"Write a test suite for a function parseCSV(text) that should handle:
- Basic comma separation
- Quoted fields containing commas
- Empty fields
- Windows line endings (CRLF)
- Files with headers"

# Step 2: Run the tests (they should fail — function doesn't exist yet)
# Step 3: Implement
"Now implement parseCSV to make these tests pass"

# Step 4: If tests fail, share the failures
"Tests are failing with: [paste failure output]. Fix the implementation."
```

### What to Observe
- **Input/output examples** converge faster than prose descriptions
- **Interview pattern** discovers requirements you didn't know you had
- **Sharing test failures** gives Claude precise information for correction

### Exam Relevance
"When should you use the interview pattern?" — Answer: in unfamiliar domains where you don't know all the considerations. "When is test-driven iteration most effective?" — Answer: for algorithmic code with clear expected behavior.

---

## Lab 7: CI/CD Mode

**Task Statement:** 3.6
**Objective:** Use the `-p` flag and `--output-format json` for non-interactive CI execution.

### Step-by-Step Instructions

**Step 1: Test non-interactive mode**
```bash
# This will hang (interactive mode):
# claude "Review this file for bugs"

# This exits correctly (non-interactive):
claude -p "Review the following code for security vulnerabilities:
\`\`\`javascript
const express = require('express');
const app = express();
app.get('/user', (req, res) => {
  const query = 'SELECT * FROM users WHERE id = ' + req.query.id;
  db.execute(query, (err, rows) => res.json(rows));
});
\`\`\`
"
```

**Step 2: Structured JSON output**
```bash
# Create a schema file
cat > review_schema.json << 'EOF'
{
  "type": "object",
  "properties": {
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "severity": {"type": "string", "enum": ["critical", "high", "medium", "low"]},
          "issue": {"type": "string"},
          "suggested_fix": {"type": "string"}
        },
        "required": ["severity", "issue", "suggested_fix"]
      }
    }
  },
  "required": ["findings"]
}
EOF

# Run with structured output
claude -p "Review the code below for security issues and produce structured findings.
[code here]" --output-format json --json-schema review_schema.json
```

**Step 3: Simulate a CI pipeline step**
```bash
#!/bin/bash
# review_pr.sh

CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD)

REVIEW_OUTPUT=$(claude -p "Review these changed files for security vulnerabilities.
Files: $CHANGED_FILES

Report only: SQL injection, authentication bypass, unvalidated user input.
Skip: style issues, naming conventions." \
  --output-format json \
  --json-schema review_schema.json)

echo "$REVIEW_OUTPUT" | python3 -c "
import json, sys
data = json.load(sys.stdin)
findings = data.get('findings', [])
critical = [f for f in findings if f['severity'] == 'critical']
if critical:
    print('BLOCKING: Critical security issues found')
    for f in critical:
        print(f'  - {f[\"issue\"]}')
    sys.exit(1)
print(f'Review passed. {len(findings)} findings logged.')
"
```

### What to Observe
- **Without `-p`**: Claude Code hangs waiting for interactive input
- **With `-p`**: Exits after producing output
- **`--output-format json --json-schema`**: Output is machine-parseable with guaranteed structure

### Exam Relevance
"What flag runs Claude Code in a CI pipeline without hanging?" — `-p` or `--print`. "How do you get structured output for automated PR comment posting?" — `--output-format json --json-schema schema.json`.

---

## Lab 8: Session Management

**Task Statement:** 1.7
**Objective:** Use `--resume`, `fork_session`, and `/compact` for long-running investigations.

### Step-by-Step Instructions

**Step 1: Start a named session**
```bash
# Begin an investigation session
claude --session-name "auth-refactor-analysis"
# Analyze several files, build up context
# Exit when done
```

**Step 2: Resume the session**
```bash
# Next work session — continue exactly where you left off
claude --resume "auth-refactor-analysis"
```

**Step 3: Practice /compact**
During a long session when context fills:
```
/compact
```
This compresses prior conversation content, freeing context budget for continued work.

**Step 4: Fork for parallel exploration**
```bash
# Start a session, analyze the codebase
# Then fork to explore two different approaches
# In the Claude Code session, use fork_session to branch
```

Use case: You've analyzed a module and want to compare "approach A: extract to a service" vs "approach B: use a middleware pattern." Fork the session to explore each approach from the same analysis baseline without mixing their contexts.

### What to Observe
- **`--resume`** restores prior conversation — useful when prior context is still valid
- **`/compact`** frees context space during long sessions
- **`fork_session`** creates independent branches from a shared baseline
- When files have changed since the last session, tell Claude about the specific changes rather than re-exploring everything

### Exam Relevance
"When should you start a fresh session with injected summaries instead of resuming?" — When prior tool results are stale (files changed, code restructured). "When is `fork_session` useful?" — Exploring two competing implementation approaches from a shared analysis baseline.

---

## Lab 9: Built-in Tools

**Task Statement:** 2.5
**Objective:** Compare Grep vs Glob, and practice Read+Write as an Edit fallback.

### Step-by-Step Instructions

**Exercise 1 — Grep vs Glob:**
```
# Task 1: Find all files that import from 'react-query'
# WRONG tool: Glob (Glob finds paths, not content)
# CORRECT tool: Grep
# In Claude Code: "Find all files that import from react-query"
# Claude should use: Grep pattern="from 'react-query'|require.*react-query" type="ts,tsx,js"

# Task 2: Find all TypeScript test files
# WRONG tool: Grep (you want paths, not content search)
# CORRECT tool: Glob
# Claude should use: Glob pattern="**/*.test.ts" or "**/*.spec.ts"
```

**Exercise 2 — Edit failure and Read+Write fallback:**
Create a test file with a repeated pattern:
```bash
cat > test_file.js << 'EOF'
const CONFIG = require('./config');
const CONFIG2 = require('./config');
// CONFIG is used here
module.exports = { CONFIG };
EOF
```

Ask Claude Code to edit `CONFIG` to `APP_CONFIG`. The word `CONFIG` appears 4 times — Edit may fail because the anchor text is not unique. Observe Claude falling back to Read + Write:
1. Read the full file
2. Make all replacements
3. Write the complete updated file

### What to Observe
- **Grep** returns file paths AND matching lines — good for finding function callers, import sources
- **Glob** returns file paths matching a pattern — good for finding all test files, all TypeScript files
- **Edit fails on non-unique text** → fallback to Read + Write is reliable

### Exam Relevance
"You need to find all callers of `validateToken()`. Which tool?" — Grep. "You need to find all `.test.tsx` files." — Glob.

---

## Lab 10: Explore Subagent

**Task Statement:** 3.4
**Objective:** Use the Explore subagent for verbose discovery to protect main context.

### Step-by-Step Instructions

**Step 1: Without Explore (observe context pollution)**
```
"Explore the entire codebase structure — every directory, file, and module.
Build a complete map of what everything does."
```
Observe: the main conversation fills with exploration output. If you then ask a specific implementation question, the context is cluttered.

**Step 2: With Explore subagent**
```
"Use the Explore subagent to explore the codebase structure and return
a concise summary. Specifically: what are the main modules, where is the
authentication logic, and which files are most frequently imported?"
```
Observe: the main context receives only the summary. Subsequent implementation questions have clean context.

**Step 3: Multi-phase pattern**
```
"Phase 1: Use the Explore subagent to map all test files and identify
which source files have no test coverage. Summarize the findings.

Phase 2 (after Phase 1 summary): Based on the coverage summary, implement
a test for [specific uncovered function]."
```

### What to Observe
- **Explore subagent** isolates verbose discovery output
- **Main context** stays clean for implementation work
- **Summarize before spawning** sub-agents for subsequent phases

### Exam Relevance
"In a multi-phase task, how do you prevent context window exhaustion during the discovery phase?" — Use the Explore subagent, which isolates verbose output and returns a summary to the main context.

---

## Summary: Tool and Mechanism Quick Reference

| Need | Use |
|------|-----|
| Search file contents | Grep |
| Find files by name/extension | Glob |
| Load full file | Read |
| Targeted modification | Edit |
| When Edit fails (non-unique text) | Read + Write |
| Always-loaded team standards | CLAUDE.md |
| Conditional conventions by file type | `.claude/rules/` with glob paths |
| On-demand task workflows | `.claude/skills/` |
| Isolate verbose skill output | `context: fork` in SKILL.md |
| Restrict skill tools | `allowed-tools` in SKILL.md |
| Non-interactive CI execution | `-p` flag |
| Structured CI output | `--output-format json --json-schema` |
| Continue prior session | `--resume` |
| Branch from shared baseline | `fork_session` |
| Free context during long sessions | `/compact` |
| Inspect loaded config | `/memory` |
