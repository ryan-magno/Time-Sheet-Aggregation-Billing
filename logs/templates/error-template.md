# Error #[ID]: [Short Descriptive Name]

**Date:** [YYYY-MM-DD]
**Session/Context:** [What were you building — keep generic, e.g., "Multi-step webhook workflow with AI enrichment"]

---

## What Happened
[2–3 sentences. What went wrong specifically? What was the observable failure — wrong output, broken expression, hallucinated node, infinite loop, etc.]

---

## User Error Category
**Primary Cause:** [Pick ONE and mark with X]

### Prompt Errors
- [ ] Ambiguous instruction — Could be interpreted multiple ways
- [ ] Missing constraints — Didn't specify what NOT to do
- [ ] Too verbose — Buried key requirements in walls of text
- [ ] Reference vs requirements — Gave reference material, expected extracted requirements
- [ ] Implicit expectations — Had requirements in head, not in prompt
- [ ] No success criteria — Didn't define what "done" looks like
- [ ] Wrong abstraction level — Too high-level or too detailed for the task

### Context Errors
- [ ] Context rot — Conversation too long, should have cleared context
- [ ] Stale context — Old information polluting new responses
- [ ] Context overflow — Too much info degraded performance
- [ ] Missing context — Assumed AI remembered something it didn't
- [ ] Wrong context — Irrelevant information drowning signal

### Harness Errors
- [ ] Subagent context loss — Critical info didn't reach subagents
- [ ] Wrong agent type — Used wrong node/agent type for the task
- [ ] No guardrails — Didn't constrain agent behavior appropriately
- [ ] Parallel when sequential needed — Launched agents/nodes that had dependencies
- [ ] Sequential when parallel possible — Slow execution from unnecessary serialization
- [ ] Missing validation — No check that output was correct before continuing
- [ ] Trusted without verification — Accepted AI output without review

### Meta Errors
- [ ] Didn't ask clarifying questions — Could have caught this earlier
- [ ] Rushed to implementation — Skipped planning or validation
- [ ] Assumed competence — Expected AI to infer too much

---

## The Triggering Prompt

```text
[Exact prompt — verbatim. Do not paraphrase.]
```

---

## What Was Wrong With This Prompt
[Be specific and critical. What was missing, ambiguous, or wrong? What would a good prompt engineer immediately see as the problem?]

---

## What The User Should Have Said Instead

```text
[Rewritten prompt that would have prevented this error. Make it concrete.]
```

---

## The Gap
**What user expected:** [The intended outcome]
**What user got:** [The actual output or failure]
**Why the gap exists:** [Tie it directly to the user error category above. Be precise.]

---

## Impact
**Time lost:** [X minutes]
**Rework required:** [What needs to be redone or fixed]

---

## Prevention — User Action Items
- [Specific behavior to change next time]
- [Another concrete action]
- [Consider adding this to your personal workflow or instructions.md]

---

## Pattern Check
**Seen this before?** [Yes / No — if Yes, this is a habit to break, not a one-off]
**Predictable?** [Should the user have anticipated this failure given what they know?]

---

## One-Line Lesson
[One actionable sentence about prompting, context management, or harnessing. NOT about model behavior.]

---
*Logged on [timestamp]*