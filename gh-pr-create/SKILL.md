---
name: gh-pr-create
description: "Create or update GitHub Pull Requests using the gh CLI. Use when the user asks to create/open a PR, update PR title/body, or prepare a PR body from a template. Defaults: base=develop, head=feature/* unless specified."
---

# GH PR Create

## Overview

Create or update a GitHub Pull Request with the gh CLI using a consistent body template.

## Workflow (recommended)

1. **Confirm repo + branches**
   - Check current branch: `git rev-parse --abbrev-ref HEAD`
   - Default base is `develop`.
   - Default head is the current branch and should match `feature/*` unless the user specifies otherwise.
   - If the head is not a feature branch or is a protected branch, ask for confirmation.

2. **Ensure the head branch is pushed**
   - If no upstream is set: `git push -u origin <head>`

3. **Collect PR inputs**
   - Title, Summary bullets, Testing, Notes (optional)
   - Optional: labels, reviewers, assignees, draft or not

4. **Build PR body**
   - Read `references/pr-body-template.md` and fill the placeholders.
   - If information is missing, ask the user or keep TODO markers and mention them in the response.

5. **Create or update the PR**
   - If no PR exists: `gh pr create -B develop -H <head> --title "<title>" --body-file <file>`
   - If a PR already exists: `gh pr edit <number> --title "<title>" --body-file <file>`

6. **Return the PR URL**
   - `gh pr view --json url -q .url`

## Command snippets (PowerShell)

```powershell
$head = (git rev-parse --abbrev-ref HEAD).Trim()
$title = "..."
$body = @'
## Summary
- ...

## Testing
- ...

## Notes
- ...
'@

$tmp = Join-Path $env:TEMP "pr-body.txt"
$body | Set-Content -NoNewline -Encoding UTF8 $tmp

gh pr create -B develop -H $head --title $title --body-file $tmp
```

## References

- `references/pr-body-template.md`: PR body template
