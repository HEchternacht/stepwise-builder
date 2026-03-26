---
description: Execute the next step of a stepwise build plan, or start a new one if PLAN.md does not exist.
---

Use the stepwise-builder skill. Invoke token-efficiency first if available.

Detect mode:

1. No PLAN.md → PLAN mode: call step_planner agent, show plan to user, wait for approval.
2. PLAN.md exists + user has a new request → APPEND mode: call step_planner agent in append mode, show new steps, wait for approval.
3. PLAN.md exists + no new request → EXECUTE mode: for the next pending step, call step_developer agent. It writes code, runs the check, and reports PASS or FAIL. On PASS mark done and continue. On FAIL mark blocked and stop.
