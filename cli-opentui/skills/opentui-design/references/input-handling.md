# Input Handling in OpenTUI Applications

Patterns for handling keyboard input in OpenTUI/SolidJS CLI applications.

## Overview

OpenTUI provides the `useKeyboard` hook for handling key presses. Unlike Ink.js, OpenTUI does not have a built-in `isActive` option, so key propagation must be managed explicitly.

## useKeyboard Basics

```typescript
import { useKeyboard } from "@opentui/core";

const MyComponent = () => {
  useKeyboard((key) => {
    if (key.name === "q") {
      process.exit(0);
    }

    if (key.name === "up") {
      // Handle up arrow
    }

    if (key.name === "down") {
      // Handle down arrow
    }

    if (key.name === "return") {
      // Handle Enter
    }
  });

  return <text>Press 'q' to quit</text>;
};
```

## Key Object Properties

```typescript
interface Key {
  name: string;        // Key name: "return", "up", "down", "escape", etc.
  ctrl: boolean;       // Ctrl modifier
  shift: boolean;      // Shift modifier
  meta: boolean;       // Meta/Command modifier
  sequence: string;    // Raw key sequence
  preventDefault: () => void;  // Stop propagation
}
```

## Key Propagation Problem

### Problem: Enter key fires on next screen immediately

When user presses Enter to navigate from Screen A to Screen B, the same Enter keypress can propagate to Screen B and trigger unintended actions (e.g., immediately selecting the first item).

```typescript
// User presses Enter on Screen A
// Screen B mounts
// Screen B's useKeyboard receives the same Enter -> BAD!
```

### Solution 1: preventDefault() - Recommended

```typescript
useKeyboard((key) => {
  if (key.name === "return") {
    key.preventDefault();  // Stop propagation to next screen
    navigateToNextScreen();
  }
});
```

### Solution 2: Initial Frame Delay

```typescript
import { createSignal, createEffect } from "solid-js";

const ScreenB = (props) => {
  const [isInitialFrame, setIsInitialFrame] = createSignal(true);

  createEffect(() => {
    if (props.visible) {
      setIsInitialFrame(true);
      const timer = setTimeout(() => setIsInitialFrame(false), 150);
      return () => clearTimeout(timer);
    }
  });

  useKeyboard((key) => {
    if (isInitialFrame() && key.name === "return") {
      return; // Ignore Enter on first frame
    }
    // Handle input...
  });

  return <box>Screen B content</box>;
};
```

### Solution 3: Focused Prop Control

```typescript
const ScreenB = (props) => {
  const [isReady, setIsReady] = createSignal(false);

  createEffect(() => {
    if (props.visible) {
      setIsReady(false);
      setTimeout(() => setIsReady(true), 150);
    }
  });

  return <SelectInput focused={isReady()} items={items} />;
};
```

### Best Practice: Combine Multiple Approaches

Use `preventDefault()` at the source AND initial frame delay at the destination for maximum reliability.

```typescript
// Source screen
useKeyboard((key) => {
  if (key.name === "return") {
    key.preventDefault();  // First line of defense
    setCurrentScreen("screenB");
  }
});

// Destination screen
const ScreenB = (props) => {
  const [inputReady, setInputReady] = createSignal(false);

  createEffect(() => {
    if (props.visible) {
      setInputReady(false);
      const timer = setTimeout(() => setInputReady(true), 150);
      return () => clearTimeout(timer);
    }
  });

  useKeyboard((key) => {
    if (!inputReady()) return;  // Second line of defense
    // Handle input safely...
  });
};
```

## Enter Double-Press Prevention (App Launch)

### Problem

First render may buffer an Enter press from the previous shell command:

```typescript
// User types: node cli.js[Enter]
// Enter key is captured by the first useKeyboard
```

### Solution: Buffer First Frame

```typescript
import { createSignal, onMount } from "solid-js";

export function useInputWithBuffer() {
  const [ready, setReady] = createSignal(false);

  onMount(() => {
    const timer = setTimeout(() => setReady(true), 50);
    return () => clearTimeout(timer);
  });

  return ready;
}

// Usage
const SelectScreen = () => {
  const ready = useInputWithBuffer();

  useKeyboard((key) => {
    if (!ready()) return; // Ignore buffered input

    if (key.name === "return") {
      // Safe to handle Enter
    }
  });
};
```

## Multiple useKeyboard Coordination

### Problem: All Handlers Fire

Unlike Ink.js, OpenTUI's `useKeyboard` does not have an `isActive` option. All handlers receive every key press.

```typescript
// Both handlers receive every key press
const Parent = () => {
  useKeyboard((key) => {
    console.log("Parent received:", key.name);
  });

  return <Child />;
};

const Child = () => {
  useKeyboard((key) => {
    console.log("Child received:", key.name); // Also fires!
  });

  return <text>Child</text>;
};
```

### Solution: Manual Active State Check

```typescript
const Screen = (props) => {
  useKeyboard((key) => {
    if (!props.active) return; // Skip if not active

    // Handle input only when active
  });
};
```

## Filter Mode Toggle

Implement filter/search mode:

```typescript
type InputMode = "navigation" | "filter";

const FilterableList = () => {
  const [mode, setMode] = createSignal<InputMode>("navigation");
  const [filterText, setFilterText] = createSignal("");

  useKeyboard((key) => {
    if (mode() === "navigation") {
      if (key.sequence === "/") {
        setMode("filter");
        return;
      }

      if (key.name === "up" || key.name === "down") {
        // Navigate list
      }
    }

    if (mode() === "filter") {
      if (key.name === "escape") {
        setMode("navigation");
        setFilterText("");
        return;
      }

      if (key.name === "return") {
        setMode("navigation");
        // Apply filter
        return;
      }

      if (key.name === "backspace") {
        setFilterText((prev) => prev.slice(0, -1));
        return;
      }

      // Printable character
      if (key.sequence && key.sequence.length === 1) {
        setFilterText((prev) => prev + key.sequence);
      }
    }
  });

  return (
    <box flexDirection="column">
      {mode() === "filter" && <text>Filter: {filterText()}_</text>}
      {/* List items */}
    </box>
  );
};
```

## Special Key Combinations

```typescript
useKeyboard((key) => {
  // Ctrl+C
  if (key.ctrl && key.name === "c") {
    /* ... */
  }

  // Ctrl+S
  if (key.ctrl && key.name === "s") {
    /* ... */
  }

  // Escape
  if (key.name === "escape") {
    /* ... */
  }

  // Tab
  if (key.name === "tab") {
    /* ... */
  }

  // Shift+Tab
  if (key.shift && key.name === "tab") {
    /* ... */
  }
});
```

## Best Practices

1. **Always use preventDefault()** - Call it when handling Enter to prevent propagation
2. **Buffer first frame** - Prevent capturing Enter from shell command
3. **Implement early returns** - Check active state before handling
4. **Show available shortcuts** - Display help in footer
5. **Group shortcuts by context** - Different modes have different keys
6. **Handle Escape for cancel** - Consistent back/cancel behavior
7. **Test screen transitions** - Verify no key propagation issues
