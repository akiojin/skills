# Performance Optimization

Patterns for optimizing Ink.js application performance.

## Overview

CLI applications need to feel responsive. Unlike web apps, terminal UIs have limited refresh capabilities and can appear sluggish if not optimized properly. This guide covers React optimization techniques specific to Ink.js.

## React.memo with Custom Comparator

### Basic React.memo

```typescript
import React from "react";
import { Text } from "ink";

interface ItemProps {
  name: string;
  selected: boolean;
}

const ItemComponent: React.FC<ItemProps> = ({ name, selected }) => (
  <Text inverse={selected}>{name}</Text>
);

// Shallow comparison - good for primitive props
export const Item = React.memo(ItemComponent);
```

### Custom Comparison Function

For complex props, implement custom comparison:

```typescript
interface ListItemProps<T> {
  item: T;
  index: number;
  selected: boolean;
  renderItem?: (item: T) => React.ReactNode;
}

function arePropsEqual<T>(
  prev: ListItemProps<T>,
  next: ListItemProps<T>
): boolean {
  // Compare by identity for selected state
  if (prev.selected !== next.selected) return false;

  // Compare item by value (not reference)
  if (prev.item !== next.item) {
    // Deep comparison if needed
    if (JSON.stringify(prev.item) !== JSON.stringify(next.item)) {
      return false;
    }
  }

  // Ignore renderItem changes (function identity)
  return true;
}

export const ListItem = React.memo(ListItemComponent, arePropsEqual);
```

### Select Component Optimization

```typescript
interface SelectProps<T> {
  items: Array<{ label: string; value: T }>;
  selectedIndex: number;
  onChange: (index: number) => void;
}

function areSelectPropsEqual<T>(
  prev: SelectProps<T>,
  next: SelectProps<T>
): boolean {
  // Check array length first (fast path)
  if (prev.items.length !== next.items.length) return false;

  // Compare selected index
  if (prev.selectedIndex !== next.selectedIndex) return false;

  // Compare items by value
  for (let i = 0; i < prev.items.length; i++) {
    if (prev.items[i].value !== next.items[i].value) return false;
    if (prev.items[i].label !== next.items[i].label) return false;
  }

  // Ignore onChange (function identity changes on every render)
  return true;
}

export const Select = React.memo(SelectComponent, areSelectPropsEqual);
```

## useMemo and useCallback

### Memoize Expensive Computations

```typescript
import { useMemo } from "react";

const filteredItems = useMemo(() => {
  return items
    .filter(item => item.name.includes(searchTerm))
    .sort((a, b) => a.name.localeCompare(b.name));
}, [items, searchTerm]);
```

### Memoize Callbacks

```typescript
import { useCallback } from "react";

const handleSelect = useCallback((index: number) => {
  setSelectedIndex(index);
  onSelect?.(items[index]);
}, [items, onSelect]);
```

### Content-Based Dependencies

Use computed values for dependency arrays:

```typescript
// Bad: Reference changes every render
const items = fetchedData.map(d => ({ label: d.name, value: d.id }));

// Good: Use content-based hash
const itemsHash = useMemo(
  () => fetchedData.map(d => `${d.id}:${d.name}`).join("|"),
  [fetchedData]
);

const processedItems = useMemo(() => {
  return fetchedData.map(d => ({
    label: d.name,
    value: d.id,
    processed: true,
  }));
}, [itemsHash]); // Stable dependency
```

## Spinner Dependency Isolation

Prevent spinners from causing parent re-renders:

### Problem

```typescript
// Bad: Parent re-renders every 80ms
const Parent: React.FC = () => {
  const [frameIndex, setFrameIndex] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      setFrameIndex(i => (i + 1) % FRAMES.length);
    }, 80);
    return () => clearInterval(timer);
  }, []);

  return (
    <Box>
      <Text>{FRAMES[frameIndex]}</Text>
      <ExpensiveComponent /> {/* Re-renders every 80ms! */}
    </Box>
  );
};
```

### Solution

```typescript
// Good: Isolate spinner state
const Spinner: React.FC = () => {
  const [frameIndex, setFrameIndex] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      setFrameIndex(i => (i + 1) % FRAMES.length);
    }, 80);
    return () => clearInterval(timer);
  }, []);

  return <Text>{FRAMES[frameIndex]}</Text>;
};

const Parent: React.FC = () => {
  return (
    <Box>
      <Spinner />
      <ExpensiveComponent /> {/* Stable - no re-renders */}
    </Box>
  );
};
```

## Row-Level Componentization

### Problem: Full List Re-render

```typescript
// Bad: Entire list re-renders on selection change
const List: React.FC<{ items: Item[]; selected: number }> = ({
  items,
  selected,
}) => (
  <Box flexDirection="column">
    {items.map((item, i) => (
      <Text key={item.id} inverse={i === selected}>
        {item.name}
      </Text>
    ))}
  </Box>
);
```

### Solution: Memoized Row Components

```typescript
// Good: Only affected rows re-render
const ListRow = React.memo<{ item: Item; selected: boolean }>(
  ({ item, selected }) => (
    <Text inverse={selected}>{item.name}</Text>
  )
);

const List: React.FC<{ items: Item[]; selected: number }> = ({
  items,
  selected,
}) => (
  <Box flexDirection="column">
    {items.map((item, i) => (
      <ListRow key={item.id} item={item} selected={i === selected} />
    ))}
  </Box>
);
```

## Virtualization for Long Lists

Only render visible items:

```typescript
interface VirtualListProps<T> {
  items: T[];
  visibleCount: number;
  selectedIndex: number;
  renderItem: (item: T, index: number, selected: boolean) => React.ReactNode;
}

export function VirtualList<T>({
  items,
  visibleCount,
  selectedIndex,
  renderItem,
}: VirtualListProps<T>) {
  // Calculate visible window
  const halfVisible = Math.floor(visibleCount / 2);
  let startIndex = Math.max(0, selectedIndex - halfVisible);
  const endIndex = Math.min(items.length, startIndex + visibleCount);

  if (endIndex - startIndex < visibleCount) {
    startIndex = Math.max(0, endIndex - visibleCount);
  }

  const visibleItems = items.slice(startIndex, endIndex);

  return (
    <Box flexDirection="column" height={visibleCount}>
      {visibleItems.map((item, i) => {
        const actualIndex = startIndex + i;
        return renderItem(item, actualIndex, actualIndex === selectedIndex);
      })}
    </Box>
  );
}
```

## Debouncing State Updates

Prevent rapid state changes:

```typescript
import { useState, useRef, useCallback } from "react";

function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);
  const timeoutRef = useRef<NodeJS.Timeout>();

  useEffect(() => {
    timeoutRef.current = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }
    };
  }, [value, delay]);

  return debouncedValue;
}

// Usage: Debounce search input
const [searchTerm, setSearchTerm] = useState("");
const debouncedSearch = useDebounce(searchTerm, 300);

const filteredItems = useMemo(
  () => items.filter(i => i.name.includes(debouncedSearch)),
  [items, debouncedSearch]
);
```

## Profiling and Debugging

### React DevTools

Enable React DevTools for Ink:

```bash
DEV=true node ./dist/cli.js
```

### Render Counting

Track re-renders during development:

```typescript
const renderCount = useRef(0);
renderCount.current++;

useEffect(() => {
  console.log(`Component rendered ${renderCount.current} times`);
});
```

## Best Practices

1. **Use React.memo for list items** - Prevent re-renders on selection change
2. **Implement custom comparison** - Skip function props, compare data by value
3. **Isolate animation state** - Keep spinners in separate components
4. **Use content-based hashes** - Stable dependency arrays for useMemo
5. **Virtualize long lists** - Only render visible items
6. **Debounce rapid updates** - Prevent UI jitter from fast input
7. **Profile before optimizing** - Use React DevTools to find bottlenecks
8. **Memoize expensive computations** - Sorting, filtering, transformations
