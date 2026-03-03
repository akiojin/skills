# Codex & Claude Code Skills

A collection of skills and plugins for Codex and Claude Code.

## Available Plugins

### cli-design

Ink.js CLI UI design skill with:

- **string-width handling**: Emoji/icon width calculation fixes for terminal alignment
- **Component patterns**: Screen/Part/Common component classification
- **Custom hooks**: useScreenState, useTerminalSize patterns
- **React.memo optimization**: Custom comparison functions for performance
- **Testing patterns**: ink-testing-library with Vitest

### unity-development

Unity C# development skill with:

- **unity-mcp-server tools**: Symbol-based code exploration and editing
- **VContainer DI**: Dependency injection patterns
- **UniTask**: Async processing patterns
- **Fail-Fast principle**: No null-check defensive coding

### speckit

Spec Kit skills (plugin) with:

- **Requirements workflow**: spec.md → plan.md → tasks.md flow
- **Upstream update**: GitHub Spec Kit version sync

### github

GitHub workflow skills with:

- **PR automation**: gh-pr workflow and PR body templates
- **PR status checks**: gh-pr-check workflow for unmerged PRs and post-merge commits
- **PR blocker diagnostics**: gh-fix-pr workflow and inspection scripts (compat: gh-fix-ci)
- **Issue analysis**: gh-fix-issue workflow for analyzing Issues, extracting error context, and proposing fix plans

### drawio

Draw.io diagram creation and editing skill with:

- **XML structure**: Direct .drawio file manipulation
- **Shape support**: Rectangle, ellipse, diamond, text, and connectors
- **Japanese fonts**: Noto Sans JP with proper sizing
- **Export options**: PNG, SVG, PDF via drawio-export CLI
- **Style guide**: Color palette, layout rules, and best practices

## Installation (Claude Code)

### Add Marketplace

```bash
/plugin marketplace add akiojin/skills
```

### Install Plugin

```bash
/plugin install cli-design@akiojin-skills
```

Or:

```bash
/plugin install unity-development@akiojin-skills
```

Or:

```bash
/plugin install speckit@akiojin-skills
```

Or:

```bash
/plugin install github@akiojin-skills
```

Or:

```bash
/plugin install drawio@akiojin-skills
```

Or interactively:

```bash
/plugin
# Select "Browse Plugins"
# Choose cli-design, unity-development, speckit, github, or drawio
```

## Installation (Codex)

Codex reads skills from folders under `.codex/skills` (repo or user scope).
This repo also ships packaged `.skill` files under `codex-skills/dist/`.

### Option A: Install from GitHub with skill-installer

```bash
# From GitHub repo/path
scripts/install-skill-from-github.py --repo akiojin/skills --path gh-pr --ref main
```

### Option B: Unzip .skill into $CODEX_HOME/skills

```bash
# Example (PowerShell)
$dest = "$env:USERPROFILE\.codex\skills"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Expand-Archive -Path .\codex-skills\dist\gh-pr.skill -DestinationPath $dest -Force
```

```bash
# Example (bash)
dest="$HOME/.codex/skills"
mkdir -p "$dest"
unzip -o ./codex-skills/dist/gh-pr.skill -d "$dest"
```

### Available Codex skills

- gh-pr
- gh-pr-check
- gh-fix-pr
- gh-fix-ci (legacy alias)
- gh-fix-issue
- speckit-require
- speckit-update
- drawio
- inkjs-design

After installation, restart Codex to load new skills.

## Usage (Codex)

### gh-pr

Use when creating or updating GitHub PRs with the gh CLI, or when you want a PR body generated from a template.

### gh-pr-check

Use when checking PR status for the current branch with a human-readable
summary, including unmerged PR detection and new commits after the latest
merged PR.

### gh-fix-pr

Use when diagnosing/fixing PR blockers (CI failures, conflicts, review blockers) for PRs and applying follow-up fixes.

### gh-fix-ci (legacy alias)

Kept for backward compatibility; delegates to `gh-fix-pr`.

### gh-fix-issue

Use when analyzing a GitHub Issue to extract error context, search the codebase for relevant files, and propose a concrete fix plan.

### speckit-require

Use when creating or updating requirement specs with Spec Kit (specify/clarify/plan/tasks flow).

### speckit-update

Use when updating GitHub Spec Kit versions and syncing templates/scripts while preserving local rules.

### drawio

Use when creating or editing .drawio files (XML) and exporting diagrams.

### inkjs-design

Use when designing Ink.js CLI UIs (layout, input handling, string-width/emoji alignment, tests).

## Usage (Claude Code)

### cli-design

Automatically triggered when:

- Creating Ink.js terminal UI components
- Handling emoji/icon width issues (string-width)
- Implementing keyboard input handling (useInput)
- Designing responsive terminal layouts
- Writing CLI component tests

### unity-development

Automatically triggered when:

- Editing Unity C# scripts
- Working with GameObjects and components
- Running Unity tests (EditMode/PlayMode)
- Configuring VContainer DI
- Implementing UniTask async methods

### speckit-require

Automatically triggered when:

- Creating or updating requirements specs
- Drafting specifications (specification/spec doc authoring)
- Running Spec Kit workflows (specify/clarify/plan/tasks)

### speckit-update

Automatically triggered when:

- Updating GitHub Spec Kit versions in a repo
- Syncing Spec Kit templates/commands/scripts from upstream
- Preserving local Spec Kit rules (Japanese, no branch ops, SPEC-UUID)

### github

Slash commands:

- `/github:gh-pr`
- `/github:gh-pr-check`
- `/github:gh-fix-pr`
- `/github:gh-fix-ci` (legacy alias)
- `/github:gh-fix-issue`

### speckit (slash commands)

- `/speckit:speckit-require`
- `/speckit:speckit-update`

### drawio

Automatically triggered when:

- Creating flowcharts, architecture diagrams, or sequence diagrams
- Editing .drawio files in XML format
- Setting up diagram styling and fonts
- Exporting diagrams to PNG, SVG, or PDF

## License

MIT

## Add a New Skill

### 1) Create the skill folder

- **Codex skill**: create a folder at repo root with `SKILL.md` (YAML frontmatter required).
- **Claude Code plugin**: create a plugin folder at repo root. If it contains multiple skills, place them under `<plugin>/skills/<skill-name>/SKILL.md`.

Example (plugin with multiple skills):

```
speckit/
└── skills/
    ├── speckit-require/
    │   └── SKILL.md
    └── speckit-update/
        └── SKILL.md
```

### 2) Register for Claude Code (plugin)

Add a new entry to `.claude-plugin/marketplace.json`:

- `name`, `source`, `description`, `version`, `category`, `keywords`

Example entry (plugin):

```json
{
  "name": "speckit",
  "source": "./speckit",
  "description": "Spec Kit skills: requirements workflow + upstream update.",
  "version": "0.1.0",
  "category": "development",
  "keywords": ["spec-kit", "requirements", "specification", "update", "workflow", "tdd"]
}
```

### 3) Package for Codex

Package the skill into `codex-skills/dist/`:

```bash
export PYTHONUTF8=1
codex_home="${CODEX_HOME:-$HOME/.codex}"
python "$codex_home/skills/.system/skill-creator/scripts/package_skill.py" \
  "<repo-root>/<skill-folder>" \
  "<repo-root>/codex-skills/dist"
```

Example (single skill package):

```bash
export PYTHONUTF8=1
codex_home="${CODEX_HOME:-$HOME/.codex}"
python "$codex_home/skills/.system/skill-creator/scripts/package_skill.py" \
  "$PWD/speckit/skills/speckit-update" \
  "$PWD/codex-skills/dist"
```

### 4) Update this README

- Add the skill under **Available Plugins**, **Available Codex skills**, and the **Usage** sections.

### 5) Add resources (scripts / references / assets)

Create only the directories you need under the skill folder:

- `scripts/` for executable automation (bash/python/etc.)
- `references/` for docs you want the model to read on demand
- `assets/` for templates or files to copy into outputs

Example:

```
speckit/skills/speckit-update/
├── SKILL.md
├── scripts/
│   └── sync-upstream.sh
├── references/
│   └── upstream-diff-checklist.md
└── assets/
    └── templates/
        └── spec-template.md
```
