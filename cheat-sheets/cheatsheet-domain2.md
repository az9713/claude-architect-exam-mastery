# Domain 2 Cheat Sheet: Tool Design & MCP Integration
**Weight: 18% | Task Statements: 2.1 – 2.5**

---

## Key Concepts

| Concept | Definition | Exam Relevance |
|---------|-----------|----------------|
| Tool description | Primary mechanism LLMs use for tool selection | First fix for misrouting; improve before adding routing layers |
| `isError` flag | MCP flag indicating tool execution failed | Must be `true` for all error responses |
| `errorCategory` | `transient` / `validation` / `permission` / `business` | Determines agent's recovery strategy |
| `isRetryable` | Boolean indicating retry will help | Prevents wasted retries on permanent failures |
| Scoped tool access | Agent receives only tools for its role | 4–5 tools ideal; more degrades reliability |
| `tool_choice: "auto"` | Model can call tool or return text | Default; allows conversational fallback |
| `tool_choice: "any"` | Model must call some tool | Guarantees structured output; model picks which tool |
| Forced tool selection | `{"type": "tool", "name": "X"}` | Model must call this specific tool |
| `.mcp.json` | Project-level MCP config (committed to git) | Shared team tooling; use `${ENV_VAR}` for credentials |
| `~/.claude.json` | User-level MCP config (not committed) | Personal/experimental servers only |
| `${ENV_VAR}` | Environment variable expansion in MCP config | Never hardcode credentials |
| MCP resource | Structured content catalog exposed by server | Reduces exploratory tool calls; gives upfront visibility |
| `Grep` | Built-in tool: searches file contents | Content patterns — NOT file path patterns |
| `Glob` | Built-in tool: matches file paths/names | File names/paths — NOT file contents |
| `Edit` | Built-in tool: targeted file modification | Fails on non-unique anchor text → use Read+Write |

---

## Decision Matrix

### Which `tool_choice` setting?

| Scenario | Setting |
|----------|---------|
| Agent may need to respond conversationally | `"auto"` |
| Multiple extraction schemas, unknown document type | `"any"` |
| Specific extraction must run before enrichment | `{"type": "tool", "name": "extract_metadata"}` |
| One schema, must use it | `{"type": "tool", "name": "..."}` |

### Which MCP config scope?

| Scenario | Location |
|----------|----------|
| Team uses shared GitHub, Jira, Postgres tools | `.mcp.json` (project-level) |
| Developer testing personal experimental server | `~/.claude.json` (user-level) |
| CI/CD pipeline shared tooling | `.mcp.json` (project-level) |
| Personal productivity tool not for teammates | `~/.claude.json` (user-level) |

### Which built-in tool?

| Task | Tool |
|------|------|
| Find all `.test.tsx` files | `Glob("**/*.test.tsx")` |
| Find all usages of `processRefund` | `Grep(pattern="processRefund")` |
| Read entire configuration file | `Read` |
| Change a unique function signature | `Edit` |
| Change non-unique text (Edit fails) | `Read` → modify → `Write` |
| System command / shell operation | `Bash` |

---

## Code/Config Reference

```json
// .mcp.json — project-level (committed, team-shared)
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

```python
# Structured MCP error response — 4 categories
return {
    "content": [{"type": "text", "text": json.dumps({
        "errorCategory": "transient",     # transient/validation/permission/business
        "isRetryable": True,
        "description": "Database timed out after 5s",
        "suggestion": "Retry after 2 seconds"
    })}],
    "isError": True
}

# tool_choice options
tool_choice={"type": "auto"}                    # may return text
tool_choice={"type": "any"}                     # must call a tool
tool_choice={"type": "tool", "name": "X"}       # must call this specific tool
```

---

## Anti-Patterns (Common Wrong Answers)

- **Minimal tool descriptions**: "Retrieves customer information" — causes misrouting; always include input formats, examples, boundaries
- **Consolidating overlapping tools as first fix**: not the best first step; improve descriptions first
- **Adding a routing layer before fixing descriptions**: over-engineered; descriptions are the root cause
- **Generic errors** `{"error": "Operation failed"}`: prevents intelligent recovery; always add `errorCategory` + `isRetryable`
- **Returning empty results as errors**: empty list from a successful search is NOT an error
- **18 tools for one agent**: degrades selection reliability; scope to 4–5 tools per agent
- **Hardcoding API tokens in `.mcp.json`**: security violation; always use `${ENV_VAR}` expansion
- **Custom MCP server for standard integrations** (GitHub, Jira): use community servers; custom = team-specific workflows only

---

## If You Remember Nothing Else

1. **Tool descriptions are the root cause of misrouting** — improve descriptions first; routing layers and consolidation are not the first step
2. **Structured errors require `isError: true`, `errorCategory`, `isRetryable`** — generic errors prevent recovery
3. **4–5 tools per agent** is the reliability ideal — more tools degrade selection
4. **`.mcp.json` = team/project (committed); `~/.claude.json` = personal (not committed)** — credentials always via `${ENV_VAR}`
5. **`tool_choice: "any"` guarantees a tool call; `"auto"` may return text** — use `"any"` when structured output is required
