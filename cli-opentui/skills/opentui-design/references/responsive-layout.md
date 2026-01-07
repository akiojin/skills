# Responsive Layout in OpenTUI

Patterns for building adaptive CLI layouts that work across terminal sizes.

## Overview

Terminal dimensions vary widely. Applications should adapt to different sizes gracefully.

## Getting Terminal Dimensions

```typescript
import { useStdout } from "@opentui/core";

const ResponsiveComponent = () => {
  const { columns, rows } = useStdout();

  return (
    <box>
      <text>
        Terminal: {columns()} x {rows()}
      </text>
    </box>
  );
};
```

## Flexbox Layout

OpenTUI uses flexbox-like layout system.

### Basic Flex Container

```typescript
// Horizontal layout (default)
<box flexDirection="row">
  <text>Left</text>
  <text>Right</text>
</box>

// Vertical layout
<box flexDirection="column">
  <text>Top</text>
  <text>Bottom</text>
</box>
```

### Flex Grow

```typescript
// Fill available space
<box flexDirection="row" width="100%">
  <box width={20}>Sidebar</box>
  <box flexGrow={1}>Main Content (fills remaining)</box>
</box>
```

### Flex Shrink

```typescript
// Allow shrinking when space is limited
<box flexDirection="row" width="100%">
  <box flexShrink={0} width={20}>Fixed Sidebar</box>
  <box flexShrink={1} flexGrow={1}>Shrinkable Content</box>
</box>
```

## Responsive Breakpoints

```typescript
const useBreakpoint = () => {
  const { columns } = useStdout();

  const breakpoint = createMemo(() => {
    const width = columns();
    if (width < 40) return "small";
    if (width < 80) return "medium";
    return "large";
  });

  return breakpoint;
};

const ResponsiveLayout = () => {
  const breakpoint = useBreakpoint();

  return (
    <Switch>
      <Match when={breakpoint() === "small"}>
        <CompactLayout />
      </Match>
      <Match when={breakpoint() === "medium"}>
        <StandardLayout />
      </Match>
      <Match when={breakpoint() === "large"}>
        <WideLayout />
      </Match>
    </Switch>
  );
};
```

## Percentage-Based Sizing

```typescript
// Percentage of parent
<box width="100%" height="50%">
  Content takes half height
</box>

// Minimum/maximum with percentage
const ResponsivePanel = () => {
  const { columns } = useStdout();
  const width = createMemo(() => Math.min(Math.max(columns() * 0.8, 40), 120));

  return <box width={width()}>Responsive width panel</box>;
};
```

## Text Truncation

```typescript
import stringWidth from "string-width";

const truncateText = (text: string, maxWidth: number): string => {
  let width = 0;
  let result = "";

  for (const char of text) {
    const charWidth = stringWidth(char);
    if (width + charWidth > maxWidth - 3) {
      return result + "...";
    }
    result += char;
    width += charWidth;
  }

  return result;
};

const TruncatedText = (props: { text: string; maxWidth: number }) => (
  <text>{truncateText(props.text, props.maxWidth)}</text>
);
```

## Multi-Column Layout

```typescript
const ColumnLayout = (props: { items: string[] }) => {
  const { columns } = useStdout();

  const columnCount = createMemo(() => {
    const width = columns();
    if (width < 60) return 1;
    if (width < 100) return 2;
    return 3;
  });

  const itemsPerColumn = createMemo(() =>
    Math.ceil(props.items.length / columnCount())
  );

  const columnItems = createMemo(() => {
    const result: string[][] = [];
    for (let i = 0; i < columnCount(); i++) {
      const start = i * itemsPerColumn();
      result.push(props.items.slice(start, start + itemsPerColumn()));
    }
    return result;
  });

  return (
    <box flexDirection="row">
      <For each={columnItems()}>
        {(items) => (
          <box flexGrow={1} flexDirection="column">
            <For each={items}>{(item) => <text>{item}</text>}</For>
          </box>
        )}
      </For>
    </box>
  );
};
```

## Scrollable Content

```typescript
const ScrollableBox = (props: {
  content: string;
  height: number;
}) => {
  const [scrollOffset, setScrollOffset] = createSignal(0);
  const lines = createMemo(() => props.content.split("\n"));

  const visibleLines = createMemo(() => {
    return lines().slice(scrollOffset(), scrollOffset() + props.height);
  });

  const canScrollUp = () => scrollOffset() > 0;
  const canScrollDown = () => scrollOffset() + props.height < lines().length;

  useKeyboard((key) => {
    if (key.name === "up" && canScrollUp()) {
      setScrollOffset((prev) => prev - 1);
    }
    if (key.name === "down" && canScrollDown()) {
      setScrollOffset((prev) => prev + 1);
    }
    if (key.name === "pageup") {
      setScrollOffset((prev) => Math.max(0, prev - props.height));
    }
    if (key.name === "pagedown") {
      setScrollOffset((prev) =>
        Math.min(lines().length - props.height, prev + props.height)
      );
    }
  });

  return (
    <box flexDirection="column" height={props.height}>
      <For each={visibleLines()}>{(line) => <text>{line}</text>}</For>
      <text dimColor>
        {scrollOffset() + 1}-{Math.min(scrollOffset() + props.height, lines().length)} of {lines().length}
      </text>
    </box>
  );
};
```

## Centering Content

```typescript
// Horizontal center
<box justifyContent="center">
  <text>Centered</text>
</box>

// Vertical center
<box alignItems="center" height="100%">
  <text>Vertically centered</text>
</box>

// Both
<box justifyContent="center" alignItems="center" width="100%" height="100%">
  <text>Fully centered</text>
</box>
```

## Border and Padding

```typescript
<box
  borderStyle="single"
  paddingX={1}
  paddingY={0}
>
  <text>Content with border</text>
</box>

// Border styles: "single", "double", "round", "bold", "singleDouble", "doubleSingle", "classic"
```

## Adaptive Header/Footer

```typescript
const AppLayout = (props: { children: JSX.Element }) => {
  const { rows } = useStdout();

  // Reserve space for header and footer
  const contentHeight = createMemo(() => rows() - 4);

  return (
    <box flexDirection="column" height="100%">
      <Header />
      <box flexGrow={1} height={contentHeight()}>
        {props.children}
      </box>
      <Footer />
    </box>
  );
};

const Header = () => (
  <box borderStyle="single" height={1} paddingX={1}>
    <text bold>Application Title</text>
  </box>
);

const Footer = () => (
  <box borderStyle="single" height={1} paddingX={1}>
    <text dimColor>Press ? for help</text>
  </box>
);
```

## Sidebar Layout

```typescript
const SidebarLayout = () => {
  const { columns } = useStdout();
  const showSidebar = () => columns() > 60;
  const sidebarWidth = () => Math.min(30, Math.floor(columns() * 0.25));

  return (
    <box flexDirection="row" width="100%">
      <Show when={showSidebar()}>
        <box width={sidebarWidth()} borderStyle="single">
          <Sidebar />
        </box>
      </Show>
      <box flexGrow={1}>
        <MainContent />
      </box>
    </box>
  );
};
```

## Best Practices

1. **Test at multiple sizes** - Check 80x24, 120x40, and smaller
2. **Set minimum dimensions** - Gracefully handle tiny terminals
3. **Use flexGrow for dynamic areas** - Fixed headers, flexible content
4. **Handle text overflow** - Truncate or wrap long text
5. **Provide keyboard navigation** - For scrollable areas
6. **Show dimension feedback** - Help users resize appropriately
7. **Use ASCII borders** - For consistent rendering
