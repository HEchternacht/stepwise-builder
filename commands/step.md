---
description: Execute the next step of a stepwise build plan, or start a new one if PLAN.md does not exist.
---

Use the stepwise-builder skill. First invoke token-efficiency if available.

Follow Phase 0 to detect mode:

- **No PLAN.md**: Phase 1 — invoke step_planner agent to generate the plan, present it to the user, wait for approval before executing anything.
- **PLAN.md exists + user is requesting a feature or fix**: Phase 0b — invoke step_planner in append mode, present new steps, wait for approval.
- **PLAN.md exists + no new request**: Phase 2 — run the pipeline for the next pending step:
  1. Read PLAN.md and previous handoff file (.stepwise/handoff_stepN-1.md)
  2. Invoke step_developer agent with step details + handoff as context
  3. Invoke step_tester agent to verify the check
  4. On PASS: mark done, proceed. On FAIL: mark blocked, stop and report.
