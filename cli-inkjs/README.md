# CLI Design Plugin

A comprehensive CLI UI design toolkit for terminal applications.

## Overview

This plugin provides patterns, best practices, and utilities for building terminal user interfaces. Currently focused on **Ink.js** (React for CLI), with potential for future CLI framework support.

## Available Skills

### inkjs-design

Complete Ink.js development toolkit covering:

- **Component Patterns** - Screen/Part/Common component architecture
- **Custom Hooks** - useInput, useApp, useTerminalSize patterns
- **Multi-screen Navigation** - Stack-based screen management
- **Animations** - Spinners, progress bars, frame-based animations
- **State Management** - Map-based state, controlled/uncontrolled modes
- **Responsive Layouts** - Terminal size handling, character width calculations
- **Performance Optimization** - React.memo, useMemo patterns
- **Input Handling** - Keyboard shortcuts, conflict avoidance
- **Testing** - ink-testing-library patterns

## Installation

```bash
# Using Claude Code plugin installer
/plugin install cli-design@akiojin-skills
```

## Quick Start

1. Use the `/cli-design:inkjs-cli` command to access Ink.js patterns
2. Reference the `inkjs-design` skill for component development
3. Check `references/` for detailed documentation on specific topics

## Directory Structure

```
cli-design/
├── .claude-plugin/
│   └── plugin.json
├── agents/
│   └── inkjs-component-architect.md
├── commands/
│   └── inkjs-cli.md
├── skills/
│   └── inkjs-design/
│       ├── SKILL.md
│       ├── examples/
│       └── references/
└── README.md
```

## Commands

| Command | Description |
|---------|-------------|
| `/cli-design:inkjs-cli` | Invoke Ink.js design patterns and guidance |

## Agents

| Agent | Description |
|-------|-------------|
| `inkjs-component-architect` | Design Ink.js components with best practices |

## Future Roadmap

- **blessed-design** - Blessed/blessed-contrib patterns
- **prompts-design** - Enquirer/Prompts patterns
- **chalk-design** - Terminal styling patterns

## License

MIT
