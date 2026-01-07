---
description: Invoke Ink.js CLI design patterns and component guidance
allowed-tools: Read, Glob, Grep
---

# Ink.js CLI Design Command

This command provides access to Ink.js (React for CLI) design patterns and implementation guidance.

## Usage

```
/cli-design:inkjs-cli [topic]
```

## Available Topics

### Component Design
- `component` - Component classification and patterns
- `screen` - Screen component patterns
- `part` - Part component patterns
- `common` - Common input component patterns

### Patterns & Techniques
- `navigation` - Multi-screen navigation patterns
- `animation` - Spinner and progress bar patterns
- `state` - State management patterns
- `layout` - Responsive layout patterns
- `performance` - Performance optimization
- `input` - Input handling patterns

### Troubleshooting
- `gotchas` - Common issues and solutions
- `emoji` - Emoji/icon width handling
- `ctrlc` - Ctrl+C handling
- `useinput` - useInput conflict resolution

### Testing
- `testing` - ink-testing-library patterns

## Examples

### Get component patterns
```
/cli-design:inkjs-cli component
```

### Troubleshoot emoji width issues
```
/cli-design:inkjs-cli emoji
```

### Learn about multi-screen navigation
```
/cli-design:inkjs-cli navigation
```

## Quick Reference

### Component Classification

| Type | Purpose | Key Features |
|------|---------|--------------|
| Screen | Full-page view | useInput, Header/Content/Footer |
| Part | Reusable element | React.memo, pure/stateless |
| Common | Input component | Controlled/uncontrolled modes |

### Directory Structure

```
src/cli/ui/
├── components/
│   ├── App.tsx              # Root component
│   ├── common/              # Common components
│   ├── parts/               # Part components
│   └── screens/             # Screen components
├── hooks/                   # Custom hooks
├── utils/                   # Utilities
└── types.ts                 # Type definitions
```

### Essential Hooks

| Hook | Purpose |
|------|---------|
| `useInput` | Keyboard input handling |
| `useApp` | App lifecycle (exit) |
| `useFocus` | Focus management |
| `useStdin` | Raw stdin access |

### Common Gotchas

1. **Emoji width** - Use WIDTH_OVERRIDES for string-width v8
2. **useInput conflict** - Use `isActive` option or early return
3. **Ctrl+C** - Set `exitOnCtrlC: false` and handle manually
4. **Enter double-press** - Buffer first render

## Skill Reference

For detailed documentation, reference the `inkjs-design` skill:

- `skills/inkjs-design/SKILL.md` - Core skill document
- `skills/inkjs-design/references/` - Detailed reference documents
- `skills/inkjs-design/examples/` - Practical examples
