# Scenario 3: Multi-Agent Research System

## Scenario Context

You are building a multi-agent research system using the Claude Agent SDK. A coordinator agent delegates to specialized subagents:

- **Web Search Agent** — searches the web for relevant articles and primary sources
- **Document Analysis Agent** — analyzes academic papers and documents, extracts key findings
- **Synthesis Agent** — combines findings from multiple sources into a coherent narrative
- **Report Agent** — generates comprehensive, cited reports from synthesized findings

The system researches complex topics and produces comprehensive, cited reports with proper source attribution.

**Primary Domains:** D1 (Agentic Architecture), D2 (Tool Design & MCP), D5 (Context Management & Reliability)

---

## Key Concepts for This Scenario

### Relevant Task Statements and Why They Matter

**Task Statement 1.2 — Multi-Agent Orchestration**
The hub-and-spoke architecture places the coordinator at the center: all inter-subagent communication, error handling, and information routing flows through the coordinator. Subagents do not inherit the coordinator's conversation history — context must be explicitly passed. The coordinator decomposes research topics into subtasks, delegates to subagents, and aggregates results. A narrow coordinator decomposition is the most common source of incomplete research coverage.

**Task Statement 1.3 — Subagent Invocation and Context Passing**
The `Task` tool is the mechanism for spawning subagents — the coordinator's `allowedTools` must include `"Task"`. Subagent context is explicitly provided in the prompt: the synthesis agent must receive web search results and document analysis outputs directly in its prompt, not via automatic inheritance. Parallel subagents are spawned by emitting multiple `Task` tool calls in a single coordinator response (not across separate turns).

**Task Statement 1.6 — Task Decomposition Strategies**
Fixed sequential pipelines (prompt chaining) work for predictable multi-aspect reviews. Dynamic adaptive decomposition works for open-ended investigation tasks where later subtasks depend on what earlier ones discover. Research systems often benefit from iterative refinement: the coordinator evaluates synthesis output for coverage gaps, re-delegates to search and analysis with targeted queries, and re-invokes synthesis until coverage is sufficient.

**Task Statement 2.3 — Tool Distribution**
Each subagent should have access only to the tools needed for its specialization. A synthesis agent with access to all web search tools will misuse them (attempting searches when it should be synthesizing). Providing a scoped `verify_fact` tool to the synthesis agent for the 85% of simple fact-checks handles the common case without full web search access.

**Task Statement 5.3 — Error Propagation**
When a subagent fails, structured error context (failure type, attempted query, partial results, potential alternatives) enables intelligent coordinator recovery decisions. Generic error statuses ("search unavailable") hide valuable context. Silently suppressing errors (returning empty results as success) prevents any recovery. Terminating the entire workflow on a single subagent failure discards all other agents' valid results.

**Task Statement 5.6 — Information Provenance**
When subagents summarize findings, source attribution is lost during the summarization step unless claim-source mappings are explicitly required in the output structure. Each subagent must return structured outputs including source URLs, document names, page numbers, and publication dates. The synthesis agent must preserve these mappings when combining findings. Conflicting statistics from different credible sources should both be included with attribution, not resolved by arbitrary selection.

---

## Practice Questions

### Question 1

After running the system on the topic "impact of AI on creative industries," each subagent completes successfully, but the final reports cover only visual arts, completely missing music, writing, and film production. You examine the coordinator's logs and see it decomposed the topic into three subtasks: "AI in digital art creation," "AI in graphic design," and "AI in photography." What is the most likely root cause?

**A)** The synthesis agent lacks instructions for identifying coverage gaps in the findings it receives from other agents.

**B)** The coordinator agent's task decomposition is too narrow, resulting in subagent assignments that don't cover all relevant domains of the topic.

**C)** The web search agent's queries are not comprehensive enough and need to be expanded to cover more creative industry sectors.

**D)** The document analysis agent is filtering out sources related to non-visual creative industries due to overly restrictive relevance criteria.

**Correct Answer: B**

**Explanation:** The coordinator's logs reveal the root cause directly: it decomposed "creative industries" into only visual arts subtasks (digital art, graphic design, photography), completely omitting music, writing, and film. The subagents executed their assigned tasks correctly — they are working as designed. The problem is what they were assigned, which is entirely the coordinator's responsibility. Options A, C, and D incorrectly blame downstream agents that are working correctly within their assigned scope. The fix is in the coordinator's task decomposition logic: either broadening the initial decomposition (explicitly mapping "creative industries" to all sub-sectors) or implementing an iterative gap-detection loop where the coordinator evaluates synthesis output for missing domains and re-delegates.

---

### Question 2

The web search subagent times out while researching a complex topic. You need to design how this failure information flows back to the coordinator agent. Which error propagation approach best enables intelligent recovery?

**A)** Return structured error context to the coordinator including the failure type, the attempted query, any partial results, and potential alternative approaches.

**B)** Implement automatic retry logic with exponential backoff within the subagent, returning a generic "search unavailable" status only after all retries are exhausted.

**C)** Catch the timeout within the subagent and return an empty result set marked as successful.

**D)** Propagate the timeout exception directly to a top-level handler that terminates the entire research workflow.

**Correct Answer: A**

**Explanation:** Structured error context gives the coordinator the information it needs to make intelligent recovery decisions: whether to retry with a modified query, try an alternative approach (e.g., document analysis instead of web search), or proceed with partial results while annotating the gap. Option B's generic "search unavailable" status after exhausting retries hides what was attempted, what query failed, and whether partial results exist — the coordinator cannot make informed decisions from this. Option C suppresses the error by marking failure as success, which prevents any recovery and risks producing incomplete research outputs with no indication of what was missed. Option D terminates the entire workflow unnecessarily, discarding all valid results from other subagents when intelligent partial recovery was possible.

---

### Question 3

During testing, you observe that the synthesis agent frequently needs to verify specific claims while combining findings. Currently, when verification is needed, the synthesis agent returns control to the coordinator, which invokes the web search agent, then re-invokes synthesis with results. This adds 2–3 round trips per task and increases latency by 40%. Your evaluation shows that 85% of verifications are simple fact-checks (dates, names, statistics) while 15% require deeper investigation. What is the most effective approach?

**A)** Give the synthesis agent a scoped `verify_fact` tool for simple lookups, while complex verifications continue delegating to the web search agent through the coordinator.

**B)** Have the synthesis agent accumulate all verification needs and return them as a batch to the coordinator at the end of its pass, which then sends them all to the web search agent at once.

**C)** Give the synthesis agent access to all web search tools so it can handle any verification need directly without round-trips through the coordinator.

**D)** Have the web search agent proactively cache extra context around each source during initial research, anticipating what the synthesis agent might need to verify.

**Correct Answer: A**

**Explanation:** Option A applies the principle of least privilege: give the synthesis agent only the tools it needs for the 85% common case (simple fact verification with a scoped `verify_fact` tool) while preserving the existing coordination pattern for the 15% of complex verifications that require deeper investigation. This eliminates most round-trips without over-provisioning. Option B's batching approach creates blocking dependencies: synthesis steps may depend on earlier verified facts (you cannot synthesize section 2 until you have verified a fact from section 1), so batching at the end of a pass is not always possible. Option C over-provisions the synthesis agent with full web search tools, violating separation of concerns and degrading tool selection reliability by increasing complexity. Option D relies on speculative caching that cannot reliably predict what the synthesis agent will need to verify.

---

### Question 4

You need the web search agent and document analysis agent to investigate a research topic simultaneously rather than sequentially. How should the coordinator emit these parallel subagent invocations?

**A)** Emit multiple `Task` tool calls in a single coordinator response, each spawning an independent subagent.

**B)** Invoke the web search agent first, and in its result-handling code, programmatically trigger the document analysis agent before returning to Claude.

**C)** Configure the coordinator to run in a loop, emitting one `Task` tool call per iteration, but using an `async` flag to prevent blocking.

**D)** Use a separate "orchestrator" layer that manages concurrency outside the Claude SDK.

**Correct Answer: A**

**Explanation:** The correct mechanism for parallel subagent execution in the Claude Agent SDK is emitting multiple `Task` tool calls in a single coordinator response — not across separate turns. When the coordinator's response includes two `Task` calls simultaneously, the SDK executes them concurrently. Option B moves orchestration logic outside the agentic loop into result-handling code, which creates a fragile custom orchestration layer that bypasses SDK features. Option C misunderstands how the SDK works: there is no `async` flag on `Task` calls; concurrency comes from emitting multiple calls in a single response, not from flags. Option D adds unnecessary complexity when the SDK provides the mechanism natively.

---

### Question 5

The synthesis agent is combining findings from the web search agent and document analysis agent. Two credible sources give conflicting statistics on AI adoption rates in creative industries (42% vs 67%). How should the synthesis agent handle this?

**A)** Include both statistics with their source attribution and annotate the conflict, preserving both values in the report rather than arbitrarily selecting one.

**B)** Select the statistic from the more authoritative source (journal article over blog post) and use that as the single reported value.

**C)** Average the two statistics (54.5%) and note that "studies vary" in the report.

**D)** Return a conflict error to the coordinator and request that the web search agent find additional sources to resolve the discrepancy.

**Correct Answer: A**

**Explanation:** Task Statement 5.6 is explicit: conflicting statistics from credible sources should be annotated with source attribution rather than arbitrarily selected. The report should include both values with their sources and explicit conflict annotation. This preserves information for the reader to evaluate. Option B (selecting the "more authoritative" source) is an arbitrary decision that discards valid data — the document analyst cannot always determine which source is correct, and both may be measuring different things (different time periods, different geographic regions, different definitions of "adoption"). Option C (averaging) is statistically meaningless and misleading. Option D (requesting additional sources) adds latency and may not resolve the conflict — different methodologies may legitimately produce different numbers, and more sources do not eliminate conflicting values.

---

## Guided Walkthrough

### Problem: Building the Coordinator Decomposition Logic

Your coordinator agent receives: "Research the economic impacts of remote work on urban commercial real estate markets in the United States."

**Step 1: Coordinator Design**

The coordinator's system prompt should specify research goals and quality criteria rather than step-by-step procedural instructions, to enable subagent adaptability:

```
You are a research coordinator. For each topic, you must:
1. Decompose the topic into distinct subtopics covering all major aspects
2. Assign each subtopic to the appropriate subagent
3. Evaluate synthesis output for coverage gaps
4. Re-delegate targeted follow-up queries when gaps are found
5. Produce a final report only when all major aspects have sufficient coverage

Quality criteria: Each factual claim must have source attribution. Conflicting findings must be preserved with both sources annotated. Coverage must include: market trends, geographic variation, key data points, and forward-looking projections.
```

**Step 2: Initial Decomposition — Parallel Subagent Spawn**

The coordinator analyzes the topic and identifies distinct subtopics. It emits multiple `Task` calls in a single response:

```json
{
  "tool_calls": [
    {
      "tool": "Task",
      "parameters": {
        "agent": "WebSearchAgent",
        "prompt": "Search for current data (2023-2026) on US urban commercial real estate vacancy rates, focusing on major metropolitan areas (NYC, SF, Chicago, Seattle). Retrieve articles with specific vacancy percentages and year-over-year changes. Return results as structured JSON with: claim, source_url, publication_date, relevant_excerpt."
      }
    },
    {
      "tool": "Task",
      "parameters": {
        "agent": "DocumentAnalysisAgent",
        "prompt": "Analyze the following academic papers on remote work and commercial real estate: [paper_list]. Extract: key findings, methodologies, data collection dates, limitations. Return structured JSON with: claim, evidence_excerpt, document_name, page_number, publication_date."
      }
    }
  ]
}
```

Both agents execute concurrently. The coordinator waits for both results.

**Step 3: Context Passing to Synthesis Agent**

When both results arrive, the coordinator passes ALL findings directly to the synthesis agent's prompt:

```
You are a synthesis agent. Combine the following research findings into a coherent analysis.

WEB SEARCH FINDINGS:
[complete structured JSON from web search agent]

DOCUMENT ANALYSIS FINDINGS:
[complete structured JSON from document analysis agent]

Requirements:
- Preserve all source attribution (URLs, document names, dates)
- Annotate any conflicting statistics with both values and sources
- Organize into: market overview, geographic variation, causal factors, projections
- Include coverage_gaps field listing any aspects where findings are insufficient
```

**Step 4: Gap Detection Loop**

The synthesis agent returns its output including `coverage_gaps: ["limited data on secondary markets", "no forward-looking projections found"]`.

The coordinator detects gaps and re-delegates targeted queries:

```json
{
  "tool": "Task",
  "parameters": {
    "agent": "WebSearchAgent",
    "prompt": "Search specifically for: (1) remote work impact on commercial real estate in mid-sized US cities (not major metros), (2) commercial real estate forecast reports for 2026-2030. Return the same structured format as previous results."
  }
}
```

**Step 5: Error Propagation Scenario**

The web search agent times out on the follow-up query. It returns:

```json
{
  "error": true,
  "error_type": "timeout",
  "attempted_query": "remote work commercial real estate secondary cities 2026 forecast",
  "partial_results": [2 articles found before timeout],
  "alternatives": ["try narrower geographic scope", "use document analysis agent for existing reports"]
}
```

The coordinator uses this structured context to decide: proceed with the 2 partial results and annotate the gap in the final report, rather than retrying or terminating.

---

## Try It Yourself

### Exercise: Design the Synthesis Agent's Output Schema

The synthesis agent must produce structured output that the report agent can consume while preserving full source attribution.

**Your tasks:**

1. **Design the JSON schema** for the synthesis agent's output. Include fields for: well-established findings (multiple sources agree), contested findings (sources conflict), coverage gaps (topics where sources were insufficient), and all source attribution metadata. Which fields should be required vs optional?

2. **Write the subagent context-passing prompt** for the synthesis agent. Include how you would pass web search findings AND document analysis findings so the synthesis agent can correlate and merge them. How do you ensure source attribution is preserved through this handoff?

3. **Design the iterative refinement loop.** The coordinator needs to detect when synthesis coverage is insufficient and re-delegate targeted queries. Write pseudocode for this loop including: how many iterations maximum, how the coordinator evaluates coverage completeness, and how partial results are incorporated into the final report.

4. **Handle a conflicting statistics scenario.** Two sources report AI adoption in creative industries as 42% (2024 survey) and 67% (2025 industry report). Write the structured representation the synthesis agent should produce for this conflict, preserving both values with full attribution.

**Exam Connection:** This exercise targets Task Statements 1.2, 1.3, 5.3, and 5.6. Key concepts: explicit context passing (subagents don't inherit coordinator history), parallel Task calls in a single response, structured error propagation, and conflict preservation with source attribution.
