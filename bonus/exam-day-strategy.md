# Exam Day Strategy Guide
**Claude Certified Architect – Foundations**

---

## Understanding the Exam Format

Before strategy, internalize the structure:

- **Format**: Multiple choice, 4 options (1 correct, 3 distractors)
- **Scenarios**: 4 scenarios selected at random from a pool of 6 — you don't choose
- **Questions per scenario**: ~12–14 questions each
- **Total**: ~50 questions
- **Time**: 90 minutes (estimated)
- **Pass threshold**: 720/1000 scaled score
- **Scoring**: No penalty for wrong answers — always guess

---

## The 720/1000 Insight

The scaled score of 720/1000 is the most important number to understand. Here's what it means in practice:

**You can miss roughly 28% of questions and still pass.**

That means approximately 14 wrong answers out of 50 is still a passing score. This has three implications:

1. **Do not panic when you hit a question you don't know** — you have a substantial buffer
2. **Never leave a question unanswered** — wrong answers are not penalized; an unanswered question is always zero
3. **Spend time on questions you can win** — a question you're 90% sure about is more valuable than spending 5 minutes on one you genuinely cannot determine

The exam tests architectural judgment, not perfect recall. Many questions hinge on one key concept (e.g., "programmatic enforcement for financial rules"). If you know that concept, the question is straightforward even if you missed other details.

---

## Time Management

With ~90 minutes and ~50 questions, you have approximately **1 minute 48 seconds per question**. The pacing strategy:

### Phase 1: First Pass (65 minutes)
- Target: ~90 seconds per question
- Answer questions you know immediately
- Flag questions requiring more thought
- Never spend more than 2 minutes on any single question in this phase
- Keep a mental count: you should be at question 25 by the 45-minute mark

### Phase 2: Flagged Questions (20 minutes)
- Return to flagged questions with fresh eyes
- Apply elimination strategy (described below)
- For any question where you're down to 2 options, go with your gut — second-guessing is rarely beneficial

### Phase 3: Final Review (5 minutes)
- Verify no questions left unanswered
- Do not change answers unless you spot a clear factual error in your reasoning
- Research consistently shows that first instincts are more often correct than changed answers

---

## Question Reading Technique

The exam questions follow a predictable structure. Use this reading sequence to maximize speed and comprehension:

### Step 1: Read the Question Stem Last
Most questions give you a scenario paragraph followed by the actual question. **Read the final question sentence first**, before the scenario. This tells you what to look for while reading the scenario — instead of reading everything and then asking "what was I supposed to find?"

Example question flow:
1. Read: "What's the most effective approach to reduce overhead while maintaining system reliability?"
   → Now you know to look for: efficiency + reliability tradeoff
2. Read the scenario with that filter active
3. Evaluate options with the specific goal in mind

### Step 2: Identify the Domain Signal
The scenario context tells you which domain is in play:
- "agent skips get_customer" → Domain 1/2 (agentic architecture, tool design)
- "CLAUDE.md", "slash command", "plan mode" → Domain 3 (Claude Code)
- "false positives", "batch API", "tool_use" → Domain 4 (prompt engineering)
- "context window", "escalation", "case facts" → Domain 5 (context/reliability)

Identifying the domain activates the right mental model before you read the options.

### Step 3: Read All Four Options Before Choosing
This sounds obvious but is often violated under time pressure. Scanning all four options:
- Prevents settling on a good-sounding but not-best answer
- Activates pattern recognition for the distractor types (see Common Traps guide)
- Takes less than 30 seconds

---

## Elimination Strategy

Most exam questions can be answered correctly by eliminating 2–3 wrong answers rather than identifying the one right answer. This is often faster and more reliable.

### The Four Distractor Archetypes

**Archetype 1: The Over-Engineering Trap**
Signal words: "routing classifier", "separate ML model", "train a model", "custom classifier", "caching layer", "batching system"
Rule: If an option introduces significant infrastructure for a problem that has a simpler first-step solution, it's wrong. The exam rewards proportionate responses.
Test: "Has the simpler approach been tried first? If not, the infrastructure option is too early."

**Archetype 2: The Wrong Problem Trap**
Signal: The option solves something real, but not the stated problem.
Example: Sentiment analysis for case complexity — sentiment is real, but doesn't address the right metric.
Test: "Does this option actually fix what the question identified as failing?"

**Archetype 3: The Insufficient Guarantee Trap**
Signal: "Add instructions to the system prompt", "few-shot examples", "tell the model to..."
Rule: When the question involves financial consequences, security, or a hard business rule requiring 100% compliance — prompt-based solutions are wrong. Programmatic enforcement (hooks, prerequisites) is needed.
Test: "Can the described failure happen even if the model follows the instructions? If yes, it's insufficient."

**Archetype 4: The Wrong Scope Trap**
Signal: The option is technically correct in isolation, but placed at the wrong level.
Example: Putting team-shared settings in user-level `~/.claude/CLAUDE.md` instead of project-level `.claude/CLAUDE.md`
Test: "Is the scope (user vs. project, agent vs. coordinator, local vs. global) correct for this use case?"

### Elimination Flow

```
Read question + options
↓
Does any option introduce major infrastructure for a simple first-step?  → Eliminate (Over-Engineering)
↓
Does any option solve a related but different problem?                    → Eliminate (Wrong Problem)
↓
Does any option use prompts for something requiring determinism?          → Eliminate (Insufficient Guarantee)
↓
Does any option use the right concept at the wrong scope/level?          → Eliminate (Wrong Scope)
↓
Remaining 1-2 options: choose the more proportionate / root-cause answer
```

---

## Flagging Protocol

Not all questions deserve equal time. Use the flag/skip protocol to maximize your score.

### When to Flag and Skip
- You genuinely don't recall the specific feature/config name being asked about
- You're torn between two options for more than 90 seconds
- The question requires calculation (SLA windows, batch frequency) — do these last
- You're reading the question a second time and still uncertain

### When NOT to Flag
- You're 70%+ confident — commit and move on
- You've already eliminated 2 options — guess between the remaining 2 and flag only if you want to double-check
- Time is running out in Phase 1 — skip the flag, just answer

### What Flagging Accomplishes
Flagging is only useful if you have Phase 2 time. If you flag more than 10 questions, you may not have time to revisit all of them. Be selective: only flag questions where you believe 2–3 more minutes of thought could genuinely change your answer.

---

## Mental Energy Management

The exam has 4 scenarios. Each scenario activates a different mental model (customer support agent, code generation, multi-agent research, etc.). Here is how to protect mental energy:

### Start with Your Strongest Domain
If you know which domain you're most confident in (check your quiz scores in the study materials), answer those questions first within each scenario. This builds confidence momentum.

**Domain confidence order guide** (most candidates):
- Domain 3 (Claude Code Config) — concrete and configuration-focused; most memorable
- Domain 2 (Tool Design) — principle-based; a few key rules cover most questions
- Domain 4 (Prompt Engineering) — similar principle-based patterns
- Domain 1 (Agentic Architecture) — requires understanding SDK patterns
- Domain 5 (Context/Reliability) — most nuanced; leave more time

### Scenario Warm-Up
When a new scenario begins, spend 30 seconds re-reading the scenario setup paragraph. This activates the right context before you hit the questions. Rushing into questions cold causes misreads.

### Between Scenarios
Take 10–15 seconds to exhale and reset. This clears the mental residue from the prior scenario and prevents context bleed between different system architectures.

---

## Common Exam Question Patterns

These question archetypes appear repeatedly. Recognizing them speeds up answering:

**"Agent is doing X when it should do Y — what's the root cause?"**
→ Almost always: tool descriptions (Domain 2) or task decomposition too narrow (Domain 1)
→ Fix: improve descriptions or widen decomposition scope

**"Agent is skipping step X — how do you guarantee it doesn't?"**
→ Programmatic enforcement (hook or prerequisite), NOT prompt instructions
→ Key signal: "in 12% of cases" or any non-zero failure rate for a critical step

**"New developer / new team member not getting setting — why?"**
→ Setting is at user-level, not project-level
→ Fix: move to `.claude/CLAUDE.md` or root `CLAUDE.md`

**"Inconsistent output after many attempts — what fixes it?"**
→ Few-shot examples (not more instructions, not confidence thresholds)

**"Should we use batch API?"**
→ Only if the workflow does NOT block a human or time-sensitive process
→ Blocking workflows always need synchronous API regardless of cost

**"How do you run Claude Code in CI?"**
→ `-p` flag. This is a factual question with one answer.

**"Customer says 'I want a human' — what do you do?"**
→ Escalate immediately. No investigation first. No sentiment analysis.

---

## Pre-Exam Checklist (Morning Of)

- [ ] Complete the 30-minute power review (`bonus/quick-wins-last-minute.md`)
- [ ] Review your weakest domain cheat sheet (check quiz scores)
- [ ] Recall the 5 distractor archetypes (Over-Engineer, Wrong Problem, Insufficient Guarantee, Wrong Scope, Premature Optimization)
- [ ] Remember: 720/1000 = ~72% correct = approximately 36/50 — you have a meaningful buffer
- [ ] Read the question stem before the scenario
- [ ] No penalty for guessing — always answer every question
- [ ] Trust your first instinct on questions where you're 70%+ confident

---

## On the Day

**What to do if you hit a question you truly don't know:**
1. Eliminate any option with obvious distractor patterns (over-engineering, wrong problem)
2. Look for the option that is most proportionate and addresses root cause
3. When in doubt between two options: pick the simpler, lower-infrastructure answer
4. Mark your best guess, flag if time permits, and move on

**What to do if you're running out of time:**
- Answer all remaining questions immediately (no penalty for wrong answers)
- Eliminate the most obvious wrong answer for each remaining question
- Don't leave anything blank

**What to do if a scenario is unfamiliar:**
- Read the "Primary domains" hint at the start of each scenario
- Focus on what the question is actually asking — you don't need to understand the business context deeply, you need to know the architectural principle
