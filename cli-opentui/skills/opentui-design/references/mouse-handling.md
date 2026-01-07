# Mouse Handling in OpenTUI

Patterns for handling mouse events in OpenTUI applications.

## Overview

OpenTUI provides mouse event handling through the `useMouse` option and `useSelectionHandler` hook. However, there are significant limitations to be aware of.

## useMouse Option

The `useMouse` option in the render configuration controls whether OpenTUI captures mouse events.

```typescript
import { render } from "@opentui/solid";

render(App, {
  useMouse: true, // OpenTUI captures mouse events
});
```

### useMouse: true (Default)

- OpenTUI captures all mouse events
- Wheel scroll works within the application
- Mouse click events are available
- **OS-level text selection and copy do NOT work**

### useMouse: false

- OS handles all mouse events
- Text selection and copy work normally
- **Wheel scroll does NOT work** in the application
- Mouse click events are NOT available

## Selection API Limitations

OpenTUI provides `useSelectionHandler` hook for handling text selection, but it has known limitations.

```typescript
import { useSelectionHandler } from "@opentui/core";

const MyComponent = () => {
  useSelectionHandler((selection) => {
    // This may not fire reliably!
    console.log("Selected:", selection);
  });

  return <text>Selectable text</text>;
};
```

### Known Issues

1. **Selection events may not fire** - The callback is not reliably triggered
2. **Clipboard copy may fail** - Even when selection is detected, copying may not work
3. **Terminal compatibility** - Behavior varies across terminal emulators

### Shared Issue

This limitation is shared with other terminal UI frameworks like OpenCode. It is a known issue in the terminal UI ecosystem.

## Current Recommendations

### For Applications Requiring Copy

```typescript
render(App, {
  useMouse: false, // Allow OS-level copy
});
```

Accept that wheel scroll will not work. Users must use keyboard navigation.

### For Applications Requiring Scroll

```typescript
render(App, {
  useMouse: true, // Enable wheel scroll
});
```

Provide keyboard shortcuts for copy operations instead of mouse selection.

### Keyboard-Based Copy Alternative

```typescript
const CopyableText = (props) => {
  const [copied, setCopied] = createSignal(false);

  useKeyboard((key) => {
    if (key.ctrl && key.name === "c" && props.focused) {
      // Copy to clipboard programmatically
      import("clipboardy").then(({ default: clipboard }) => {
        clipboard.writeSync(props.text);
        setCopied(true);
        setTimeout(() => setCopied(false), 2000);
      });
    }
  });

  return (
    <box>
      <text>{props.text}</text>
      {copied() && <text color="green"> (Copied!)</text>}
    </box>
  );
};
```

## Mouse Click Handling

When `useMouse: true`, you can handle click events:

```typescript
const ClickableItem = (props) => {
  return (
    <box
      onClick={() => {
        props.onSelect();
      }}
    >
      <text>{props.label}</text>
    </box>
  );
};
```

### Limitations

- Click coordinates may not be precise
- Double-click detection is not built-in
- Right-click context menus require custom implementation

## Wheel Scroll Handling

```typescript
const ScrollableList = () => {
  const [scrollOffset, setScrollOffset] = createSignal(0);

  // Wheel events are handled automatically when useMouse: true
  // Use keyboard as fallback for useMouse: false

  useKeyboard((key) => {
    if (key.name === "pageup") {
      setScrollOffset((prev) => Math.max(0, prev - 10));
    }
    if (key.name === "pagedown") {
      setScrollOffset((prev) => prev + 10);
    }
  });

  return (
    <box height={10}>
      <text wrap="wrap">{/* content with scroll offset */}</text>
    </box>
  );
};
```

## Best Practices

1. **Default to useMouse: true** - Most CLI apps benefit from wheel scroll
2. **Provide keyboard alternatives** - Don't rely solely on mouse
3. **Document copy limitations** - Let users know about selection issues
4. **Test in multiple terminals** - Mouse behavior varies
5. **Consider the use case** - Choose useMouse based on primary user needs
