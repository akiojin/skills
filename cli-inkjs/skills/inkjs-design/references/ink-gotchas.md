# Ink.js Gotchas and Common Issues

## Table of Contents

1. [Icon/Emoji Width Issues](#1-iconemoji-width-issues)
2. [Ctrl+C Handling](#2-ctrlc-handling)
3. [useInput Conflicts](#3-useinput-conflicts)
4. [Enter Double-Press Issue](#4-enter-double-press-issue)
5. [CJK Cursor Movement](#5-cjk-cursor-movement)
6. [Layout Breaking](#6-layout-breaking)
7. [console.log Interference](#7-consolelog-interference)
8. [Ink Color Type Errors](#8-ink-color-type-errors)
9. [Rendering Flicker Issues](#9-rendering-flicker-issues)

## 1. Icon/Emoji Width Issues

### Problem

Since `string-width` v8+, handling of Variation Selectors (VS15/VS16) has changed, causing emoji width calculations to be off. In terminals, emojis may take up 2 character widths, but `stringWidth()` may return 1.

### Solution: WIDTH_OVERRIDES Pattern

```typescript
import stringWidth from "string-width";

const iconWidthOverrides: Record<string, number> = {
  "⚡": 1,
  "✨": 1,
  "🐛": 1,
  "🔥": 1,
  "🚀": 1,
  "📌": 1,
  "🟢": 1,
  "🟠": 1,
  "👉": 1,
  "💾": 1,
  "📤": 1,
  "🔃": 1,
  "✅": 1,
  "⚠️": 1,
  "🔗": 1,
  "💻": 1,
  "☁️": 1,
};

const getIconWidth = (icon: string): number => {
  const baseWidth = stringWidth(icon);
  const override = iconWidthOverrides[icon];
  return override !== undefined ? Math.max(baseWidth, override) : baseWidth;
};

// Padding for fixed-width columns
const COLUMN_WIDTH = 2;
const padIcon = (icon: string): string => {
  const width = getIconWidth(icon);
  const padding = Math.max(0, COLUMN_WIDTH - width);
  return icon + " ".repeat(padding);
};
```

### Affected Areas

- Branch list display
- Status icon display
- Progress bars
- All fixed-width columns

## 2. Ctrl+C Handling

### Problem

Ink handles `Ctrl+C` by default to exit the app, but this can fire twice or conflict with custom exit processing.

### Solution: exitOnCtrlC: false + useInput + sigint

```typescript
import { render, useApp, useInput } from "ink";
import process from "node:process";

function App({ onExit }: { onExit: () => void }) {
  const { exit } = useApp();

  useInput((input, key) => {
    if (key.ctrl && input === "c") {
      onExit();
      exit();
    }
  });

  // Also handle SIGINT (another signal for Ctrl+C)
  useEffect(() => {
    const handler = () => {
      onExit();
      exit();
    };
    process.on("SIGINT", handler);
    return () => process.off("SIGINT", handler);
  }, [exit, onExit]);

  return <Box>...</Box>;
}

// Disable exitOnCtrlC when rendering
render(<App onExit={cleanup} />, { exitOnCtrlC: false });
```

## 3. useInput Conflicts

### Problem

When multiple `useInput` hooks exist, **all handlers are called**. Both parent and child components process the same key.

### Solution

**Pattern 1: disabled prop**

```typescript
useInput((input, key) => {
  if (disabled) return;  // Do nothing when disabled
  // Key processing...
});
```

**Pattern 2: Mode flag**

```typescript
const [filterMode, setFilterMode] = useState(false);

useInput((input, key) => {
  if (filterMode) return;  // Disable global shortcuts during filter input
  if (input === "c") onCleanupCommand?.();
});
```

**Pattern 3: blockKeys prop (Input component)**

```typescript
// Inside Input component
useInput((input) => {
  if (blockKeys && blockKeys.includes(input)) {
    return;  // Consume key without propagating to parent
  }
});

// Usage
<Input blockKeys={["c", "r", "f"]} ... />
```

## 4. Enter Double-Press Issue

### Problem

After selecting in a Select component, Enter also responds on the next screen. The initial Enter event propagates.

### Solution: Initial Acceptance Buffering

```typescript
const [ready, setReady] = useState(false);

useEffect(() => {
  // Set ready to true after first render
  const timer = setTimeout(() => setReady(true), 50);
  return () => clearTimeout(timer);
}, []);

useInput((input, key) => {
  if (!ready) return;  // Ignore input until initialization complete
  if (key.return) {
    onSelect(selectedItem);
  }
});
```

## 5. CJK Cursor Movement

### Problem

CJK characters have display width of 2 but character count of 1. When calculating cursor position by character count, the cursor shifts during CJK input.

### Solution: Display Width-Based Position Calculation

```typescript
// Get character display width
function getCharWidth(char: string): number {
  const code = char.codePointAt(0);
  if (!code) return 1;

  // CJK characters, emojis, etc. have width 2
  if (
    (code >= 0x1100 && code <= 0x115F) ||  // Hangul
    (code >= 0x2E80 && code <= 0x9FFF) ||  // CJK
    (code >= 0xAC00 && code <= 0xD7AF) ||  // Hangul
    (code >= 0xF900 && code <= 0xFAFF) ||  // CJK Compatibility
    (code >= 0xFE10 && code <= 0xFE1F) ||  // Vertical Forms
    (code >= 0x1F300 && code <= 0x1F9FF)   // Emojis
  ) {
    return 2;
  }
  return 1;
}

// Calculate string display width
function getDisplayWidth(str: string): number {
  return [...str].reduce((width, char) => width + getCharWidth(char), 0);
}

// Convert cursor position (character count) to display width
function toDisplayColumn(text: string, charPosition: number): number {
  return getDisplayWidth(text.slice(0, charPosition));
}
```

## 6. Layout Breaking

### Problem

- Long lines wrap beyond terminal width
- Timestamps get pushed to the next line
- Column positions shift

### Solution

**Use safe margins**

```typescript
const { stdout } = useStdout();
const columns = Math.max(20, (stdout?.columns ?? 80) - 1);  // 1 character margin

// Or use 90% width (Gemini CLI pattern)
const safeColumns = Math.floor(columns * 0.9);
```

**Use fixed-width columns**

```typescript
const COLUMN_WIDTH = 2;
const padToWidth = (content: string, width: number): string => {
  const actualWidth = stringWidth(content);
  const padding = Math.max(0, width - actualWidth);
  return content + " ".repeat(padding);
};
```

**Truncate strings**

```typescript
function truncateToWidth(text: string, maxWidth: number): string {
  let width = 0;
  let result = "";
  for (const char of text) {
    const charWidth = getCharWidth(char);
    if (width + charWidth > maxWidth) {
      return result + "…";
    }
    width += charWidth;
    result += char;
  }
  return result;
}
```

## 7. console.log Interference

### Problem

`console.log` output conflicts with Ink's rendering, breaking the UI. Ink periodically re-renders the screen, overwriting log output.

### Solution

**Output structured logs to a separate stream**

```typescript
import pino from "pino";

// Output logs to stderr
const logger = pino({
  transport: {
    target: "pino-pretty",
    options: { destination: 2 }  // stderr
  }
});
```

**Conditional logging**

```typescript
if (process.env.DEBUG) {
  console.error("Debug:", data);  // Use stderr
}
```

## 8. Ink Color Type Errors

### Problem

Using `<Text color="cyan">` etc. in TypeScript may cause errors due to strict color types.

### Solution

```typescript
// Use type assertion
<Text color={"cyan" as const}>...</Text>

// Or define as variable
const selectedColor = "cyan" as const;
<Text color={selectedColor}>...</Text>

// Or use chalk
import chalk from "chalk";
<Text>{chalk.cyan("Selected")}</Text>
```

## 9. Rendering Flicker Issues

### Problem

Screen flickers during cursor movement (up/down keys) in list display. Especially noticeable during spinner animations.

### Causes

1. **React re-render chains**: State changes cause all components to re-render
2. **Insufficient component memoization**: Parent updates propagate to child components
3. **useCallback dependency array issues**: Functions regenerate when frequently updated values (like spinner frames) are included
4. **Spinner updates (every 120ms)**: All rows get re-rendered

### Solutions

**1. Memoize components**

```typescript
// Memoize entire screen to prevent unnecessary re-renders from parent
export const MyScreen = React.memo(function MyScreen({
  items,
  onSelect,
}: MyScreenProps) {
  // ...
});
```

**2. Disable patchConsole in render options**

```typescript
import { render } from "ink";

render(<App />, {
  stdin: process.stdin,
  stdout: process.stdout,
  stderr: process.stderr,
  patchConsole: false,  // Prevent console.log from affecting UI
});
```

**3. Isolate spinner dependencies**

Remove spinner frames from useCallback dependency arrays and use static icons where animation is needed.

```typescript
// Bad: Function regenerates every 120ms when spinnerFrame is in dependencies
const renderRow = useCallback(
  (item) => {
    const icon = item.isSpinning ? spinnerFrame : item.icon;
    return <Text>{icon} {item.name}</Text>;
  },
  [spinnerFrame]  // ← This is the problem
);

// Good: Use static icon to remove spinnerFrame dependency
const renderRow = useCallback(
  (item) => {
    // Use static first frame "⠋" during spinning
    const icon = item.isSpinning ? "⠋" : item.icon;
    return <Text>{icon} {item.name}</Text>;
  },
  []  // ← No dependencies, no regeneration
);
```

**4. Componentize each row (more fundamental solution)**

By making each row a separate memoized component, only necessary rows are re-rendered.

```typescript
const RowComponent = React.memo(function RowComponent({
  item,
  isSelected,
}: RowProps) {
  // Only rows that need spinners use useSpinnerFrame internally
  const spinnerFrame = useSpinnerFrame(item.isSpinning);

  return (
    <Box>
      <Text>{item.isSpinning ? spinnerFrame : item.icon}</Text>
      <Text>{item.name}</Text>
    </Box>
  );
}, (prev, next) => {
  // Custom comparison function for re-rendering only when needed
  return prev.isSelected === next.isSelected &&
         prev.item.isSpinning === next.item.isSpinning;
});
```

### Priority

1. **Component memoization** - Easiest and most effective
2. **Spinner dependency isolation** - Fundamental solution
3. **patchConsole: false** - Secondary effect
