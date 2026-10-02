---
description: Review the current diff (or given files) against the pxkit conventions and report violations
argument-hint: [files, or empty for the working diff]
---

Review against the pxkit conventions: $ARGUMENTS (default: `git diff` + `git diff --staged`; if both are empty, the last commit).

Read `${CLAUDE_PLUGIN_ROOT}/skills/pxkit-conventions/references/review-checklist.md` and `${CLAUDE_PLUGIN_ROOT}/skills/pxkit-conventions/SKILL.md`, then only the other rule files the checklist names for the areas the diff touches. Follow the checklist's ground rules, check order, and output format.
