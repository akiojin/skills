# Responsive Layout

Patterns for building terminal-responsive layouts in Ink.js applications.

## Overview

Terminal applications must adapt to varying terminal sizes. Unlike web browsers, terminals have strict character-based dimensions that change when users resize their windows.

## useTerminalSize Hook

Ink provides `useStdout` for accessing terminal dimensions:

```typescript
import { useStdout } from "ink";

export function useTerminalSize() {
  const { stdout } = useStdout();

  return {
    columns: stdout?.columns ?? 80,
    rows: stdout?.rows ?? 24,
  };
}
```

### Fallback for Non-TTY

Handle environments without TTY (e.g., CI, piped output):

```typescript
export function useTerminalSize(defaults = { columns: 80, rows: 24 }) {
  const { stdout } = useStdout();

  if (!stdout?.isTTY) {
    return defaults;
  }

  return {
    columns: stdout.columns ?? defaults.columns,
    rows: stdout.rows ?? defaults.rows,
  };
}
```

## Dynamic Height Calculation

Calculate content area height:

```typescript
const HEADER_LINES = 3;  // Logo + title + blank
const FOOTER_LINES = 2;  // Help text + status

export const MainScreen: React.FC = () => {
  const { rows } = useTerminalSize();

  const contentHeight = Math.max(5, rows - HEADER_LINES - FOOTER_LINES);

  return (
    <Box flexDirection="column" height={rows}>
      <Header />
      <Box height={contentHeight} flexDirection="column">
        {/* Scrollable content */}
      </Box>
      <Footer />
    </Box>
  );
};
```

## Character Width Handling

### string-width Issues

The `string-width` library (v8+) sometimes miscalculates emoji widths:

```typescript
import stringWidth from "string-width";

// Problem: Some emojis report incorrect widths
stringWidth("🔥"); // May return 1 instead of 2
```

### WIDTH_OVERRIDES Pattern

Create a lookup table for known problematic characters:

```typescript
const WIDTH_OVERRIDES: Record<string, number> = {
  // Status icons
  "⚡": 1,
  "✨": 1,
  "🐛": 1,
  "🔥": 1,
  "🚀": 1,
  // Indicators
  "🟢": 1,
  "🟠": 1,
  "🔴": 1,
  "✅": 1,
  "❌": 1,
  "⚠️": 1,
  // Arrows
  "→": 1,
  "←": 1,
  "↑": 1,
  "↓": 1,
};

export function getDisplayWidth(text: string): number {
  let width = 0;
  for (const char of text) {
    const override = WIDTH_OVERRIDES[char];
    width += override ?? stringWidth(char);
  }
  return width;
}
```

### Icon Width Helper

```typescript
export function getIconWidth(icon: string): number {
  const baseWidth = stringWidth(icon);
  const override = WIDTH_OVERRIDES[icon];
  return override !== undefined ? Math.max(baseWidth, override) : baseWidth;
}
```

## Text Truncation

### truncateToWidth Function

Truncate text to fit within a specified width:

```typescript
export function truncateToWidth(
  text: string,
  maxWidth: number,
  suffix = "…"
): string {
  const textWidth = getDisplayWidth(text);

  if (textWidth <= maxWidth) {
    return text;
  }

  const suffixWidth = getDisplayWidth(suffix);
  const targetWidth = maxWidth - suffixWidth;

  if (targetWidth <= 0) {
    return suffix.slice(0, maxWidth);
  }

  let result = "";
  let currentWidth = 0;

  for (const char of text) {
    const charWidth = WIDTH_OVERRIDES[char] ?? stringWidth(char);
    if (currentWidth + charWidth > targetWidth) {
      break;
    }
    result += char;
    currentWidth += charWidth;
  }

  return result + suffix;
}
```

### TruncatedText Component

```typescript
import React from "react";
import { Text } from "ink";

interface TruncatedTextProps {
  children: string;
  maxWidth: number;
  suffix?: string;
}

export const TruncatedText: React.FC<TruncatedTextProps> = ({
  children,
  maxWidth,
  suffix = "…",
}) => {
  const truncated = truncateToWidth(children, maxWidth, suffix);
  return <Text>{truncated}</Text>;
};
```

## Safe Margins

Leave margin for terminal edges:

```typescript
const SAFE_MARGIN = 2; // Characters from edge

export const ResponsiveBox: React.FC<{ children: React.ReactNode }> = ({
  children,
}) => {
  const { columns } = useTerminalSize();
  const safeWidth = columns - SAFE_MARGIN * 2;

  return (
    <Box width={safeWidth} marginLeft={SAFE_MARGIN}>
      {children}
    </Box>
  );
};
```

## Responsive Column Layouts

### Fixed + Flexible Columns

```typescript
const ICON_WIDTH = 3;
const STATUS_WIDTH = 10;

export const ListItem: React.FC<{
  icon: string;
  name: string;
  status: string;
}> = ({ icon, name, status }) => {
  const { columns } = useTerminalSize();
  const nameWidth = columns - ICON_WIDTH - STATUS_WIDTH - 4; // 4 for padding

  return (
    <Box>
      <Box width={ICON_WIDTH}>
        <Text>{icon}</Text>
      </Box>
      <Box width={nameWidth}>
        <TruncatedText maxWidth={nameWidth}>{name}</TruncatedText>
      </Box>
      <Box width={STATUS_WIDTH}>
        <Text dimColor>{status}</Text>
      </Box>
    </Box>
  );
};
```

### Breakpoint-Based Layout

```typescript
type LayoutMode = "compact" | "normal" | "wide";

function getLayoutMode(columns: number): LayoutMode {
  if (columns < 80) return "compact";
  if (columns < 120) return "normal";
  return "wide";
}

export const AdaptiveLayout: React.FC = () => {
  const { columns } = useTerminalSize();
  const mode = getLayoutMode(columns);

  return (
    <Box flexDirection={mode === "compact" ? "column" : "row"}>
      <MainPanel flex={mode === "wide" ? 2 : 1} />
      {mode !== "compact" && <SidePanel flex={1} />}
    </Box>
  );
};
```

## Scrollable Lists

### Visible Items Calculation

```typescript
const ITEMS_PER_ROW = 1;
const MIN_VISIBLE = 5;

export const ScrollableList: React.FC<{
  items: string[];
  selectedIndex: number;
}> = ({ items, selectedIndex }) => {
  const { rows } = useTerminalSize();
  const availableRows = rows - HEADER_LINES - FOOTER_LINES;
  const visibleCount = Math.max(MIN_VISIBLE, availableRows);

  // Calculate scroll offset
  const halfVisible = Math.floor(visibleCount / 2);
  let startIndex = Math.max(0, selectedIndex - halfVisible);
  const endIndex = Math.min(items.length, startIndex + visibleCount);

  // Adjust if at end of list
  if (endIndex - startIndex < visibleCount) {
    startIndex = Math.max(0, endIndex - visibleCount);
  }

  const visibleItems = items.slice(startIndex, endIndex);

  return (
    <Box flexDirection="column" height={visibleCount}>
      {visibleItems.map((item, i) => (
        <Text
          key={startIndex + i}
          inverse={startIndex + i === selectedIndex}
        >
          {item}
        </Text>
      ))}
    </Box>
  );
};
```

## Best Practices

1. **Always provide fallback dimensions** - Handle non-TTY environments
2. **Use WIDTH_OVERRIDES for emojis** - Fix string-width miscalculations
3. **Leave safe margins** - Content shouldn't touch terminal edges
4. **Calculate fixed vs. flexible widths** - Icons/status fixed, names flexible
5. **Truncate long text** - Prevent layout breaking from long strings
6. **Use breakpoints for major layout changes** - Adapt to available space
7. **Keep minimum dimensions** - Ensure usability on small terminals
8. **Test at various sizes** - Resize terminal during development
