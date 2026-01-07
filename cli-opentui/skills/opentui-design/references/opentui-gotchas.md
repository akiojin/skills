# OpenTUI Gotchas

Common issues and pitfalls when developing with OpenTUI and SolidJS.

## 1. Mouse Selection/Copy Not Working

**Problem**: Users cannot select text with mouse and copy to clipboard.

**Cause**: OpenTUI's Selection API (`useSelectionHandler`) does not reliably trigger on mouse selection. This is a known issue shared with OpenCode and other terminal UI frameworks.

**Workaround**:
```typescript
render(App, {
  useMouse: false, // Allow OS-level copy
});
```

**Trade-off**: This disables wheel scroll control within the application.

## 2. Key Event Propagation

**Problem**: When pressing Enter to navigate to a new screen, the Enter key fires on the new screen immediately, causing unintended actions.

**Cause**: Unlike Ink.js's `useInput({ isActive })`, OpenTUI's `useKeyboard` always fires for all mounted components.

**Solution**:
```typescript
// At the source screen
useKeyboard((key) => {
  if (key.name === "return") {
    key.preventDefault(); // Critical!
    navigateToNextScreen();
  }
});

// At the destination screen
const [ready, setReady] = createSignal(false);
onMount(() => {
  setTimeout(() => setReady(true), 150);
});
useKeyboard((key) => {
  if (!ready()) return;
  // Handle keys safely
});
```

## 3. SolidJS Props Destructuring

**Problem**: Component stops reacting to prop changes.

**Cause**: Destructuring props breaks SolidJS reactivity.

**Wrong**:
```typescript
const MyComponent = ({ value, onChange }) => {
  // value and onChange are now static!
  return <text>{value}</text>;
};
```

**Correct**:
```typescript
const MyComponent = (props) => {
  // Access via props.xxx to maintain reactivity
  return <text>{props.value}</text>;
};
```

## 4. String Width Issues

**Problem**: Layout breaks with emoji or CJK characters.

**Cause**: Terminal columns count by display width, not character count. Emoji and CJK characters take 2 columns.

**Solution**:
```typescript
import stringWidth from "string-width";

const truncateToWidth = (str: string, maxWidth: number): string => {
  let width = 0;
  let result = "";
  for (const char of str) {
    const charWidth = stringWidth(char);
    if (width + charWidth > maxWidth) break;
    result += char;
    width += charWidth;
  }
  return result;
};
```

## 5. createEffect Cleanup

**Problem**: Memory leaks or stale timers after component unmount.

**Cause**: Missing cleanup in createEffect.

**Wrong**:
```typescript
createEffect(() => {
  const timer = setInterval(() => {
    // Update something
  }, 1000);
  // Timer never cleared!
});
```

**Correct**:
```typescript
createEffect(() => {
  const timer = setInterval(() => {
    // Update something
  }, 1000);
  onCleanup(() => clearInterval(timer));
});
```

## 6. Conditional Rendering with Signals

**Problem**: Component crashes when accessing properties of undefined signal value.

**Cause**: Signal might be undefined during render.

**Wrong**:
```typescript
const [data, setData] = createSignal<Data | undefined>();

return <text>{data().name}</text>; // Crash if undefined!
```

**Correct**:
```typescript
return <Show when={data()}>{(d) => <text>{d().name}</text>}</Show>;
```

## 7. Focus Management

**Problem**: Multiple components respond to keyboard input simultaneously.

**Cause**: No centralized focus management.

**Solution**:
```typescript
const [focusedId, setFocusedId] = createSignal<string>("list");

const FocusableComponent = (props) => {
  const isFocused = () => focusedId() === props.id;

  useKeyboard((key) => {
    if (!isFocused()) return;
    // Handle input only when focused
  });

  return (
    <box borderStyle={isFocused() ? "single" : "round"}>
      {props.children}
    </box>
  );
};
```

## 8. Async Operations in Effects

**Problem**: State updates after component unmount cause errors.

**Cause**: Async operation completes after cleanup.

**Solution**:
```typescript
createEffect(() => {
  let cancelled = false;

  fetchData().then((result) => {
    if (!cancelled) {
      setData(result);
    }
  });

  onCleanup(() => {
    cancelled = true;
  });
});
```

## 9. Box Sizing and Flexbox

**Problem**: Components don't size as expected.

**Cause**: Missing flex properties or incorrect box model understanding.

**Tips**:
```typescript
// Fill available space
<box flexGrow={1}>Content</box>

// Fixed size
<box width={20} height={5}>Content</box>

// Percentage (of parent)
<box width="50%">Content</box>
```

## 10. Terminal Resize Handling

**Problem**: Layout breaks on terminal resize.

**Solution**:
```typescript
import { useStdout } from "@opentui/core";

const ResponsiveComponent = () => {
  const { columns, rows } = useStdout();

  return (
    <box width={columns()} height={rows()}>
      {/* Content adapts to terminal size */}
    </box>
  );
};
```

## 11. ASCII-Only Icons

**Problem**: Icons display incorrectly in some terminals.

**Cause**: Emoji and special Unicode characters have inconsistent rendering.

**Best Practice**:
```typescript
// Good - ASCII only
const ICONS = {
  check: "[x]",
  uncheck: "[ ]",
  arrow: "->",
  bullet: "*",
};

// Avoid
const BAD_ICONS = {
  check: "✓",      // May not render
  uncheck: "☐",    // Width issues
  arrow: "→",      // Inconsistent
};
```

## 12. Color Support Detection

**Problem**: Colors don't display in some terminals.

**Solution**:
```typescript
import supportsColor from "supports-color";

const useColors = () => {
  return supportsColor.stdout !== false;
};

const ColoredText = (props) => {
  const colors = useColors();

  return (
    <text color={colors ? props.color : undefined}>
      {props.children}
    </text>
  );
};
```
