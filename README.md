# Claude Code Skills

A collection of Claude Code plugins and skills for software development.

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

## Installation

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

Or interactively:

```bash
/plugin
# Select "Browse Plugins"
# Choose cli-design, unity-development, or requirements-spec-kit
```

## Usage

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

## License

MIT
