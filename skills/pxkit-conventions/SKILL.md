---
name: pxkit-conventions
description: The pxkit house style for TypeScript, React, and Next.js — readability red lines, NASA's Power of Ten rules adapted to TypeScript, naming and types, components and styling with shadcn/ui and Tailwind, hooks and state. Use this whenever you write, edit, refactor, or review any .ts/.tsx file — a component, hook, service, server action, route handler, type, or stylesheet — even for a "small" change or when the user does not mention conventions. Also use it when the user asks whether code "follows the house style" or "matches the repo".
---

# pxkit Conventions

The rules in this file apply to every line of code. Each topic rule is written in exactly one place; the table at the end says where.

## Red lines

- **Traceability budget: two files.** A reader understands a function or component by opening at most two other files. Shared infrastructure the whole repo uses (`Result`, the `http` client, `env`, `cn`, error copy, UI primitives) doesn't count. Over budget means the structure is wrong: inline it, colocate it, or pass the data directly.
- **Duplication over the wrong abstraction.** Two similar blocks are fine. A function that needs a flag, an options bag, or a type parameter to serve two callers is two functions. Moving existing code to a shared file when a second file imports it is fine; unifying similar code into a new abstraction waits for the third identical case.
- **Simple over clever.** If a reviewer has to think to see that it works, rewrite it.
- **Written for humans.** Names say the business meaning (`isEligibleForRenewal`, not `flag2`). Comments say why, never what. A file reads in dependency order: private pieces above the export that uses them, so nothing is read before it's defined. An export and the private pieces only it uses form one group; pieces shared by several groups go above all of them.

## NASA's Power of Ten, for TypeScript and React

Gerard Holzmann's rules for safety-critical code at NASA/JPL, adapted from C. Where a rule doesn't translate literally, the adaptation keeps its purpose: code you can verify by reading it.

1. **Simple control flow.** Guard clauses and early returns; the happy path stays unindented and there is no `else` after a `return`. No nested ternaries. No recursion except over data that is itself a bounded tree. Exceptions signal programmer errors, never control flow.
2. **Every loop, retry, and wait has a fixed bound.** No `while (true)`. A retry has a max attempt count, polling has a stop condition, every request has a timeout, a page-walking loop has a max page count.
3. **Build once, then don't mutate.** What never changes (`Intl` formatters, zod schemas, lookup maps) is created once at module scope, not per call or per render. Shared data is never mutated: copy, then change (`[...rows].sort(...)`). No module-level `let`.
4. **A function fits on one screen: 60 lines at most**, components included. Split a longer one by responsibility into private pieces in the same file. Files have no limit; one long cohesive file reads better than five short ones.
5. **Check assumptions in code, not in comments.** Untrusted input (request bodies, API responses, URL params, storage) is parsed with zod before use. An invariant is a guard that throws a developer-facing message. A `switch` over a union ends in an exhaustive `never` check.
6. **Declare everything at the smallest scope.** A value lives in the block that uses it; a helper, type, or constant in the file that uses it; state in the lowest component that owns it; a file in the feature that uses it. Promote one level only when a second consumer appears.
7. **Check every return value.** Read `result.data` only after `if (!result.ok)`. No floating promises; a promise deliberately not awaited is marked `void`.
8. **No metaprogramming.** No decorators, no `eval` or `new Function`, no behavior looked up by a string built at runtime (``handlers[`on${event}`]``). No `as` casts except `as const`: narrow with a check or a schema; if a library type forces a cast, comment why. Conditional and mapped types only where a declared type can't express the shape.
9. **One level of indirection.** Call the function you mean. No factories that return functions (`createHttp(config)`), no higher-order functions or components that wrap behavior (`withAuth(handler)`), no barrel files, no re-exports: import every symbol from the file that defines it. Passing a callback one level (`onSelect`, `rows.map(parseRow)`) is fine. The one allowed re-export adds a directive the source lacks (`components/motion.tsx` adding `'use client'`).
10. **Zero warnings.** TypeScript `strict`, no `any`, no `@ts-ignore`; a lint disable carries a comment saying why. The repo's `typecheck`, `lint`, and `test` scripts pass before work is called done.

## How to work

1. **Inspect the repo first.** Layout, path aliases, UI layer, test runner, and the shared layers already present (`http`, `Result`, error copy, `format*` helpers). Match them; a second copy of a shared layer is worse than either. The repo wins over any example path here.
2. **Plan, then confirm.** Restate the task as verifiable targets, list the files that change and what is out of scope, and wait for confirmation. Ask when the request is ambiguous, say "I don't know" rather than guess, and push back when a simpler approach exists.
3. **Read the rule file for the area** (table below) before writing in it.
4. **Make surgical changes.** Every changed line traces to the request. Leave neighboring code alone.
5. **Self-review.** Run the repo's `typecheck`, `lint`, and `test`, then walk `references/review-checklist.md` over your diff.

## Tests

- Use the repo's runner (Vitest + Testing Library when none is set). A test sits next to the file it tests (`invoice.test.ts`).
- Test decisions (parsers, calculators, query builders), not component wiring. `describe('<subject>')` + `it('should …')`.
- Keep decisions (pure functions) apart from actions (I/O, state, toasts) so a decision tests without mocks. A decision stays in the file that uses it; it moves to the feature's `lib/` when it needs its own tests or a second consumer.
- Formatting follows the repo; when unset: single quotes, semicolons, 2-space indent.

## Where each rule lives

| You are about to… | Read |
| --- | --- |
| name files or identifiers, write types | `references/typescript.md` |
| write or restyle a component, pick a UI primitive, add a client boundary | `references/components.md` (the `pxkit-component` skill to design a new one) |
| write a hook, hold state, derive a list view, add optimistic UI or context | `references/state.md` |
| build a form | the `pxkit-form` skill |
| call an API, DB, or SDK; add a server action or route handler; add error keys | the `pxkit-service` skill |
| add a page, route, layout, or metadata | the `pxkit-nextjs-page` skill |
| start a feature, create a feature folder or package | the `pxkit-feature` skill |
| investigate a bug | the `pxkit-debug` skill |
| review a diff | `references/review-checklist.md` |
