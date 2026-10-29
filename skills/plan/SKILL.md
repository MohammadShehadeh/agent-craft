---
name: plan
description: Plan before coding — restate the task as verifiable targets with done criteria, scope boundaries, and stop conditions; confirm before writing code. Run as /agent-craft:plan <task>.
argument-hint: <task description>
disable-model-invocation: true
---

Plan this task before writing any code: $ARGUMENTS

1. **Restate it as verifiable targets.** "Add validation" becomes "these inputs are rejected with these error keys, covered by these tests".
2. **State assumptions.** If the request is ambiguous, ask. Say "I don't know" instead of guessing.
3. **List the change set.** Each file that changes and why it traces to the request, plus an out-of-scope list of related-looking files and behaviors that stay untouched.
4. **Propose the simplest approach.** No speculative abstraction. If a simpler approach than the one implied exists, say so.
5. **Check the traceability budget.** For each new function or component, name the files a reader will need to open. More than two (not counting shared infrastructure) means the plan needs a flatter shape.
6. **Define done as commands.** The repo's own scripts with expected results (`pnpm typecheck` exits 0, the test command passes including N new tests, `rg` for the old pattern returns nothing).
7. **Name the stop conditions.** The assumptions the plan rests on; if one turns out false during implementation, stop and report.
8. **Wait for confirmation.**

Be concise.
