# Domain 2: Tool Design & MCP Integration
**Exam Weight: 18% | Task Statements: 2.1 – 2.5**

---

## Task Statement 2.1: Design effective tool interfaces with clear descriptions and boundaries

### What You Need to Know
- Tool descriptions are the **primary mechanism** LLMs use for tool selection; minimal or vague descriptions lead to unreliable selection among similar tools
- Effective descriptions include: input formats, example queries, edge cases, and boundary explanations (when to use this tool vs. similar tools)
- Ambiguous or overlapping descriptions cause misrouting (e.g., `analyze_content` vs `analyze_document` with near-identical descriptions)
- System prompt wording can create unintended tool associations; keyword-sensitive instructions can override well-written descriptions

### What You Need to Be Able to Do
- Write tool descriptions that clearly differentiate purpose, expected inputs, outputs, and when to use vs. similar alternatives
- Rename tools and update descriptions to eliminate functional overlap (e.g., rename `analyze_content` to `extract_web_results` with a web-specific description)
- Split generic tools into purpose-specific tools with defined input/output contracts (e.g., split `analyze_document` into `extract_data_points`, `summarize_content`, `verify_claim_against_source`)
- Review system prompts for keyword-sensitive instructions that might override well-written tool descriptions

### Deep Dive

**The Description-First Principle**

When Claude encounters multiple tools with similar names or scopes, the tool description is consulted first to decide which tool is appropriate. A minimal description like `"Retrieves customer information"` gives the model nothing to distinguish `get_customer` from `lookup_order` when both accept identifier inputs.

**Poor description:** `"Retrieves customer information"` — gives the model nothing to distinguish `get_customer` from `lookup_order` when both accept identifier inputs.

**Strong description structure (5 elements):**
1. What the tool does (one sentence)
2. Input formats accepted — specify accepted identifier formats (e.g., CUST-XXXXX vs ORD-XXXXX)
3. Example queries that should trigger it
4. What it does NOT handle (boundary cases)
5. When to use it versus similar tools — require prerequisite steps (e.g., customer must be verified via `get_customer` before calling `lookup_order`), and state what the tool returns.

**Splitting Generic Tools**

Before: one `analyze_document` tool forces the model to interpret what kind of analysis is needed. After: three purpose-specific tools with unambiguous intent — `extract_data_points` (returns JSON with structured facts and figures), `summarize_content` (returns plain text narrative), and `verify_claim_against_source` (returns supported/contradicted/absent).

> **Exam Tip:** Questions about tool misrouting always point to tool descriptions as the root cause. The correct answer involves improving descriptions, not adding routing layers or few-shot examples. Few-shot examples add overhead without fixing the root cause — descriptions first.

> **Anti-Pattern Alert:** Do not consolidate overlapping tools into one mega-tool as a "first step" fix. While architecturally valid in some cases, it requires more effort than description refinement and is never the best first step according to exam logic.

> **Cross-Domain Connection:** Task 1.4 covers programmatic enforcement to guarantee tool call ordering — this complements good descriptions by ensuring sequence compliance when descriptions alone are insufficient.

---

## Task Statement 2.2: Implement structured error responses for MCP tools

### What You Need to Know
- The MCP `isError` flag pattern communicates tool failures back to the agent
- Four error categories: **transient** (timeouts, service unavailability), **validation** (invalid input), **business** (policy violations), **permission** (access denied)
- Uniform generic errors (`"Operation failed"`) prevent the agent from making appropriate recovery decisions
- The difference between **retryable** and **non-retryable** errors — returning structured metadata prevents wasted retry attempts

### What You Need to Be Able to Do
- Return structured error metadata including `errorCategory` (transient/validation/permission/business), `isRetryable` boolean, and human-readable descriptions
- Include `retriable: false` flags and customer-friendly explanations for business rule violations
- Implement local error recovery within subagents for transient failures; propagate only unresolvable errors with partial results and what was attempted
- Distinguish between access failures (needing retry decisions) and valid empty results (successful queries with no matches)

### Deep Dive

**The Four Error Categories**

| Error Category | Example | isRetryable | Agent Action |
|---|---|---|---|
| Transient | Database timeout | Yes | Retry after backoff |
| Validation | Invalid order ID format | No | Ask customer to confirm |
| Business | Refund exceeds $500 limit | No | Escalate to supervisor |
| Permission | Lacks VIP account access | No | Escalate to senior agent |

All errors use `isError: true` in the MCP response, with a payload containing `errorCategory`, `isRetryable`, `description`, and `suggestion`.

**Empty Results vs. Access Failures**

An empty list (`[]`) is a successful query result — the customer simply has no matching orders. A `None` response or timeout exception is an access failure, flagged `isError: true` with `isRetryable: true`. Conflating the two causes the agent to either retry a legitimately empty search (wasted calls) or silently drop a real failure (missed error handling).

**Subagent Error Propagation Pattern**

Subagents handle transient errors locally — retry with exponential backoff up to a maximum of 2 attempts — before propagating anything to the coordinator. Non-retryable errors skip retries and propagate immediately. When propagating, the subagent includes error category, the failed query, attempt count, any partial results collected, and suggested alternatives for the coordinator to act on.

> **Exam Tip:** When a question asks about the agent retrying a refund or taking the wrong escalation path, check if the error response includes `errorCategory` and `isRetryable`. The correct answer will always add structured error metadata, not just improve the prompt.

> **Anti-Pattern Alert:** Returning `{"error": "Operation failed"}` is the prototypical bad answer in Domain 2 questions. It prevents the agent from distinguishing between a timeout (retry) and a policy violation (escalate immediately).

> **Cross-Domain Connection:** Task 5.3 covers error propagation across multi-agent systems — same structured error pattern applies at the coordinator level.

---

## Task Statement 2.3: Distribute tools appropriately across agents and configure tool choice

### What You Need to Know
- Too many tools (e.g., 18 instead of 4–5) **degrades tool selection reliability** by increasing decision complexity
- Agents with out-of-scope tools tend to misuse them (e.g., a synthesis agent attempting web searches)
- Scoped tool access: give agents only the tools needed for their role
- `tool_choice` options: `"auto"` (model may return text), `"any"` (model must call a tool), `{"type": "tool", "name": "..."}` (forced specific tool)

### What You Need to Be Able to Do
- Restrict each subagent's tool set to those relevant to its role
- Replace generic tools with constrained alternatives (e.g., replace `fetch_url` with `load_document` that validates document URLs)
- Provide scoped cross-role tools for high-frequency needs (e.g., a `verify_fact` tool for the synthesis agent) while routing complex cases through the coordinator
- Use forced tool selection to ensure a specific tool is called first, then process subsequent steps in follow-up turns
- Set `tool_choice: "any"` to guarantee the model calls a tool rather than returning conversational text

### Deep Dive

**Scoped Tool Distribution Pattern**

In a multi-agent research system, each agent receives only the tools for its role:

- **Coordinator:** orchestration tools only — `Task`, `summarize_findings`, `compile_report`
- **Search subagent:** search tools only — `web_search`, `fetch_url`, `search_database`
- **Analysis subagent:** analysis tools only — `extract_data_points`, `verify_claim_against_source`, `parse_citation`
- **Synthesis subagent:** synthesis tools plus one scoped cross-role tool — `combine_findings`, `structure_report`, `verify_fact`

> **Anti-Pattern Alert:** Loading all 12+ tools onto a single agent or giving every agent the full toolkit is the wrong answer. Cross-role tools should be scoped and constrained (like `verify_fact`), not the full search or analysis toolkit.

**tool_choice Configuration**

| Setting | Syntax | When to Use |
|---|---|---|
| `"auto"` | `tool_choice: {"type": "auto"}` | Model may respond conversationally; tool call is optional |
| `"any"` | `tool_choice: {"type": "any"}` | Must call a tool; model picks which one |
| Forced | `tool_choice: {"type": "tool", "name": "X"}` | Must call this specific tool first; follow-up turns handle remaining steps |

**Constrained Tool Replacement**

Replace the generic `fetch_url` (accepts any URL) with `load_document`, which validates that the URL matches the internal document domain only (`https://docs.internal/`), states its purpose (loading coordinator-provided research documents), and explicitly rejects external URLs. The constrained tool prevents the synthesis agent from performing arbitrary web fetches while still covering its legitimate access need.

> **Exam Tip:** When asked about reliability degradation as tool count grows, remember the 4–5 tool ideal per agent. Questions will present agents with 10–20 tools and ask what to change; the correct answer always involves scoping, not improving descriptions for all tools.

> **Anti-Pattern Alert:** Giving the synthesis agent `web_search` "just in case" is the wrong answer. Cross-role tools should be scoped and constrained (like `verify_fact`), not the full search toolkit.

> **Cross-Domain Connection:** Task 1.3 covers `AgentDefinition` configuration where tool restrictions per subagent type are set during agent setup — same principle applied at SDK configuration level.

---

## Task Statement 2.4: Integrate MCP servers into Claude Code and agent workflows

### What You Need to Know
- MCP server scoping: **project-level** (`.mcp.json`) for shared team tooling vs **user-level** (`~/.claude.json`) for personal/experimental servers
- Environment variable expansion in `.mcp.json` (e.g., `${GITHUB_TOKEN}`) for credential management without committing secrets
- All configured MCP server tools are discovered at connection time and available simultaneously
- **MCP resources** expose content catalogs (issue summaries, documentation hierarchies, database schemas) to reduce exploratory tool calls

### What You Need to Be Able to Do
- Configure shared MCP servers in project-scoped `.mcp.json` with environment variable expansion for authentication tokens
- Configure personal/experimental MCP servers in user-scoped `~/.claude.json`
- Enhance MCP tool descriptions to explain capabilities, preventing the agent from preferring built-in tools (like `Grep`) over more capable MCP tools
- Choose existing community MCP servers over custom implementations for standard integrations (e.g., Jira)
- Expose content catalogs as MCP resources to give agents visibility into available data without exploratory tool calls

### Deep Dive

**Project vs. User-Level MCP Configuration**

`.mcp.json` is committed to version control and shared with the team. Credentials are never hardcoded — they use `${ENV_VAR}` expansion:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    }
  }
}
```

`~/.claude.json` is the user-level config file (not committed, not shared) used for personal or experimental MCP servers that should not appear in the team's project configuration.

**When to Use Community vs. Custom MCP Servers**

```
Decision Rule:
- Standard integration (GitHub, Jira, Slack, databases)? → Use community MCP server
- Team-specific business workflow? → Custom MCP server

Reason: Community servers are maintained, battle-tested, and faster to implement.
Custom servers require maintenance and are only justified when no community
option fits the team's specific API conventions or data schemas.
```

**MCP Resources for Content Catalogs**

Without resources, the agent must make exploratory tool calls to discover what data exists (e.g., `list_projects()` → iterate each project → scan issues), consuming tokens and time before any real work begins. With resources, the MCP server exposes a catalog upfront:

```json
{
  "uri": "issues://sprint-summary",
  "name": "Current Sprint Issue Summary",
  "description": "Summary of all open issues in current sprint, grouped by team",
  "contents": { "sprintId": "SP-2026-12", "teamSummaries": { ... } }
}
```

Resources reduce exploratory tool calls by giving agents upfront visibility into what data is available before they decide what to query.

**Enhancing MCP Tool Descriptions**

The `semantic_code_search` description must explicitly distinguish it from `Grep`: unlike Grep (which matches literal text), this tool understands intent — searching "authentication flow" finds login, session, JWT, and OAuth code even when those exact words do not appear together. Best for conceptual discovery; use Grep for exact strings, imports, or error messages.

> **Exam Tip:** Scoping questions always have two choices: `.mcp.json` (shared/team) vs. `~/.claude.json` (personal). Team = project-level. Personal/experimental = user-level. Never commit credentials — always use `${ENV_VAR}` expansion.

> **Anti-Pattern Alert:** Hardcoding API tokens in `.mcp.json` is a security violation and a wrong exam answer. Environment variable expansion (`${TOKEN}`) is always correct.

> **Cross-Domain Connection:** Task 3.1 covers CLAUDE.md hierarchy (project vs. user-level) — the same project/user scoping logic applies to MCP configuration.

---

## Task Statement 2.5: Select and apply built-in tools (Read, Write, Edit, Bash, Grep, Glob) effectively

### What You Need to Know
- **Grep**: search file *contents* for patterns (function names, error messages, import statements)
- **Glob**: find files by *name/path patterns* (extensions, directory patterns)
- **Read/Write**: full file operations (load entire file; write entire file)
- **Edit**: targeted modification using unique text matching — fails if anchor text is not unique
- **When Edit fails** due to non-unique text matches: use Read + Write as fallback

### What You Need to Be Able to Do
- Select Grep for searching code content across a codebase (find all callers of a function, locate error messages)
- Select Glob for finding files matching naming patterns (e.g., `**/*.test.tsx`)
- Use Read to load full file contents followed by Write when Edit cannot find unique anchor text
- Build codebase understanding incrementally: start with Grep to find entry points, then use Read to follow imports and trace flows
- Trace function usage across wrapper modules by first identifying all exported names, then searching for each name across the codebase

### Deep Dive

**Tool Selection Decision Tree**

```
Need to find files?
  ├─ By name/path pattern → Glob  (e.g., **/*.test.tsx, src/api/*.ts)
  └─ By content → Grep  (e.g., all files importing 'useAuth')

Need to work with file content?
  ├─ Read full file → Read
  ├─ Make targeted change → Edit (if anchor text is unique)
  └─ Edit fails (non-unique anchor) → Read then Write

Need system operations? → Bash
```

**Built-in Tool Quick Reference**

| Task | Tool | Example |
|---|---|---|
| Find all callers of `processRefund` | Grep | `Grep(pattern="processRefund(", path="src/")` |
| Find all `.test.tsx` files | Glob | `Glob(pattern="**/*.test.tsx")` |
| Change unique function signature | Edit | `old_string` / `new_string` targeting that signature |
| Non-unique text edit | Read + Write | Load full file, modify in memory, write entire file back |

**Incremental Codebase Exploration**

```
Efficient pattern for unfamiliar codebases:
1. Glob("**/*.ts") — get file list
2. Grep("export default", path="src/") — find main exports
3. Read(entry_file) — understand structure
4. Grep("import.*from", path=entry_file) — find dependencies
5. Read(dependency) — trace the flow

NOT efficient:
- Reading all files upfront (context exhaustion)
- Using Bash with find (less efficient than Glob)
```

> **Exam Tip:** Tool selection questions give you a task (find all callers of X, find files with Y extension, modify a specific line) and ask which tool. The answer map is strict: content search = Grep, path matching = Glob, targeted edit = Edit (with Read+Write fallback).

> **Anti-Pattern Alert:** Using Bash with `grep` or `find` commands when dedicated tools exist is slower and less observable. Similarly, using `Read` on every file before searching is context-expensive.

> **Cross-Domain Connection:** Task 3.4 covers the Explore subagent — which uses these same tools internally during verbose discovery phases to prevent main context window exhaustion.

---

## Key Terminology Quick Reference

| Term | Definition | Exam Relevance |
|------|-----------|----------------|
| `isError` flag | MCP protocol flag indicating tool execution failed | Must be set to `true` for all error responses |
| `errorCategory` | Error type: transient/validation/permission/business | Determines agent's recovery strategy |
| `isRetryable` | Boolean indicating if retry will help | Prevents wasted retry attempts on permanent failures |
| `tool_choice: "auto"` | Model may call a tool or return text | Default; allows conversational fallback |
| `tool_choice: "any"` | Model must call some tool | Guarantees structured output when tools defined |
| `tool_choice: {"type": "tool", "name": "X"}` | Model must call this specific tool | Enforces ordered extraction steps |
| `.mcp.json` | Project-level MCP config (version controlled) | Shared team tooling, environment variable expansion |
| `~/.claude.json` | User-level MCP config (not version controlled) | Personal/experimental servers |
| MCP resource | Structured content catalog exposed by MCP server | Reduces exploratory tool calls; gives upfront visibility |
| Scoped tool access | Agent receives only tools for its role | 4–5 tools ideal; prevents cross-role misuse |
| `Grep` | Built-in tool for content search | File content patterns — NOT file name patterns |
| `Glob` | Built-in tool for path pattern matching | File names/paths — NOT file contents |
| `Edit` | Targeted file modification via unique anchor | Fails on non-unique anchor; fallback to Read+Write |

---

## Decision Matrix

### Which `tool_choice` option?

| Scenario | Choice | Reason |
|----------|--------|--------|
| Agent may need to respond conversationally | `"auto"` | Allows text fallback |
| Multiple extraction schemas, unknown document type | `"any"` | Forces a tool call; model picks the right schema |
| Metadata must be extracted before enrichment | `{"type": "tool", "name": "extract_metadata"}` | Guarantees ordering |
| Guaranteed structured output, single schema | `{"type": "tool", "name": "..."}` or `"any"` | Either works |

### Which MCP configuration scope?

| Scenario | Config Location |
|----------|----------------|
| Team uses same Jira, GitHub, Postgres tools | `.mcp.json` (project-level) |
| Developer testing a personal experimental server | `~/.claude.json` (user-level) |
| CI/CD pipeline shared tooling | `.mcp.json` (project-level) |
| Personal productivity tool not for teammates | `~/.claude.json` (user-level) |

### Which built-in tool?

| Task | Tool |
|------|------|
| Find all `.test.tsx` files | `Glob("**/*.test.tsx")` |
| Find all usages of `processRefund` | `Grep(pattern="processRefund")` |
| Read an entire configuration file | `Read` |
| Change a specific unique function signature | `Edit` |
| Change non-unique text in a file | `Read` + `Write` |
| Run a shell command | `Bash` |

---

## If You're Short on Time

1. **Tool descriptions are the root cause of misrouting** — improving descriptions is always the first fix, never routing layers or consolidation.
2. **Structured error responses require `isError: true`, `errorCategory`, and `isRetryable`** — generic errors prevent intelligent agent recovery.
3. **4–5 tools per agent is the reliability sweet spot** — more tools degrade selection; scope tool access to each agent's role.
4. **`.mcp.json` = team/project (committed); `~/.claude.json` = personal (not committed)** — always use `${ENV_VAR}` for credentials, never hardcode.
5. **`tool_choice: "any"` guarantees a tool is called; `tool_choice: "auto"` may return text** — use `"any"` when structured output is required.
