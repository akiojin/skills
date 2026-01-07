# Testing Patterns for OpenTUI

Patterns for testing CLI applications built with OpenTUI and SolidJS.

## Overview

Testing CLI applications requires different approaches than web applications. This guide covers unit testing, integration testing, and end-to-end testing patterns.

## Unit Testing Components

### Setup with Bun Test

```typescript
import { describe, it, expect, beforeEach, afterEach } from "bun:test";
import { render, cleanup } from "@opentui/solid/testing";

afterEach(() => {
  cleanup();
});

describe("Counter", () => {
  it("should render initial count", () => {
    const { getByText } = render(() => <Counter initialCount={5} />);
    expect(getByText("Count: 5")).toBeTruthy();
  });
});
```

### Testing State Changes

```typescript
import { fireEvent } from "@opentui/solid/testing";

describe("Counter", () => {
  it("should increment count on key press", async () => {
    const { getByText, container } = render(() => <Counter />);

    // Simulate key press
    fireEvent.keyDown(container, { key: "+" });

    expect(getByText("Count: 1")).toBeTruthy();
  });
});
```

### Testing with Props

```typescript
describe("List", () => {
  it("should render items", () => {
    const items = ["Item 1", "Item 2", "Item 3"];
    const { getByText } = render(() => <List items={items} />);

    items.forEach((item) => {
      expect(getByText(item)).toBeTruthy();
    });
  });

  it("should highlight selected item", () => {
    const items = ["A", "B", "C"];
    const { container } = render(() => (
      <List items={items} selectedIndex={1} />
    ));

    const selected = container.querySelector("[data-selected]");
    expect(selected?.textContent).toContain("B");
  });
});
```

## Testing Hooks

### Custom Hook Testing

```typescript
import { createRoot } from "solid-js";

const testHook = <T>(hook: () => T): T => {
  let result: T;
  createRoot((dispose) => {
    result = hook();
    dispose();
  });
  return result!;
};

describe("useDebounce", () => {
  it("should debounce value changes", async () => {
    const [value, setValue] = createSignal("initial");

    let debounced: Accessor<string>;
    createRoot(() => {
      debounced = useDebounce(value, 100);
    });

    setValue("changed");
    expect(debounced!()).toBe("initial");

    await new Promise((r) => setTimeout(r, 150));
    expect(debounced!()).toBe("changed");
  });
});
```

### Testing useKeyboard

```typescript
import { vi } from "vitest"; // or use Bun's mock

describe("useKeyboard", () => {
  it("should call handler on key press", () => {
    const handler = vi.fn();

    render(() => {
      useKeyboard(handler);
      return <box>Test</box>;
    });

    // Simulate stdin key event
    process.stdin.emit("data", "\r"); // Enter key

    expect(handler).toHaveBeenCalledWith(
      expect.objectContaining({ name: "return" })
    );
  });
});
```

## Integration Testing

### Testing Screen Navigation

```typescript
describe("App Navigation", () => {
  it("should navigate from main to detail screen", async () => {
    const { getByText, queryByText } = render(() => <App />);

    // Verify main screen
    expect(getByText("Main Screen")).toBeTruthy();
    expect(queryByText("Detail Screen")).toBeNull();

    // Navigate
    fireEvent.keyDown(document, { key: "Enter" });

    // Wait for transition
    await new Promise((r) => setTimeout(r, 200));

    // Verify detail screen
    expect(queryByText("Main Screen")).toBeNull();
    expect(getByText("Detail Screen")).toBeTruthy();
  });

  it("should go back on Escape", async () => {
    const { getByText, queryByText } = render(() => <App />);

    // Navigate to detail
    fireEvent.keyDown(document, { key: "Enter" });
    await new Promise((r) => setTimeout(r, 200));

    // Go back
    fireEvent.keyDown(document, { key: "Escape" });
    await new Promise((r) => setTimeout(r, 200));

    // Verify back on main
    expect(getByText("Main Screen")).toBeTruthy();
  });
});
```

### Testing Forms

```typescript
describe("LoginForm", () => {
  it("should validate required fields", () => {
    const { getByText, getByTestId } = render(() => <LoginForm />);

    const submitButton = getByText("Submit");
    fireEvent.click(submitButton);

    expect(getByText("Username is required")).toBeTruthy();
    expect(getByText("Password is required")).toBeTruthy();
  });

  it("should submit valid form", async () => {
    const onSubmit = vi.fn();
    const { getByTestId, getByText } = render(() => (
      <LoginForm onSubmit={onSubmit} />
    ));

    // Fill form
    const usernameInput = getByTestId("username-input");
    const passwordInput = getByTestId("password-input");

    fireEvent.input(usernameInput, { target: { value: "user" } });
    fireEvent.input(passwordInput, { target: { value: "password123" } });

    // Submit
    fireEvent.click(getByText("Submit"));

    expect(onSubmit).toHaveBeenCalledWith({
      username: "user",
      password: "password123",
    });
  });
});
```

## Mocking

### Mocking External Dependencies

```typescript
import { mock } from "bun:test";

const mockApi = {
  fetchUsers: mock(() =>
    Promise.resolve([
      { id: "1", name: "User 1" },
      { id: "2", name: "User 2" },
    ])
  ),
};

describe("UserList", () => {
  it("should display fetched users", async () => {
    const { getByText, findByText } = render(() => (
      <UserList api={mockApi} />
    ));

    // Wait for loading
    await findByText("User 1");

    expect(getByText("User 1")).toBeTruthy();
    expect(getByText("User 2")).toBeTruthy();
    expect(mockApi.fetchUsers).toHaveBeenCalled();
  });
});
```

### Mocking File System

```typescript
import { mock } from "bun:test";
import * as fs from "fs";

mock.module("fs", () => ({
  readFileSync: mock(() => "mocked content"),
  writeFileSync: mock(),
  existsSync: mock(() => true),
}));

describe("FileManager", () => {
  it("should read file content", () => {
    const { getByText } = render(() => <FileViewer path="/test.txt" />);

    expect(getByText("mocked content")).toBeTruthy();
    expect(fs.readFileSync).toHaveBeenCalledWith("/test.txt", "utf-8");
  });
});
```

## Snapshot Testing

```typescript
import { renderToString } from "@opentui/solid";

describe("Header", () => {
  it("should match snapshot", () => {
    const output = renderToString(() => <Header title="Test" />);

    expect(output).toMatchSnapshot();
  });
});
```

## E2E Testing

### Testing CLI Commands

```typescript
import { spawn } from "child_process";

const runCli = (args: string[]): Promise<{ stdout: string; exitCode: number }> => {
  return new Promise((resolve) => {
    const process = spawn("bun", ["run", "cli.ts", ...args]);
    let stdout = "";

    process.stdout.on("data", (data) => {
      stdout += data.toString();
    });

    process.on("close", (exitCode) => {
      resolve({ stdout, exitCode: exitCode ?? 0 });
    });
  });
};

describe("CLI", () => {
  it("should show help with --help", async () => {
    const { stdout, exitCode } = await runCli(["--help"]);

    expect(exitCode).toBe(0);
    expect(stdout).toContain("Usage:");
    expect(stdout).toContain("Options:");
  });

  it("should exit with error for invalid command", async () => {
    const { exitCode, stdout } = await runCli(["invalid"]);

    expect(exitCode).toBe(1);
    expect(stdout).toContain("Unknown command");
  });
});
```

### Interactive Testing

```typescript
import { spawn } from "child_process";
import { Writable } from "stream";

const interactiveCli = () => {
  const process = spawn("bun", ["run", "cli.ts"], {
    stdio: ["pipe", "pipe", "pipe"],
  });

  return {
    write: (input: string) => process.stdin.write(input),
    read: (): Promise<string> =>
      new Promise((resolve) => {
        process.stdout.once("data", (data) => resolve(data.toString()));
      }),
    close: () => process.kill(),
  };
};

describe("Interactive CLI", () => {
  it("should navigate list with arrow keys", async () => {
    const cli = interactiveCli();

    // Wait for initial render
    const initial = await cli.read();
    expect(initial).toContain("> Item 1");

    // Press down arrow
    cli.write("\x1B[B"); // Down arrow escape sequence

    const after = await cli.read();
    expect(after).toContain("> Item 2");

    cli.close();
  });
});
```

## Test Utilities

```typescript
// test-utils.ts
export const wait = (ms: number) =>
  new Promise((resolve) => setTimeout(resolve, ms));

export const pressKey = (key: string) => {
  fireEvent.keyDown(document, { key });
};

export const pressEnter = () => pressKey("Enter");
export const pressEscape = () => pressKey("Escape");
export const pressUp = () => pressKey("ArrowUp");
export const pressDown = () => pressKey("ArrowDown");

export const renderWithProviders = (component: () => JSX.Element) => {
  return render(() => (
    <AppProvider>
      <ThemeProvider>{component()}</ThemeProvider>
    </AppProvider>
  ));
};
```

## Best Practices

1. **Test behavior, not implementation** - Focus on what users see
2. **Use data-testid sparingly** - Prefer accessible queries
3. **Clean up after each test** - Prevent state leakage
4. **Mock external dependencies** - Keep tests fast and isolated
5. **Test edge cases** - Empty states, errors, loading
6. **Use snapshot tests carefully** - Only for stable UI
7. **Test keyboard navigation** - Critical for CLI apps
8. **Test async operations** - Handle loading and error states
