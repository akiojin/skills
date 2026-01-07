# Testing Patterns and Best Practices

## Table of Contents

1. [Test Environment Setup](#test-environment-setup)
2. [Component Testing](#component-testing)
3. [Mock Patterns](#mock-patterns)
4. [Integration Testing](#integration-testing)

## Test Environment Setup

### Vitest + happy-dom

```typescript
/**
 * @vitest-environment happy-dom
 */
import { describe, it, expect, beforeEach, vi } from "vitest";
import { render } from "@testing-library/react";
import { Window } from "happy-dom";

describe("Component", () => {
  beforeEach(() => {
    const window = new Window();
    // @ts-expect-error - happy-dom type mismatch
    globalThis.window = window;
    // @ts-expect-error - happy-dom type mismatch
    globalThis.document = window.document;
  });
});
```

### ink-testing-library

```typescript
import { render } from "ink-testing-library";

describe("Select", () => {
  it("should render items", () => {
    const items = [
      { label: "Item 1", value: "1" },
      { label: "Item 2", value: "2" },
    ];

    const { lastFrame } = render(
      <Select items={items} onSelect={vi.fn()} />
    );

    expect(lastFrame()).toContain("Item 1");
    expect(lastFrame()).toContain("Item 2");
  });
});
```

## Component Testing

### Basic Pattern

```typescript
import { render } from "ink-testing-library";
import { describe, it, expect, vi } from "vitest";

describe("Header", () => {
  it("should render title", () => {
    const { lastFrame } = render(<Header title="Test" />);
    expect(lastFrame()).toContain("Test");
  });

  it("should render version when provided", () => {
    const { lastFrame } = render(
      <Header title="Test" version="1.0.0" />
    );
    expect(lastFrame()).toContain("1.0.0");
  });
});
```

### Key Input Testing

```typescript
import { render } from "ink-testing-library";

describe("Select", () => {
  it("should move selection down on j key", async () => {
    const items = [
      { label: "Item 1", value: "1" },
      { label: "Item 2", value: "2" },
    ];

    const { stdin, lastFrame } = render(
      <Select items={items} onSelect={vi.fn()} />
    );

    // Initial state
    expect(lastFrame()).toContain("› Item 1");

    // Press j key to move down
    stdin.write("j");

    // Verify selection moved
    expect(lastFrame()).toContain("› Item 2");
  });

  it("should call onSelect on Enter", () => {
    const onSelect = vi.fn();
    const items = [{ label: "Item 1", value: "1" }];

    const { stdin } = render(
      <Select items={items} onSelect={onSelect} />
    );

    stdin.write("\r");  // Enter key

    expect(onSelect).toHaveBeenCalledWith(items[0]);
  });
});
```

### Async Testing

```typescript
import { render } from "ink-testing-library";
import { vi } from "vitest";

describe("LoadingIndicator", () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it("should show after delay", async () => {
    const { lastFrame } = render(
      <LoadingIndicator isLoading={true} delay={300} />
    );

    // Not displayed initially
    expect(lastFrame()).toBe("");

    // Displayed after 300ms
    vi.advanceTimersByTime(300);
    expect(lastFrame()).toContain("Loading");
  });
});
```

## Mock Patterns

### Mocking External Dependencies

```typescript
vi.mock("../../../git.js", () => ({
  getRepositoryRoot: vi.fn().mockResolvedValue("/repo"),
  getAllBranches: vi.fn().mockResolvedValue([]),
  fetchAllRemotes: vi.fn().mockResolvedValue(undefined),
}));
```

### Mocking Screen Components

Mock Screens other than the test target:

```typescript
// Array to capture props
const capturedProps: BranchListScreenProps[] = [];

vi.mock("../../components/screens/BranchListScreen.js", () => ({
  BranchListScreen: (props: BranchListScreenProps) => {
    capturedProps.push(props);
    return <div>Mocked BranchListScreen</div>;
  },
}));

describe("App", () => {
  beforeEach(() => {
    capturedProps.length = 0;  // Reset
  });

  it("should pass branches to BranchListScreen", async () => {
    render(<App />);

    await waitFor(() => {
      expect(capturedProps.length).toBeGreaterThan(0);
    });

    expect(capturedProps[0].branches).toHaveLength(2);
  });
});
```

### Mocking Hooks

```typescript
vi.mock("../../hooks/useGitData.js", () => ({
  useGitData: () => ({
    branches: [
      { name: "main", branchType: "main" },
      { name: "develop", branchType: "develop" },
    ],
    loading: false,
    error: null,
    refresh: vi.fn(),
  }),
}));
```

### Conditional Mock Responses

```typescript
const mockGetBranches = vi.fn();

vi.mock("../../../git.js", () => ({
  getAllBranches: mockGetBranches,
}));

describe("useGitData", () => {
  it("should handle error", async () => {
    mockGetBranches.mockRejectedValueOnce(new Error("Git error"));

    // Verify error is set
  });

  it("should return branches", async () => {
    mockGetBranches.mockResolvedValueOnce([
      { name: "main" },
    ]);

    // Verify branches are returned
  });
});
```

## Integration Testing

### Navigation Testing

```typescript
describe("Navigation", () => {
  it("should navigate from branch list to action selector", async () => {
    const { stdin, lastFrame } = render(<App />);

    // Wait for branch list to display
    await waitFor(() => {
      expect(lastFrame()).toContain("Branch List");
    });

    // Select with Enter
    stdin.write("\r");

    // Navigate to action selector
    await waitFor(() => {
      expect(lastFrame()).toContain("Action Selector");
    });
  });

  it("should go back on Escape", async () => {
    const { stdin, lastFrame } = render(<App />);

    // Navigate
    stdin.write("\r");
    await waitFor(() => {
      expect(lastFrame()).toContain("Action Selector");
    });

    // Go back with Escape
    stdin.write("\x1B");  // Escape key

    await waitFor(() => {
      expect(lastFrame()).toContain("Branch List");
    });
  });
});
```

### Edge Case Testing

```typescript
describe("Edge Cases", () => {
  it("should handle empty list", () => {
    const { lastFrame } = render(
      <Select items={[]} onSelect={vi.fn()} />
    );

    expect(lastFrame()).not.toContain("undefined");
  });

  it("should handle very long labels", () => {
    const items = [{
      label: "A".repeat(200),
      value: "1",
    }];

    const { lastFrame } = render(
      <Select items={items} onSelect={vi.fn()} />
    );

    // Verify display is not broken
    expect(lastFrame()).toBeDefined();
  });
});
```

### Performance Testing

```typescript
describe("Performance", () => {
  it("should handle 1000 items without significant lag", async () => {
    const items = Array.from({ length: 1000 }, (_, i) => ({
      label: `Item ${i}`,
      value: String(i),
    }));

    const start = performance.now();

    const { stdin } = render(
      <Select items={items} onSelect={vi.fn()} limit={20} />
    );

    // 100 movements
    for (let i = 0; i < 100; i++) {
      stdin.write("j");
    }

    const duration = performance.now() - start;

    // Should complete within 1 second
    expect(duration).toBeLessThan(1000);
  });
});
```

## Test Utilities

### waitFor Helper

```typescript
async function waitFor(
  condition: () => boolean | void,
  timeout = 1000
): Promise<void> {
  const start = Date.now();

  while (Date.now() - start < timeout) {
    try {
      if (condition()) return;
    } catch {
      // Condition not met
    }
    await new Promise((r) => setTimeout(r, 10));
  }

  throw new Error("waitFor timed out");
}
```

### Key Constants for Testing

```typescript
const KEYS = {
  ENTER: "\r",
  ESCAPE: "\x1B",
  UP: "\x1B[A",
  DOWN: "\x1B[B",
  LEFT: "\x1B[D",
  RIGHT: "\x1B[C",
  TAB: "\t",
  BACKSPACE: "\x7F",
  CTRL_C: "\x03",
};
```
