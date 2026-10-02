# pxkit

House conventions for TypeScript, React, and Next.js, packaged for Claude Code. The goal is code you can understand without jumping between files: a two-file traceability budget, flat call stacks, and NASA's [Power of Ten](https://spinroot.com/gerard/pdf/P10.pdf) rules adapted to TypeScript.

## What's inside

| Path | What it holds |
| --- | --- |
| `skills/pxkit-conventions/SKILL.md` | The red lines, the Power of Ten rules, how to work, tests, and a map of where every other rule lives |
| `skills/pxkit-conventions/references/typescript.md` | Naming and types |
| `skills/pxkit-conventions/references/components.md` | Components, flat trees, client boundaries, shadcn/ui, styling, icons, accessibility, UX states |
| `skills/pxkit-conventions/references/state.md` | Hooks, status unions, the query cache, list derivations, formatting, optimistic UI, context |
| `skills/pxkit-conventions/references/review-checklist.md` | The review checks, in order, and the output format |
| `skills/pxkit-service/` | `http` client, `Result`, error keys, parsing, server actions, route handlers, env |
| `skills/pxkit-form/` | Forms: schema first, controls, Field markup, submit |
| `skills/pxkit-nextjs-page/` | Pages, sections, routing, metadata |
| `skills/pxkit-feature/` | Feature planning, folder layout, packages |
| `skills/pxkit-component/` | Designing a new component |
| `skills/pxkit-debug/` | Reproduce → localize → trace → blast radius → verify |
| `commands/plan-task.md` | `/plan-task`: verifiable targets and done criteria before code |
| `commands/review-conventions.md` | `/review-conventions`: review a diff against the checklist |
| `agents/conventions-reviewer.md` | The same review as a subagent |
| `templates/CLAUDE.md` | A minimal project `CLAUDE.md` |

Each rule is written in exactly one file. Workflow skills hold their own topic's rules and load `pxkit-conventions` for the rest.

## Install

```text
/plugin marketplace add MohammadShehadeh/pxkit
/plugin install pxkit
```

The skills depend on `pxkit-conventions`, and the command and agent read the checklist through `${CLAUDE_PLUGIN_ROOT}`. Install them together as the plugin.

## License

MIT © Mohammad Shehadeh. See [LICENSE.md](LICENSE.md).
