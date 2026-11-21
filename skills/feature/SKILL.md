---
name: feature
description: Plan and build a feature the agent-craft way — user journey and business rules first, a flat component tree, an explicit state and communication map, then a colocated feature folder with direct imports, and a shared package only when 2+ consumers exist. Use whenever the user asks to add a feature, module, domain area, or "section of the app", or to add hooks, actions, or services to an existing one, and whenever a task would create a new folder under app/, features/, or packages/, even if they don't say "feature".
---

# agent-craft Feature

How a feature is planned, where it lives, and how its folder is laid out. Load `agent-craft:conventions` for the rules inside each file.

## Gates

- **No code before the plan (steps 1–3) is confirmed.**
- **No files before the home (step 4) is answered from the repo.** A second layout pattern in one repo costs every reader.
- **No package for a single consumer.**
- **No `types/`, `constants/`, or `lib/` file with a single consumer.** It stays in the file that uses it.

## 1. Journey and business rules

- Walk the user journey step by step: what the user sees, does, and expects at each point, including loading, empty, and failure. "A Gmail-like inbox" becomes: scan the list → open a message → read → reply, archive, or delete → back to the list.
- Pull out the business rules behind each step (what marks a message read, who can archive, what orders the list). These are pure decisions and get tests.
- Restate both as verifiable targets: behaviors, inputs and outputs, error keys.
- Search for what exists: similar features, services, hooks, primitives. Extend before duplicating; duplicate a small pure thing rather than reach into another feature's files.
- Ask about anything ambiguous.

## 2. Component tree

Outline top-down, build bottom-up:

- **Pages**: one per journey destination (`/inbox`, `/inbox/[messageId]`), thin server components.
- **Sections**: one per journey step or screen region (`MessageList`, `ThreadView`, `ComposePanel`).
- **Primitives**: the existing UI layer first; a new one only when nothing fits.

Page → section → primitive, nothing in between that only forwards props. Name every unit by business meaning (`MessageRow`, not `ListItem2`) and note what data each renders.

## 3. State and communication

For every piece of state, write down:

- **Owner**: the lowest common owner of what reads it. Selected message → the URL param; a row's hover → the row.
- **Server or client**: server data is fetched where it's used or held in the query cache, never copied into `useState`.
- **Communication**: parent → child by props, child → parent by `on*` callbacks, siblings through their shared owner, cross-page through the URL, a subtree-wide concern through one context. Write the chains out: `MessageRow → onArchive → MessageList → archiveMessage action`.
- **Optimistic or not**: reversible, low-stakes writes (archive, star) are optimistic; destructive or server-decided ones wait.

**Present steps 1–3 as the plan**, with the files that change and an out-of-scope list. Wait for confirmation.

## 4. Home

| Situation | Home |
| --- | --- |
| Default: one app uses it | The route folder under `app/`, or `features/<name>/`, whichever the repo already uses |
| Two or more apps use it, or it ships on its own version | A package |

## 5. Folder layout

```
features/billing/
  components/                 # JSX
    invoice-list.tsx          # a concept that fits in one file
    checkout/                 # a concept that outgrew one file
      checkout.tsx            # entry: the only file imported from outside checkout/
      checkout-summary.tsx
      checkout-payment.tsx
  hooks/                      # use-*.ts, only when two components share one
  actions/                    # 'use server' mutations
  services/                   # reads and shared I/O
  lib/                        # schemas and decisions shared by 2+ files, or that need their own tests
```

- **Focused directories.** A feature directory holds one domain area. Under it, a role directory (`components/`, `hooks/`, …) holds one kind of file, and a concept directory holds one concept. Three levels at most: feature → role → concept. A directory exists only when a file lives in it.
- **Focused files.** A file holds one public concept and everything private to it (`references/typescript.md`).
- **When a file becomes a directory.** When a concept's file passes 300 lines, or its private pieces have grown into units with their own props and state, the concept moves to a directory named after it. The entry file repeats the directory name (`checkout/checkout.tsx`), so the import path says what it is. Each piece goes to a sibling file named `<concept>-<piece>.tsx`. A piece that needs its own directory is a second concept: promote it to a sibling of the first.
- **Boundaries.** Code outside a concept directory imports only its entry file. The entry file defines and exports everything consumers use, compound parts included; sibling files hold only pieces private to the concept, so no re-export is ever needed. Files inside the directory import each other freely; nothing reaches into another concept's internals. A feature never imports another feature's internals; code two features need moves to a shared top-level folder.
- **Traceability holds.** The entry file plus at most two sibling files explain the concept. If it takes more, the directory holds two concepts.
- Types, constants, and error keys live in the file that uses or produces them. A `types/` or `constants/` directory appears only for things two files share.
- Consumers import the defining file through the repo's alias (`@/features/billing/components/checkout/checkout`). There is no `index.ts` and no "public" layer.
- Two known cases are a `switch` or a lookup map. A plugin-style registry only when outside code must register new cases.

## 6. Package — only when extracted

Same layout, plus per-file wildcard exports so nothing needs a barrel:

```jsonc
{
  "name": "@scope/upload",
  "type": "module",
  "sideEffects": false,
  "exports": {
    "./components/*": { "types": "./dist/components/*.d.ts", "default": "./src/components/*.tsx" },
    "./hooks/*": { "types": "./dist/hooks/*.d.ts", "default": "./src/hooks/*.ts" },
    "./actions/*": { "types": "./dist/actions/*.d.ts", "default": "./src/actions/*.ts" }
  }
}
```

A wildcard resolves one extension per directory: JSX only in `components/`, everything else `.ts`. Follow the repo's workspace and naming conventions; don't introduce workspaces into a single-package repo.

## 7. Verify

- Every journey step from step 1 works, including loading, empty, and failure.
- `rg --files <feature> | rg "index\.ts$"` returns nothing.
- `rg "export default" <feature>` matches only Next.js file conventions.
- Pick any file in the feature: it can be understood by opening at most two other files.
- No import from outside a concept directory names a file other than its entry, and no directory sits deeper than feature → role → concept.
- The repo's `typecheck`, `lint`, and `test` scripts pass, including tests for the business rules from step 1.
- Every changed line traces to the plan.
