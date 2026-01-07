# OpenTUI Design Skill

## Description

Comprehensive toolkit for designing and implementing CLI applications with OpenTUI and SolidJS.

## When to Use

- Building new CLI screens or components with OpenTUI
- Debugging input handling issues
- Implementing screen navigation
- Handling mouse events and limitations
- Optimizing CLI performance

## Key References

- `input-handling.md` - Key input patterns and propagation prevention
- `mouse-handling.md` - Mouse events and Selection API limitations
- `opentui-gotchas.md` - Common issues and workarounds
- `component-patterns.md` - Component structure and composition
- `multi-screen-navigation.md` - Screen transition patterns
- `state-management.md` - SolidJS state patterns
- `hooks-guide.md` - OpenTUI hooks reference
- `performance-optimization.md` - Performance best practices
- `responsive-layout.md` - Terminal layout patterns
- `testing-patterns.md` - Testing CLI components

## Critical Knowledge

### Key Propagation Prevention

When navigating between screens with Enter key, the same keypress can propagate to the new screen. Always use:

1. `key.preventDefault()` at the source
2. Initial frame delay at the destination
3. `focused` prop control on interactive elements

### SolidJS Reactivity

Never destructure props - always access via `props.xxx` to maintain reactivity.

### Mouse Limitations

OpenTUI's Selection API (`useSelectionHandler`) does not reliably work. Set `useMouse: false` to allow OS-level copy, but this disables wheel scroll.
