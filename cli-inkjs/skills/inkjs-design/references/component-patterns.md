# Component Design Patterns

## Table of Contents

1. [Directory Structure](#directory-structure)
2. [Component Classification](#component-classification)
3. [Three-Layer Layout](#three-layer-layout)
4. [React.memo Optimization](#reactmemo-optimization)
5. [Controlled/Uncontrolled Mode](#controlleduncontrolled-mode)

## Directory Structure

```
src/cli/ui/
├── components/
│   ├── App.tsx              # Root component (screen management)
│   ├── common/              # Generic components
│   │   ├── Select.tsx       # Selection list
│   │   ├── Input.tsx        # Text input
│   │   ├── Confirm.tsx      # Yes/No confirmation
│   │   ├── LoadingIndicator.tsx
│   │   └── ErrorBoundary.tsx
│   ├── parts/               # UI parts
│   │   ├── Header.tsx
│   │   ├── Footer.tsx
│   │   ├── ProgressBar.tsx
│   │   ├── Stats.tsx
│   │   └── ScrollableList.tsx
│   └── screens/             # Screen components
│       ├── BranchListScreen.tsx
│       ├── ModelSelectorScreen.tsx
│       └── BranchCreatorScreen.tsx
├── hooks/                   # Custom hooks
├── utils/                   # Utility functions
├── types.ts                 # Type definitions
└── __tests__/               # Tests
```

## Component Classification

### Screen

Components representing complete screens.

**Responsibilities:**
- Handle keyboard input with `useInput`
- Three-layer layout with Header/Content/Footer
- Screen-specific business logic

**Naming:** `<Concept>Screen.tsx`

```typescript
interface BranchListScreenProps {
  branches: BranchItem[];
  stats: Statistics;
  onSelect: (branch: BranchItem) => void;
  onQuit?: () => void;
  onRefresh?: () => void;
  loading?: boolean;
  error?: Error | null;
}

export function BranchListScreen({
  branches,
  stats,
  onSelect,
  onQuit,
  onRefresh,
  loading,
  error,
}: BranchListScreenProps) {
  const { rows } = useTerminalSize();

  useInput((input, key) => {
    if (key.escape || input === "q") {
      onQuit?.();
    }
    if (input === "r") {
      onRefresh?.();
    }
  });

  return (
    <Box flexDirection="column" height={rows}>
      <Header title="Branch List" />
      <Box flexDirection="column" flexGrow={1}>
        {loading ? (
          <LoadingIndicator />
        ) : (
          <Select items={branches} onSelect={onSelect} />
        )}
      </Box>
      <Footer actions={[{ key: "q", label: "Quit" }]} />
    </Box>
  );
}
```

### Part

Reusable UI parts. Pure components without state.

**Responsibilities:**
- Display only
- Optimize with `React.memo`
- Receive all data from props

**Naming:** `<Feature>.tsx`

```typescript
interface HeaderProps {
  title: string;
  version?: string;
}

export const Header = React.memo(function Header({
  title,
  version,
}: HeaderProps) {
  return (
    <Box borderStyle="single" paddingX={1}>
      <Text bold>{title}</Text>
      {version && <Text dimColor> v{version}</Text>}
    </Box>
  );
});
```

### Common

Basic input components.

**Responsibilities:**
- Support both controlled and uncontrolled modes
- Generic interface
- Reusable in other projects

**Naming:** `<BasicType>.tsx`

## Three-Layer Layout

Standard screen layout pattern.

```typescript
export function StandardScreen({ children }: PropsWithChildren) {
  const { rows } = useTerminalSize();

  return (
    <Box flexDirection="column" height={rows}>
      {/* Layer 1: Header - fixed height */}
      <Header title="App Title" />

      {/* Layer 2: Content - fill remaining height with flexGrow */}
      <Box flexDirection="column" flexGrow={1}>
        {children}
      </Box>

      {/* Layer 3: Footer - fixed height */}
      <Footer actions={footerActions} />
    </Box>
  );
}
```

### Dynamic Height Calculation

```typescript
const { rows } = useTerminalSize();

// Calculate fixed portion line count
const headerLines = 2;    // Border + title
const filterLines = 1;    // Filter input
const statsLines = 1;     // Statistics info
const footerLines = 1;    // Footer

const fixedLines = headerLines + filterLines + statsLines + footerLines;
const contentHeight = rows - fixedLines;
const listLimit = Math.max(5, contentHeight);  // Ensure minimum 5 lines

return (
  <Select
    items={items}
    limit={listLimit}
    onSelect={handleSelect}
  />
);
```

## React.memo Optimization

### Basic Pattern

```typescript
export const Header = React.memo(function Header(props: HeaderProps) {
  return <Box>...</Box>;
});
```

### Custom Comparison Function

> **Important**: When using custom comparison functions, it's assumed that callback props (like `onSelect`) have stable references. Use `useCallback` in parent components to memoize callbacks.

For comparing array contents rather than references:

```typescript
function arePropsEqual<T extends SelectItem>(
  prevProps: SelectProps<T>,
  nextProps: SelectProps<T>,
): boolean {
  // Simple value comparison
  if (
    prevProps.limit !== nextProps.limit ||
    prevProps.disabled !== nextProps.disabled ||
    prevProps.onSelect !== nextProps.onSelect
  ) {
    return false;
  }

  // Compare array length
  if (prevProps.items.length !== nextProps.items.length) {
    return false;
  }

  // Compare array contents
  for (let i = 0; i < prevProps.items.length; i++) {
    const prevItem = prevProps.items[i];
    const nextItem = nextProps.items[i];
    if (
      prevItem.value !== nextItem.value ||
      prevItem.label !== nextItem.label
    ) {
      return false;
    }
  }

  return true;
}

export const Select = React.memo(
  SelectComponent,
  arePropsEqual,
) as typeof SelectComponent;
```

## Controlled/Uncontrolled Mode

Enable parent control of state in components like Select.

```typescript
interface SelectProps<T> {
  items: T[];
  onSelect: (item: T) => void;
  // For uncontrolled mode
  initialIndex?: number;
  // For controlled mode
  selectedIndex?: number;
  onSelectedIndexChange?: (index: number) => void;
}

function SelectComponent<T>({
  items,
  onSelect,
  initialIndex = 0,
  selectedIndex: externalSelectedIndex,
  onSelectedIndexChange,
}: SelectProps<T>) {
  // Internal state (for uncontrolled mode)
  const [internalSelectedIndex, setInternalSelectedIndex] =
    useState(initialIndex);

  // Determine controlled vs uncontrolled mode
  const isControlled = externalSelectedIndex !== undefined;
  const selectedIndex = isControlled
    ? externalSelectedIndex
    : internalSelectedIndex;

  // Unified update function
  const updateSelectedIndex = (
    value: number | ((prev: number) => number)
  ) => {
    const newIndex = typeof value === "function"
      ? value(selectedIndex)
      : value;

    // Update internal state in uncontrolled mode
    if (!isControlled) {
      setInternalSelectedIndex(newIndex);
    }

    // Call callback if provided
    if (onSelectedIndexChange) {
      onSelectedIndexChange(newIndex);
    }
  };

  // ...
}
```

## Non-Looping Navigation

Pattern where list stops at edges (no looping):

```typescript
useInput((input, key) => {
  if (key.upArrow || input === "k") {
    // Move up (stop at 0)
    updateSelectedIndex((current) => Math.max(0, current - 1));
  } else if (key.downArrow || input === "j") {
    // Move down (stop at end)
    updateSelectedIndex((current) =>
      Math.min(items.length - 1, current + 1)
    );
  }
});
```

## ErrorBoundary Pattern

> **Note**: Due to React constraints, ErrorBoundary can only be implemented as a class component. This is the only exception to the "functional components + hooks" principle.

Implementation with class component (not possible with hooks):

```typescript
interface Props {
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error("ErrorBoundary caught:", error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <Box>
          <Text color="red">Error: {this.state.error?.message}</Text>
        </Box>
      );
    }
    return this.props.children;
  }
}
```
