---
name: debug
description: The agent-craft debugging method for TypeScript, React, and Next.js — reproduce, localize top-down (page → section → primitive), map references both ways, trace to the root cause (asking where, why, and how for wrong-value bugs), and account for the blast radius before fixing. Use whenever the user reports a bug, regression, crash, flaky behavior, "wrong value", "it used to work", or asks why something renders or computes incorrectly, even when they only ask for a quick fix.
---

# agent-craft Debug

A bug is found by narrowing its location, fixed by tracing its cause, and shipped after checking who else it touches. Work the five steps in order and report in the format at the end.

## Gates

- **No fix without a reproduction.** If you can't reproduce it, say so and ask for the input, route, or account state that triggers it.
- **No edit before the root cause is named** as "correct input becomes wrong output at `file:line` because <assumption that doesn't hold>".
- **No edit to shared code before its consumers are listed** (step 4).
- **No flag added to a shared function** so two callers can disagree. That is two functions.

## 1. Localize: walk the tree down

Reproduce, then narrow along page → section → primitive. At each level ask whether the bug still reproduces below this point, and descend. The result is the **defect site**: where the bug shows, not yet where it lives. If localizing takes more than a few jumps, report that as a traceability finding too.

## 2. Gather: map references both ways

- **Inbound**: grep the defect file's import path. With direct imports and no barrels, `rg "components/date-range-picker"` is the complete consumer list.
- **Outbound**: its imports — hooks, services, context, constants.

## 3. Trace: follow references to the cause

Walk outbound from the defect site toward the data: component → hook → action or service → `http`. The bug usually lives where a decision is made, not where its result renders.

- The `'Error in <fn>::'` log prefix tells you which layer produced a failure.
- Stop at the first place where correct input becomes wrong output. That is the bug; everything after it is symptom.

### Wrong-value bugs: where, why, how

A crash comes with a stack trace; a wrong value doesn't. At each site on the path from step 2, ask:

- **Where** does this value come from? Every transform between the boundary and the render is a candidate: the parser, a `??` default, a cast, a `new Date(...)`, a `.find()`, a cache, an optimistic overlay.
- **Why** is it shaped this way here? Each transform assumes something (this field is always present, this string is a date, this list has the row). Name the assumption; the one that fails for this input is the bug.
- **How** does each consumer read it? A value that is correct at the source but read in the wrong unit, shape, or timezone by one consumer is a coupling bug; take it to step 4.

## 4. Blast radius: account before fixing

- List every consumer of the file you are about to change (step 2's grep, run on the cause file).
- For each one: does the fix change a contract it relies on, or only the broken behavior?
- Fix once at the shared root rather than patching each caller.
- If some consumers depend on the broken behavior, the shared code was two things. Split it into two functions; don't add a flag.

## 5. Verify

- The original reproduction is clean.
- Every consumer from step 4 still behaves: typecheck, tests, and a look at the screens that share the code.
- A regression test sits next to where the decision lives, not where the symptom rendered.
- The diff holds only the fix and its test.

## Report format

```text
Reproduction: <route / input / state> → <observed> (expected <…>)
Defect site:  <file:line> — where it shows
Root cause:   <file:line> — <assumption that doesn't hold for this input>
Blast radius: <N consumers of the cause file> — <unchanged | contract change: …>
Fix:          <one line>
Regression:   <test file> — <what it pins>
```
