---
name: review
description: Review the current diff (or given files) against the agent-craft conventions and report violations. Run as /agent-craft:review [files].
argument-hint: [files, or empty for the working diff]
disable-model-invocation: true
---

Review against the agent-craft conventions: $ARGUMENTS (default: `git diff` + `git diff --staged`; if both are empty, the last commit).

Read `${CLAUDE_PLUGIN_ROOT}/skills/conventions/references/review-checklist.md` and `${CLAUDE_PLUGIN_ROOT}/skills/conventions/SKILL.md`, then only the other rule files the checklist names for the areas the diff touches. Follow the checklist's ground rules, check order, and output format.
