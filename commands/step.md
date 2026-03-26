---
description: Execute the next step of a stepwise build plan, or start a new one if PLAN.md does not exist.
---

Use the stepwise-builder skill.

This system builds incrementally — working code over clean code, fewer files over more files, simple logic over abstractions. Each step must leave the project runnable.

Detect mode:

1. No PLAN.md → PLAN mode: call step_planner agent, show plan to user, wait for approval.
2. PLAN.md exists + user has a new request → APPEND mode: call step_planner agent in append mode, show new steps, wait for approval.
3. PLAN.md exists + no new request → EXECUTE mode: for the next pending step, call step_developer agent (writes code + handoff), then call step_smoketest agent (runs check + updates PLAN.md). On PASS continue. On FAIL stop and report.
