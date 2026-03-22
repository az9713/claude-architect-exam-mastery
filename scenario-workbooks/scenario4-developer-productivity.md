# Scenario 4: Developer Productivity with Claude

## Scenario Context

You are building developer productivity tools using the Claude Agent SDK. The agent helps engineers:

- **Explore unfamiliar codebases** — navigating large, undocumented legacy systems
- **Understand legacy systems** — tracing data flows and dependencies
- **Generate boilerplate code** — creating consistent scaffolding from patterns
- **Automate repetitive tasks** — running checks, generating summaries, filing reports

The agent uses built-in tools (Read, Write, Edit, Bash, Grep, Glob) and integrates with Model Context Protocol (MCP) servers for team-specific backend access.

**Primary Domains:** D2 (Tool Design & MCP), D3 (Claude Code Config), D1 (Agentic Architecture)

---

## Key Concepts for This Scenario

### Relevant Task Statements and Why They Matter

**Task Statement 2.4 — MCP Server Integration**
Project-level `.mcp.json` stores shared team tooling (committed to version control, available to all developers). User-level `~/.claude.json` stores personal or experimental servers (not shared). Environment variable expansion (`${GITHUB_TOKEN}`) in `.mcp.json` handles credentials without committing secrets. All configured MCP servers are discovered at connection time and available simultaneously. MCP resources expose content catalogs (issue summaries, documentation hierarchies, database schemas) that reduce exploratory tool calls — the agent can see what's available without having to guess.

**Task Statement 2.5 — Built-in Tool Selection**
Grep searches file *contents* (function names, error messages, import statements). Glob matches file *paths* by name or extension patterns. Read/Write handle full file operations; Edit makes targeted modifications using unique text matching. When Edit fails because the anchor text is not unique in the file, fall back to Read + Write for reliable modifications. Building codebase understanding incrementally (Grep to find entry points, Read to follow imports) is more efficient than reading all files upfront.

**Task Statement 3.1 — CLAUDE.md Configuration**
For developer productivity tools, the project-level CLAUDE.md should describe the codebase structure, key modules, architectural patterns, and which areas are safe to modify versus read-only. This context is always loaded, giving Claude the orientation it needs to navigate an unfamiliar codebase without exhaustive exploration every session.

**Task Statement 3.2 — Custom Skills**
Skills in `.claude/skills/` with `context: fork` run in isolated subagents — ideal for verbose codebase analysis tasks that should not pollute the main conversation. The `allowed-tools` frontmatter restricts which tools the skill can use during its execution.

**Task Statement 1.6 — Task Decomposition**
When exploring a legacy codebase to add comprehensive tests, the right decomposition is: first map structure (what modules exist, what they do), identify high-impact areas (most-used, most-risky code), then create a prioritized plan that adapts as dependencies are discovered. This dynamic adaptive decomposition contrasts with fixed sequential pipelines, which are appropriate for predictable tasks.

---

## Practice Questions

### Question 1

A developer needs to find all call sites for a function named `calculateDiscount` across a large codebase. What is the correct tool to use?

**A)** Grep, searching file contents for the pattern `calculateDiscount`

**B)** Glob, using the pattern `**/*.js` to find all JavaScript files, then reading each one

**C)** Read on each file in sequence, building up a list of all callers

**D)** Bash with `find . -name "*.js"` piped to a content search

**Correct Answer: A**

**Explanation:** Grep is the correct tool for searching file *contents* — finding all occurrences of a function name, import statement, or error message across a codebase. It is designed for exactly this purpose. Glob (Option B) is for path pattern matching — finding files by name or extension, not searching inside files. Using Glob followed by reading every file (Option C) to find content is inefficient and uses excessive tokens when Grep can find all matches directly. Option D uses the Bash tool for a search task when the dedicated Grep tool exists — the system instructions are explicit that dedicated tools should be preferred over Bash for tasks they cover.

---

### Question 2

You are configuring an MCP server that connects to your team's Jira instance for issue tracking. The server needs a Jira API token. You want all team members to be able to use this server when they clone the repository. How should you configure it?

**A)** Add the MCP server to `.mcp.json` in the project root using `${JIRA_TOKEN}` for the credential, and instruct team members to set `JIRA_TOKEN` in their environment.

**B)** Add the MCP server to `~/.claude.json` on each developer's machine with the API token hardcoded.

**C)** Add the MCP server to the project's CLAUDE.md file with environment variable instructions.

**D)** Add the MCP server to `.mcp.json` with the API token value directly committed to the repository.

**Correct Answer: A**

**Explanation:** Project-level `.mcp.json` is version-controlled and available to all developers when they clone the repository — the correct location for shared team tooling. Environment variable expansion (`${JIRA_TOKEN}`) handles the credential without committing it to the repo. Each developer sets the environment variable locally. Option B (`~/.claude.json`) is for personal or experimental servers and is not shared via version control — each developer would need to manually configure it. Option C (CLAUDE.md) provides configuration instructions, not actual MCP server configuration — CLAUDE.md is for project context and standards, not `.mcp.json` server definitions. Option D commits the API token to the repository, which is a security vulnerability.

---

### Question 3

When building codebase understanding incrementally, what is the recommended order of operations for exploring an unfamiliar module?

**A)** Use Grep to find entry points and relevant function names, then use Read to follow imports and trace data flows from those entry points.

**B)** Use Read to load all files in the module directory upfront, then use Grep to find specific patterns within the loaded content.

**C)** Use Glob to find all files in the module directory, then Read each file in alphabetical order to build a complete picture.

**D)** Use Bash to run any available documentation generators, then Read the generated documentation files.

**Correct Answer: A**

**Explanation:** Task Statement 2.5 describes the incremental approach: start with Grep to find entry points and key function names (this is fast and identifies what to read), then use Read to follow imports and trace flows from those specific entry points. This targeted approach avoids loading irrelevant files into context. Option B (loading all files first, then searching) is the reverse of the efficient approach — it consumes tokens loading files that may be irrelevant, then searches content that is already in context. Option C (alphabetical order) has no relationship to code dependency structure and will often read low-level utilities before understanding high-level architecture. Option D assumes documentation generators exist, which is often not the case in legacy systems.

---

### Question 4

A developer wants to add an MCP server for GitHub but your company already uses a standard open-source GitHub MCP server. A teammate suggests building a custom GitHub MCP server for more control. What is the best architectural decision?

**A)** Use the existing community GitHub MCP server; reserve custom MCP servers for team-specific workflows that community servers do not cover.

**B)** Build a custom GitHub MCP server to ensure full control over the tool interface, descriptions, and behavior.

**C)** Use both — the community server for standard operations and a custom server for advanced operations — to maximize capability.

**D)** Neither; prefer native Claude Code GitHub integrations over MCP server configuration.

**Correct Answer: A**

**Explanation:** Task Statement 2.4 is explicit: choose existing community MCP servers over custom implementations for standard integrations (Jira, GitHub, Slack). Reserving custom servers for team-specific workflows — workflows that no existing server covers — is the right architectural principle. Custom MCP servers require development, testing, and maintenance effort. For a standard integration like GitHub that already has a well-maintained community server, this effort is unnecessary. Option B (building custom for control) is over-engineering when a working solution exists. Option C (using both) is needlessly complex and creates potential tool confusion with overlapping capabilities. Option D mischaracterizes how Claude Code is designed — MCP server integration is a core feature, not a workaround.

---

### Question 5

You are using the Edit tool to modify a configuration constant that appears three times in a large configuration file. The Edit tool fails with an error indicating that the target text is not unique. What is the correct fallback approach?

**A)** Use Read to load the full file, modify the content in memory, then use Write to save the complete updated file.

**B)** Use Grep to find the exact line number where the first occurrence appears, then use Bash with a line-number-based sed command.

**C)** Use Glob to find similar configuration files and check if the pattern appears there instead.

**D)** Expand the Edit anchor text to include more surrounding context until it becomes unique.

**Correct Answer: A**

**Explanation:** Task Statement 2.5 explicitly describes this fallback: when Edit fails because anchor text is not unique, use Read + Write for reliable file modification. Read loads the full current file content, the model makes the necessary changes, and Write saves the complete updated content. This is reliable regardless of text uniqueness. Option D (expanding anchor text) is worth trying first if the duplication is caused by similar but not identical surrounding text — but when there are three identical occurrences of the constant being changed, expanding context won't make it unique. The fallback is Read + Write. Option B (Bash + sed) uses the Bash tool for a file modification task when dedicated tools exist. Option C is irrelevant to the problem — the constant is in the file we are editing.

---

## Guided Walkthrough

### Problem: Exploring a Legacy Codebase to Map the Refund Flow

A developer asks: "I need to understand how the refund process works in this legacy Node.js application so I can safely add logging to it."

**Step 1: Start with Grep — Find the Entry Point**

Rather than reading all files, start with a targeted content search:

```
Grep: pattern="processRefund|refund.*process|handleRefund"
      type="js"
```

This returns matches in 4 files: `refundController.js`, `refundService.js`, `paymentGateway.js`, and `orderModel.js`.

**Step 2: Read the Entry Point**

Read `refundController.js` — this is likely the entry point (controller pattern suggests it handles the HTTP layer). The file shows it calls `refundService.processRefund(orderId, amount)`.

**Step 3: Trace the Import Chain**

Read `refundService.js`. Find it imports from `paymentGateway.js` and `orderModel.js`. The service orchestrates: validate order → check refund eligibility → call payment gateway → update order status.

**Step 4: Glob for Test Coverage**

```
Glob: pattern="**/*.test.js"
```

Find `refundService.test.js` exists but `refundController.test.js` does not. Read the test file to understand expected behavior and edge cases that are already tested.

**Step 5: Scratchpad for Complex Systems**

For a complex legacy system with many interconnected modules, create a scratchpad file during exploration:

```
Write: path="refund-flow-analysis.md"
content: "
## Refund Flow Map
1. POST /refunds → refundController.js:handleRefund
2. refundController calls refundService.processRefund(orderId, amount)
3. refundService:
   - validates order exists via orderModel.findById
   - checks eligibility: order.status === 'delivered' && !order.refundProcessed
   - calls paymentGateway.initiateRefund(paymentId, amount)
   - updates order.refundProcessed = true on success
4. paymentGateway makes external Stripe API call
...
Safe to add logging: refundService.js lines 45-67 (before/after gateway call)
"
```

This scratchpad persists findings across context boundaries. In a subsequent session, reading the scratchpad restores the analysis without re-exploring the codebase.

**Step 6: MCP Integration for Issue Tracking**

With the refund flow mapped, create a Jira ticket for the logging task using the configured GitHub MCP server:

```
mcp_jira_create_issue(
  project="BACKEND",
  summary="Add observability logging to refund flow",
  description="Safe insertion points identified: refundService.js:45-67",
  type="Story"
)
```

---

## Try It Yourself

### Exercise: Design the Productivity Agent's Tool Strategy

You are building a developer productivity agent that helps engineers:
1. Find all usages of deprecated API methods across a 50,000-line TypeScript codebase
2. Generate replacement boilerplate for each usage
3. Create a migration tracking document

**Your tasks:**

1. **Tool selection sequence.** For finding deprecated API usages, what is the correct sequence: Grep then Read, or Glob then Read? What specific Grep pattern would you use if the deprecated method is `legacyFetch`? How do you then trace its callers?

2. **MCP server configuration.** The agent needs to post migration progress updates to a Slack channel and create GitHub issues for complex migrations. Should these be in `.mcp.json` or `~/.claude.json`? Write a sample `.mcp.json` with environment variable expansion for both servers.

3. **Context management.** The 50,000-line codebase analysis will generate extensive tool output. Design a scratchpad file structure to persist findings across context boundaries. What fields should each entry include?

4. **Edit vs Read+Write decision.** The agent needs to replace `legacyFetch('/api/users')` with `modernFetch('/api/users', {version: 2})` in 23 files. In some files, the pattern appears once (unique); in others, it appears multiple times (not unique for Edit). Design the agent's decision logic for choosing between Edit and Read+Write on each file.

**Exam Connection:** This exercise targets Task Statements 2.4, 2.5, and 5.4. Key concepts: Grep for content, Glob for paths, Read+Write fallback when Edit fails, scratchpad files for context persistence, and `.mcp.json` for team-shared MCP configuration.
