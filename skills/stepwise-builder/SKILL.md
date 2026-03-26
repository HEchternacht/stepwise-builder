---
name: stepwise-builder
description: "Agentic step-by-step project builder. Plans first, gets approval, then executes one verified step at a time via subagents. Use for any build, feature, fix, or multi-step coding task."
---

# Stepwise Builder

An orchestrator. It does not write code, plan, or test directly. It delegates all work to three dedicated agents and enforces the pipeline.

## On start — always do this first

If the `token-efficiency` skill is available, invoke it before anything else.

---

## Phase 0 — Detect mode

Check if `PLAN.md` exists in the project root.

| Condition | Action |
|---|---|
| No `PLAN.md` | Phase 1 — new plan |
| `PLAN.md` exists + user requesting feature/fix | Phase 0b — append steps |
| `PLAN.md` exists + no new request | Phase 2 — resume execution |

### Phase 0b — Append mode

Invoke `step_planner` in append mode:

```
Agent(
  subagent_type: "step_planner",
  description: "Append steps for: <request summary>",
  prompt: "APPEND MODE. The user wants to add the following to an existing project: <user request>.

Here is the current PLAN.md:
<paste full PLAN.md contents>

Append only the new steps needed. Do not rewrite existing steps."
)
```

After the planner writes the updated `PLAN.md`, present the new steps to the user and ask for approval. Do not proceed to Phase 2 until approved.

---

## Phase 1 — Plan

Invoke the `step_planner` agent. Do not plan yourself.

```
Agent(
  subagent_type: "step_planner",
  description: "Plan: <request summary>",
  prompt: "The user wants to build the following: <user request verbatim>.

Project root: <absolute path>

Produce a complete PLAN.md for this project."
)
```

After `step_planner` completes, read `PLAN.md` and present the plan to the user. Ask: "Does this plan look right? Any changes before I start?"

Do not proceed to Phase 2 until the user approves. If changes are requested, invoke `step_planner` again in append/edit mode with the requested changes.

---

## Phase 2 — Execute

This is the core loop. Repeat for every `pending` step in `PLAN.md`, strictly in order.

**Never execute a step yourself. Never skip the tester. Never start the next step until the tester returns PASS.**

### For each step, run this exact pipeline:

#### 2a — Prepare context

Before invoking any agent:

1. Read `PLAN.md` — extract the current step's `What`, `Files`, `Check`
2. Read `.stepwise/handoff_stepN-1.md` if it exists (previous step's handoff) — this is the only prior context to pass
3. Update the step's `Status` to `in progress` in `PLAN.md`

Do not read source files. Do not scan the project. The handoff is enough.

#### 2b — Invoke step_developer

```
Agent(
  subagent_type: "step_developer",
  description: "Step N — <title>",
  prompt: "You are executing Step N of a build plan.

**What**: <copy from PLAN.md>
**Files**: <copy from PLAN.md>
**Check**: <copy from PLAN.md>

**Handoff from previous step**:
<paste contents of .stepwise/handoff_stepN-1.md, or 'No previous handoff — this is step 1.' if none>

Working directory: <absolute path to project root>"
)
```

#### 2c — Invoke step_tester

After `step_developer` completes, always invoke `step_tester` — even if the developer reports success:

```
Agent(
  subagent_type: "step_tester",
  description: "Test Step N — <title>",
  prompt: "Run the sanity check for Step N.

**Step**: N — <title>
**Check**: <exact command from PLAN.md>
**Working directory**: <absolute path to project root>"
)
```

#### 2d — Evaluate result

| Tester result | Action |
|---|---|
| PASS | Update step to `done` in `PLAN.md`. Proceed to next step. |
| FAIL | Update step to `blocked` in `PLAN.md`. Stop. Report to user with full tester output. Wait for instructions. |

---

## Handoff files

Location: `.stepwise/handoff_stepN.md`
Written by: `step_developer`
Read by: orchestrator (to pass to the next `step_developer`)
Max: 10 lines — exports, files created, env vars, critical notes only

---

## Tracking

`PLAN.md` is the live source of truth:
- `pending` — not started
- `in progress` — agent running
- `done` — tester returned PASS
- `blocked` — tester returned FAIL, needs user intervention

---

## Stack detection (for step_planner)

Pass the stack to `step_planner` if known. If unknown, the planner will ask before producing the plan.
