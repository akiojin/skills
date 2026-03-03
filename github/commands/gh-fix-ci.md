---
description: Backward-compatible alias for the `gh-fix-pr` command.
author: akiojin
allowed-tools: Read, Glob, Grep, Bash
---

# GitHub PR Fix Command (compatibility)

This command is a compatibility wrapper for `gh-fix-pr`.
Prefer using `/github:gh-fix-pr` going forward.

## Usage

```
/github:gh-fix-ci [pr-number|pr-url|optional context]
```

## Steps

1. Load `github/skills/gh-fix-ci/SKILL.md` and follow its workflow.
2. The workflow delegates to `github/skills/gh-fix-pr/SKILL.md`.
3. Run the same inspection and fix steps via `gh-fix-pr`.

## Notes

`/github:gh-fix-ci` will use the same behavior as `/github:gh-fix-pr` so existing
workflows continue to work unchanged.
