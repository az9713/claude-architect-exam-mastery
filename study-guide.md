# Master Study Guide
**Claude Certified Architect – Foundations**

*Start here. This guide is your navigation hub for the entire study package.*

---

## Quick Navigation

| I want to... | Go to |
|-------------|-------|
| Start studying from scratch | [Study Plans](#study-plans) → pick your timeframe |
| Study a specific task statement | [By Domain & Task Statement](#study-by-domain-and-task-statement) |
| Prepare for a specific exam scenario | [By Scenario](#study-by-exam-scenario) |
| Take practice tests | [Quizzes](quizzes/) and [Mock Exams](mock-exams/) |
| Cram the night before | [Cheat Sheets](cheat-sheets/) + [Pro Tips](bonus/pro-tips-by-domain.md) |
| Prep 30 min before exam | [Power Review](bonus/quick-wins-last-minute.md) |
| Understand wrong answers | [Common Traps Guide](bonus/common-traps-and-pitfalls.md) |
| Exam day tactics | [Exam Day Strategy](bonus/exam-day-strategy.md) |
| Understand how this package was built | [Preparation Process](preparation-process.md) |

---

## Package Overview

This package contains all materials needed to pass the Claude Certified Architect – Foundations exam (pass threshold: 720/1000 scaled score). The exam covers 5 domains with 30 task statements and uses 4 randomly selected scenarios from a pool of 6.

```
claude_certifie_architect/
├── study-guide.md                          ← You are here
├── preparation-process.md                  ← How this package was built
│
├── study-notes/                            ← Deep conceptual coverage
│   ├── domain1-agentic-architecture.md     ← 27% of exam
│   ├── domain2-tool-design-mcp.md          ← 18% of exam
│   ├── domain3-claude-code-config.md       ← 20% of exam
│   ├── domain4-prompt-engineering.md       ← 20% of exam
│   └── domain5-context-reliability.md     ← 15% of exam
│
├── cheat-sheets/                           ← One-page quick reference per domain
├── quizzes/                               ← 5 domain quizzes (HTML, ~12 questions each)
├── mock-exams/                            ← Full exam simulations (HTML, ~50 questions)
├── scenario-workbooks/                    ← Guided walkthroughs of 6 exam scenarios
├── labs/                                  ← Hands-on CLI and SDK exercises
├── interactive-sessions/                  ← Guided practice across Claude surfaces
├── infographics/                          ← Visual architecture diagrams (HTML)
├── flashcards/                            ← 120+ spaced repetition cards (HTML)
├── concept-map/                           ← Cross-domain relationship map (HTML)
│
└── bonus/
    ├── exam-day-strategy.md               ← Time management, elimination strategy
    ├── common-traps-and-pitfalls.md       ← 5 distractor patterns decoded
    ├── pro-tips-by-domain.md              ← Insider tips per domain
    └── quick-wins-last-minute.md          ← 30-minute pre-exam power review
```

---

## Study Plans

Choose the plan that matches your available time.

### 1-Week Intensive (20–30 hours)
*Full deep-dive through all materials. Best score outcome.*

**Days 1–2: Foundation**
- Read all 5 study notes (2–3 hours each): `study-notes/domain1` through `domain5`
- Review infographics after each domain: `infographics/`
- Goal: understand all 30 task statements conceptually

**Days 3–4: Active Recall**
- Complete all 5 domain quizzes: `quizzes/quiz-domain1.html` through `quiz-domain5.html`
- For any domain scoring below 70%: re-read that domain's study notes + cheat sheet
- Review [Common Traps Guide](bonus/common-traps-and-pitfalls.md) — identify which patterns you fell for

**Days 5–6: Hands-On Practice**
- Complete CLI lab guide: `labs/lab-guide-claude-code-cli.md`
- Complete Agent SDK lab guide: `labs/lab-guide-agent-sdk.md`
- Work through 2–3 scenario workbooks relevant to your target domain: `scenario-workbooks/`
- Take Mock Exam A: `mock-exams/mock-exam-A.html` (timed)

**Day 7: Final Prep**
- Review weak domain cheat sheets based on Mock Exam A results
- Read [Pro Tips by Domain](bonus/pro-tips-by-domain.md)
- Take Mock Exam B: `mock-exams/mock-exam-B.html` (timed)
- Before bed: [Quick Wins Power Review](bonus/quick-wins-last-minute.md)

---

### 3-Day Focused (12–15 hours)
*Study notes + quizzes + 1 mock exam + targeted review. Solid preparation.*

**Day 1 (4–5 hours): Study Notes**
- Domain 1 study notes (1h): `study-notes/domain1-agentic-architecture.md`
- Domain 2 study notes (45m): `study-notes/domain2-tool-design-mcp.md`
- Domain 3 study notes (1h): `study-notes/domain3-claude-code-config.md`
- Domain 4 study notes (1h): `study-notes/domain4-prompt-engineering.md`
- Domain 5 study notes (45m): `study-notes/domain5-context-reliability.md`

**Day 2 (4–5 hours): Active Recall + Traps**
- All 5 domain quizzes: `quizzes/` (2.5h)
- Review wrong answers using study notes (1h)
- Read [Common Traps Guide](bonus/common-traps-and-pitfalls.md) (30m)
- Read [Pro Tips by Domain](bonus/pro-tips-by-domain.md) (45m)

**Day 3 (4–5 hours): Mock Exam + Review**
- Take Mock Exam A: `mock-exams/mock-exam-A.html` (90m timed)
- Review all wrong answers against study notes (1.5h)
- Re-read weakest domain's cheat sheet (30m)
- Read [Exam Day Strategy](bonus/exam-day-strategy.md) (30m)
- Before bed: [Power Review](bonus/quick-wins-last-minute.md) (30m)

---

### 1-Day Crash Course (6–8 hours)
*Cheat sheets + pro tips + mock exam + power review. Maximum efficiency.*

**Morning (3 hours): Cheat Sheets + Pro Tips**
- All 5 cheat sheets (15m each): `cheat-sheets/`
- [Common Traps Guide](bonus/common-traps-and-pitfalls.md) (30m)
- [Pro Tips by Domain](bonus/pro-tips-by-domain.md) (45m)
- [Exam Day Strategy](bonus/exam-day-strategy.md) (20m)

**Afternoon (3–4 hours): Mock Exam + Review**
- Take Mock Exam A: `mock-exams/mock-exam-A.html` (90m timed)
- Review wrong answers using cheat sheets (1h)
- Focus on your 2 weakest domains: re-read those cheat sheets

**Evening (1 hour): Final Review**
- [Power Review](bonus/quick-wins-last-minute.md) — read twice
- Sleep

---

## Learning Style Paths

### Visual Learners
Start here before reading study notes:
1. `infographics/infographic-agentic-loop.html` → then study Domain 1
2. `infographics/infographic-mcp-architecture.html` → then study Domain 2
3. `infographics/infographic-claude-code-hierarchy.html` → then study Domain 3
4. `infographics/infographic-decision-trees.html` → covers Domains 3, 4, 1
5. `infographics/infographic-error-handling.html` → covers Domains 2, 5
6. `concept-map/concept-map.html` → see cross-domain relationships

### Reading Learners
Linear path:
1. Study notes in domain order (Domain 1 → 5)
2. Cheat sheet after each domain as a summary check
3. Domain quiz after each cheat sheet
4. Mock exam after all 5 domains

### Hands-On Learners
Start with practice:
1. `labs/lab-guide-claude-code-cli.md` (Domain 3 — most hands-on domain)
2. `labs/lab-guide-agent-sdk.md` (Domain 1)
3. Interactive sessions: `interactive-sessions/session-cli-agentic-loop.md`
4. Study notes to fill gaps revealed by lab exercises

### Test-Oriented Learners
Test first, study gaps:
1. Take Mock Exam A: `mock-exams/mock-exam-A.html` cold (baseline)
2. Identify missed questions by domain
3. Study only the domains where you scored below 70%
4. Retake with Mock Exam B to verify improvement

---

## Study by Domain and Task Statement

For each task statement, the table below shows which materials cover it and links to each.

### Domain 1: Agentic Architecture & Orchestration (27%)

| Task Statement | Study | Practice | Hands-On | Visual |
|---------------|-------|---------|---------|--------|
| **1.1** Agentic loops | [Study Notes](study-notes/domain1-agentic-architecture.md#task-11) | [Quiz](quizzes/quiz-domain1.html) | [SDK Lab](labs/lab-guide-agent-sdk.md) / [CLI Session](interactive-sessions/session-cli-agentic-loop.md) | [Infographic](infographics/infographic-agentic-loop.html) |
| **1.2** Multi-agent orchestration | [Study Notes](study-notes/domain1-agentic-architecture.md#task-12) | [Quiz](quizzes/quiz-domain1.html) | [SDK Lab](labs/lab-guide-agent-sdk.md) | [Infographic](infographics/infographic-agentic-loop.html) |
| **1.3** Subagent invocation & context | [Study Notes](study-notes/domain1-agentic-architecture.md#task-13) | [Quiz](quizzes/quiz-domain1.html) | [SDK Lab](labs/lab-guide-agent-sdk.md) / [CLI Session](interactive-sessions/session-cli-agentic-loop.md) | [Infographic](infographics/infographic-agentic-loop.html) |
| **1.4** Multi-step workflows & enforcement | [Study Notes](study-notes/domain1-agentic-architecture.md#task-14) | [Quiz](quizzes/quiz-domain1.html) | [SDK Lab](labs/lab-guide-agent-sdk.md) | — |
| **1.5** Agent SDK hooks | [Study Notes](study-notes/domain1-agentic-architecture.md#task-15) | [Quiz](quizzes/quiz-domain1.html) | [SDK Lab](labs/lab-guide-agent-sdk.md) | [Error Handling](infographics/infographic-error-handling.html) |
| **1.6** Task decomposition | [Study Notes](study-notes/domain1-agentic-architecture.md#task-16) | [Quiz](quizzes/quiz-domain1.html) | — | [Decision Trees](infographics/infographic-decision-trees.html) |
| **1.7** Session state & forking | [Study Notes](study-notes/domain1-agentic-architecture.md#task-17) | [Quiz](quizzes/quiz-domain1.html) | [CLI Lab](labs/lab-guide-claude-code-cli.md) | — |

**Cheat Sheet**: [Domain 1 Cheat Sheet](cheat-sheets/cheatsheet-domain1.md)
**Pro Tips**: [Domain 1 Tips](bonus/pro-tips-by-domain.md#domain-1-agentic-architecture--orchestration-27)

---

### Domain 2: Tool Design & MCP Integration (18%)

| Task Statement | Study | Practice | Hands-On | Visual |
|---------------|-------|---------|---------|--------|
| **2.1** Tool interface design | [Study Notes](study-notes/domain2-tool-design-mcp.md#task-statement-21-design-effective-tool-interfaces-with-clear-descriptions-and-boundaries) | [Quiz](quizzes/quiz-domain2.html) | — | [MCP Architecture](infographics/infographic-mcp-architecture.html) |
| **2.2** Structured error responses | [Study Notes](study-notes/domain2-tool-design-mcp.md#task-statement-22-implement-structured-error-responses-for-mcp-tools) | [Quiz](quizzes/quiz-domain2.html) | — | [Error Handling](infographics/infographic-error-handling.html) |
| **2.3** Tool distribution & tool_choice | [Study Notes](study-notes/domain2-tool-design-mcp.md#task-statement-23-distribute-tools-appropriately-across-agents-and-configure-tool-choice) | [Quiz](quizzes/quiz-domain2.html) | — | — |
| **2.4** MCP server integration | [Study Notes](study-notes/domain2-tool-design-mcp.md#task-statement-24-integrate-mcp-servers-into-claude-code-and-agent-workflows) | [Quiz](quizzes/quiz-domain2.html) | [CLI Lab](labs/lab-guide-claude-code-cli.md) / [MCP Session](interactive-sessions/session-cli-mcp-config.md) | [MCP Architecture](infographics/infographic-mcp-architecture.html) |
| **2.5** Built-in tools | [Study Notes](study-notes/domain2-tool-design-mcp.md#task-statement-25-select-and-apply-built-in-tools-read-write-edit-bash-grep-glob-effectively) | [Quiz](quizzes/quiz-domain2.html) | [CLI Lab](labs/lab-guide-claude-code-cli.md) | — |

**Cheat Sheet**: [Domain 2 Cheat Sheet](cheat-sheets/cheatsheet-domain2.md)
**Pro Tips**: [Domain 2 Tips](bonus/pro-tips-by-domain.md#domain-2-tool-design--mcp-integration-18)

---

### Domain 3: Claude Code Configuration & Workflows (20%)

| Task Statement | Study | Practice | Hands-On | Visual |
|---------------|-------|---------|---------|--------|
| **3.1** CLAUDE.md hierarchy | [Study Notes](study-notes/domain3-claude-code-config.md#task-statement-31-configure-claudemd-files-with-appropriate-hierarchy-scoping-and-modular-organization) | [Quiz](quizzes/quiz-domain3.html) | [CLI Lab](labs/lab-guide-claude-code-cli.md) / [MCP Session](interactive-sessions/session-cli-mcp-config.md) | [Hierarchy](infographics/infographic-claude-code-hierarchy.html) |
| **3.2** Commands & skills | [Study Notes](study-notes/domain3-claude-code-config.md#task-statement-32-create-and-configure-custom-slash-commands-and-skills) | [Quiz](quizzes/quiz-domain3.html) | [CLI Lab](labs/lab-guide-claude-code-cli.md) | [Hierarchy](infographics/infographic-claude-code-hierarchy.html) |
| **3.3** Path-specific rules | [Study Notes](study-notes/domain3-claude-code-config.md#task-statement-33-apply-path-specific-rules-for-conditional-convention-loading) | [Quiz](quizzes/quiz-domain3.html) | [CLI Lab](labs/lab-guide-claude-code-cli.md) | [Hierarchy](infographics/infographic-claude-code-hierarchy.html) |
| **3.4** Plan mode vs direct execution | [Study Notes](study-notes/domain3-claude-code-config.md#task-statement-34-determine-when-to-use-plan-mode-vs-direct-execution) | [Quiz](quizzes/quiz-domain3.html) | [CLI Lab](labs/lab-guide-claude-code-cli.md) | [Decision Trees](infographics/infographic-decision-trees.html) |
| **3.5** Iterative refinement | [Study Notes](study-notes/domain3-claude-code-config.md#task-statement-35-apply-iterative-refinement-techniques-for-progressive-improvement) | [Quiz](quizzes/quiz-domain3.html) | [CLI Lab](labs/lab-guide-claude-code-cli.md) | — |
| **3.6** CI/CD integration | [Study Notes](study-notes/domain3-claude-code-config.md#task-statement-36-integrate-claude-code-into-cicd-pipelines) | [Quiz](quizzes/quiz-domain3.html) | [Cowork Session](interactive-sessions/session-cowork-code-review.md) | — |

**Cheat Sheet**: [Domain 3 Cheat Sheet](cheat-sheets/cheatsheet-domain3.md)
**Pro Tips**: [Domain 3 Tips](bonus/pro-tips-by-domain.md#domain-3-claude-code-configuration--workflows-20)

---

### Domain 4: Prompt Engineering & Structured Output (20%)

| Task Statement | Study | Practice | Hands-On | Visual |
|---------------|-------|---------|---------|--------|
| **4.1** Explicit criteria | [Study Notes](study-notes/domain4-prompt-engineering.md#task-statement-41-design-prompts-with-explicit-criteria-to-improve-precision-and-reduce-false-positives) | [Quiz](quizzes/quiz-domain4.html) | [Web Session](interactive-sessions/session-web-prompt-engineering.md) | — |
| **4.2** Few-shot prompting | [Study Notes](study-notes/domain4-prompt-engineering.md#task-statement-42-apply-few-shot-prompting-to-improve-output-consistency-and-quality) | [Quiz](quizzes/quiz-domain4.html) | [Web Session](interactive-sessions/session-web-prompt-engineering.md) | — |
| **4.3** Structured output / tool_use | [Study Notes](study-notes/domain4-prompt-engineering.md#task-statement-43-enforce-structured-output-using-tool-use-and-json-schemas) | [Quiz](quizzes/quiz-domain4.html) | [Web Session](interactive-sessions/session-web-structured-output.md) | — |
| **4.4** Validation & retry loops | [Study Notes](study-notes/domain4-prompt-engineering.md#task-statement-44-implement-validation-retry-and-feedback-loops-for-extraction-quality) | [Quiz](quizzes/quiz-domain4.html) | [Web Session](interactive-sessions/session-web-structured-output.md) | — |
| **4.5** Batch processing | [Study Notes](study-notes/domain4-prompt-engineering.md#task-statement-45-design-efficient-batch-processing-strategies) | [Quiz](quizzes/quiz-domain4.html) | — | [Decision Trees](infographics/infographic-decision-trees.html) |
| **4.6** Multi-pass review | [Study Notes](study-notes/domain4-prompt-engineering.md#task-statement-46-design-multi-instance-and-multi-pass-review-architectures) | [Quiz](quizzes/quiz-domain4.html) | [Cowork Session](interactive-sessions/session-cowork-code-review.md) | — |

**Cheat Sheet**: [Domain 4 Cheat Sheet](cheat-sheets/cheatsheet-domain4.md)
**Pro Tips**: [Domain 4 Tips](bonus/pro-tips-by-domain.md#domain-4-prompt-engineering--structured-output-20)

---

### Domain 5: Context Management & Reliability (15%)

| Task Statement | Study | Practice | Hands-On | Visual |
|---------------|-------|---------|---------|--------|
| **5.1** Conversation context management | [Study Notes](study-notes/domain5-context-reliability.md#task-statement-51-manage-conversation-context-to-preserve-critical-information-across-long-interactions) | [Quiz](quizzes/quiz-domain5.html) | — | — |
| **5.2** Escalation & ambiguity resolution | [Study Notes](study-notes/domain5-context-reliability.md#task-statement-52-design-effective-escalation-and-ambiguity-resolution-patterns) | [Quiz](quizzes/quiz-domain5.html) | — | [Error Handling](infographics/infographic-error-handling.html) |
| **5.3** Error propagation | [Study Notes](study-notes/domain5-context-reliability.md#task-statement-53-implement-error-propagation-strategies-across-multi-agent-systems) | [Quiz](quizzes/quiz-domain5.html) | — | [Error Handling](infographics/infographic-error-handling.html) |
| **5.4** Large codebase exploration | [Study Notes](study-notes/domain5-context-reliability.md#task-statement-54-manage-context-effectively-in-large-codebase-exploration) | [Quiz](quizzes/quiz-domain5.html) | [CLI Lab](labs/lab-guide-claude-code-cli.md) | — |
| **5.5** Human review & confidence | [Study Notes](study-notes/domain5-context-reliability.md#task-statement-55-design-human-review-workflows-and-confidence-calibration) | [Quiz](quizzes/quiz-domain5.html) | — | — |
| **5.6** Information provenance | [Study Notes](study-notes/domain5-context-reliability.md#task-statement-56-preserve-information-provenance-and-handle-uncertainty-in-multi-source-synthesis) | [Quiz](quizzes/quiz-domain5.html) | — | — |

**Cheat Sheet**: [Domain 5 Cheat Sheet](cheat-sheets/cheatsheet-domain5.md)
**Pro Tips**: [Domain 5 Tips](bonus/pro-tips-by-domain.md#domain-5-context-management--reliability-15)

---

## Study by Exam Scenario

Each exam uses 4 randomly selected scenarios from the pool of 6. Study all 6 scenarios, but pay extra attention to the 2–3 scenarios covering your weakest domains.

### Scenario 1: Customer Support Resolution Agent
**Primary Domains**: D1 (Agentic Architecture), D2 (Tool Design), D5 (Context & Reliability)
**Workbook**: [scenario-workbooks/scenario1-customer-support.md](scenario-workbooks/scenario1-customer-support.md)
**Related Task Statements**: 1.1, 1.2, 1.4, 1.5, 2.1, 2.2, 2.3, 5.1, 5.2, 5.3
**Sample Questions from Exam Guide**: Q1, Q2, Q3 (customer support scenario)
**Mock Exam Exposure**: Mock A, Mock B

### Scenario 2: Code Generation with Claude Code
**Primary Domains**: D3 (Claude Code Config), D5 (Context & Reliability)
**Workbook**: [scenario-workbooks/scenario2-code-generation.md](scenario-workbooks/scenario2-code-generation.md)
**Related Task Statements**: 3.1, 3.2, 3.3, 3.4, 3.5, 5.4
**Sample Questions from Exam Guide**: Q4, Q5, Q6 (code generation scenario)
**Mock Exam Exposure**: Mock A, Mock C

### Scenario 3: Multi-Agent Research System
**Primary Domains**: D1 (Agentic Architecture), D2 (Tool Design), D5 (Context & Reliability)
**Workbook**: [scenario-workbooks/scenario3-multi-agent-research.md](scenario-workbooks/scenario3-multi-agent-research.md)
**Related Task Statements**: 1.2, 1.3, 1.6, 2.1, 2.3, 5.3, 5.6
**Sample Questions from Exam Guide**: Q7, Q8, Q9 (multi-agent research scenario)
**Mock Exam Exposure**: Mock A, Mock B

### Scenario 4: Developer Productivity with Claude
**Primary Domains**: D2 (Tool Design), D3 (Claude Code Config), D1 (Agentic Architecture)
**Workbook**: [scenario-workbooks/scenario4-developer-productivity.md](scenario-workbooks/scenario4-developer-productivity.md)
**Related Task Statements**: 2.1, 2.4, 2.5, 3.1, 3.2, 3.4, 1.3
**Mock Exam Exposure**: Mock B, Mock C

### Scenario 5: Claude Code for Continuous Integration
**Primary Domains**: D3 (Claude Code Config), D4 (Prompt Engineering)
**Workbook**: [scenario-workbooks/scenario5-ci-cd.md](scenario-workbooks/scenario5-ci-cd.md)
**Related Task Statements**: 3.5, 3.6, 4.1, 4.2, 4.5, 4.6
**Sample Questions from Exam Guide**: Q10, Q11, Q12 (CI/CD scenario)
**Mock Exam Exposure**: Mock A, Mock C

### Scenario 6: Structured Data Extraction
**Primary Domains**: D4 (Prompt Engineering), D5 (Context & Reliability)
**Workbook**: [scenario-workbooks/scenario6-structured-extraction.md](scenario-workbooks/scenario6-structured-extraction.md)
**Related Task Statements**: 4.3, 4.4, 4.5, 5.5, 5.6
**Mock Exam Exposure**: Mock B, Mock C

---

## Mock Exams

Mock exams simulate the real exam format: 4 scenarios, ~50 questions, 90-minute timer, pass/fail projection at 720/1000.

| Exam | Scenarios | Best Used For |
|------|-----------|--------------|
| [Mock Exam A](mock-exams/mock-exam-A.html) | S1, S2, S3, S5 | Initial baseline before deep study |
| [Mock Exam B](mock-exams/mock-exam-B.html) | S1, S3, S4, S6 | Mid-study progress check |
| Mock Exam C *(in progress)* | S2, S4, S5, S6 | Final readiness check |

**Instructions for timed mock exams:**
1. Set a 90-minute timer
2. Work through the exam without referring to study materials
3. Review wrong answers after completion using the relevant study notes section
4. Note which domains you missed most — review those cheat sheets

---

## Domain Quizzes

Use domain quizzes to identify gaps after reading study notes. Target: ≥70% per domain before the mock exam.

| Quiz | Questions | Topics |
|------|-----------|--------|
| [Domain 1 Quiz](quizzes/quiz-domain1.html) | ~13 | Agentic loops, multi-agent, hooks, sessions |
| [Domain 2 Quiz](quizzes/quiz-domain2.html) | ~11 | Tool design, MCP, tool_choice, built-in tools |
| [Domain 3 Quiz](quizzes/quiz-domain3.html) | ~13 | CLAUDE.md, skills, commands, plan mode, CI/CD |
| [Domain 4 Quiz](quizzes/quiz-domain4.html) | ~13 | Criteria, few-shot, schemas, batch API, multi-pass |
| [Domain 5 Quiz](quizzes/quiz-domain5.html) | ~11 | Context, escalation, error propagation, provenance |

**Strength Assessment**:
```
Take Domain Quiz → Score < 70%?
  YES → Re-read study notes for that domain
        → Re-take quiz
        → Score ≥ 70%? → Move to next domain
  NO  → Move to next domain
        → After all 5: take Mock Exam A
```

---

## Pre-Exam Checklist

### 48 Hours Before
- [ ] Complete Mock Exam B (or A if not taken yet)
- [ ] Review all wrong answers
- [ ] Re-read cheat sheets for your 2 weakest domains
- [ ] Read [Exam Day Strategy](bonus/exam-day-strategy.md) fully

### Evening Before
- [ ] Read [Pro Tips by Domain](bonus/pro-tips-by-domain.md) — the full guide once
- [ ] Read [Common Traps Guide](bonus/common-traps-and-pitfalls.md) — refresh the 5 patterns
- [ ] Read [Power Review](bonus/quick-wins-last-minute.md) — first pass
- [ ] Sleep — rest matters more than last-minute cramming

### Morning Of (30 minutes before exam)
- [ ] Read [Power Review](bonus/quick-wins-last-minute.md) — second pass
- [ ] Recall the 3 escalation triggers (explicit request / policy gap / cannot progress)
- [ ] Recall the 1 golden rule (programmatic enforcement for financial/security)
- [ ] Recall the 5 distractor patterns (Over-Engineer / Wrong Problem / Insufficient Guarantee / Wrong Scope / Premature Optimization)
- [ ] Remind yourself: 720/1000 = ~72% correct — you have a meaningful buffer

---

## Frequently Asked Questions

**Q: How many questions are on the real exam?**
A: Not publicly disclosed. Based on the 4-scenario structure (random selection from 6) and pass threshold mathematics, approximately 50 questions is a reasonable estimate.

**Q: What is the 720/1000 scaled score in practical terms?**
A: The exact conversion varies by exam form due to equating. As a rough guide: scoring ~72% of questions correct should place you near the passing threshold. This means approximately 36/50 correct. You have a meaningful buffer — roughly 14 wrong answers in a typical 50-question exam.

**Q: Which domains should I prioritize if I'm short on time?**
A: By weight: Domain 1 (27%), then Domain 3 (20%), Domain 4 (20%), Domain 2 (18%), Domain 5 (15%). However, Domain 3 (Claude Code Configuration) tends to be the most concrete and therefore the easiest to study efficiently. Domain 1 has the highest weight and therefore the highest return per hour studied.

**Q: Can I use the `-p` flag to test knowledge without taking a full mock exam?**
A: Yes — the labs and interactive sessions use this approach. The lab guide exercises are shorter, targeted practices for specific task statements.

**Q: What if I'm scoring below 60% on mock exams?**
A: Focus on the study notes for the domains where you're missing the most questions. The [Common Traps Guide](bonus/common-traps-and-pitfalls.md) can help identify if you're falling for specific distractor patterns. Take domain quizzes for your weakest areas to isolate which task statements need more attention.
