# Lab Guide: Claude Agent SDK

5 exercises focused on core Agent SDK concepts: agentic loops, tool calling, hooks, subagents, and error handling.

**Prerequisites:** Python 3.9+ or Node.js 18+, Anthropic SDK installed (`pip install anthropic` or `npm install @anthropic-ai/sdk`), API key set as `ANTHROPIC_API_KEY`.

---

## Exercise 1: Implementing the Agentic Loop

**Task Statement:** 1.1
**Objective:** Build a correct agentic loop that handles `stop_reason` properly.

### Background

The agentic loop has four steps:
1. Send a request to Claude (with tools available)
2. Inspect `stop_reason` in the response
3. If `"tool_use"` → execute the tools, append results to history, go to step 1
4. If `"end_turn"` → present the final response to the user

### Exercise

Implement a weather-checking agent with two tools: `get_current_weather` and `get_forecast`.

```python
import anthropic
import json

client = anthropic.Anthropic()

# Mock tool implementations
def get_current_weather(city: str) -> dict:
    return {"city": city, "temperature": 22, "condition": "Sunny", "humidity": 60}

def get_forecast(city: str, days: int) -> dict:
    return {"city": city, "days": days, "forecast": [{"day": i+1, "high": 20+i, "low": 15} for i in range(days)]}

tools = [
    {
        "name": "get_current_weather",
        "description": "Get the current weather conditions for a city. Use this when the user asks about current/right now weather.",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "City name"}
            },
            "required": ["city"]
        }
    },
    {
        "name": "get_forecast",
        "description": "Get a multi-day weather forecast for a city. Use this when the user asks about future weather or a forecast.",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "City name"},
                "days": {"type": "integer", "description": "Number of days (1-7)", "minimum": 1, "maximum": 7}
            },
            "required": ["city", "days"]
        }
    }
]

def run_agent(user_message: str) -> str:
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=1024,
            tools=tools,
            messages=messages
        )

        # STEP 2: Inspect stop_reason
        if response.stop_reason == "end_turn":
            # Extract text content from the final response
            for block in response.content:
                if hasattr(block, 'text'):
                    return block.text

        elif response.stop_reason == "tool_use":
            # STEP 3: Execute tools and append results
            # First, append the assistant's response (with tool_use blocks) to history
            messages.append({"role": "assistant", "content": response.content})

            # Execute all requested tools
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    tool_name = block.name
                    tool_input = block.input

                    # Execute the tool
                    if tool_name == "get_current_weather":
                        result = get_current_weather(**tool_input)
                    elif tool_name == "get_forecast":
                        result = get_forecast(**tool_input)
                    else:
                        result = {"error": f"Unknown tool: {tool_name}"}

                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": json.dumps(result)
                    })

            # Append tool results to history
            messages.append({"role": "user", "content": tool_results})
            # Loop continues — sends updated messages back to Claude

        else:
            # Unexpected stop reason
            raise ValueError(f"Unexpected stop_reason: {response.stop_reason}")

# Test
print(run_agent("What's the weather like in London right now, and what will it be like for the next 3 days?"))
```

### What to Observe

1. **The loop continues as long as stop_reason is "tool_use"** — the agent may call multiple tools across several iterations
2. **Tool results are appended as "user" role messages** — Claude needs to see them to reason about what to do next
3. **The assistant's response content (including tool_use blocks) must be appended before tool results** — this maintains the correct conversation structure

### Anti-Patterns to Avoid
- Checking for text content in the response to decide if the loop is done (use `stop_reason` instead)
- Setting arbitrary iteration caps as the primary stopping mechanism
- Parsing natural language like "I'm done" to determine loop termination

---

## Exercise 2: Tool Call Interception with Hooks

**Task Statement:** 1.5
**Objective:** Implement pre-call hooks to enforce business rules and PostToolUse hooks for data normalization.

### Exercise A — Pre-Call Hook (Business Rule Enforcement)

```python
from anthropic import Anthropic
import json
from functools import wraps

client = Anthropic()

# Business rule: block refunds above $500
REFUND_LIMIT = 500.0

# Hook: intercept process_refund calls before execution
def pre_call_hook(tool_name: str, tool_input: dict) -> dict | None:
    """
    Returns None to allow the call.
    Returns a modified tool_input dict to redirect the call.
    Raises an exception to block the call.
    """
    if tool_name == "process_refund":
        amount = tool_input.get("amount", 0)
        if amount > REFUND_LIMIT:
            # Block and redirect to escalation
            return {
                "blocked": True,
                "reason": f"Refund amount ${amount} exceeds the ${REFUND_LIMIT} automated limit",
                "action": "escalate_to_human",
                "required_approver": "manager"
            }
    return None  # Allow all other calls

def execute_tool_with_hook(tool_name: str, tool_input: dict, tool_executor) -> dict:
    """Execute a tool, applying the pre-call hook first."""
    # Apply pre-call hook
    hook_result = pre_call_hook(tool_name, tool_input)

    if hook_result and hook_result.get("blocked"):
        # Return the hook result as the "tool result" instead of executing the tool
        return {
            "success": False,
            "blocked_by_policy": True,
            "redirect_to": hook_result["action"],
            "reason": hook_result["reason"]
        }

    # Execute the actual tool
    return tool_executor(tool_name, tool_input)
```

### Exercise B — PostToolUse Hook (Data Normalization)

```python
import datetime

def post_tool_use_hook(tool_name: str, raw_result: dict) -> dict:
    """
    Normalize tool outputs before the model processes them.
    Converts Unix timestamps to ISO 8601, normalizes status codes, etc.
    """
    if tool_name == "lookup_order":
        normalized = raw_result.copy()

        # Normalize Unix timestamps to ISO 8601
        for field in ["created_at", "updated_at", "shipped_at"]:
            if field in normalized and isinstance(normalized[field], (int, float)):
                normalized[field] = datetime.datetime.fromtimestamp(
                    normalized[field], tz=datetime.timezone.utc
                ).isoformat()

        # Normalize numeric status codes to human-readable strings
        status_map = {1: "pending", 2: "processing", 3: "shipped", 4: "delivered", 5: "refunded"}
        if "status_code" in normalized:
            normalized["status"] = status_map.get(normalized["status_code"], "unknown")
            del normalized["status_code"]  # Remove the raw code

        # Trim to only relevant fields (reduce context usage)
        relevant_fields = {"order_id", "status", "created_at", "total_amount", "refund_eligible"}
        normalized = {k: v for k, v in normalized.items() if k in relevant_fields}

        return normalized

    return raw_result  # No normalization for other tools

# Usage in the agentic loop:
# raw_result = execute_tool(tool_name, tool_input)
# normalized_result = post_tool_use_hook(tool_name, raw_result)
# Append normalized_result to conversation history
```

### What to Observe

1. **Pre-call hooks provide deterministic enforcement** — the `process_refund` tool is never called for amounts above the threshold
2. **PostToolUse hooks normalize data** — Claude receives clean, consistent data regardless of what the backend returns
3. **Hooks replace prompt-based rules** for scenarios requiring guaranteed compliance

---

## Exercise 3: Subagent Spawning with Task Tool

**Task Statement:** 1.2, 1.3
**Objective:** Build a coordinator-subagent pattern with explicit context passing.

### Exercise

```python
from anthropic import Anthropic
import json

client = Anthropic()

# Coordinator system prompt
COORDINATOR_PROMPT = """You are a research coordinator.
For research tasks, you delegate to specialized subagents using the Task tool.
Your allowedTools includes "Task" — use it to spawn subagents.
All subagent communication routes through you.

Available subagents:
- WebSearchAgent: searches for recent information on the web
- DocumentAnalysisAgent: analyzes provided documents and extracts key findings

After receiving subagent results, synthesize a comprehensive answer."""

# Subagent system prompts
SUBAGENT_PROMPTS = {
    "WebSearchAgent": """You are a web search specialist.
You receive a research query and return structured findings.
Return JSON with: [{claim, source_url, publication_date, relevant_excerpt}]
If your search fails, return structured error context.""",

    "DocumentAnalysisAgent": """You are a document analysis specialist.
You receive documents and extract key findings.
Return JSON with: [{claim, evidence_excerpt, document_section, confidence}]"""
}

# Task tool definition (for the coordinator)
task_tool = {
    "name": "Task",
    "description": "Spawn a specialized subagent to handle a delegated task. The subagent runs with isolated context — include ALL necessary information in the prompt.",
    "input_schema": {
        "type": "object",
        "properties": {
            "agent": {
                "type": "string",
                "enum": ["WebSearchAgent", "DocumentAnalysisAgent"],
                "description": "Which subagent to invoke"
            },
            "prompt": {
                "type": "string",
                "description": "Complete task instructions for the subagent. Include all context — subagents do NOT inherit coordinator context."
            }
        },
        "required": ["agent", "prompt"]
    }
}

def invoke_subagent(agent_name: str, prompt: str) -> str:
    """Execute a subagent with isolated context."""
    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=2048,
        system=SUBAGENT_PROMPTS[agent_name],
        messages=[{"role": "user", "content": prompt}]
        # NOTE: Subagent gets NO coordinator conversation history
        # All needed context must be in the prompt
    )
    return response.content[0].text

def run_coordinator(research_question: str) -> str:
    """Run the coordinator, which delegates to subagents."""
    messages = [{"role": "user", "content": research_question}]

    while True:
        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=2048,
            system=COORDINATOR_PROMPT,
            tools=[task_tool],
            messages=messages
        )

        if response.stop_reason == "end_turn":
            return response.content[0].text

        elif response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})

            tool_results = []
            # Check for parallel Task calls in a single response
            task_calls = [b for b in response.content if b.type == "tool_use" and b.name == "Task"]

            # Execute all Task calls (could be parallel subagents)
            for task_call in task_calls:
                agent = task_call.input["agent"]
                prompt = task_call.input["prompt"]
                # In production: run these concurrently with asyncio.gather
                result = invoke_subagent(agent, prompt)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": task_call.id,
                    "content": result
                })

            messages.append({"role": "user", "content": tool_results})

# Test
result = run_coordinator("What are the main economic impacts of remote work adoption in 2024?")
print(result)
```

### What to Observe

1. **Subagents receive NO coordinator history** — they only know what is in their prompt
2. **Multiple Task calls in a single response** enable parallel execution
3. **All subagent communication routes through the coordinator** for observability

---

## Exercise 4: Structured Error Propagation

**Task Statement:** 2.2, 5.3
**Objective:** Implement structured error responses that enable intelligent recovery.

### Exercise

```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional, Any

class ErrorCategory(Enum):
    TRANSIENT = "transient"      # Timeout, service unavailable — may retry
    VALIDATION = "validation"    # Invalid input — don't retry, fix the input
    PERMISSION = "permission"    # Access denied — don't retry, escalate
    BUSINESS = "business"        # Policy violation — don't retry, explain

@dataclass
class StructuredToolResult:
    success: bool
    data: Optional[Any] = None
    error_category: Optional[ErrorCategory] = None
    is_retryable: bool = False
    error_message: Optional[str] = None
    customer_message: Optional[str] = None
    partial_results: Optional[Any] = None
    attempted_operation: Optional[str] = None

    def to_dict(self) -> dict:
        result = {"success": self.success}
        if self.success:
            result["data"] = self.data
        else:
            result["error_category"] = self.error_category.value if self.error_category else None
            result["is_retryable"] = self.is_retryable
            result["error_message"] = self.error_message
            result["customer_message"] = self.customer_message
            if self.partial_results:
                result["partial_results"] = self.partial_results
            if self.attempted_operation:
                result["attempted_operation"] = self.attempted_operation
        return result

# Tool implementations with structured errors
def process_refund_tool(order_id: str, amount: float, customer_id: str) -> dict:
    """Process a refund with structured error responses."""

    # Simulate different error scenarios
    if amount > 500:
        return StructuredToolResult(
            success=False,
            error_category=ErrorCategory.BUSINESS,
            is_retryable=False,
            error_message=f"Refund amount ${amount} exceeds automated processing limit of $500",
            customer_message="Your refund requires manager approval. A team member will contact you within 24 hours.",
            attempted_operation=f"process_refund(order_id={order_id}, amount={amount})"
        ).to_dict()

    if order_id == "TIMEOUT_SIM":
        return StructuredToolResult(
            success=False,
            error_category=ErrorCategory.TRANSIENT,
            is_retryable=True,
            error_message="Payment gateway timed out after 30 seconds",
            attempted_operation=f"process_refund(order_id={order_id})",
            partial_results=None  # No partial results for a failed initiation
        ).to_dict()

    # Success
    return StructuredToolResult(
        success=True,
        data={"refund_id": "REF-001", "processing_time": "3-5 business days"}
    ).to_dict()

# Test structured error handling
result = process_refund_tool("ORD-123", 750, "CUST-001")
print(f"Business error (non-retryable): {result}")

result = process_refund_tool("TIMEOUT_SIM", 50, "CUST-001")
print(f"Transient error (retryable): {result}")
```

### What to Observe

1. **Error category enables recovery decisions** — the agent knows whether to retry, escalate, or explain
2. **`is_retryable` prevents wasted retry attempts** on business rule violations
3. **`customer_message`** gives the agent language to use with the user
4. **Distinguishing empty results from errors** — a valid empty result (`data: []`) is different from a failed access

---

## Exercise 5: Context Management in Multi-Turn Sessions

**Task Statement:** 5.1
**Objective:** Implement context trimming and case-facts persistence to prevent context degradation.

### Exercise

```python
from anthropic import Anthropic
import json

client = Anthropic()

class CustomerSupportSession:
    def __init__(self):
        self.messages = []
        self.case_facts = {}  # Persistent facts extracted from tool results

    def trim_tool_result(self, tool_name: str, raw_result: dict) -> dict:
        """Keep only relevant fields from verbose tool outputs."""
        if tool_name == "lookup_order":
            # An order record might have 40+ fields — keep only what matters
            relevant_fields = {
                "order_id", "status", "order_date", "total_amount",
                "refund_eligible", "return_window_days", "shipped_at"
            }
            return {k: v for k, v in raw_result.items() if k in relevant_fields}

        if tool_name == "get_customer":
            relevant_fields = {"customer_id", "name", "account_status", "verified"}
            return {k: v for k, v in raw_result.items() if k in relevant_fields}

        return raw_result

    def extract_and_persist_facts(self, tool_name: str, result: dict):
        """Extract transactional facts that must not be lost to summarization."""
        if tool_name == "get_customer":
            self.case_facts["customer_id"] = result.get("customer_id")
            self.case_facts["customer_name"] = result.get("name")
            self.case_facts["verified"] = result.get("verified")

        if tool_name == "lookup_order":
            self.case_facts["order_id"] = result.get("order_id")
            self.case_facts["order_amount"] = result.get("total_amount")
            self.case_facts["order_status"] = result.get("status")
            self.case_facts["refund_eligible"] = result.get("refund_eligible")

    def build_system_prompt(self) -> str:
        """Build system prompt including persistent case facts."""
        base_prompt = "You are a customer support agent with access to order management tools."

        if self.case_facts:
            facts_section = f"""
## Current Case Facts (DO NOT LOSE — preserved across summarization)
{json.dumps(self.case_facts, indent=2)}

These facts have been verified from system tools and must be maintained accurately.
Do not summarize these into vague descriptions.
"""
            return base_prompt + facts_section

        return base_prompt

    def process_tool_result(self, tool_name: str, raw_result: dict) -> dict:
        """Apply trimming and fact extraction before appending to context."""
        trimmed = self.trim_tool_result(tool_name, raw_result)
        self.extract_and_persist_facts(tool_name, trimmed)
        return trimmed
```

### What to Observe

1. **Trimming verbose outputs** prevents context from filling with irrelevant order fields
2. **Persistent case facts** survive even if conversation history is compacted or summarized
3. **"Lost in the middle" mitigation** — place key facts at the beginning of the system prompt, not in the middle of a long conversation

### Summary

| Exercise | Key Concept | Task Statement |
|----------|-------------|----------------|
| 1 — Agentic Loop | stop_reason handling, tool result appending | 1.1 |
| 2 — Hooks | Pre-call enforcement, PostToolUse normalization | 1.5 |
| 3 — Subagents | Task tool, explicit context passing, parallel calls | 1.2, 1.3 |
| 4 — Error Propagation | Structured errors, errorCategory, is_retryable | 2.2, 5.3 |
| 5 — Context Management | Trimming, case facts persistence | 5.1 |
