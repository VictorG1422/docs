---
name: repository-engineering
description: Use for implementation, debugging, testing, review, and maintenance tasks that need a general repository workflow.
---

# Repository engineering

Follow `.github/copilot-instructions.md` and any matching path-scoped
instructions in `.github/instructions/`.

## Workflow

1. Inspect Git status, then locate the owning code and the smallest relevant
   set of callers, tests, and documentation.
2. Summarize the requested behavior and relevant constraints. Ask only when an
   unresolved choice materially changes behavior or risk.
3. Make the smallest complete change using nearby patterns. Update directly
   affected tests and docs; avoid unrelated refactors.
4. Run focused validation, inspect the final diff, and report verified outcomes
   plus anything not checked.

Keep context lean: search narrowly, read only needed sections, and avoid
repeating source or checks that already produced usable results.
