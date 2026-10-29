---
name: component
description: Design and build a React component the agent-craft way — place it in the tree, list its states, design the props API before the markup, compose existing primitives, and keep everything it uses in one file. Use whenever the user asks for a new component, card, list, row, panel, widget, or UI piece, or asks to split or restructure a component, even if they only say "add a button that…".
---

# agent-craft Component

The steps for designing a new component. The rules it must follow live in `agent-craft:conventions` → `references/components.md`; read that first.

## Gates

- **Steps 1–2 before any markup.**
- **No new primitive while an existing one fits.** Search the UI layer and the repo first.
- **No prop nobody passes today.** No speculative variants, sizes, or flags.

## 1. Role

- **Place it in the tree.** Is it a section or a primitive? If it would only forward props to one child, it shouldn't exist.
- **Search first.** An existing shadcn/ui primitive or house component may already do this; compose or extend it.
- **List its states**: default, hover, focus, disabled, loading, empty, error, whichever apply. Each one maps to a prop or a state value. States found mid-build are how boolean props pile up.
- **Name it by business meaning**: `PlanCard`, `MessageRow`, not `Wrapper` or `ListItem2`.

## 2. Props API

- Only the props today's callers need, and only the fields the component reads.
- `children` or slot props over configuration props; compound parts only when callers arrange several slots.
- Variants are one typed union prop.
- State stays with its owner: the component receives values and emits `on*` callbacks. It doesn't fetch or hold state its siblings also need.

## 3. Server or client

Server component unless it needs interactivity; then `'use client'` on this leaf only.

## 4. Build

- One kebab-case file colocated with its feature, in dependency order: private pieces first, then the props interface and exported component.
- Composition, styling, icons, and accessibility per `components.md`.
- Over 60 lines: split into private sub-components in the same file.

## 5. Verify

- Every state from step 1 renders, including loading, empty, and error.
- The tree stayed flat: no new pass-through layer, no prop drilled past one intermediate.
- The component can be understood from its own file plus at most two others.
- The repo's `typecheck` and `lint` scripts pass.
