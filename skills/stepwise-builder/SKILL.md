---
name: stepwise-builder
description: Agentic step-by-step project builder that breaks any implementation request into small, independently testable increments. Use this skill whenever a user asks to build a project, implement a feature from scratch, scaffold a new application, create an API, set up a service, or any multi-step coding task — even if they just say "build X", "create a Y app", or "implement Z". This skill is fundamentally different from normal one-shot coding: it plans exhaustively first, gets user approval, then executes one step at a time via sequential subagents, each verified by a smoke test before proceeding to the next. Always use this skill for any implementation request that involves more than one logical piece of work.
---

# Stepwise Builder

A disciplined, incremental build workflow. The core idea: small steps, verified at each boundary, minimize error accumulation.

## Why this matters

When a system is built all at once, errors compound invisibly. By the time something breaks, it's impossible to know which layer introduced the problem. Building in verified increments keeps each boundary clean: if step N passes its smoke test, it's a known-good foundation for step N+1.

## Phase 1 — Plan

Before writing any code, generate a complete step-by-step plan and get the user's approval.

### How to decompose a request into steps

Think in layers, smallest to largest. A good step:
- Does exactly one logical thing (install deps, scaffold a file, add a route)
- Can be verified with a quick smoke test (run it, hit an endpoint, import a module)
- Is small enough that if it breaks, the cause is obvious
- Leaves prior code untouched unless integration is the explicit purpose of that step

Typical step categories, roughly in order:
1. Environment / tooling setup (language version, virtualenv, package manager init)
2. Dependency declaration and installation
3. "Hello world" scaffold — the smallest runnable skeleton
4. Core domain logic (pure functions, utilities, algorithms) — isolated from the framework
5. Framework integration — wire the domain logic into the app
6. Each additional route / feature / module — one at a time
7. Configuration, env vars, secrets handling
8. Final integration and end-to-end check

Not every project needs all categories. Adapt to what makes sense. The number of steps should match the complexity of the request — a simple CRUD API might be 6 steps, a complex multi-service system might be 15+.

### Writing the plan

Save the plan to `PLAN.md` in the project root with this structure:

```markdown
# Build Plan: <project name>

## Goal
<one paragraph describing what we're building and why this plan is structured this way>

## Steps

### Step 1 — <short title>
**What**: <what this step does>
**Why**: <why this step comes before the next one>
**Smoke test**: <exact command or action that proves this step worked>
**Files touched**: <list of files this step creates or modifies>
**Status**: pending

### Step 2 — <short title>
...
```

After writing `PLAN.md`, present the plan to the user clearly and ask: "Does this plan look right? Any steps to add, remove, or reorder before I start?"

Do not proceed to Phase 2 until the user approves. If they request changes, update `PLAN.md` and re-present.

---

## Phase 2 — Execute

Once the plan is approved, execute each step by spawning a subagent. Steps are strictly sequential — never start step N+1 until step N's subagent has completed and its smoke test passed.

### Before spawning each subagent

1. Read `PLAN.md` and note the current step
2. Collect the current code state: list and read all relevant files the subagent will need context on (don't dump the entire project, just files that relate to this step)
3. Note what the smoke test for this step must verify

### Subagent prompt structure

Each subagent receives a prompt like this:

```
You are executing step N of a stepwise build plan.

## Your task
<copy the step's "What" from PLAN.md verbatim>

## Context: what has been built so far
<paste relevant file contents — only files this step touches or depends on>

## Constraints
- Only create or modify the files listed in "Files touched" for this step
- Do not refactor, rename, or clean up code from previous steps unless it is strictly necessary to make this step work
- If you must touch a prior file (e.g., to register a new route), change as little as possible

## Smoke test
When your code is written, run this smoke test and confirm it passes:
<exact smoke test command from PLAN.md>

The smoke test must pass before you finish. If it fails, see the failure protocol below.

## Smoke test failure protocol
1. Read the error carefully and attempt a fix (up to 3 attempts)
2. If still failing, search online for the error (use WebSearch if available)
3. If still failing after searching, stop and clearly report: what you tried, what the error is, and what you think might be wrong — so the user can intervene

## When you are done
Report:
- What files were created or modified
- The smoke test command you ran and its output (paste the relevant part)
- Any notes for the next step (e.g., "the math utils module is at src/math_utils.py and exports these functions: ...")
```

### After each subagent completes

1. Read the subagent's report
2. Update `PLAN.md` — change the step's `Status` from `pending` to `done` (or `blocked` if it failed)
3. If smoke test passed: proceed to the next step
4. If smoke test failed and the subagent could not recover: pause and report clearly to the user before proceeding

### Tracking progress

Keep `PLAN.md` as the live source of truth. After every step, its statuses should reflect reality:
- `pending` → not started
- `in progress` → subagent currently running
- `done` → smoke test passed
- `blocked` → failed, needs user intervention

---

## Smoke test philosophy

Smoke tests are not unit tests. They answer one question: **"Is this step basically working?"**

Good smoke tests:
- `python -c "import mymodule; print('ok')"` — does the module import without errors?
- `curl -s http://localhost:8000/health` — does the server respond?
- `node -e "const x = require('./utils'); console.log(x.add(1,2))"` — does the function return something?
- Running the app and checking it starts without crashing

Bad smoke tests:
- Full test suites with many edge cases (too slow, too much coverage required)
- Tests that depend on external services not yet set up
- Tests that check correctness deeply (that's for later)

If a project has its own test runner already set up by the time a step runs, a `--smoke` or minimal subset is fine. Otherwise, a quick one-liner is ideal.

---

## Step isolation rules

These rules minimize the chance that one step's agent breaks another's work:

1. **Each step declares its files upfront** in the plan — the subagent must respect this
2. **Integration steps are explicit steps** — if step 5 is "couple math utils with the API", that's a dedicated step, not a side effect of another
3. **Read freely, write minimally** — subagents can read any file for context, but only write to their declared files
4. **No opportunistic refactoring** — if earlier code looks messy, that's fine; touching it introduces risk

---

## Adapting to the project type

The plan structure adapts to the language and framework:

- **Python**: steps for `venv`, `requirements.txt`/`pyproject.toml`, then framework scaffold
- **Node/TypeScript**: steps for `npm init`, `tsconfig.json`, then framework scaffold
- **Rust**: steps for `cargo new`, dependency declarations in `Cargo.toml`, then module structure
- **Go**: steps for `go mod init`, then package structure
- Any other: follow the same pattern — environment, deps, skeleton, domain, integration

When uncertain about the stack, ask the user before writing the plan.
