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

**Strong description structure (6 elements):**
1. What the tool does (one sentence)
2. Input formats accepted — specify accepted identifier formats (e.g., CUST-XXXXX vs ORD-XXXXX)
3. What the tool RETURNS — format, structure, and data shape of the output (e.g., "returns a JSON object with fields: `orderId`, `status`, `lineItems`")
4. Example queries that should trigger it
5. What it does NOT handle (boundary cases)
6. When to use it versus similar tools — require prerequisite steps (e.g., customer must be verified via `get_customer` before calling `lookup_order`)

**MCP Tools Must Outcompete Built-in Tools**

In systems where both built-in tools (Grep, Read, Glob) and MCP tools coexist, Claude defaults to familiar built-in tools unless the MCP tool description explicitly explains its advantage. An MCP `semantic_code_search` tool with a minimal description will lose to Grep every time. The MCP description must clarify: what it returns, why it outperforms the built-in alternative, and what query patterns it handles that Grep cannot.

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

**The `isError` Flag in the MCP Protocol**

`isError` is part of the `tool_result` content block — it is the standard MCP mechanism for communicating tool failures back through the protocol. Setting `isError: true` signals to the calling agent that the tool call did not succeed, without throwing an exception or collapsing the conversation. The payload alongside it (`errorCategory`, `isRetryable`, etc.) is what drives intelligent recovery. Without `isError: true`, the agent has no way to distinguish a failure from a valid empty result.

**The Four Error Categories and Recovery Strategies**

| Error Category | Example | isRetryable | Agent Recovery Strategy |
|---|---|---|---|
| Transient | Database timeout | Yes | Retry with exponential backoff (max 2 attempts locally); propagate if still failing |
| Validation | Invalid order ID format | No | Stop, surface the input error to the user for correction |
| Business | Refund exceeds $500 limit | No | Escalate to supervisor; include rule violated in propagated message |
| Permission | Lacks VIP account access | No | Escalate to senior agent; do not retry under current credentials |

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
- Subagent frontmatter fields: `tools` (allowlist) and `disallowedTools` (denylist) — when both are set, `disallowedTools` is applied first, then `tools` is resolved against the remaining pool
- `Agent(agent_type)` syntax in a `tools` list restricts which subagent types can be spawned (e.g., `tools: [Agent(worker), Agent(researcher), Read, Bash]`)
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

**Frontmatter Tool Control Fields**

`tools` is an allowlist — only listed tools are available to the subagent. `disallowedTools` is a denylist — listed tools are blocked even if they would otherwise be available. When both appear in the same frontmatter, the denylist is applied first to the full tool pool, and the allowlist is then resolved against what remains. `Agent(type)` entries in `tools` restrict which subagent types can be spawned — only the named types are permitted, preventing a subagent from spawning arbitrary agent types.

**tool_choice Configuration**

**When to use each setting:**
- `"auto"`: Default. Model may call a tool OR return text. Use when conversational responses are acceptable alongside tool use.
- `"any"`: Model MUST call a tool but chooses which. Use when guaranteed structured output is required and multiple extraction schemas are valid (e.g., unknown document type where any schema is correct).
- `{"type": "tool", "name": "X"}`: Model MUST call this specific tool. Use when a specific extraction must happen before subsequent enrichment steps (e.g., force `extract_metadata` first, then enrich in follow-up turns).

Key exam pattern: "Guaranteed structured output with unknown document type" → `"any"`. "Specific extraction must run first" → forced. "May respond conversationally" → `"auto"`.

| Setting | Syntax | When to Use |
|---|---|---|
| `"auto"` | `tool_choice: {"type": "auto"}` | Conversational responses are acceptable; tool call is optional |
| `"any"` | `tool_choice: {"type": "any"}` | Structured output required; model picks which tool; multiple valid schemas |
| Forced | `tool_choice: {"type": "tool", "name": "X"}` | Specific tool must run first; step ordering is critical; follow-up turns handle remaining steps |

**Constrained Tool Replacement**

Replace the generic `fetch_url` (accepts any URL) with `load_document`, which validates that the URL matches the internal document domain only (`https://docs.internal/`), states its purpose (loading coordinator-provided research documents), and explicitly rejects external URLs. The constrained tool prevents the synthesis agent from performing arbitrary web fetches while still covering its legitimate access need.

> **Exam Tip:** When asked about reliability degradation as tool count grows, remember the 4–5 tool ideal per agent. Questions will present agents with 10–20 tools and ask what to change; the correct answer always involves scoping, not improving descriptions for all tools.

> **Anti-Pattern Alert:** Giving the synthesis agent `web_search` "just in case" is the wrong answer. Cross-role tools should be scoped and constrained (like `verify_fact`), not the full search toolkit.

> **Cross-Domain Connection:** Task 1.3 covers `AgentDefinition` configuration where tool restrictions per subagent type are set during agent setup — same principle applied at SDK configuration level.

---

## Task Statement 2.4: Integrate MCP servers into Claude Code and agent workflows

### What You Need to Know
- MCP server scoping has **three levels**, with a defined override hierarchy:
  1. **Local scope** — `claude mcp add --local`, stored in `.claude/mcp.json` (NOT `.mcp.json`), current directory only, not committed
  2. **Project scope** — `.mcp.json` in the project root, committed to version control, shared with the team
  3. **User scope** — `~/.claude.json`, personal, applies across all projects
- Scope override order: **local > project > user** (local overrides project, project overrides user)
- Environment variable expansion in `.mcp.json` (e.g., `${GITHUB_TOKEN}`) for credential management without committing secrets
- MCP servers can be scoped to specific subagents using the `mcpServers` field in subagent frontmatter — entries can be inline definitions (new server config) or string references (reuse an existing named server)
- **MCP Tool Search (deferred tool loading):** for servers with large numbers of tools, tools are discovered lazily rather than all at connection time — reduces startup overhead
- **MCP prompts** can be surfaced as slash commands inside Claude Code
- **Dynamic tool updates:** MCP tools can change at runtime; the agent reflects the updated set without reconnecting
- **Push messages via channels:** MCP servers can push messages to the agent proactively through channels
- **Managed MCP configuration:** enterprise deployments can enforce allowlists/denylists on MCP servers centrally
- **MCP resources** expose content catalogs (issue summaries, documentation hierarchies, database schemas) to reduce exploratory tool calls

### What You Need to Be Able to Do
- Configure shared MCP servers in project-scoped `.mcp.json` with environment variable expansion for authentication tokens
- Configure personal/experimental MCP servers in user-scoped `~/.claude.json`
- Configure directory-local MCP servers using `claude mcp add --local` (stored in `.claude/mcp.json`)
- Apply the correct scope given a scenario: local (current dir only, not committed), project (team-shared, committed), user (personal, all projects)
- Scope an MCP server to a specific subagent using the `mcpServers` field in that subagent's frontmatter
- Enhance MCP tool descriptions to explain capabilities, preventing the agent from preferring built-in tools (like `Grep`) over more capable MCP tools
- Choose existing community MCP servers over custom implementations for standard integrations (e.g., Jira)
- Expose content catalogs as MCP resources to give agents visibility into available data without exploratory tool calls

### Deep Dive

**Three-Scope MCP Configuration**

| Scope | File Location | Committed? | Applies To | Added Via |
|---|---|---|---|---|
| Local | `.claude/mcp.json` | No | Current directory only | `claude mcp add --local` |
| Project | `.mcp.json` (project root) | Yes | All team members on this project | Edit file directly |
| User | `~/.claude.json` | No | All projects for this user | `claude mcp add` (no flag) |

Override order: **local > project > user**. A local server definition shadows a project-level server with the same name.

Key scoping decision: "Team uses same tools" → `.mcp.json` (project-level, committed). "Personal experimental server" → `~/.claude.json` (user-level, not committed).

Never hardcode API tokens in `.mcp.json` — always use `${ENV_VAR}` expansion for credentials:

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

**Subagent-Scoped MCP Servers**

The `mcpServers` field in a subagent's frontmatter restricts which MCP servers that subagent can access. Entries can be string references (reuse an already-configured server by name) or inline definitions (a new server configuration just for this subagent). This prevents a restricted subagent from calling MCP tools that belong to a different role.

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

**Resources vs. Tools distinction:** MCP resources are read-only structured data catalogs — they expose content for the agent to consult (issue summaries, database schemas, documentation). MCP tools are for actions — they execute operations, query live data, or trigger side effects. Resources are consumed passively; tools are invoked actively.

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
Need to find files by name/path? → Glob  (e.g., **/*.test.tsx, src/api/*.ts)
Need to search file contents? → Grep  (e.g., all files importing 'useAuth')
Need to read full file? → Read
Need targeted edit with unique anchor? → Edit
Edit fails (non-unique anchor)? → Read + Write
Need to run shell command? → Bash
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
