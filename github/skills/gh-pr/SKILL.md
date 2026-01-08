---
name: gh-pr
description: "Create or update GitHub Pull Requests with the gh CLI, including deciding whether to create a new PR or only push based on existing PR merge status. Use when the user asks to open/create/edit a PR, generate a PR body/template, or says 'PRを出して/PR作成/gh pr'. Defaults: base=develop, head=current branch (same-branch only; never create/switch branches)."
---

# GH PR

## Overview

Create or update GitHub Pull Requests with the gh CLI using a detailed body template and strict same-branch rules.

## Decision rules (must follow)

1. **Do not create or switch branches.** Always use the current branch as the PR head.
2. **Check for an existing PR for the current head branch.**
   - `gh pr list --head <head> --state all --json number,state,mergedAt,updatedAt,url,title`
3. **If no PR exists** → create a new PR.
4. **If any PR exists and is NOT merged** (`mergedAt` is null) → push only and finish (do **not** create a new PR).
   - This applies to OPEN or CLOSED (unmerged) PRs.
   - Only update title/body/labels if the user explicitly requests changes.
5. **If all PRs for the head are merged** → create a new PR from the same head branch.
6. **If multiple PRs exist for the head** → use the most recently updated PR for reporting, but the create vs push decision is based on `mergedAt`.

## Workflow (recommended)

1. **Confirm repo + branches**
   - Repo root: `git rev-parse --show-toplevel`
   - Current branch (head): `git rev-parse --abbrev-ref HEAD`
   - Base branch defaults to `develop` unless user specifies.

2. **Check existing PR for head branch**
   - Use decision rules above to pick action.
   - Treat `mergedAt` as the source of truth for "merged".

3. **Ensure the head branch is pushed**
   - If no upstream: `git push -u origin <head>`
   - Otherwise: `git push`

4. **Collect PR inputs (for new PR or explicit update)**
   - Title, Summary, Context, Changes, Testing, Risk/Impact, Deployment, Screenshots, Related Links, Notes
   - Optional: labels, reviewers, assignees, draft

5. **Build PR body from template**
   - Read `references/pr-body-template.md` and fill placeholders.
   - If info is missing, keep TODO markers and explicitly mention them in the response.

6. **Create or update the PR**
   - Create: `gh pr create -B <base> -H <head> --title "<title>" --body-file <file>`
   - Update (only if user asked): `gh pr edit <number> --title "<title>" --body-file <file>`

7. **Return PR URL**
   - `gh pr view <number> --json url -q .url`

## Command snippets (bash)

```bash
head=$(git rev-parse --abbrev-ref HEAD)
base=develop

# Check existing PRs for the head branch
pr_count=$(gh pr list --head "$head" --state all --json number -q 'length')
unmerged_count=$(gh pr list --head "$head" --state all --json mergedAt -q 'map(select(.mergedAt == null)) | length')

if [ "$pr_count" -eq 0 ]; then
  action=create
elif [ "$unmerged_count" -gt 0 ]; then
  action=push_only
else
  action=create
fi

# Create PR body from template (edit as needed)
cat > /tmp/pr-body.md <<'BODY'
## Summary
- TODO (one-sentence outcome)
- TODO (user-visible change, if any)

## Context
- TODO (why this PR is needed)
- TODO (background, ticket, or incident link)

## Changes
- TODO (key changes, bullets)
- TODO (notable refactors or cleanup)

## Testing
- TODO (commands run)
- TODO (manual steps, if any)

## Risk / Impact
- TODO (areas impacted)
- TODO (rollback plan / mitigation)

## Deployment
- TODO (steps, flags, or "none")

## Screenshots
- TODO (UI changes only)

## Related Issues / Links
- TODO (issues, specs, docs)

## Checklist
- [ ] Tests added/updated
- [ ] Lint/format checked
- [ ] Docs updated
- [ ] Migration/backfill plan included (if needed)
- [ ] Monitoring/alerts updated (if needed)

## Notes
- TODO (optional)
BODY

# Create new PR
# gh pr create -B "$base" -H "$head" --title "..." --body-file /tmp/pr-body.md
```

## References

- `references/pr-body-template.md`: PR body template
