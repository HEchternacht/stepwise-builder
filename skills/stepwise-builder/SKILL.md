---
name: stepwise-builder
description: "Agentic step-by-step project builder. Plans first, gets approval, then executes one verified step at a time via subagents. Use for any build, feature, fix, or multi-step coding task."
---

# Stepwise Builder

A disciplined, incremental build workflow. Small steps, verified at each boundary, minimal token usage between steps.

## On start

If the `/token-efficiency` skill is available, invoke it before doing anything else.

---

## Phase 0 — Detect mode

Before planning, check if `PLAN.md` already exists in the project root.

- **No PLAN.md** → proceed to Phase 1 (new project)
- **PLAN.md exists + user is adding a feature or fix** → proceed to Phase 0b (append steps)
- **PLAN.md exists + user typed `/step`** → proceed to Phase 2 (resume execution)

### Phase 0b — Append steps to existing plan

When the user wants to add a feature or fix something in an already-planned project, do NOT start a new plan. Instead:

1. Read `PLAN.md`
2. Design only the new steps needed (same format as below)
3. Append them to `PLAN.md` with status `pending`
4. Tell the user: "Added N steps to the plan. Ready to execute?"
5. On approval, proceed to Phase 2 at the first new pending step

---

## Phase 1 — Plan

Generate a complete step-by-step plan and get user approval before writing any code.

### How to decompose into steps

A good step:
- Does exactly one logical thing
- Can be verified with a one-liner sanity check
- Is small enough that if it breaks, the cause is obvious
- Leaves prior files untouched unless integration is the explicit purpose of that step

Typical order: env setup → deps → skeleton → domain logic → framework integration → each feature → config → final check.

Adapt to complexity: a simple API might be 6 steps, a multi-service system 15+.

### PLAN.md format

```markdown
# Build Plan: <project name>

## Goal
<one paragraph>

## Steps

### Step 1 — <title>
**What**: <what this step does>
**Files**: <files created or modified>
**Check**: <one-liner to verify it works>
**Status**: pending
```

After writing `PLAN.md`, present the plan and ask: "Does this look right? Any changes before I start?"

Do not proceed until the user approves.

---

## Phase 2 — Execute

Execute steps strictly sequentially. Never start step N+1 until step N is done and its check passed.

**Each step MUST be executed by invoking the `Agent` tool** (subagent_type: `general-purpose`). Do not execute steps directly in the main conversation — always spawn a dedicated agent. This is not optional.

### Token-efficient execution

**Before invoking the Agent tool for each step**, the orchestrator must:

1. Read `PLAN.md` — identify the current pending step
2. Read **only** the handoff file from the previous step (`.stepwise/handoff_stepN.md`) — do not re-read source files unless they are in the current step's `Files`
3. Read source files **only** if they are listed in the current step's `Files` — nothing else
4. Mark the step `in progress` in `PLAN.md`

This means: **no full project scans between steps**. The handoff is the only inter-step context.

### Agent tool usage

Call the Agent tool like this for every step:

```
Agent(
  subagent_type: "general-purpose",
  description: "Step N — <title>",
  prompt: <subagent prompt below>
)
```

### Subagent prompt structure

```
You are executing Step N of a stepwise build plan.

## Task
<copy "What" from PLAN.md verbatim>

## Files you may create or modify
<list from "Files" in PLAN.md>

## Context from previous step
<paste contents of .stepwise/handoff_stepN-1.md — nothing else>

## Constraints
- Only touch files listed above
- Do not refactor prior code unless strictly required
- Minimal changes to any file not owned by this step

## Sanity check
Run this when done and confirm it passes:
<"Check" from PLAN.md>

If it fails: attempt fix up to 2 times, then stop and report the error clearly.

## When done, write .stepwise/handoff_stepN.md with this exact format:
**Exports**: <functions/classes/vars other steps may need, one line each>
**Env vars**: <any env vars consumed or produced>
**Notes**: <one or two lines the next step must know — nothing else>
```

### After each subagent completes

1. Update step status in `PLAN.md` to `done` or `blocked`
2. If `blocked`: stop and report to user before continuing
3. If `done`: proceed to next pending step

### Sanity check philosophy

The check answers one question: **"Did this step basically work?"**

- `python -c "import mymodule"` — imports without error
- `node -e "require('./utils')"` — loads without error
- `cargo check` — compiles
- A process starts and exits 0

Not unit tests. Not coverage. Not edge cases. Just: does it run?

Prefer single-command checks that exit in under 2 seconds.

---

## Handoff files

Stored in `.stepwise/handoff_stepN.md`. Written by each subagent, read by the next.

**Rules:**
- Max 10 lines
- Only what the next step actually needs
- No code, no file contents — only interface facts
- Orchestrator never modifies them, only subagents write them

These files are the memory of the build. They replace file re-reading between steps.

---

## Tracking

`PLAN.md` is the live source of truth:
- `pending` — not started
- `in progress` — subagent running
- `done` — check passed
- `blocked` — failed, needs user intervention

---

## Step isolation rules

1. Each step declares its files upfront — subagent must respect this
2. Integration steps are explicit — coupling is a dedicated step, not a side effect
3. Read freely, write minimally — subagents can read anything, but only write to declared files
4. No opportunistic refactoring

---

## Stack detection

- **Python**: venv → requirements/pyproject → skeleton → domain → integration
- **Node/TypeScript**: npm init → tsconfig → skeleton → domain → integration
- **Rust**: cargo new → Cargo.toml deps → module structure
- **Go**: go mod init → package structure

When uncertain about the stack, ask before writing the plan.
