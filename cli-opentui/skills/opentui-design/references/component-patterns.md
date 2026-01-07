# Component Patterns in OpenTUI

Patterns for building reusable and maintainable CLI components with OpenTUI and SolidJS.

## Component Categories

### 1. Screen Components

Full-page views that handle their own input and state.

```typescript
interface ScreenProps {
  visible: boolean;
  onNavigate: (screen: string) => void;
}

const MainScreen = (props: ScreenProps) => {
  const [selectedIndex, setSelectedIndex] = createSignal(0);

  useKeyboard((key) => {
    if (!props.visible) return;

    if (key.name === "return") {
      key.preventDefault();
      props.onNavigate("detail");
    }
  });

  return (
    <Show when={props.visible}>
      <box flexDirection="column" flexGrow={1}>
        <Header title="Main" />
        <List
          items={items}
          selectedIndex={selectedIndex()}
          onSelect={setSelectedIndex}
        />
        <Footer shortcuts={["Enter: Select", "q: Quit"]} />
      </box>
    </Show>
  );
};
```

### 2. Part Components

Reusable UI components that receive data and callbacks via props.

```typescript
interface ListProps {
  items: string[];
  selectedIndex: number;
  onSelect: (index: number) => void;
  focused?: boolean;
}

const List = (props: ListProps) => {
  const focused = () => props.focused ?? true;

  useKeyboard((key) => {
    if (!focused()) return;

    if (key.name === "up") {
      props.onSelect(Math.max(0, props.selectedIndex - 1));
    }
    if (key.name === "down") {
      props.onSelect(Math.min(props.items.length - 1, props.selectedIndex + 1));
    }
  });

  return (
    <box flexDirection="column">
      <For each={props.items}>
        {(item, index) => (
          <text
            backgroundColor={
              index() === props.selectedIndex ? "blue" : undefined
            }
          >
            {index() === props.selectedIndex ? "> " : "  "}
            {item}
          </text>
        )}
      </For>
    </box>
  );
};
```

### 3. Common Components

Shared utilities and layout components.

```typescript
// Header component
const Header = (props: { title: string }) => (
  <box borderStyle="single" paddingX={1}>
    <text bold>{props.title}</text>
  </box>
);

// Footer component
const Footer = (props: { shortcuts: string[] }) => (
  <box borderStyle="single" paddingX={1}>
    <text dimColor>{props.shortcuts.join(" | ")}</text>
  </box>
);

// Spinner component
const Spinner = () => {
  const frames = ["-", "\\", "|", "/"];
  const [frameIndex, setFrameIndex] = createSignal(0);

  createEffect(() => {
    const timer = setInterval(() => {
      setFrameIndex((prev) => (prev + 1) % frames.length);
    }, 100);
    onCleanup(() => clearInterval(timer));
  });

  return <text>{frames[frameIndex()]}</text>;
};
```

## Composition Patterns

### Container/Presenter Pattern

Separate logic from presentation.

```typescript
// Container - handles logic
const UserListContainer = () => {
  const [users, setUsers] = createSignal<User[]>([]);
  const [loading, setLoading] = createSignal(true);

  onMount(async () => {
    const data = await fetchUsers();
    setUsers(data);
    setLoading(false);
  });

  return <UserListPresenter users={users()} loading={loading()} />;
};

// Presenter - handles rendering
const UserListPresenter = (props: { users: User[]; loading: boolean }) => (
  <box flexDirection="column">
    <Show when={props.loading} fallback={<UserList users={props.users} />}>
      <Spinner />
      <text>Loading users...</text>
    </Show>
  </box>
);
```

### Compound Components

Components that work together as a unit.

```typescript
// Parent provides context
const Tabs = (props: { children: JSX.Element }) => {
  const [activeTab, setActiveTab] = createSignal(0);

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <box flexDirection="column">{props.children}</box>
    </TabsContext.Provider>
  );
};

// Child components consume context
const TabList = (props: { children: JSX.Element }) => (
  <box flexDirection="row">{props.children}</box>
);

const Tab = (props: { index: number; label: string }) => {
  const { activeTab, setActiveTab } = useContext(TabsContext);

  return (
    <text
      backgroundColor={activeTab() === props.index ? "blue" : undefined}
      onClick={() => setActiveTab(props.index)}
    >
      {props.label}
    </text>
  );
};

const TabPanel = (props: { index: number; children: JSX.Element }) => {
  const { activeTab } = useContext(TabsContext);

  return <Show when={activeTab() === props.index}>{props.children}</Show>;
};

// Usage
<Tabs>
  <TabList>
    <Tab index={0} label="Files" />
    <Tab index={1} label="Search" />
  </TabList>
  <TabPanel index={0}>
    <FileList />
  </TabPanel>
  <TabPanel index={1}>
    <SearchPanel />
  </TabPanel>
</Tabs>;
```

### Render Props Pattern

Pass render function as prop for flexibility.

```typescript
interface SelectProps<T> {
  items: T[];
  renderItem: (item: T, selected: boolean) => JSX.Element;
  onSelect: (item: T) => void;
}

const Select = <T,>(props: SelectProps<T>) => {
  const [selectedIndex, setSelectedIndex] = createSignal(0);

  useKeyboard((key) => {
    if (key.name === "return") {
      props.onSelect(props.items[selectedIndex()]);
    }
  });

  return (
    <box flexDirection="column">
      <For each={props.items}>
        {(item, index) => props.renderItem(item, index() === selectedIndex())}
      </For>
    </box>
  );
};

// Usage
<Select
  items={branches}
  renderItem={(branch, selected) => (
    <text backgroundColor={selected ? "blue" : undefined}>
      {selected ? "> " : "  "}
      {branch.name}
      {branch.current && " (current)"}
    </text>
  )}
  onSelect={(branch) => checkout(branch)}
/>;
```

## State Management Patterns

### Local State

```typescript
const Counter = () => {
  const [count, setCount] = createSignal(0);

  return (
    <box>
      <text>Count: {count()}</text>
    </box>
  );
};
```

### Lifted State

```typescript
const Parent = () => {
  const [value, setValue] = createSignal("");

  return (
    <box flexDirection="column">
      <Input value={value()} onChange={setValue} />
      <Display value={value()} />
    </box>
  );
};
```

### Context State

```typescript
const AppContext = createContext<{
  theme: Accessor<string>;
  setTheme: Setter<string>;
}>();

const AppProvider = (props: { children: JSX.Element }) => {
  const [theme, setTheme] = createSignal("dark");

  return (
    <AppContext.Provider value={{ theme, setTheme }}>
      {props.children}
    </AppContext.Provider>
  );
};

const useAppContext = () => useContext(AppContext)!;
```

## Best Practices

1. **Keep components small and focused** - One responsibility per component
2. **Use TypeScript interfaces** - Define clear prop types
3. **Avoid prop drilling** - Use context for deeply nested data
4. **Memoize expensive computations** - Use createMemo
5. **Handle loading and error states** - Always show feedback
6. **Make components keyboard-accessible** - Don't rely on mouse
7. **Use ASCII for icons** - Ensure terminal compatibility
