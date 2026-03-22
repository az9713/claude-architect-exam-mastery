# Scenario 2: Code Generation with Claude Code

## Scenario Context

You are using Claude Code to accelerate software development. Your team uses it for code generation, refactoring, debugging, and documentation. You need to integrate it into your development workflow with:

- **Custom slash commands** in `.claude/commands/` for repeatable team workflows
- **CLAUDE.md configurations** for project and directory-level coding standards
- **Skills** in `.claude/skills/` for on-demand task-specific workflows
- **Plan mode vs direct execution** decisions based on task complexity
- **Iterative refinement techniques** for progressive improvement

**Primary Domains:** D3 (Claude Code Configuration), D5 (Context Management & Reliability)

---

## Key Concepts for This Scenario

### Relevant Task Statements and Why They Matter

**Task Statement 3.1 — CLAUDE.md Hierarchy**
Claude Code loads configuration files at three levels: user (~/.claude/CLAUDE.md), project (.claude/CLAUDE.md or root CLAUDE.md), and directory (subdirectory CLAUDE.md files). Understanding the hierarchy matters because: user-level settings apply only to that user and are NOT shared via version control, while project-level settings are committed to the repo and received by every team member. A new developer not receiving expected behavior is often a hierarchy diagnostic problem.

**Task Statement 3.2 — Custom Slash Commands and Skills**
Project-scoped commands live in `.claude/commands/` and are version-controlled — available to every developer who clones the repo. Skills in `.claude/skills/` support frontmatter configuration: `context: fork` runs the skill in an isolated subagent context (preventing verbose skill output from polluting the main conversation), `allowed-tools` restricts which tools the skill can use, and `argument-hint` prompts for required parameters. The distinction between skills (on-demand invocation) and CLAUDE.md (always-loaded standards) matters for the exam.

**Task Statement 3.3 — Path-Specific Rules**
`.claude/rules/` files use YAML frontmatter `paths:` fields with glob patterns for conditional rule activation. A rule with `paths: ["**/*.test.tsx"]` loads only when editing test files — regardless of which directory they are in. This is the key advantage over subdirectory CLAUDE.md files, which cannot span multiple directories. When test conventions must apply everywhere tests exist, path-specific rules are the right mechanism.

**Task Statement 3.4 — Plan Mode vs Direct Execution**
Plan mode is for complex tasks with large-scale changes, multiple valid approaches, architectural decisions, and multi-file modifications. Direct execution is for simple, well-scoped changes. The decision should be made *before* starting based on stated requirements — waiting until "unexpected complexity emerges" during direct execution risks costly rework. The Explore subagent is useful for isolating verbose discovery output during multi-phase tasks.

**Task Statement 3.5 — Iterative Refinement**
When prose descriptions produce inconsistent results, 2–3 concrete input/output examples are the most effective clarification technique. The interview pattern (having Claude ask questions before implementing) surfaces design considerations in unfamiliar domains. Test-driven iteration (write tests first, share failures to guide improvement) applies well to algorithmic code. When multiple issues interact, address them in a single message rather than sequentially.

**Task Statement 5.4 — Context Management in Large Codebase Exploration**
In extended Claude Code sessions, context degrades: models start referencing "typical patterns" rather than specific classes discovered earlier. Scratchpad files persist key findings across context boundaries. The `/compact` command reduces context usage. Subagent delegation (using the Explore subagent) isolates verbose discovery while the main agent maintains high-level coordination.

---

## Practice Questions

### Question 1

You want to create a custom `/review` slash command that runs your team's standard code review checklist. This command should be available to every developer when they clone or pull the repository. Where should you create this command file?

**A)** In the `.claude/commands/` directory in the project repository

**B)** In `~/.claude/commands/` in each developer's home directory

**C)** In the CLAUDE.md file at the project root

**D)** In a `.claude/config.json` file with a `commands` array

**Correct Answer: A**

**Explanation:** Project-scoped custom slash commands go in `.claude/commands/` within the repository. This directory is version-controlled, so commands are automatically available to all developers when they clone or pull the repo — exactly what the requirement specifies. Option B (`~/.claude/commands/`) stores personal commands that are not shared via version control; each developer would need to manually create the command file, which defeats the purpose of team standardization. Option C (CLAUDE.md) is for always-loaded project instructions and context, not command definitions — you cannot define executable slash commands there. Option D describes a configuration mechanism that does not exist in Claude Code.

---

### Question 2

You have been assigned to restructure the team's monolithic application into microservices. This involves changes across dozens of files and requires decisions about service boundaries and module dependencies. Which approach should you take?

**A)** Enter plan mode to explore the codebase, understand dependencies, and design an implementation approach before making changes.

**B)** Start with direct execution and make changes incrementally, letting the implementation reveal the natural service boundaries.

**C)** Use direct execution with comprehensive upfront instructions detailing exactly how each service should be structured.

**D)** Begin in direct execution mode and only switch to plan mode if you encounter unexpected complexity during implementation.

**Correct Answer: A**

**Explanation:** Plan mode is designed for complex tasks with large-scale changes, multiple valid approaches, and architectural decisions — exactly what monolith-to-microservices restructuring requires. It enables safe codebase exploration and design before committing to any changes, preventing costly rework when the wrong service boundary is chosen mid-implementation. Option B risks exactly this: discovering wrong dependencies after making 20+ file changes. Option C assumes you already know the right structure without exploring the actual code — comprehensive upfront instructions without exploration are often wrong. Option D ignores that the complexity is already stated in the requirements, not something that might emerge later. Starting in direct execution when complexity is known upfront is backward.

---

### Question 3

Your codebase has distinct areas with different coding conventions. Test files are spread throughout the codebase alongside the code they test (e.g., `Button.test.tsx` next to `Button.tsx`). You want all tests to follow the same conventions regardless of location. What is the most maintainable way to ensure Claude automatically applies the correct test conventions when generating code?

**A)** Create rule files in `.claude/rules/` with YAML frontmatter specifying glob patterns to conditionally apply conventions based on file paths.

**B)** Consolidate all conventions in the root CLAUDE.md file under headers for each area, relying on Claude to infer which section applies.

**C)** Create skills in `.claude/skills/` for each code type that include the relevant conventions in their SKILL.md files.

**D)** Place a separate CLAUDE.md file in each subdirectory containing that area's specific conventions.

**Correct Answer: A**

**Explanation:** `.claude/rules/` with glob patterns (e.g., `paths: ["**/*.test.tsx"]`) allows conventions to be automatically applied based on file paths regardless of directory location — essential when test files are scattered throughout the codebase. When Claude Code opens `src/components/Button.test.tsx`, the test-conventions rule activates automatically. Option B relies on inference: the model must read the full CLAUDE.md, recognize which section applies, and apply it correctly — this is unreliable and loads irrelevant context. Option C requires manual skill invocation; the requirement says "automatically applied," which rules out on-demand invocation. Option D cannot handle files spread across many directories: a CLAUDE.md in `/src/components/` won't load when editing a test in `/src/hooks/`.

---

### Question 4

Your team creates a `/analyze-codebase` skill that does an exhaustive exploration of the project, producing verbose output with detailed findings. After using it, you notice that the main conversation context is now cluttered with thousands of lines of exploration output, which degrades the quality of subsequent interactions. How should you fix the skill configuration?

**A)** Add `context: fork` to the skill's YAML frontmatter so the skill runs in an isolated subagent context and returns only a summary to the main conversation.

**B)** Add `allowed-tools: []` to the skill's YAML frontmatter to prevent it from making any tool calls that generate verbose output.

**C)** Move the skill file from `.claude/skills/` to `~/.claude/skills/` so it runs with personal rather than project scope.

**D)** Use the `/compact` command after running the skill each time to reduce context usage.

**Correct Answer: A**

**Explanation:** `context: fork` is precisely the mechanism for skills that produce verbose output or exploratory context. With `context: fork`, the skill runs in an isolated subagent context — all the verbose exploration happens in that isolated context, and only a summary is returned to the main conversation. This prevents skill output from polluting and degrading subsequent interactions. Option B (`allowed-tools: []`) would prevent the skill from using tools at all, making codebase analysis impossible — it restricts which tools are available, not how verbose output is handled. Option C changing the scope from project to personal doesn't affect the context pollution problem. Option D using `/compact` manually each time is a workaround, not a fix — it also compresses other valuable context that should be retained.

---

### Question 5

You are asking Claude Code to implement a caching layer for your API service. After two attempts, the implementation is still wrong — Claude misunderstood the transformation you wanted. The issue is that your prose description ("cache responses for 5 minutes unless invalidated") is being interpreted differently each time. What is the most effective technique to fix this?

**A)** Provide 2–3 concrete input/output examples showing exactly what the transformation should look like for representative cases, including edge cases like manual invalidation.

**B)** Rewrite the prose description in more detail, adding three paragraphs explaining the caching logic in depth.

**C)** Use the interview pattern — ask Claude to ask you questions about the caching requirements before implementing.

**D)** Switch to plan mode so Claude can analyze the existing codebase structure before proposing the implementation.

**Correct Answer: A**

**Explanation:** When natural language descriptions are interpreted inconsistently across multiple attempts, concrete input/output examples are the most effective correction technique. Examples show the exact transformation: "given this API call with these parameters, the cache should return this stored response (not make a new API call)." Examples remove ambiguity that prose creates. Option B (more detailed prose) compounds the problem — if two attempts with prose didn't converge, more prose is unlikely to help. Option C (interview pattern) is excellent for *unknown* requirements in unfamiliar domains but is less effective when you already know what you want and the issue is communication precision. The problem is not missing requirements — it is imprecise specification. Option D (plan mode) is useful for architectural decisions, not for clarifying a single transformation rule.

---

## Guided Walkthrough

### Problem: Setting Up a Multi-Language Monorepo Configuration

Your monorepo has a React frontend (`/frontend`), a Node.js API (`/api`), and Terraform infrastructure (`/infra`). Each area has distinct conventions and test patterns. You need Claude Code to automatically apply the right rules in each area.

**Step 1: Establish the Project-Level Baseline**

Create `/CLAUDE.md` at the project root with universal standards that apply everywhere:

```markdown
# Project Standards

This is a monorepo with frontend (React), API (Node.js), and infrastructure (Terraform) areas.

## Universal Standards
- All code must pass existing linter checks before committing
- Use descriptive names; avoid abbreviations except widely-known ones (URL, API, ID)
- Write tests alongside all new features

## File Structure
/frontend — React + TypeScript UI
/api — Node.js Express API
/infra — Terraform infrastructure
```

**Step 2: Create Path-Specific Rule Files**

Instead of cluttering CLAUDE.md with all conventions, create focused rule files:

`.claude/rules/frontend.md`:
```yaml
---
paths: ["frontend/**/*"]
---
# Frontend Conventions
- Use functional React components with hooks
- TypeScript strict mode — no `any` types
- Component files: PascalCase (Button.tsx)
- CSS: Tailwind utility classes only
```

`.claude/rules/api.md`:
```yaml
---
paths: ["api/**/*"]
---
# API Conventions
- async/await for all async operations (not callbacks)
- Express error handler pattern: next(error) for all caught exceptions
- Input validation via Joi before any database operation
```

`.claude/rules/tests.md`:
```yaml
---
paths: ["**/*.test.ts", "**/*.test.tsx", "**/*.spec.ts"]
---
# Test Conventions
- Jest + React Testing Library for frontend
- Supertest for API integration tests
- Each test file must have a describe block matching the file under test
- Test names: "should [behavior] when [condition]"
```

**Step 3: Create a Project-Scoped Skill**

Create `.claude/skills/SKILL.md` for a `/generate-component` skill:

```markdown
---
name: generate-component
description: Generate a complete React component with TypeScript types and tests
argument-hint: "ComponentName"
context: fork
allowed-tools: ["Write", "Read"]
---

Generate a complete React component based on the component name provided.
Include: TypeScript interface, functional component, CSS module, test file.
Follow the conventions in frontend.md.
```

The `context: fork` ensures the verbose component generation happens in an isolated subagent, returning just the created files to the main context. The `allowed-tools` restriction prevents the skill from running shell commands or doing destructive operations.

**Step 4: Use Plan Mode for the Right Tasks**

Adding a button validation rule? Direct execution — it is a simple, well-scoped single-file change.

Migrating from `react-query` v3 to `tanstack-query` v5 across 45+ files? Plan mode — architectural implications, multiple valid migration patterns, multi-file changes. Enter plan mode, let Claude explore all usages, propose a migration plan, and get approval before executing.

**Step 5: Verify Configuration with /memory**

Run `/memory` inside Claude Code to inspect which memory files are loaded. This is the diagnostic tool for "why isn't Claude following my instructions?" — it shows exactly which CLAUDE.md files and rule files are currently active for the file you are editing.

---

## Try It Yourself

### Exercise: Configure a TypeScript API Project

You have a Node.js TypeScript API project with these characteristics:
- Source files in `/src` following strict TypeScript conventions
- Tests in `/tests` using Jest with a custom test helper at `tests/helpers/`
- Infrastructure scripts in `/scripts` that can use shell commands
- A CI workflow that should only use `claude -p` non-interactively

**Your tasks:**

1. **Design the CLAUDE.md hierarchy.** What goes in the project-level CLAUDE.md? What goes in path-specific rule files? Create the file structure with sample content for each.

2. **Create a skill definition** for a `/generate-endpoint` skill that creates a new Express route with TypeScript types and a test file. Should it use `context: fork`? Why or why not? What `allowed-tools` should you set?

3. **Diagnose a hierarchy bug.** A new team member reports that Claude Code is not following the TypeScript strict mode conventions. They have set this in their `~/.claude/CLAUDE.md`. What is the problem and how do you fix it?

4. **Plan mode decision.** You need to refactor error handling across all API routes to use a new centralized error handler pattern (affects 23 route files). Should you use plan mode or direct execution? Justify your answer using the specific criteria from Task Statement 3.4.

**Exam Connection:** This exercise targets Task Statements 3.1, 3.2, 3.3, and 3.4. The key diagnostic insight (question 3) is that `~/.claude/CLAUDE.md` is user-scoped and not shared via version control — project standards must live in the project-level CLAUDE.md to be received by all team members.
