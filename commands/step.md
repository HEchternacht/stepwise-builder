---
description: Execute the next step of a stepwise build plan, or start a new one if PLAN.md does not exist.
---

Use the stepwise-builder skill. Follow Phase 0 to detect the correct mode:

- If PLAN.md does not exist: Phase 1 — ask what to build, generate the plan, wait for approval.
- If PLAN.md exists and the user is requesting a new feature or fix: Phase 0b — append new steps to the existing plan, do not restart.
- If PLAN.md exists and no new request: Phase 2 — execute the next pending step using the handoff file from the previous step as context. Do not re-read source files unless they are listed in the current step's Files field.
