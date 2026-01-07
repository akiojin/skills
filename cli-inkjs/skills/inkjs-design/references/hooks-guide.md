# Custom Hooks Design Guide

## Table of Contents

1. [Design Principles](#design-principles)
2. [useScreenState](#usescreenstate)
3. [useTerminalSize](#useterminalsize)
4. [useGitData (Data Fetching Pattern)](#usegitdata)
5. [useInput Patterns](#useinput-patterns)

## Design Principles

### Single Responsibility

Each hook handles only one concern.

```typescript
// Good: Single responsibility
function useScreenState() { ... }  // Screen navigation only
function useTerminalSize() { ... } // Size monitoring only

// Bad: Multiple responsibilities
function useAppState() {
  // Screen navigation, data fetching, terminal size...
}
```

### Return Value Object

Return related state and functions together as an object.

```typescript
interface UseGitDataResult {
  branches: BranchInfo[];
  loading: boolean;
  error: Error | null;
  refresh: () => Promise<void>;
}

function useGitData(): UseGitDataResult {
  // ...
  return { branches, loading, error, refresh };
}
```

### Cleanup

Always return a cleanup function from `useEffect`.

```typescript
useEffect(() => {
  const handler = () => { ... };
  process.stdout.on("resize", handler);

  // Cleanup
  return () => {
    process.stdout.removeListener("resize", handler);
  };
}, []);
```

### Dependency Array Accuracy

Specify dependency arrays accurately for `useCallback`/`useEffect`/`useMemo`.

```typescript
const loadData = useCallback(async () => {
  setLoading(true);
  const data = await fetchData(options);  // uses options
  setData(data);
  setLoading(false);
}, [options]);  // include options in dependencies
```

## useScreenState

Stack-based screen navigation management.

```typescript
import { useState, useCallback } from "react";
import type { ScreenType } from "../types.js";

export interface ScreenStateResult {
  currentScreen: ScreenType;
  navigateTo: (screen: ScreenType) => void;
  goBack: () => void;
  reset: () => void;
}

const INITIAL_SCREEN: ScreenType = "branch-list";

export function useScreenState(): ScreenStateResult {
  const [history, setHistory] = useState<ScreenType[]>([INITIAL_SCREEN]);

  const currentScreen = history[history.length - 1] ?? INITIAL_SCREEN;

  const navigateTo = useCallback((screen: ScreenType) => {
    setHistory((prev) => [...prev, screen]);
  }, []);

  const goBack = useCallback(() => {
    setHistory((prev) => {
      if (prev.length <= 1) {
        return prev;  // Cannot go back from initial screen
      }
      return prev.slice(0, -1);
    });
  }, []);

  const reset = useCallback(() => {
    setHistory([INITIAL_SCREEN]);
  }, []);

  return {
    currentScreen,
    navigateTo,
    goBack,
    reset,
  };
}
```

### Usage Example

```typescript
function App() {
  const { currentScreen, navigateTo, goBack } = useScreenState();

  const renderScreen = () => {
    switch (currentScreen) {
      case "branch-list":
        return (
          <BranchListScreen
            onSelect={(branch) => {
              setBranch(branch);
              navigateTo("action-selector");
            }}
          />
        );
      case "action-selector":
        return (
          <ActionSelectorScreen
            onBack={goBack}
            onSelect={handleAction}
          />
        );
      default:
        return null;
    }
  };

  return <Box>{renderScreen()}</Box>;
}
```

## useTerminalSize

Get terminal size and monitor resize events.

> **Note**: In non-TTY environments such as CI/test environments, `process.stdout.rows`/`columns` may be `undefined`. Always set fallback values. It's recommended to use `process.stdout.isTTY` for TTY detection and skip resize events in non-TTY environments.

```typescript
import { useState, useEffect } from "react";

export interface TerminalSize {
  rows: number;
  columns: number;
}

export function useTerminalSize(): TerminalSize {
  const [size, setSize] = useState<TerminalSize>(() => ({
    rows: process.stdout.rows || 24,
    columns: process.stdout.columns || 80,
  }));

  useEffect(() => {
    const handleResize = () => {
      setSize({
        rows: process.stdout.rows || 24,
        columns: process.stdout.columns || 80,
      });
    };

    process.stdout.on("resize", handleResize);
    return () => {
      process.stdout.removeListener("resize", handleResize);
    };
  }, []);

  return size;
}
```

### Usage Example

```typescript
function BranchListScreen() {
  const { rows, columns } = useTerminalSize();

  // Calculate display rows dynamically
  const headerHeight = 2;
  const footerHeight = 1;
  const availableRows = rows - headerHeight - footerHeight;

  return (
    <Box flexDirection="column">
      <Header />
      <Select items={items} limit={Math.max(5, availableRows)} />
      <Footer />
    </Box>
  );
}
```

## useGitData

Pattern for async data fetching and caching.

```typescript
import { useState, useCallback, useEffect } from "react";

export interface UseGitDataOptions {
  enableAutoRefresh?: boolean;
  refreshInterval?: number;
}

export interface UseGitDataResult {
  branches: BranchInfo[];
  worktrees: WorktreeInfo[];
  loading: boolean;
  error: Error | null;
  lastUpdated: Date | null;
  refresh: () => Promise<void>;
}

export function useGitData(
  options?: UseGitDataOptions
): UseGitDataResult {
  const {
    enableAutoRefresh = false,
    refreshInterval = 30000,
  } = options ?? {};

  const [branches, setBranches] = useState<BranchInfo[]>([]);
  const [worktrees, setWorktrees] = useState<WorktreeInfo[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  const [lastUpdated, setLastUpdated] = useState<Date | null>(null);

  const loadData = useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      // Fetch data in parallel
      const [branchData, worktreeData] = await Promise.all([
        fetchBranches(),
        fetchWorktrees(),
      ]);

      setBranches(branchData);
      setWorktrees(worktreeData);
      setLastUpdated(new Date());
    } catch (err) {
      setError(err instanceof Error ? err : new Error(String(err)));
    } finally {
      setLoading(false);
    }
  }, []);

  // Initial load
  useEffect(() => {
    loadData();
  }, [loadData]);

  // Auto refresh
  useEffect(() => {
    if (!enableAutoRefresh) return;

    const intervalId = setInterval(loadData, refreshInterval);
    return () => clearInterval(intervalId);
  }, [enableAutoRefresh, refreshInterval, loadData]);

  return {
    branches,
    worktrees,
    loading,
    error,
    lastUpdated,
    refresh: loadData,
  };
}
```

### Cancellation Pattern

Prevent state updates after component unmounts:

```typescript
const loadData = useCallback(async () => {
  let cancelled = false;

  setLoading(true);

  try {
    const data = await fetchData();

    if (!cancelled) {
      setData(data);
    }
  } catch (err) {
    if (!cancelled) {
      setError(err);
    }
  } finally {
    if (!cancelled) {
      setLoading(false);
    }
  }

  return () => {
    cancelled = true;
  };
}, []);
```

## useInput Patterns

### Basic Pattern

```typescript
import { useInput } from "ink";

function MyComponent() {
  useInput((input, key) => {
    if (key.escape) {
      handleEscape();
    }
    if (key.return) {
      handleEnter();
    }
    if (key.upArrow || input === "k") {
      moveUp();
    }
    if (key.downArrow || input === "j") {
      moveDown();
    }
  });
}
```

### Conditional Handling

```typescript
function MyComponent({ disabled, mode }: Props) {
  useInput((input, key) => {
    // Do nothing when disabled
    if (disabled) return;

    // Branch processing by mode
    if (mode === "edit") {
      // Edit mode processing
    } else {
      // Normal mode processing
    }
  });
}
```

### Global Shortcuts

```typescript
function App() {
  const [filterMode, setFilterMode] = useState(false);

  // Global shortcuts (active except during filter mode)
  useInput((input, key) => {
    if (filterMode) return;  // Disable during filter input

    switch (input) {
      case "q":
        handleQuit();
        break;
      case "r":
        handleRefresh();
        break;
      case "c":
        handleCleanup();
        break;
    }
  });

  return (
    <Box>
      <FilterInput
        onFocus={() => setFilterMode(true)}
        onBlur={() => setFilterMode(false)}
      />
    </Box>
  );
}
```

### Spinner Animation

```typescript
const SPINNER_FRAMES = ["⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧"];

function useSpinner(isActive: boolean) {
  const [frameIndex, setFrameIndex] = useState(0);
  const frameRef = useRef(0);

  useEffect(() => {
    if (!isActive) return;

    const interval = setInterval(() => {
      frameRef.current = (frameRef.current + 1) % SPINNER_FRAMES.length;
      setFrameIndex(frameRef.current);
    }, 80);  // 80ms = 12.5 FPS

    return () => clearInterval(interval);
  }, [isActive]);

  return SPINNER_FRAMES[frameIndex];
}

// Usage
function LoadingIndicator({ loading }: { loading: boolean }) {
  const frame = useSpinner(loading);

  if (!loading) return null;

  return <Text>{frame} Loading...</Text>;
}
```
