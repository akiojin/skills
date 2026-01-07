# Performance Optimization in OpenTUI

Techniques for building fast and responsive CLI applications.

## Overview

Terminal UI performance is critical for user experience. This guide covers optimization techniques for OpenTUI applications.

## Signal Optimization

### Avoid Unnecessary Re-renders

```typescript
// Bad: Creates new object every render
const [config, setConfig] = createSignal({ theme: "dark", size: "medium" });

// To update theme:
setConfig({ ...config(), theme: "light" }); // Entire config object changes

// Good: Granular signals
const [theme, setTheme] = createSignal("dark");
const [size, setSize] = createSignal("medium");

// Only theme signal updates
setTheme("light");
```

### Use createStore for Complex State

```typescript
import { createStore } from "solid-js/store";

// Better for nested objects
const [state, setState] = createStore({
  user: { name: "John", settings: { theme: "dark" } },
  items: [],
});

// Granular update - only affected parts re-render
setState("user", "settings", "theme", "light");
```

### Memoize Expensive Computations

```typescript
import { createMemo } from "solid-js";

// Bad: Recomputes on every access
const filteredItems = () =>
  items().filter((item) => item.name.includes(searchTerm()));

// Good: Only recomputes when dependencies change
const filteredItems = createMemo(() =>
  items().filter((item) => item.name.includes(searchTerm()))
);
```

## List Rendering

### Virtualization for Long Lists

```typescript
const VirtualList = (props: { items: string[]; height: number }) => {
  const [scrollOffset, setScrollOffset] = createSignal(0);

  const visibleItems = createMemo(() => {
    const start = scrollOffset();
    const end = start + props.height;
    return props.items.slice(start, end);
  });

  useKeyboard((key) => {
    if (key.name === "up") {
      setScrollOffset((prev) => Math.max(0, prev - 1));
    }
    if (key.name === "down") {
      setScrollOffset((prev) =>
        Math.min(props.items.length - props.height, prev + 1)
      );
    }
  });

  return (
    <box flexDirection="column" height={props.height}>
      <For each={visibleItems()}>
        {(item, index) => <text>{item}</text>}
      </For>
    </box>
  );
};
```

### Use Index for Static Lists

```typescript
import { Index } from "solid-js";

// When items don't change identity, use Index
// More efficient than For for value updates
<Index each={values()}>
  {(value, index) => <text>{value()}</text>}
</Index>
```

## Batch Updates

```typescript
import { batch } from "solid-js";

// Bad: 3 separate re-renders
setA(1);
setB(2);
setC(3);

// Good: 1 re-render
batch(() => {
  setA(1);
  setB(2);
  setC(3);
});
```

## Async Operations

### Debounce Input

```typescript
const useDebounce = <T>(signal: Accessor<T>, delay: number) => {
  const [debounced, setDebounced] = createSignal(signal());

  createEffect(() => {
    const value = signal();
    const timer = setTimeout(() => setDebounced(() => value), delay);
    onCleanup(() => clearTimeout(timer));
  });

  return debounced;
};

// Usage for search
const SearchInput = () => {
  const [input, setInput] = createSignal("");
  const debouncedInput = useDebounce(input, 300);

  createEffect(() => {
    const query = debouncedInput();
    if (query) {
      performSearch(query);
    }
  });
};
```

### Throttle Updates

```typescript
const useThrottle = <T>(signal: Accessor<T>, interval: number) => {
  const [throttled, setThrottled] = createSignal(signal());
  let lastUpdate = 0;

  createEffect(() => {
    const value = signal();
    const now = Date.now();

    if (now - lastUpdate >= interval) {
      setThrottled(() => value);
      lastUpdate = now;
    }
  });

  return throttled;
};
```

### Cancel Stale Requests

```typescript
const SearchResults = (props: { query: string }) => {
  const [results, setResults] = createSignal<string[]>([]);

  createEffect(() => {
    const query = props.query;
    let cancelled = false;

    if (query) {
      search(query).then((data) => {
        if (!cancelled) {
          setResults(data);
        }
      });
    }

    onCleanup(() => {
      cancelled = true;
    });
  });

  return <List items={results()} />;
};
```

## Render Optimization

### Lazy Components

```typescript
import { lazy, Suspense } from "solid-js";

const HeavyComponent = lazy(() => import("./HeavyComponent"));

const App = () => (
  <Suspense fallback={<text>Loading...</text>}>
    <HeavyComponent />
  </Suspense>
);
```

### Conditional Rendering

```typescript
// Good: Component unmounts when hidden
<Show when={isVisible()}>
  <HeavyComponent />
</Show>

// Alternative: Keep mounted but hide
// Use when remounting is expensive
<box display={isVisible() ? "flex" : "none"}>
  <HeavyComponent />
</box>
```

### Avoid Inline Functions

```typescript
// Bad: New function created every render
<Button onClick={() => handleClick(item.id)} />

// Good: Stable reference
const handleItemClick = (id: string) => () => handleClick(id);

// Or use data attributes
<Button data-id={item.id} onClick={handleButtonClick} />
```

## Memory Management

### Clean Up Effects

```typescript
createEffect(() => {
  const subscription = eventEmitter.on("data", handleData);

  onCleanup(() => {
    subscription.unsubscribe();
  });
});
```

### Avoid Memory Leaks in Timers

```typescript
createEffect(() => {
  const timer = setInterval(() => {
    // Update
  }, 1000);

  onCleanup(() => clearInterval(timer));
});
```

### Dispose Resources on Unmount

```typescript
const FileWatcher = (props: { path: string }) => {
  let watcher: FSWatcher | null = null;

  onMount(() => {
    watcher = fs.watch(props.path, handleChange);
  });

  onCleanup(() => {
    watcher?.close();
  });
};
```

## Profiling

### Measure Render Time

```typescript
const ProfiledComponent = (props) => {
  const start = performance.now();

  onMount(() => {
    const duration = performance.now() - start;
    console.log(`Render took ${duration}ms`);
  });

  return <box>{props.children}</box>;
};
```

### Track Signal Updates

```typescript
const createTrackedSignal = <T>(initial: T, name: string) => {
  const [value, setValue] = createSignal(initial);

  const trackedSet = (newValue: T | ((prev: T) => T)) => {
    console.log(`[${name}] Signal updated`);
    setValue(newValue as T);
  };

  return [value, trackedSet] as const;
};
```

## Best Practices Summary

1. **Use granular signals** - Avoid large object signals
2. **Memoize derived values** - Use createMemo
3. **Batch updates** - Group related state changes
4. **Virtualize long lists** - Only render visible items
5. **Debounce user input** - Reduce computation frequency
6. **Cancel stale async** - Prevent race conditions
7. **Clean up resources** - Use onCleanup consistently
8. **Profile bottlenecks** - Measure before optimizing
