# Claude Code Skills

A collection of Claude Code plugins and skills for software development.

## Available Plugins

### unity-development

Unity C# development skill with:

- **unity-mcp-server tools**: Symbol-based code exploration and editing
- **VContainer DI**: Dependency injection patterns
- **UniTask**: Async processing patterns
- **Fail-Fast principle**: No null-check defensive coding

## Installation

### Add Marketplace

```bash
/plugin marketplace add akiojin/skills
```

### Install Plugin

```bash
/plugin install unity-development@akiojin-skills
```

Or interactively:

```bash
/plugin
# Select "Browse Plugins"
# Choose unity-development
```

## Usage

Once installed, the skill is automatically triggered when:

- Editing Unity C# scripts
- Working with GameObjects and components
- Running Unity tests (EditMode/PlayMode)
- Configuring VContainer DI
- Implementing UniTask async methods

## License

MIT
