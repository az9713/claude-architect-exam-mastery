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

A high-quality description follows this structure:
- What the tool does (one sentence)
- What input formats it accepts (IDs, names, dates, etc.)
- Example queries that should trigger it
- What it does NOT handle (boundary cases)
- When to use it versus similar tools

```python
# Poor description — causes misrouting
tools = [
    {
        "name": "get_customer",
        "description": "Retrieves customer information",
        "input_schema": {
            "type": "object",
            "properties": {
                "identifier": {"type": "string"}
            }
        }
    },
    {
        "name": "lookup_order",
        "description": "Retrieves order details",
        "input_schema": {
            "type": "object",
            "properties": {
                "identifier": {"type": "string"}
            }
        }
    }
]

# Strong descriptions — reliable selection
tools = [
    {
        "name": "get_customer",
        "description": (
            "Look up a customer account by customer ID, email, or phone number. "
            "Use this tool FIRST before any order operations to verify identity. "
            "Returns: customer_id, name, account_status, tier. "
            "Use when: the user asks about their account, wants to verify identity, "
            "or before processing refunds/changes. "
            "Do NOT use for order number lookups — use lookup_order instead."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "identifier": {
                    "type": "string",
                    "description": "Customer ID (CUST-XXXXX), email, or phone"
                }
            },
            "required": ["identifier"]
        }
    },
    {
        "name": "lookup_order",
        "description": (
            "Look up order details by order number (e.g., ORD-12345). "
            "Use when the customer references a specific order number. "
            "Returns: order_id, status, items, amounts, shipping. "
            "Requires: a customer must already be verified via get_customer. "
            "Do NOT use for customer account information."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "order_id": {
                    "type": "string",
                    "description": "Order ID in format ORD-XXXXX"
                },
                "customer_id": {
                    "type": "string",
                    "description": "Verified customer ID from get_customer"
                }
            },
            "required": ["order_id", "customer_id"]
        }
    }
]
```

**Splitting Generic Tools**

A generic `analyze_document` tool forces the model to interpret what kind of analysis is needed. Purpose-specific tools communicate intent unambiguously:

```python
# Before: one generic tool
{
    "name": "analyze_document",
    "description": "Analyzes a document"
}

# After: three purpose-specific tools with clear boundaries
[
    {
        "name": "extract_data_points",
        "description": (
            "Extract structured numerical data, dates, and named entities from a document. "
            "Use when: you need facts, figures, statistics, or structured data from the text. "
            "Returns: a JSON object with extracted fields."
        )
    },
    {
        "name": "summarize_content",
        "description": (
            "Generate a concise prose summary of a document's main points. "
            "Use when: you need a narrative overview for human consumption. "
            "Returns: a plain text summary of 1-3 paragraphs."
        )
    },
    {
        "name": "verify_claim_against_source",
        "description": (
            "Check whether a specific claim is supported, contradicted, or absent in a document. "
            "Use when: you need to fact-check or validate a specific statement. "
            "Returns: {supported: bool, quote: str, confidence: float}"
        )
    }
]
```

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

```python
# MCP tool error response structure
def handle_tool_error(error_type, context):
    if error_type == "timeout":
        return {
            "content": [{
                "type": "text",
                "text": json.dumps({
                    "errorCategory": "transient",
                    "isRetryable": True,
                    "description": "Database connection timed out after 5s",
                    "suggestion": "Retry after 2 seconds; if persists, report to ops",
                    "attemptedQuery": context.get("query")
                })
            }],
            "isError": True
        }

    elif error_type == "invalid_order_id":
        return {
            "content": [{
                "type": "text",
                "text": json.dumps({
                    "errorCategory": "validation",
                    "isRetryable": False,
                    "description": "Order ID format invalid. Expected ORD-XXXXX",
                    "receivedValue": context.get("order_id"),
                    "suggestion": "Ask the customer to confirm their order number"
                })
            }],
            "isError": True
        }

    elif error_type == "refund_exceeds_limit":
        return {
            "content": [{
                "type": "text",
                "text": json.dumps({
                    "errorCategory": "business",
                    "isRetryable": False,
                    "description": "Refund amount ($750) exceeds automated limit ($500)",
                    "customerMessage": (
                        "Your refund request requires supervisor approval. "
                        "I'm connecting you with our team now."
                    ),
                    "escalationRequired": True
                })
            }],
            "isError": True
        }

    elif error_type == "unauthorized":
        return {
            "content": [{
                "type": "text",
                "text": json.dumps({
                    "errorCategory": "permission",
                    "isRetryable": False,
                    "description": "Agent lacks permission to access VIP account records",
                    "suggestion": "Escalate to senior agent with VIP access"
                })
            }],
            "isError": True
        }
```

**Empty Results vs. Access Failures**

This is a critical distinction. When a search returns no results legitimately (the customer has no orders), that is a successful operation. When a database is unavailable, that is an access failure. Conflating them causes the agent to either retry a genuinely empty search or silently drop real failures.

```python
def search_orders(customer_id, status_filter):
    try:
        results = db.query(customer_id, status_filter)

        if results is None:
            # Access failure — db returned None, not empty list
            return {
                "content": [{"type": "text", "text": json.dumps({
                    "errorCategory": "transient",
                    "isRetryable": True,
                    "description": "Order database returned unexpected null response"
                })}],
                "isError": True
            }

        # Empty list is a valid result — NOT an error
        return {
            "content": [{"type": "text", "text": json.dumps({
                "orders": results,          # [] is valid
                "totalFound": len(results),
                "queryDetails": {
                    "customerId": customer_id,
                    "statusFilter": status_filter
                }
            })}],
            "isError": False
        }

    except TimeoutError:
        return {
            "content": [{"type": "text", "text": json.dumps({
                "errorCategory": "transient",
                "isRetryable": True,
                "description": "Query timed out"
            })}],
            "isError": True
        }
```

**Subagent Error Propagation Pattern**

Subagents should resolve transient errors locally (retry once or twice) and only escalate errors they cannot resolve, always including partial results and what was attempted.

```python
class SearchSubagent:
    def search_with_recovery(self, query):
        # Attempt local recovery for transient errors
        for attempt in range(2):
            result = self.search_tool(query)
            if not result.get("isError"):
                return {"success": True, "data": result}

            error = json.loads(result["content"][0]["text"])
            if not error.get("isRetryable", False):
                break  # Non-retryable: propagate immediately

            time.sleep(2 ** attempt)  # Backoff

        # Propagate with context for coordinator
        return {
            "success": False,
            "errorCategory": error["errorCategory"],
            "failedQuery": query,
            "attemptsCount": attempt + 1,
            "partialResults": [],
            "alternatives": [
                "Try narrowing date range",
                "Search by order ID instead of customer name"
            ]
        }
```

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

In a multi-agent research system, each agent should receive only the tools it needs:

```python
# Coordinator agent — orchestration tools only
coordinator_tools = ["Task", "summarize_findings", "compile_report"]

# Search subagent — search tools only
search_agent_tools = ["web_search", "fetch_url", "search_database"]

# Analysis subagent — analysis tools only
analysis_agent_tools = ["extract_data_points", "verify_claim_against_source", "parse_citation"]

# Synthesis subagent — synthesis tools + one cross-role tool
synthesis_agent_tools = [
    "combine_findings",
    "structure_report",
    "verify_fact"        # cross-role: high-frequency need, simpler than routing to coordinator
]

# NOT this — one agent with all 12+ tools
bad_universal_tools = [
    "web_search", "fetch_url", "search_database",
    "extract_data_points", "verify_claim_against_source", "parse_citation",
    "combine_findings", "structure_report", "verify_fact",
    "compile_report", "summarize_findings", "Task"
]
```

**tool_choice Configuration**

```python
import anthropic

client = anthropic.Anthropic()

# tool_choice: "auto" — model can call a tool OR return text
# Use when: conversational responses are acceptable
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "auto"},
    messages=[{"role": "user", "content": "What can you do?"}]
)

# tool_choice: "any" — model MUST call some tool (no plain text)
# Use when: you need structured output and have multiple tool options
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    tools=[extract_invoice_tool, extract_receipt_tool, extract_report_tool],
    tool_choice={"type": "any"},  # Guarantees a tool is called
    messages=[{"role": "user", "content": document_text}]
)

# tool_choice: forced — model MUST call this specific tool
# Use when: a specific extraction must happen before enrichment steps
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    tools=all_tools,
    tool_choice={"type": "tool", "name": "extract_metadata"},  # Forced first
    messages=[{"role": "user", "content": document_text}]
)
# Process metadata, then send follow-up for enrichment
```

**Constrained Tool Replacement**

Replacing a generic tool with a constrained alternative limits misuse:

```python
# Generic tool — the synthesis agent can fetch arbitrary URLs
{
    "name": "fetch_url",
    "description": "Fetch content from any URL"
}

# Constrained replacement — validates only document storage URLs
{
    "name": "load_document",
    "description": (
        "Load a document from the team's document storage system. "
        "Accepts only URLs matching https://docs.internal/. "
        "Use for loading research documents passed by the coordinator. "
        "Will reject external URLs — use this instead of web_search for documents."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "document_url": {
                "type": "string",
                "pattern": "^https://docs\\.internal/",
                "description": "Internal document URL (docs.internal only)"
            }
        },
        "required": ["document_url"]
    }
}
```

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

```json
// .mcp.json (project-level, committed to version control, shared with team)
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"    // Expanded from environment — never hardcoded
      }
    },
    "jira": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-jira"],
      "env": {
        "JIRA_URL": "${JIRA_URL}",
        "JIRA_TOKEN": "${JIRA_TOKEN}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${DATABASE_URL}"]
    }
  }
}
```

```json
// ~/.claude.json (user-level, NOT committed, personal/experimental)
{
  "mcpServers": {
    "my-experimental-tool": {
      "command": "node",
      "args": ["/home/simon/dev/experimental-mcp/index.js"],
      "env": {
        "API_KEY": "${MY_PERSONAL_API_KEY}"
      }
    }
  }
}
```

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

MCP resources expose structured content to the agent without requiring exploratory tool calls. This is especially valuable when the agent needs to know what data exists before deciding what to query.

```python
# Without MCP resources: agent calls tools to discover what exists
# Exploration phase consumes tokens and time:
# 1. list_projects() → 47 projects
# 2. get_project_issues(project_1) → scan all...
# 3. get_project_issues(project_2) → scan all...

# With MCP resources: agent gets catalog upfront
# The MCP server exposes a resource:
{
    "uri": "issues://sprint-summary",
    "name": "Current Sprint Issue Summary",
    "description": "Summary of all open issues in current sprint, grouped by team",
    "mimeType": "application/json",
    "contents": {
        "sprintId": "SP-2026-12",
        "teamSummaries": {
            "backend": {
                "open": 14,
                "inProgress": 6,
                "highPriority": ["BACK-234", "BACK-891"]
            },
            "frontend": {
                "open": 9,
                "inProgress": 4,
                "highPriority": ["FRONT-102"]
            }
        }
    }
}
```

**Enhancing MCP Tool Descriptions**

When Claude Code has both built-in tools (like `Grep`) and MCP tools for similar tasks, the MCP tool needs a richer description to compete:

```json
{
  "name": "semantic_code_search",
  "description": (
    "Search the codebase using semantic meaning, not just text patterns. "
    "Unlike Grep (which matches literal text), this tool understands intent: "
    "searching 'authentication flow' finds login, session, JWT, and OAuth code "
    "even if those exact words don't appear together. "
    "Best for: understanding unfamiliar codebases, finding conceptually related code. "
    "Use Grep instead for: exact string matches, import statements, error messages."
  )
}
```

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

**Grep for Content Discovery**

```python
# Finding all callers of a function
Grep(pattern="processRefund(", path="src/")

# Finding import statements
Grep(pattern="from ['\"].*auth.*['\"]", path="src/", type="ts")

# Finding error message definitions
Grep(pattern="ERR_PAYMENT_", path="src/constants/")

# Building incremental understanding
# Step 1: Find entry point
Grep(pattern="export.*Router", path="src/routes/")
# Step 2: Read the route file
Read(file_path="src/routes/payment.ts")
# Step 3: Follow the import to the service
Read(file_path="src/services/payment-service.ts")
```

**Glob for File Discovery**

```python
# Find all test files regardless of directory
Glob(pattern="**/*.test.tsx")

# Find all TypeScript files in the API layer
Glob(pattern="src/api/**/*.ts")

# Find all CLAUDE.md configuration files
Glob(pattern="**/.claude/CLAUDE.md")
Glob(pattern="**/CLAUDE.md")

# Find all MCP configuration files
Glob(pattern="**/.mcp.json")
```

**Edit vs. Read+Write**

```python
# Edit — works when anchor text is unique
Edit(
    file_path="src/utils/validator.ts",
    old_string="function validateEmail(email: string) {",
    new_string="function validateEmail(email: string): boolean {"
)

# Edit fails when anchor text appears multiple times
# e.g., if there are multiple functions named the same
# → fallback: Read full file, modify in memory, Write entire file
content = Read(file_path="src/utils/validator.ts")
# modify content string
Write(file_path="src/utils/validator.ts", content=modified_content)
```

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
