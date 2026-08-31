# Global Skills — Session Management + Coding Guidelines

> **This file is a global skill layer.** It works alongside your project-level CLAUDE.md. It does NOT replace CLAUDE.md. Place this file in `~/.claude/CLAUDE.md` to apply it globally across all projects, or in a specific project's root directory to scope it to that project.
>
> **Loading order:** Claude Code reads your project-level CLAUDE.md first, then this file.

---

## Skill 1: Session Relay
*Credit: [Andy Tozier](https://github.com/Andytoizer/session-relay), Head of Growth @ [Freckle.io](https://freckle.io/) | Writing [AgentOperator](https://agentoperator.substack.com/) (building GTM with agentic tools)*

**Purpose:** Manage multi-session projects so context is never lost — either by compacting within a session or handing off between sessions.

**When to activate:**

- When a project will clearly take more than one session (multi-phase work, 3+ distinct phases)
- When context is getting heavy but you're still mid-workflow
- When you've completed a distinct phase and the next phase is genuinely different work
- Don't wait to be asked — proactively propose splitting when the scope is large

### Choosing Compact vs. New Session

**Compact when:**
- You're iterating on the same work — refining, adjusting, re-running with tweaks
- The user gave feedback that shaped your approach and it hasn't been captured in files yet
- You're mid-loop on something that isn't done yet
- Tool state matters — file paths, data shapes, variable names are in context
- The next step is a continuation of the current workflow, not a new workflow

**New session when:**
- You're starting a genuinely different task with different tools and a different workflow
- Everything the next step needs is already captured in the README and data files
- The current context would be noise for the next phase
- The next task has its own setup, iteration loop, and completion criteria

**Rule of thumb:** If you could hand the README to a colleague and they'd know exactly what to do without asking questions, it's a new session. If they'd need to ask "wait, what did the user say about X?" — compact.

### Project Folder Structure

For any project that spans sessions, create a dedicated folder with a README as the single source of truth:

```
<project-name>/
├── README.md          ← Status tracker, decisions, next steps
├── [phase outputs]    ← Whatever artifacts each phase produces
└── [subfolders]       ← Organize by phase or type as needed
```

### README Template

```markdown
# [Project Name]

## Overview
[1-2 sentences: what this project is and what the goal is]

## Status Tracker
### Phase 1: [Name] ← COMPLETED
- [x] Task 1
- [x] Task 2

### Phase 2: [Name] ← CURRENT
- [x] Task 3
- [ ] Task 4
- [ ] Task 5

### Phase 3: [Name] ← PENDING
- [ ] Task 6

## Decisions Made
[Numbered list of key decisions with brief rationale]

## Open Questions
[Anything unresolved that the next session needs to address]

## Session Log
### Session 1 — [Date]
**Completed:** [bullet list]
**Next:** [what the next session should do first]
```

### Compaction Rules

1. Update the README — mark completed tasks, add decisions, append to session log
2. Capture any uncaptured preferences — if the user gave feedback that only lives in the conversation, write it to the README before compacting
3. Compact the conversation
4. Continue working — pick up where you left off

### Session Handoff Rules

1. Summarize what was completed — specific deliverables, not vague descriptions
2. Summarize what's next — the first 2-3 concrete steps for the next session
3. Update the README — mark completed tasks, add any new decisions or open questions, append to session log
4. Flag blockers — if the next session needs something from the user (an API key, a decision, access to a tool), call it out explicitly
5. Generate a ready-to-paste prompt — a short code block the user can copy into the next session

### Session Pickup

When starting a new session on an existing project, read the project README first. It tells you what's been done, what's next, what decisions were already made, what tools were chosen, and any blockers or open questions. Read it before doing anything else.

### What a Good Handoff Looks Like

**Good:**
> **Completed:** Built and tested `data_pull.py`. Pulls all records from the past 7 days. Output saved to `.tmp/data.csv` with 156 rows.
>
> **Next session should:**
> 1. Read `directives/analysis.md` for full context
> 2. Build `execution/transform.py` — normalize the data and save to `.tmp/clean.csv`
> 3. Test it and verify row counts match

**Bad:**
> We made good progress on the automation. Next time we should keep building scripts.

### Key Principles

- The README is the source of truth. Not memory, not conversation history.
- Update before you compact or close. Never compact or end a session without updating the README first. Non-negotiable.
- Prefer compaction over handoffs. Compaction preserves nuance. Use handoffs only at natural phase boundaries.
- Be specific about what's next. "Continue the project" is useless. "Build `execution/transform.py`, test it, save to `.tmp/clean.csv`" is useful.
- Decisions are permanent unless revisited. If a previous session decided to use Tool X over Tool Y, respect that unless the user explicitly wants to reconsider.
- Don't over-split. If it can fit in 2-3 sessions with compaction, that's better. The goal is focused context, not busywork.

---

## Skill 2: Karpathy Coding Guidelines

**Purpose:** Reduce common LLM coding mistakes. Apply these when writing, reviewing, or refactoring any code.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

### 1. Think Before Coding

Don't assume. Don't hide confusion. Surface tradeoffs.

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

### 2. Simplicity First

Minimum code that solves the problem. Nothing speculative.

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

### 3. Surgical Changes

Touch only what you must. Clean up only your own mess.

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it — don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

### 4. Goal-Driven Execution

Define success criteria. Loop until verified.

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

## Quick Reference

| Situation | What To Do |
|---|---|
| Starting a multi-session project | Create project folder + README. Propose session split. |
| Context getting heavy, same workflow | Compact. Update README first. |
| Completed a phase, next phase is different | Session handoff. Update README. Generate pickup prompt. |
| Writing a new script | State assumptions. Write minimal code. Test. |
| Editing an existing script | Surgical changes only. Don't touch unrelated code. Match existing style. |
| Something broke | Fix → test → update docs → move on. |
| Unclear request | Stop. Name what's confusing. Ask. Don't guess. |
| About to close a session | Update README. Always. Non-negotiable. |
