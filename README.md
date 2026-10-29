# agent-craft

House conventions for TypeScript, React, and Next.js, packaged for Claude Code. The goal is code you can understand without jumping between files: a two-file traceability budget, flat call stacks, and NASA's [Power of Ten](https://spinroot.com/gerard/pdf/P10.pdf) rules adapted to TypeScript.

## Install

One step, on Claude Code 2.1.275 or later:

```text
/plugin install agent-craft --marketplace MohammadShehadeh/agent-craft
```

Or in two:

```text
/plugin marketplace add MohammadShehadeh/agent-craft
/plugin install agent-craft@agent-craft
```

The skills depend on `agent-craft:conventions`, and the review skill and agent read the checklist through `${CLAUDE_PLUGIN_ROOT}`. Install them together as the plugin.

## What's inside

Skills load on their own when a task matches; the two marked with a slash command run only when you call them.

| Path | What it holds |
| --- | --- |
| `skills/conventions/SKILL.md` | The red lines, the Power of Ten rules, how to work, tests, and a map of where every other rule lives |
| `skills/conventions/references/typescript.md` | Naming and types |
| `skills/conventions/references/components.md` | Components, flat trees, client boundaries, shadcn/ui, styling, icons, accessibility, UX states |
| `skills/conventions/references/state.md` | Hooks, status unions, the query cache, list derivations, formatting, optimistic UI, context |
| `skills/conventions/references/review-checklist.md` | The review checks, in order, and the output format |
| `skills/service/` | `http` client, `Result`, error keys, parsing, server actions, route handlers, env |
| `skills/form/` | Forms: schema first, controls, Field markup, submit |
| `skills/nextjs-page/` | Pages, sections, routing, metadata |
| `skills/feature/` | Feature planning, folder layout, packages |
| `skills/component/` | Designing a new component |
| `skills/debug/` | Reproduce → localize → trace → blast radius → verify |
| `skills/plan/` | `/agent-craft:plan <task>`: verifiable targets and done criteria before code |
| `skills/review/` | `/agent-craft:review [files]`: review a diff against the checklist |
| `agents/conventions-reviewer.md` | The same review as a subagent: `@agent-agent-craft:conventions-reviewer` |
| `templates/CLAUDE.md` | A minimal project `CLAUDE.md` |

Each rule is written in exactly one file. Workflow skills hold their own topic's rules and load `agent-craft:conventions` for the rest.

## License

MIT © Mohammad Shehadeh. See [LICENSE.md](LICENSE.md).
