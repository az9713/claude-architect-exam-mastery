# Interactive Session: MCP Configuration and Claude Code Setup

**Surface:** Claude Code CLI
**Duration:** 45–60 minutes
**Task Statements:** 2.4, 3.1, 3.2

---

## Learning Objectives

By the end of this session, you will be able to:
- Configure MCP servers at project and user scope with environment variable expansion
- Verify CLAUDE.md hierarchy with /memory
- Create and use project-scoped slash commands
- Understand the difference between skills, commands, and CLAUDE.md

---

## Prerequisites

- Claude Code CLI installed
- A test project directory (create one if needed: `mkdir mcp-lab && cd mcp-lab`)
- Terminal access with ability to create files and directories

---

## Session Steps

### Step 1 — Initialize a Test Project

```bash
mkdir -p ~/mcp-lab/src/components
mkdir -p ~/mcp-lab/src/services
mkdir -p ~/mcp-lab/.claude/commands
mkdir -p ~/mcp-lab/.claude/rules
mkdir -p ~/mcp-lab/.claude/skills

cd ~/mcp-lab
echo "console.log('hello');" > src/index.js
echo "export const greet = (name) => \`Hello \${name}\`;" > src/services/greeter.js
echo "// Button component" > src/components/Button.jsx
```

---

### Step 2 — Verify Empty State with /memory

```bash
claude
```

Once in Claude Code:
```
/memory
```

**Observe:** No CLAUDE.md files are loaded. The output should show an empty or minimal memory state.

**Checkpoint 1:** What is shown in `/memory` when no CLAUDE.md files exist?

---

### Step 3 — Create Project-Level CLAUDE.md

Exit Claude Code, then:
```bash
cat > ~/mcp-lab/CLAUDE.md << 'EOF'
# MCP Lab Project

## Project Context
This is a small JavaScript project used for Claude Code configuration experiments.

## Coding Standards
- Use ES6+ syntax (arrow functions, template literals, destructuring)
- No semicolons at end of lines (we use ASI)
- Use single quotes for strings

## Architecture
- src/components/ — UI components
- src/services/ — Business logic and utilities
EOF
```

Reopen Claude Code and run `/memory` again:
```bash
claude
/memory
```

**Observe:** The project-level CLAUDE.md should now appear in the loaded files.

**Checkpoint 2:** Did the project-level CLAUDE.md appear? Note the file path shown.

---

### Step 4 — Create User-Level CLAUDE.md (Personal)

Exit Claude Code:
```bash
mkdir -p ~/.claude
cat > ~/.claude/CLAUDE.md << 'EOF'
# Personal Preferences (NOT shared with team)

I prefer very detailed explanations when generating code.
Always add JSDoc comments to new functions.
EOF
```

Reopen Claude Code and check `/memory`:
```bash
claude
/memory
```

**Observe:** Both the user-level AND project-level CLAUDE.md should appear.

**Checkpoint 3:** Both files should be loaded. Now consider: if you put team coding standards in `~/.claude/CLAUDE.md` instead of `./CLAUDE.md`, what would happen for a new team member?

---

### Step 5 — Create Directory-Level CLAUDE.md

Exit Claude Code:
```bash
cat > ~/mcp-lab/src/services/CLAUDE.md << 'EOF'
# Service Layer Conventions

Services must export pure functions only.
All function parameters must be destructured if an object.
EOF
```

Test directory-level loading — open Claude Code while in the services directory:
```bash
cd ~/mcp-lab/src/services
claude
/memory
```

Then compare from the project root:
```bash
cd ~/mcp-lab
claude
/memory
```

**Observe:** From `src/services/`, all three levels should load. From the project root, only user and project levels load.

**Checkpoint 4:** How does the directory-level CLAUDE.md loading behavior differ between the two locations?

---

### Step 6 — Create Project-Scoped Slash Command

Exit and create a team-shared command:
```bash
cat > ~/mcp-lab/.claude/commands/review.md << 'EOF'
# Code Review Command

Review the specified file or recent changes for:
1. Code style issues (based on project standards in CLAUDE.md)
2. Potential bugs or logic errors
3. Security concerns

Format output as:
- STYLE: [issue]
- BUG: [issue]
- SECURITY: [issue]

If no issues found in a category, output "STYLE: None found"
EOF
```

Test the command:
```bash
cd ~/mcp-lab
claude
/review src/services/greeter.js
```

**Observe:** The slash command should run with the instructions defined in the file.

**Checkpoint 5:** This command is version-controlled in `.claude/commands/`. What happens when a new team member clones the repo?

---

### Step 7 — Create a Skill with context: fork

```bash
mkdir -p ~/mcp-lab/.claude/skills/analyze-deps
cat > ~/mcp-lab/.claude/skills/analyze-deps/SKILL.md << 'EOF'
---
name: analyze-deps
description: Analyze all import statements in the project and create a dependency map
argument-hint: "Optional: path to specific directory (defaults to src/)"
context: fork
allowed-tools: ["Grep", "Glob", "Read"]
---

# Dependency Analysis Skill

1. Use Glob to find all .js and .jsx files in the target directory
2. Use Grep to extract all import statements from each file
3. Build a dependency map showing which files import from which
4. Identify any circular dependencies
5. Return a summary (not the full verbose output) to the main conversation

Target directory: {{argument}} or src/ if not specified
EOF
```

Test the skill:
```bash
claude
/analyze-deps
```

**Observe:** The skill should run in a forked context. The main conversation should receive a summary rather than verbose output.

**Checkpoint 6:** What is the difference between `/review` (a command) and `/analyze-deps` (a skill with `context: fork`)? When would you choose one over the other?

---

### Step 8 — Configure MCP Server (Project-Level)

```bash
cat > ~/mcp-lab/.mcp.json << 'EOF'
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp/allowed-path"],
      "env": {}
    }
  }
}
EOF
```

For a server requiring credentials:
```bash
cat > ~/mcp-lab/.mcp.json << 'EOF'
{
  "mcpServers": {
    "github-mock": {
      "command": "echo",
      "args": ["MCP server would start here"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}",
        "GITHUB_ORG": "${GITHUB_ORG}"
      }
    }
  }
}
EOF
```

**Checkpoint 7:** Where does `${GITHUB_TOKEN}` come from at runtime? Is this value stored in the `.mcp.json` file? What should happen in version control with this file?

---

### Step 9 — Compare Project vs User MCP Scope

**Prompt (to discuss in Claude Code):**
```
I have two MCP servers to configure:
1. A GitHub server used by our entire team for PR reviews — configured in .mcp.json
2. A personal database exploration tool I'm experimenting with — configured in ~/.claude.json

Can you explain the difference in scope and why each configuration location is appropriate for each use case?
```

**Observe:** Claude should explain:
- `.mcp.json` is committed to version control → available to all team members
- `~/.claude.json` is personal → only available on your machine

**Checkpoint 8:** If the experimental database tool works well and you want the team to use it, what would you need to do?

---

### Step 10 — Path-Specific Rules Exercise

```bash
cat > ~/mcp-lab/.claude/rules/components.md << 'EOF'
---
paths: ["src/components/**/*"]
---
# Component Conventions

React components only (no class-based).
Props must be destructured in the function signature.
Component name must match file name.
EOF

cat > ~/mcp-lab/.claude/rules/services.md << 'EOF'
---
paths: ["src/services/**/*"]
---
# Service Layer Rules

No React imports in service files.
All exported functions must have JSDoc.
EOF
```

Test conditional loading:
```bash
cd ~/mcp-lab
claude src/components/Button.jsx
/memory
# Should show components.md rule loaded

claude src/services/greeter.js
/memory
# Should show services.md rule loaded, NOT components.md
```

**Checkpoint 9:** What is the advantage of `.claude/rules/` over putting all conventions in the root CLAUDE.md? When is a directory-level CLAUDE.md better than a path-specific rule?

---

### Step 11 — Variation: Diagnose a Missing Configuration Bug

**Scenario:**
```
A new team member reports that Claude Code is not following the "no semicolons" rule you defined in the project CLAUDE.md. When they generate code, Claude adds semicolons.

They have their own ~/.claude/CLAUDE.md that says "Always add semicolons for clarity."

Run /memory and diagnose the issue.
```

**Prompt:**
```
/memory
```

**Checkpoint 10:** What does the `/memory` output reveal about why the user-level preference overrides the project-level standard? What is the fix?

---

## Session Debrief

**Configuration Hierarchy Summary:**

| Level | File Location | Scope | Version Controlled? |
|-------|---------------|-------|---------------------|
| User | `~/.claude/CLAUDE.md` | Personal only | No |
| Project | `./CLAUDE.md` or `./.claude/CLAUDE.md` | All team members | Yes |
| Directory | `./src/services/CLAUDE.md` | That directory | Yes |
| Path Rules | `./.claude/rules/*.md` | Files matching glob | Yes |

**When to Use Each:**

| Need | Solution |
|------|----------|
| Team standards (always loaded) | Project-level CLAUDE.md |
| File-type-specific conventions | `.claude/rules/` with glob paths |
| On-demand task workflow (simple) | `.claude/commands/` |
| On-demand task workflow (verbose output) | `.claude/skills/` with `context: fork` |
| Team MCP server | `.mcp.json` with env var expansion |
| Personal MCP server | `~/.claude.json` |

**Exam Connection:**
- Task Statement 2.4: MCP server scoping, env var expansion, project vs user scope
- Task Statement 3.1: CLAUDE.md hierarchy, /memory diagnostic, user vs project scope
- Task Statement 3.2: commands vs skills, `context: fork`, `allowed-tools`
