# State Management

Patterns for managing complex state in Ink.js applications.

## Overview

Effective state management in CLI applications balances simplicity with scalability. This guide covers patterns from basic useState to complex Map-based state.

## Map-Based State Management

For collections of entities, use Map for O(1) lookups:

```typescript
interface Branch {
  name: string;
  current: boolean;
  upstream?: string;
}

const [branchMap, setBranchMap] = useState<Map<string, Branch>>(new Map());

// Update single branch
const updateBranch = (name: string, updates: Partial<Branch>) => {
  setBranchMap(prev => {
    const newMap = new Map(prev);
    const existing = newMap.get(name);
    if (existing) {
      newMap.set(name, { ...existing, ...updates });
    }
    return newMap;
  });
};

// Batch update
const updateBranches = (updates: Array<[string, Partial<Branch>]>) => {
  setBranchMap(prev => {
    const newMap = new Map(prev);
    for (const [name, update] of updates) {
      const existing = newMap.get(name);
      if (existing) {
        newMap.set(name, { ...existing, ...update });
      }
    }
    return newMap;
  });
};
```

## Hash-Based Dependency Arrays

Use content hashes for efficient memoization:

```typescript
import { useMemo } from "react";

const branchList = useMemo(() => {
  return Array.from(branchMap.values()).sort((a, b) =>
    a.name.localeCompare(b.name)
  );
}, [branchMap]);

// Hash for deep comparison
const branchHash = useMemo(() => {
  return Array.from(branchMap.entries())
    .map(([k, v]) => `${k}:${v.current}:${v.upstream ?? ""}`)
    .join("|");
}, [branchMap]);

// Use hash in dependent computations
const processedBranches = useMemo(() => {
  return branchList.filter(b => !b.name.startsWith("_"));
}, [branchHash]); // Uses hash instead of full object
```

## Controlled vs Uncontrolled Components

Support both modes for flexibility:

```typescript
interface SelectProps<T> {
  items: Array<{ label: string; value: T }>;
  // Controlled mode
  value?: T;
  onChange?: (value: T) => void;
  // Uncontrolled mode
  defaultValue?: T;
  onSelect?: (value: T) => void;
}

export function Select<T>({
  items,
  value,
  onChange,
  defaultValue,
  onSelect,
}: SelectProps<T>) {
  const isControlled = value !== undefined;
  const [internalValue, setInternalValue] = useState(defaultValue);

  const currentValue = isControlled ? value : internalValue;

  const handleSelect = (newValue: T) => {
    if (isControlled) {
      onChange?.(newValue);
    } else {
      setInternalValue(newValue);
      onSelect?.(newValue);
    }
  };

  // Unified update function works for both modes
  const selectedIndex = items.findIndex(item => item.value === currentValue);

  // ... rest of implementation
}
```

## Selection State Management

### Single Selection

```typescript
const [selectedIndex, setSelectedIndex] = useState(0);

useInput((input, key) => {
  if (key.upArrow) {
    setSelectedIndex(prev => Math.max(0, prev - 1));
  }
  if (key.downArrow) {
    setSelectedIndex(prev => Math.min(items.length - 1, prev + 1));
  }
});
```

### Multi-Selection with Set

```typescript
const [selectedItems, setSelectedItems] = useState<Set<string>>(new Set());

const toggleSelection = (id: string) => {
  setSelectedItems(prev => {
    const newSet = new Set(prev);
    if (newSet.has(id)) {
      newSet.delete(id);
    } else {
      newSet.add(id);
    }
    return newSet;
  });
};

const selectAll = () => {
  setSelectedItems(new Set(items.map(item => item.id)));
};

const clearSelection = () => {
  setSelectedItems(new Set());
};
```

## Async State Updates

### Cancelled Flag Pattern

Prevent state updates after unmount:

```typescript
useEffect(() => {
  let cancelled = false;

  async function fetchData() {
    try {
      const result = await api.fetchBranches();
      if (!cancelled) {
        setBranches(result);
      }
    } catch (error) {
      if (!cancelled) {
        setError(error.message);
      }
    }
  }

  fetchData();

  return () => {
    cancelled = true;
  };
}, []);
```

### withTimeout Utility

Add timeout to async operations:

```typescript
export async function withTimeout<T>(
  promise: Promise<T>,
  timeoutMs: number,
  timeoutMessage = "Operation timed out"
): Promise<T> {
  let timeoutId: NodeJS.Timeout;

  const timeoutPromise = new Promise<never>((_, reject) => {
    timeoutId = setTimeout(() => {
      reject(new Error(timeoutMessage));
    }, timeoutMs);
  });

  try {
    return await Promise.race([promise, timeoutPromise]);
  } finally {
    clearTimeout(timeoutId);
  }
}

// Usage
const result = await withTimeout(
  fetchBranches(),
  5000,
  "Failed to fetch branches within 5 seconds"
);
```

## Loading States

### Unified Loading State

```typescript
type LoadingState = "idle" | "loading" | "success" | "error";

interface AsyncState<T> {
  state: LoadingState;
  data: T | null;
  error: string | null;
}

function useAsyncData<T>(
  fetcher: () => Promise<T>
): AsyncState<T> & { refetch: () => void } {
  const [state, setState] = useState<AsyncState<T>>({
    state: "idle",
    data: null,
    error: null,
  });

  const fetch = async () => {
    setState(prev => ({ ...prev, state: "loading", error: null }));

    try {
      const data = await fetcher();
      setState({ state: "success", data, error: null });
    } catch (error) {
      setState(prev => ({
        ...prev,
        state: "error",
        error: error.message,
      }));
    }
  };

  useEffect(() => {
    fetch();
  }, []);

  return { ...state, refetch: fetch };
}
```

## State Indicators

Visual feedback for item states:

```typescript
interface ItemState {
  loading: boolean;
  success: boolean;
  error: string | null;
}

const [itemStates, setItemStates] = useState<Map<string, ItemState>>(new Map());

const setItemLoading = (id: string) => {
  setItemStates(prev => {
    const newMap = new Map(prev);
    newMap.set(id, { loading: true, success: false, error: null });
    return newMap;
  });
};

const setItemSuccess = (id: string) => {
  setItemStates(prev => {
    const newMap = new Map(prev);
    newMap.set(id, { loading: false, success: true, error: null });
    return newMap;
  });
};

const setItemError = (id: string, error: string) => {
  setItemStates(prev => {
    const newMap = new Map(prev);
    newMap.set(id, { loading: false, success: false, error });
    return newMap;
  });
};
```

## Best Practices

1. **Use Map for collections** - O(1) lookups vs O(n) array searches
2. **Prefer immutable updates** - Always create new Map/Set instances
3. **Use hash-based memoization** - For complex dependency arrays
4. **Support both controlled/uncontrolled modes** - Flexibility for consumers
5. **Handle async cancellation** - Prevent updates after unmount
6. **Add timeouts to async operations** - Prevent hanging states
7. **Centralize loading states** - Consistent loading/error handling
8. **Batch updates when possible** - Reduce re-renders
