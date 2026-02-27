---
description: Create or update GitHub PRs with the gh CLI using the gh-pr skill
author: akiojin
allowed-tools: Read, Glob, Grep, Bash
---

# GitHub PR Command

Use this command to draft or update a GitHub PR with the gh CLI.

## Usage

```
/github:gh-pr [optional context]
```

## Steps

1. Load `github/skills/gh-pr/SKILL.md` and follow the workflow.
2. Ensure `gh auth status` succeeds before running PR commands.
3. Run the local working tree preflight from the skill (`git status --porcelain`); if changes exist, confirm with the user before push/PR operations.
4. Generate or update the PR body using the provided templates.

## Examples

```
/github:gh-pr create draft for current branch
```

```
/github:gh-pr update PR body only
```
