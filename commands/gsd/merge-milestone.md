---
name: gsd:merge-milestone
description: Merge milestone branch into target branch with AI-assisted conflict resolution (team mode)
argument-hint: "[target branch] (if omitted, will ask)"
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
---

<objective>
Merge a milestone branch into a target branch using squash merge.
Resolves conflicts with AI-assisted, context-aware merge logic.
Designed for team mode workflows where each developer works on an isolated branch.
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/merge-milestone.md
</execution_context>

Execute the merge-milestone workflow from @~/.claude/get-shit-done/workflows/merge-milestone.md end-to-end.
