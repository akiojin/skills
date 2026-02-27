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

Always return a structured summary that includes:

- `status`
- `recommended_action`
- `reason`
- `head`
- `base`
- `pr_count`
- `evidence` (commands and key values)
- `worktree_dirty` (boolean)

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

Example response format:

```json
{
  "status": "ALL_MERGED_WITH_NEW_COMMITS",
  "recommended_action": "CREATE_PR",
  "reason": "3 commits exist after merge commit abcdef1",
  "head": "feature/my-branch",
  "base": "develop",
  "pr_count": 2,
  "worktree_dirty": false,
  "evidence": {
    "latest_merged_pr": 123,
    "merge_commit": "abcdef1234567890",
    "new_commits_after_merge": 3
  }
}
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
5. Print final status + recommended action + evidence.

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

echo "status=$status action=$action reason=$reason worktree_dirty=$dirty"
```

## Related skill

- `gh-pr`: creates/updates PRs
- `gh-pr-check`: inspects PR status only
