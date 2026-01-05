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

### requirements-spec-kit

Spec Kit workflow skill with:

- **Requirements definition**: New or updated requirement specs
- **Specification authoring**: spec.md → plan.md → tasks.md flow
- **TDD alignment**: Test-first requirements capture

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
/plugin install requirements-spec-kit@akiojin-skills
```

Or:

```bash
/plugin install drawio@akiojin-skills
```

Or interactively:

```bash
/plugin
# Select "Browse Plugins"
# Choose cli-design, unity-development, requirements-spec-kit, or drawio
```

## Installation (Codex)

Codex reads skills from folders under `.codex/skills` (repo or user scope).
This repo also ships packaged `.skill` files under `codex-skills/dist/`.

### Option A: Install from GitHub with skill-installer

```bash
# From GitHub repo/path
scripts/install-skill-from-github.py --repo akiojin/skills --path gh-pr-create --ref main
```

### Option B: Unzip .skill into $CODEX_HOME/skills

```bash
# Example (PowerShell)
$dest = "$env:USERPROFILE\.codex\skills"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Expand-Archive -Path .\codex-skills\dist\gh-pr-create.skill -DestinationPath $dest -Force
```

```bash
# Example (bash)
dest="$HOME/.codex/skills"
mkdir -p "$dest"
unzip -o ./codex-skills/dist/gh-pr-create.skill -d "$dest"
```

### Available Codex skills

- gh-pr-create
- gh-fix-ci
- requirements-spec-kit
- drawio
- inkjs-design

After installation, restart Codex to load new skills.

## Usage (Codex)

### gh-pr-create

Use when creating or updating GitHub PRs with the gh CLI, or when you want a PR body generated from a template.

### gh-fix-ci

Use when debugging failing GitHub Actions checks for a PR and you want a fix plan + code changes.

### requirements-spec-kit

Use when creating or updating requirement specs with Spec Kit (specify/clarify/plan/tasks flow).

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

### requirements-spec-kit

Automatically triggered when:

- Creating or updating requirements specs
- Drafting specifications (specification/spec doc authoring)
- Running Spec Kit workflows (specify/clarify/plan/tasks)

### drawio

Automatically triggered when:

- Creating flowcharts, architecture diagrams, or sequence diagrams
- Editing .drawio files in XML format
- Setting up diagram styling and fonts
- Exporting diagrams to PNG, SVG, or PDF

## License

MIT
