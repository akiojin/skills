---
name: gh-pr-check
description: "Check GitHub PR status with the gh CLI, including unmerged PR detection and post-merge new-commit detection for the current branch."
---

# GH PR Check

## Overview

Check PR status for the current branch with `gh` and report a recommended next action.

This skill is **check-only**:

- Do not create/switch branches
- Do not push
- Do not create/edit PRs

## Decision rules (must follow)

1. Resolve repository, `head` branch, and `base` branch.
   - `head`: current branch (`git rev-parse --abbrev-ref HEAD`)
   - `base`: default `develop` unless user specifies
2. Optionally collect local working tree state:
   - `git status --porcelain`
   - Report as context only; do not mutate files.
3. Fetch latest remote refs before comparing:
   - `git fetch origin`
4. List PRs for head branch:
   - `gh pr list --head <head> --state all --json`
   - `number,state,mergedAt,updatedAt,url,title,mergeCommit,baseRefName,headRefName`
5. Classify:
   - No PR found -> `NO_PR` + recommended action `CREATE_PR`
   - Any PR where `mergedAt == null`
     -> `UNMERGED_PR_EXISTS` + recommended action `PUSH_ONLY`
   - All PRs merged -> perform post-merge commit check
6. Post-merge commit check (critical when all PRs are merged):
   - Select latest merged PR by `mergedAt`
   - Get merge commit SHA from `mergeCommit.oid`
   - Count commits after merge:
     - `git rev-list --count <merge_commit>..HEAD`
   - If count > 0 -> `ALL_MERGED_WITH_NEW_COMMITS` + `CREATE_PR`
   - If count == 0 -> `ALL_MERGED_NO_NEW_COMMITS` + `NO_ACTION`
7. Fallback when merge commit SHA is missing:
   - Compare against base:
     - `git rev-list --count origin/<base>..HEAD`
   - Count > 0 -> `ALL_MERGED_WITH_NEW_COMMITS` + `CREATE_PR` (fallback)
   - Count == 0 -> `ALL_MERGED_NO_NEW_COMMITS` + `NO_ACTION` (fallback)
   - If comparison fails -> `CHECK_FAILED` + `MANUAL_CHECK`

## Output contract

Return a human-readable summary by default.

Do not return raw JSON as the default output.
If JSON is explicitly requested by the user, append it after the human summary.

Recommended status values:

- `NO_PR`
- `UNMERGED_PR_EXISTS`
- `ALL_MERGED_WITH_NEW_COMMITS`
- `ALL_MERGED_NO_NEW_COMMITS`
- `CHECK_FAILED`

Recommended action values:

- `CREATE_PR`
- `PUSH_ONLY`
- `NO_ACTION`
- `MANUAL_CHECK`

### Language rule

- Follow the user's input language for all headings and messages.
- If the language is ambiguous, use English.

### Default output template

Use this section order:

1. Result (one line)
2. Recommended next step (one line)
3. Why (1-2 lines)
4. Context
5. Evidence
6. Open PRs (only when PRs exist)

Include these fields in the rendered text:

- `status` (mapped to a natural sentence)
- `recommended_action` (mapped to concrete next action)
- `reason`
- `head` / `base`
- `pr_count`
- `worktree_dirty`
- key evidence values:
  - `unmerged_count`
  - `latest_merged_pr`
  - `merge_commit`
  - `new_commits_after_merge`

### Status-to-message mapping (must use)

- `NO_PR`
  - Result: "No PR exists for this branch."
  - Next step: "Create a new PR after pushing the branch if needed."
- `UNMERGED_PR_EXISTS`
  - Result: "At least one PR for this branch is still unmerged."
  - Next step: "Push updates to the existing PR; do not create a new PR."
- `ALL_MERGED_WITH_NEW_COMMITS`
  - Result: "All previous PRs are merged, and new commits were found."
  - Next step: "Create a new PR for the commits made after the last merge."
- `ALL_MERGED_NO_NEW_COMMITS`
  - Result: "All previous PRs are merged, and no new commits were found."
  - Next step: "No PR action is needed."
- `CHECK_FAILED`
  - Result: "PR status check could not be completed."
  - Next step: "Run manual checks for merge commit and branch diff."

### Example human-readable output

```text
PR Check Result: All previous PRs are merged, and new commits were found.
Recommended next step: Create a new PR for the commits made after the last merge.
Why: 3 commits were detected after merge commit abcdef1.

Context
- head: feature/my-branch
- base: develop
- worktree dirty: no

Evidence
- PR count for head branch: 2
- unmerged PR count: 0
- latest merged PR: #123
- merge commit: abcdef1234567890
- commits after merge: 3

Open PRs
- #123 merged https://github.com/org/repo/pull/123
- #120 merged https://github.com/org/repo/pull/120
```

## Workflow (recommended)

1. Verify repo context:
   - `git rev-parse --show-toplevel`
   - `git rev-parse --abbrev-ref HEAD`
2. Confirm auth:
   - `gh auth status`
3. Collect context:
   - `git status --porcelain`
   - `git fetch origin`
4. List PRs for head branch and classify using rules above.
5. Print human-readable result using the default template.
6. Append JSON only if the user explicitly asks for machine-readable output.

## Command snippet (bash)

```bash
head="${HEAD_BRANCH:-$(git rev-parse --abbrev-ref HEAD)}"
base="${BASE_BRANCH:-develop}"

dirty=0
if [ -n "$(git status --porcelain)" ]; then
  dirty=1
fi

git fetch origin

pr_json="$(gh pr list --head "$head" --state all --json number,state,mergedAt,updatedAt,url,title,mergeCommit)"
pr_count="$(echo "$pr_json" | jq 'length')"
unmerged_count="$(echo "$pr_json" | jq 'map(select(.mergedAt == null)) | length')"

if [ "$pr_count" -eq 0 ]; then
  status="NO_PR"
  action="CREATE_PR"
  reason="No PR found for head branch"
elif [ "$unmerged_count" -gt 0 ]; then
  status="UNMERGED_PR_EXISTS"
  action="PUSH_ONLY"
  reason="At least one PR for the head branch is not merged"
else
  merge_commit="$(echo "$pr_json" | jq -r 'sort_by(.mergedAt) | last | .mergeCommit.oid')"
  if [ -n "$merge_commit" ] && [ "$merge_commit" != "null" ]; then
    new_commits="$(
      git rev-list --count "$merge_commit"..HEAD 2>/dev/null || echo ""
    )"
  else
    new_commits=""
  fi

  if [ -n "$new_commits" ]; then
    if [ "$new_commits" -gt 0 ]; then
      status="ALL_MERGED_WITH_NEW_COMMITS"
      action="CREATE_PR"
      reason="$new_commits commits found after last merge"
    else
      status="ALL_MERGED_NO_NEW_COMMITS"
      action="NO_ACTION"
      reason="No commits found after last merge"
    fi
  else
    fallback_commits="$(
      git rev-list --count "origin/$base"..HEAD 2>/dev/null || echo ""
    )"
    if [ -n "$fallback_commits" ]; then
      if [ "$fallback_commits" -gt 0 ]; then
        status="ALL_MERGED_WITH_NEW_COMMITS"
        action="CREATE_PR"
        reason="Fallback check found commits ahead of origin/$base"
      else
        status="ALL_MERGED_NO_NEW_COMMITS"
        action="NO_ACTION"
        reason="Fallback check found no commits ahead of origin/$base"
      fi
    else
      status="CHECK_FAILED"
      action="MANUAL_CHECK"
      reason="Could not resolve merge commit and fallback comparison failed"
    fi
  fi
fi

case "$status" in
  NO_PR)
    result_msg="No PR exists for this branch."
    next_msg="Create a new PR after pushing the branch if needed."
    ;;
  UNMERGED_PR_EXISTS)
    result_msg="At least one PR for this branch is still unmerged."
    next_msg="Push updates to the existing PR; do not create a new PR."
    ;;
  ALL_MERGED_WITH_NEW_COMMITS)
    result_msg="All previous PRs are merged, and new commits were found."
    next_msg="Create a new PR for the commits made after the last merge."
    ;;
  ALL_MERGED_NO_NEW_COMMITS)
    result_msg="All previous PRs are merged, and no new commits were found."
    next_msg="No PR action is needed."
    ;;
  *)
    result_msg="PR status check could not be completed."
    next_msg="Run manual checks for merge commit and branch diff."
    ;;
esac

echo "PR Check Result: $result_msg"
echo "Recommended next step: $next_msg"
echo "Why: $reason"
echo "Context:"
echo "- head: $head"
echo "- base: $base"
echo "- worktree dirty: $dirty"
echo "- PR count for head branch: $pr_count"
```

## Related skill

- `gh-pr`: creates/updates PRs
- `gh-pr-check`: inspects PR status only
