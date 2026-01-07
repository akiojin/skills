# Hooks Guide for OpenTUI

Reference guide for OpenTUI hooks and SolidJS primitives.

## OpenTUI Hooks

### useKeyboard

Handle keyboard input.

```typescript
import { useKeyboard } from "@opentui/core";

useKeyboard((key) => {
  console.log("Key pressed:", key.name);
  console.log("With Ctrl:", key.ctrl);
  console.log("With Shift:", key.shift);
  console.log("Raw sequence:", key.sequence);

  // Prevent propagation to other handlers
  key.preventDefault();
});
```

### useStdout

Get terminal dimensions.

```typescript
import { useStdout } from "@opentui/core";

const MyComponent = () => {
  const { columns, rows } = useStdout();

  return (
    <box width={columns()} height={rows()}>
      Terminal size: {columns()} x {rows()}
    </box>
  );
};
```

### useApp

Access application controls.

```typescript
import { useApp } from "@opentui/core";

const MyComponent = () => {
  const { exit } = useApp();

  useKeyboard((key) => {
    if (key.name === "q") {
      exit();
    }
  });
};
```

### useFocus

Manage focus state.

```typescript
import { useFocus } from "@opentui/core";

const FocusableInput = (props: { id: string }) => {
  const { isFocused, focus } = useFocus(props.id);

  return (
    <box
      borderStyle={isFocused() ? "double" : "single"}
      onClick={() => focus()}
    >
      {isFocused() ? "Focused" : "Not focused"}
    </box>
  );
};
```

### useSelectionHandler

Handle text selection (with limitations).

```typescript
import { useSelectionHandler } from "@opentui/core";

// Note: This hook may not reliably fire
useSelectionHandler((selection) => {
  console.log("Selected text:", selection);
});
```

## SolidJS Primitives

### createSignal

Create reactive state.

```typescript
import { createSignal } from "solid-js";

const [count, setCount] = createSignal(0);

// Read value
console.log(count()); // 0

// Set value
setCount(1);
setCount((prev) => prev + 1);
```

### createEffect

Run side effects when dependencies change.

```typescript
import { createEffect, onCleanup } from "solid-js";

createEffect(() => {
  const value = someSignal(); // Dependency tracked automatically
  console.log("Value changed:", value);

  // Cleanup function
  onCleanup(() => {
    console.log("Cleaning up");
  });
});
```

### createMemo

Create derived/computed values.

```typescript
import { createMemo } from "solid-js";

const [items, setItems] = createSignal([1, 2, 3]);

const total = createMemo(() => {
  return items().reduce((sum, item) => sum + item, 0);
});

console.log(total()); // 6
```

### createResource

Handle async data fetching.

```typescript
import { createResource, Suspense } from "solid-js";

const fetchUser = async (id: string) => {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
};

const UserProfile = (props: { userId: string }) => {
  const [user] = createResource(() => props.userId, fetchUser);

  return (
    <Suspense fallback={<text>Loading...</text>}>
      <Show when={user()}>
        <text>{user()!.name}</text>
      </Show>
    </Suspense>
  );
};
```

### createStore

Manage complex state objects.

```typescript
import { createStore } from "solid-js/store";

interface AppState {
  user: { name: string; email: string } | null;
  settings: { theme: string; language: string };
}

const [state, setState] = createStore<AppState>({
  user: null,
  settings: { theme: "dark", language: "en" },
});

// Update nested properties
setState("settings", "theme", "light");
setState("user", { name: "John", email: "john@example.com" });
```

### onMount / onCleanup

Lifecycle hooks.

```typescript
import { onMount, onCleanup } from "solid-js";

const MyComponent = () => {
  onMount(() => {
    console.log("Component mounted");
    const timer = setInterval(() => {
      // Do something
    }, 1000);

    onCleanup(() => {
      console.log("Component unmounting");
      clearInterval(timer);
    });
  });

  return <text>Hello</text>;
};
```

### batch

Batch multiple updates.

```typescript
import { batch } from "solid-js";

const [a, setA] = createSignal(0);
const [b, setB] = createSignal(0);

// Without batch - triggers 2 re-renders
setA(1);
setB(2);

// With batch - triggers 1 re-render
batch(() => {
  setA(1);
  setB(2);
});
```

### untrack

Read signal without creating dependency.

```typescript
import { untrack } from "solid-js";

createEffect(() => {
  const trackedValue = trackedSignal(); // Creates dependency
  const untrackedValue = untrack(() => untrackedSignal()); // No dependency

  console.log(trackedValue, untrackedValue);
});
```

## Control Flow Components

### Show

Conditional rendering.

```typescript
import { Show } from "solid-js";

<Show when={isVisible()} fallback={<text>Hidden</text>}>
  <text>Visible</text>
</Show>;

// With callback for non-null assertion
<Show when={user()}>
  {(u) => <text>{u().name}</text>}
</Show>;
```

### For

List rendering.

```typescript
import { For } from "solid-js";

<For each={items()}>
  {(item, index) => (
    <text>
      {index()}: {item.name}
    </text>
  )}
</For>;
```

### Switch / Match

Multiple conditions.

```typescript
import { Switch, Match } from "solid-js";

<Switch fallback={<text>Unknown</text>}>
  <Match when={status() === "loading"}>
    <Spinner />
  </Match>
  <Match when={status() === "error"}>
    <text color="red">Error occurred</text>
  </Match>
  <Match when={status() === "success"}>
    <text color="green">Success!</text>
  </Match>
</Switch>;
```

### Index

List rendering with index tracking.

```typescript
import { Index } from "solid-js";

// Use Index when items don't change identity but values do
<Index each={values()}>
  {(value, index) => (
    <text>
      {index}: {value()}
    </text>
  )}
</Index>;
```

### Dynamic

Dynamic component rendering.

```typescript
import { Dynamic } from "solid-js/web";

const components = {
  text: TextScreen,
  list: ListScreen,
  form: FormScreen,
};

<Dynamic component={components[screenType()]} />;
```

## Custom Hooks

### useDebounce

```typescript
const useDebounce = <T>(value: Accessor<T>, delay: number): Accessor<T> => {
  const [debouncedValue, setDebouncedValue] = createSignal(value());

  createEffect(() => {
    const v = value();
    const timer = setTimeout(() => setDebouncedValue(() => v), delay);
    onCleanup(() => clearTimeout(timer));
  });

  return debouncedValue;
};
```

### useLocalStorage

```typescript
const useLocalStorage = <T>(key: string, initialValue: T) => {
  const stored = localStorage.getItem(key);
  const initial = stored ? JSON.parse(stored) : initialValue;

  const [value, setValue] = createSignal<T>(initial);

  createEffect(() => {
    localStorage.setItem(key, JSON.stringify(value()));
  });

  return [value, setValue] as const;
};
```

### useInputBuffer

Prevent initial Enter key capture.

```typescript
const useInputBuffer = (delay = 100) => {
  const [ready, setReady] = createSignal(false);

  onMount(() => {
    const timer = setTimeout(() => setReady(true), delay);
    onCleanup(() => clearTimeout(timer));
  });

  return ready;
};
```
