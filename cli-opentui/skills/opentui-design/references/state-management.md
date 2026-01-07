# State Management in OpenTUI

Patterns for managing application state with SolidJS.

## Overview

SolidJS provides fine-grained reactivity that makes state management straightforward. This guide covers patterns from simple local state to complex application state.

## Local State with Signals

```typescript
import { createSignal } from "solid-js";

const Counter = () => {
  const [count, setCount] = createSignal(0);

  const increment = () => setCount((c) => c + 1);
  const decrement = () => setCount((c) => c - 1);

  return (
    <box>
      <text>Count: {count()}</text>
    </box>
  );
};
```

## Derived State with Memos

```typescript
import { createSignal, createMemo } from "solid-js";

const TodoList = () => {
  const [todos, setTodos] = createSignal<Todo[]>([]);
  const [filter, setFilter] = createSignal<"all" | "active" | "completed">("all");

  const filteredTodos = createMemo(() => {
    const list = todos();
    switch (filter()) {
      case "active":
        return list.filter((t) => !t.completed);
      case "completed":
        return list.filter((t) => t.completed);
      default:
        return list;
    }
  });

  const stats = createMemo(() => ({
    total: todos().length,
    completed: todos().filter((t) => t.completed).length,
    active: todos().filter((t) => !t.completed).length,
  }));

  return (
    <box flexDirection="column">
      <text>
        {stats().active} active, {stats().completed} completed
      </text>
      <List items={filteredTodos()} />
    </box>
  );
};
```

## Complex State with Stores

```typescript
import { createStore } from "solid-js/store";

interface AppState {
  user: {
    name: string;
    preferences: {
      theme: "light" | "dark";
      showHelp: boolean;
    };
  } | null;
  currentScreen: string;
  notifications: Array<{ id: string; message: string }>;
}

const [state, setState] = createStore<AppState>({
  user: null,
  currentScreen: "main",
  notifications: [],
});

// Update nested property
setState("user", "preferences", "theme", "dark");

// Update array
setState("notifications", (n) => [...n, { id: "1", message: "Hello" }]);

// Remove from array
setState("notifications", (n) => n.filter((item) => item.id !== "1"));
```

## Context for Global State

```typescript
import { createContext, useContext, ParentComponent } from "solid-js";

// Define context type
interface AppContextType {
  theme: Accessor<string>;
  setTheme: (theme: string) => void;
  currentScreen: Accessor<string>;
  navigate: (screen: string) => void;
}

// Create context
const AppContext = createContext<AppContextType>();

// Provider component
const AppProvider: ParentComponent = (props) => {
  const [theme, setTheme] = createSignal("dark");
  const [currentScreen, setCurrentScreen] = createSignal("main");

  const value: AppContextType = {
    theme,
    setTheme,
    currentScreen,
    navigate: setCurrentScreen,
  };

  return (
    <AppContext.Provider value={value}>
      {props.children}
    </AppContext.Provider>
  );
};

// Hook to use context
const useApp = () => {
  const context = useContext(AppContext);
  if (!context) {
    throw new Error("useApp must be used within AppProvider");
  }
  return context;
};

// Usage
const ThemeToggle = () => {
  const { theme, setTheme } = useApp();

  return (
    <text>
      Theme: {theme()}
    </text>
  );
};
```

## State Machine Pattern

```typescript
type State = "idle" | "loading" | "success" | "error";
type Event = { type: "FETCH" } | { type: "SUCCESS"; data: unknown } | { type: "ERROR"; error: Error };

const createStateMachine = () => {
  const [state, setState] = createSignal<State>("idle");
  const [data, setData] = createSignal<unknown>(null);
  const [error, setError] = createSignal<Error | null>(null);

  const send = (event: Event) => {
    switch (event.type) {
      case "FETCH":
        setState("loading");
        break;
      case "SUCCESS":
        setState("success");
        setData(event.data);
        break;
      case "ERROR":
        setState("error");
        setError(event.error);
        break;
    }
  };

  return { state, data, error, send };
};

// Usage
const DataFetcher = () => {
  const { state, data, error, send } = createStateMachine();

  const fetchData = async () => {
    send({ type: "FETCH" });
    try {
      const result = await api.getData();
      send({ type: "SUCCESS", data: result });
    } catch (e) {
      send({ type: "ERROR", error: e as Error });
    }
  };

  return (
    <Switch>
      <Match when={state() === "idle"}>
        <text>Press Enter to fetch</text>
      </Match>
      <Match when={state() === "loading"}>
        <Spinner />
      </Match>
      <Match when={state() === "success"}>
        <text>{JSON.stringify(data())}</text>
      </Match>
      <Match when={state() === "error"}>
        <text color="red">{error()?.message}</text>
      </Match>
    </Switch>
  );
};
```

## Form State Management

```typescript
interface FormState {
  values: Record<string, string>;
  errors: Record<string, string>;
  touched: Record<string, boolean>;
  isValid: boolean;
}

const createForm = <T extends Record<string, string>>(
  initialValues: T,
  validate: (values: T) => Record<string, string>
) => {
  const [state, setState] = createStore<FormState>({
    values: { ...initialValues },
    errors: {},
    touched: {},
    isValid: false,
  });

  const setField = (name: string, value: string) => {
    setState("values", name, value);
    setState("touched", name, true);
    const errors = validate(state.values as T);
    setState("errors", errors);
    setState("isValid", Object.keys(errors).length === 0);
  };

  const reset = () => {
    setState({
      values: { ...initialValues },
      errors: {},
      touched: {},
      isValid: false,
    });
  };

  return { state, setField, reset };
};

// Usage
const LoginForm = () => {
  const { state, setField, reset } = createForm(
    { username: "", password: "" },
    (values) => {
      const errors: Record<string, string> = {};
      if (!values.username) errors.username = "Required";
      if (!values.password) errors.password = "Required";
      if (values.password.length < 6) errors.password = "Too short";
      return errors;
    }
  );

  return (
    <box flexDirection="column">
      <TextInput
        label="Username"
        value={state.values.username}
        onChange={(v) => setField("username", v)}
        error={state.touched.username ? state.errors.username : undefined}
      />
      <TextInput
        label="Password"
        value={state.values.password}
        onChange={(v) => setField("password", v)}
        error={state.touched.password ? state.errors.password : undefined}
      />
      <text>Valid: {state.isValid ? "Yes" : "No"}</text>
    </box>
  );
};
```

## Async State with Resources

```typescript
import { createResource, Suspense } from "solid-js";

const fetchUser = async (id: string): Promise<User> => {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
};

const UserProfile = (props: { userId: string }) => {
  const [user, { refetch, mutate }] = createResource(
    () => props.userId,
    fetchUser
  );

  return (
    <Suspense fallback={<Spinner />}>
      <Show when={user()}>
        {(u) => (
          <box flexDirection="column">
            <text>Name: {u().name}</text>
            <text>Email: {u().email}</text>
          </box>
        )}
      </Show>
    </Suspense>
  );
};
```

## State Persistence

```typescript
const createPersistedSignal = <T>(key: string, initialValue: T) => {
  // Load from storage
  const stored = localStorage.getItem(key);
  const initial = stored ? JSON.parse(stored) : initialValue;

  const [value, setValue] = createSignal<T>(initial);

  // Persist on change
  createEffect(() => {
    localStorage.setItem(key, JSON.stringify(value()));
  });

  return [value, setValue] as const;
};

// Usage
const Settings = () => {
  const [theme, setTheme] = createPersistedSignal("theme", "dark");

  return <text>Theme: {theme()}</text>;
};
```

## Best Practices

1. **Start simple** - Use signals before reaching for stores
2. **Colocate state** - Keep state close to where it's used
3. **Lift when needed** - Move state up only when sharing is required
4. **Use context sparingly** - Only for truly global state
5. **Memoize derived values** - Prevent unnecessary recomputation
6. **Type your state** - Use TypeScript interfaces
7. **Handle async properly** - Use resources or explicit loading states
8. **Clean up subscriptions** - Use onCleanup for side effects
