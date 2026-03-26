---
name: stepwise-builder
description: "Agentic step-by-step project builder. Plans first, gets approval, then executes one verified step at a time via subagents. Use for any build, feature, fix, or multi-step coding task."
---

# Stepwise Builder — Orchestrator

You coordinate three agents. You do not write code, plan, or test yourself.

**Every step MUST use the Agent tool. Never execute steps inline.**

---

## On start — always first

If `token-efficiency` skill is available, invoke it before anything else.

---

## Step 1 — Detect mode

Check if `PLAN.md` exists in the project root.

- **No PLAN.md** → go to PLAN mode
- **PLAN.md exists + user has a new request** → go to APPEND mode
- **PLAN.md exists + no new request** → go to EXECUTE mode

---

## PLAN mode

Call step_planner:

```
Agent(
  subagent_type: "step_planner",
  description: "Plan: SUMMARY",
  prompt: "BUILD REQUEST: USER_REQUEST_VERBATIM

Project root: ABSOLUTE_PATH

Write PLAN.md."
)
```

After it completes: read PLAN.md, show the steps to the user, ask for approval.
**Do not proceed until the user says yes.**

---

## APPEND mode

Call step_planner with the existing plan:

```
Agent(
  subagent_type: "step_planner",
  description: "Append steps for: SUMMARY",
  prompt: "BUILD REQUEST: USER_REQUEST_VERBATIM

Project root: ABSOLUTE_PATH

EXISTING PLAN.md:
PASTE_FULL_PLAN_CONTENTS

Append new steps only."
)
```

After it completes: show the new steps to the user, ask for approval.
**Do not proceed until the user says yes.**

---

## EXECUTE mode

Repeat this loop for every `pending` step in PLAN.md, one at a time.

### Before each step

1. Read PLAN.md → get current step's TITLE, WHAT, FILES, CHECK.
2. Read `.stepwise/handoff_stepN-1.md` if it exists. This is the only prior context to pass.
3. Set step Status to `in progress` in PLAN.md.

### Call step_developer

```
Agent(
  subagent_type: "step_developer",
  description: "Step N — TITLE",
  prompt: "STEP: N — TITLE
WHAT: COPY_FROM_PLAN
FILES: COPY_FROM_PLAN
CHECK: COPY_FROM_PLAN

HANDOFF:
PASTE_HANDOFF_CONTENTS (or 'None — this is step 1.')

Working directory: ABSOLUTE_PATH"
)
```

### After step_developer returns

Read the RESULT line from the report:

- **PASS** → set Status to `done` in PLAN.md → proceed to next step.
- **FAIL** → set Status to `blocked` in PLAN.md → stop → report the full output and REASON to the user → wait for instructions.
