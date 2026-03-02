---
description: >-
  Inspect GitHub PR for CI failures, merge conflicts, update-branch requirements,
  reviewer comments, change requests, and unresolved review threads.
  Create fix plans and implement after user approval.
  Use when: (1) user explicitly asks to fix CI/PR issues,
  (2) after creating or pushing to a PR and CI checks fail or reviews are requested,
  (3) user says 'CIを直して/fix ci/fix pr/マージできない'.
  PROACTIVE: After any PR creation or push, if CI failures or merge blockers
  are detected, automatically invoke this skill without waiting for user request.
author: akiojin
allowed-tools: Read, Glob, Grep, Bash
---

# GitHub CI Fix Command

Use this command to diagnose and fix CI failures for a PR.

## Usage

```
/github:gh-fix-ci [pr-number|pr-url|optional context]
```

## Steps

1. Load `github/skills/gh-fix-ci/SKILL.md` and follow the workflow.
2. Run the inspection script to gather CI results.
3. Propose fixes and apply them after user approval.

## Proactive Trigger Examples

<example>
Context: LLM just created a PR using gh-pr, and CI checks may fail
user: "PRを作成して"
assistant: (gh-pr でPR作成完了後) "PR #123 を作成しました。CI チェックの状態を確認します。"
<commentary>
PR作成直後、自動的に gh-fix-ci で検査する。
</commentary>
</example>

<example>
Context: User pushed commits and mentions the PR isn't passing
user: "pushしたけどCIが通らない"
assistant: "gh-fix-ci で PR の状態を診断します。"
</example>

<example>
Context: User mentions PR can't be merged
user: "PRがマージできない"
assistant: "gh-fix-ci で blocking items を診断します。"
</example>

## Examples

```
/github:gh-fix-ci 123
```

```
/github:gh-fix-ci https://github.com/org/repo/pull/123
```
