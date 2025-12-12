# テストパターン・ベストプラクティス

## 目次

1. [テスト環境セットアップ](#テスト環境セットアップ)
2. [コンポーネントテスト](#コンポーネントテスト)
3. [モックパターン](#モックパターン)
4. [統合テスト](#統合テスト)

## テスト環境セットアップ

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

## コンポーネントテスト

### 基本パターン

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

### キー入力テスト

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

    // 初期状態
    expect(lastFrame()).toContain("› Item 1");

    // j キーで下に移動
    stdin.write("j");

    // 選択が移動したことを確認
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

### 非同期テスト

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

    // 初期状態では表示されない
    expect(lastFrame()).toBe("");

    // 300ms 後に表示
    vi.advanceTimersByTime(300);
    expect(lastFrame()).toContain("Loading");
  });
});
```

## モックパターン

### 外部依存のモック

```typescript
vi.mock("../../../git.js", () => ({
  getRepositoryRoot: vi.fn().mockResolvedValue("/repo"),
  getAllBranches: vi.fn().mockResolvedValue([]),
  fetchAllRemotes: vi.fn().mockResolvedValue(undefined),
}));
```

### Screenコンポーネントのモック

テスト対象以外のScreenをモック化:

```typescript
// Props をキャプチャするための配列
const capturedProps: BranchListScreenProps[] = [];

vi.mock("../../components/screens/BranchListScreen.js", () => ({
  BranchListScreen: (props: BranchListScreenProps) => {
    capturedProps.push(props);
    return <div>Mocked BranchListScreen</div>;
  },
}));

describe("App", () => {
  beforeEach(() => {
    capturedProps.length = 0;  // リセット
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

### フックのモック

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

### 条件付きモック応答

```typescript
const mockGetBranches = vi.fn();

vi.mock("../../../git.js", () => ({
  getAllBranches: mockGetBranches,
}));

describe("useGitData", () => {
  it("should handle error", async () => {
    mockGetBranches.mockRejectedValueOnce(new Error("Git error"));

    // エラーが設定されることを確認
  });

  it("should return branches", async () => {
    mockGetBranches.mockResolvedValueOnce([
      { name: "main" },
    ]);

    // ブランチが返されることを確認
  });
});
```

## 統合テスト

### ナビゲーションテスト

```typescript
describe("Navigation", () => {
  it("should navigate from branch list to action selector", async () => {
    const { stdin, lastFrame } = render(<App />);

    // ブランチリストが表示されるのを待つ
    await waitFor(() => {
      expect(lastFrame()).toContain("Branch List");
    });

    // Enter で選択
    stdin.write("\r");

    // アクションセレクタに遷移
    await waitFor(() => {
      expect(lastFrame()).toContain("Action Selector");
    });
  });

  it("should go back on Escape", async () => {
    const { stdin, lastFrame } = render(<App />);

    // 遷移
    stdin.write("\r");
    await waitFor(() => {
      expect(lastFrame()).toContain("Action Selector");
    });

    // Escape で戻る
    stdin.write("\x1B");  // Escape key

    await waitFor(() => {
      expect(lastFrame()).toContain("Branch List");
    });
  });
});
```

### エッジケーステスト

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

    // 表示が壊れていないことを確認
    expect(lastFrame()).toBeDefined();
  });
});
```

### パフォーマンステスト

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

    // 100回の移動
    for (let i = 0; i < 100; i++) {
      stdin.write("j");
    }

    const duration = performance.now() - start;

    // 1秒以内に完了すること
    expect(duration).toBeLessThan(1000);
  });
});
```

## テストユーティリティ

### waitFor ヘルパー

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
      // 条件が満たされない
    }
    await new Promise((r) => setTimeout(r, 10));
  }

  throw new Error("waitFor timed out");
}
```

### テスト用のキー定数

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
