# Domain 3 Cheat Sheet: Claude Code Configuration & Workflows
**Weight: 20% | Task Statements: 3.1 – 3.6**

---

## Key Concepts

| Concept | Definition | Exam Relevance |
|---------|-----------|----------------|
| `~/.claude/CLAUDE.md` | User-level config | NOT in git; personal only; not shared with team |
| `.claude/CLAUDE.md` or `CLAUDE.md` | Project-level config | Committed to git; ALL developers get this |
| Directory `CLAUDE.md` | Subdirectory-level rules | Applies only within that directory subtree |
| `@import` | Include external file in CLAUDE.md | Modular organization; import by context |
| `.claude/rules/` | Topic-specific rule files | Alternative to monolithic CLAUDE.md |
| `paths:` frontmatter | YAML glob field in rules files | Loads rule ONLY when editing matching files |
| `/memory` | Claude Code command | Shows which memory files are loaded for this session |
| `.claude/commands/` | Project-scoped slash commands | Version-controlled; available to all team members |
| `~/.claude/commands/` | User-scoped slash commands | Personal; not shared via version control |
| `.claude/skills/` | Skill definition files | YAML frontmatter: context, allowed-tools, argument-hint |
| `context: fork` | Skill frontmatter option | Runs skill in isolated sub-agent; output stays separate |
| `allowed-tools` | Skill frontmatter option | Restricts tools available during skill execution |
| `argument-hint` | Skill frontmatter option | Prompts developer for parameter when invoked without args |
| Plan mode | Exploration before execution | Multi-file, architectural, multiple valid approaches |
| Direct execution | Immediate implementation | Single file, clear scope, obvious approach |
| Explore subagent | Isolates verbose discovery output | Returns summary to main agent; preserves context |
| `-p` / `--print` flag | Non-interactive (print) mode | Required for CI/CD; prevents interactive hang |
| `--output-format json` | Structured JSON output | Machine-parseable; pair with `--json-schema` |
| Interview pattern | Claude asks questions before implementing | Surfaces unstated requirements in unfamiliar domains |
| Test-driven iteration | Write tests first; share failures | Most effective for precise behavioral specification |

---

## Decision Matrix

### CLAUDE.md level for a requirement

| Requirement | Level | Location |
|-------------|-------|----------|
| Universal team coding standards | Project-level | `.claude/CLAUDE.md` |
| Personal response style preference | User-level | `~/.claude/CLAUDE.md` |
| Backend-specific Python conventions | Directory-level | `packages/backend/CLAUDE.md` |
| Test conventions for test files everywhere | Path-scoped rule | `.claude/rules/testing.md` with `paths: ["**/*.test.ts"]` |
| Terraform conventions for infra files | Path-scoped rule | `.claude/rules/terraform.md` with `paths: ["terraform/**/*"]` |

### Plan Mode vs. Direct Execution

| Plan Mode Signals | Direct Execution Signals |
|-------------------|--------------------------|
| Multi-file changes (5+) | Single file |
| Architectural decisions | Implementation-only |
| Multiple valid approaches | Approach is obvious |
| Must map dependencies first | Clear stack trace |
| "Restructure", "migrate", "refactor across" | "Fix", "add validation", "update" |
| High rework cost if wrong | Easy to reverse |

### Skills vs. CLAUDE.md vs. Commands

| Use Case | Mechanism |
|----------|-----------|
| Code style applied to every interaction | CLAUDE.md |
| On-demand PR review workflow | Skill or command |
| Verbose codebase analysis | Skill with `context: fork` |
| Personal variant of team skill | `~/.claude/skills/` with different name |
| Team-wide `/deploy-check` | `.claude/commands/deploy-check.md` |

---

## Config/CLI Reference

```yaml
# .claude/rules/testing.md — path-scoped rule
---
paths:
  - "**/*.test.ts"
  - "**/*.test.tsx"
  - "tests/**/*"
---

Use Jest + React Testing Library. Coverage: 80% minimum for new code.
```

```markdown
<!-- .claude/skills/analyze-codebase.md -->
---
context: fork
allowed-tools: ["Read", "Grep", "Glob"]
argument-hint: "Module or component to analyze"
---
Analyze the architecture of {{argument}}. Return concise summary.
```

```bash
# CI/CD non-interactive mode
claude -p "Review this diff for security issues" \
  --output-format json \
  --json-schema ./schemas/review-findings.json \
  < diff_output.txt

# Without -p: hangs waiting for interactive input — breaks CI pipeline
```

```markdown
# CLAUDE.md @import syntax
@import .claude/rules/testing.md
@import .claude/rules/api-conventions.md
@import packages/shared/standards/security.md
```

---

## Anti-Patterns (Common Wrong Answers)

- **Team standards in `~/.claude/CLAUDE.md`**: personal config; new team members don't get it
- **Monolithic giant CLAUDE.md**: loads all context every session; split into `.claude/rules/`
- **Verbose skills without `context: fork`**: pollutes main conversation; use `context: fork` for verbose skills
- **Directory CLAUDE.md for cross-directory conventions**: one path-scoped rule file beats N directory CLAUDE.md files
- **Running CI without `-p` flag**: Claude Code hangs waiting for interactive input
- **Same Claude session reviewing its own generated code**: retains generation context; use independent instance
- **Starting direct execution on multi-file architectural task**: costly rework risk; use plan mode first

---

## If You Remember Nothing Else

1. **CLAUDE.md hierarchy**: `~/.claude/CLAUDE.md` = personal (not shared); `.claude/CLAUDE.md` = team (git); subdirectory = local scope
2. **`context: fork`** isolates verbose skill output — use for any skill generating extended output
3. **Path-scoped rules** in `.claude/rules/` with `paths:` glob beat N directory CLAUDE.md files for cross-directory file-type conventions
4. **`-p` flag** is required for non-interactive CI — without it, Claude Code hangs
5. **Plan mode** = architectural/multi-file/multiple valid approaches; **direct execution** = clear, bounded, single-file tasks
