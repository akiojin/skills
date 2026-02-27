---
description: Check GitHub PR status and post-merge commits using the gh-pr-check skill
author: akiojin
allowed-tools: Read, Glob, Grep, Bash
---

# GitHub PR Check Command

Use this command to inspect PR status for the current branch with the gh CLI.

## Usage

```text
/github:gh-pr-check [optional context]
```

## Steps

1. Load `github/skills/gh-pr-check/SKILL.md` and follow the workflow.
2. Ensure `gh auth status` succeeds before running PR checks.
3. Run checks and report `status`, `recommended_action`, and evidence.
4. Do not push or create/edit PRs in this command.

## Examples

```text
/github:gh-pr-check
```

```text
/github:gh-pr-check check current branch after merge
```
