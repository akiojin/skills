# カスタムフック設計ガイド

## 目次

1. [設計原則](#設計原則)
2. [useScreenState](#usescreenstate)
3. [useTerminalSize](#useterminalsize)
4. [useGitData（データ取得パターン）](#usegitdata)
5. [useInput活用パターン](#useinput活用パターン)

## 設計原則

### 単一責務

各フックは1つの関心事のみを担当する。

```typescript
// Good: 単一責務
function useScreenState() { ... }  // 画面遷移のみ
function useTerminalSize() { ... } // サイズ監視のみ

// Bad: 複数の責務
function useAppState() {
  // 画面遷移、データ取得、ターミナルサイズ...
}
```

### 戻り値オブジェクト

関連する状態と関数をオブジェクトでまとめて返す。

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

### クリーンアップ

`useEffect`内で必ずクリーンアップ関数を返す。

```typescript
useEffect(() => {
  const handler = () => { ... };
  process.stdout.on("resize", handler);

  // クリーンアップ
  return () => {
    process.stdout.removeListener("resize", handler);
  };
}, []);
```

### 依存配列の正確性

`useCallback`/`useEffect`/`useMemo`の依存配列を正確に指定する。

```typescript
const loadData = useCallback(async () => {
  setLoading(true);
  const data = await fetchData(options);  // options を使用
  setData(data);
  setLoading(false);
}, [options]);  // options を依存に含める
```

## useScreenState

スタック型の画面ナビゲーション管理。

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
        return prev;  // 初期画面から戻れない
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

### 使用例

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

ターミナルサイズの取得とリサイズ監視。

> **注**: CI/テスト環境など非TTY環境では`process.stdout.rows`/`columns`が`undefined`になります。必ずフォールバック値を設定してください。また、`process.stdout.isTTY`でTTY判定を行い、非TTY環境ではリサイズイベントをスキップすることを推奨します。

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

### 使用例

```typescript
function BranchListScreen() {
  const { rows, columns } = useTerminalSize();

  // 動的に表示行数を計算
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

非同期データ取得とキャッシュのパターン。

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
      // 並列でデータ取得
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

  // 初回ロード
  useEffect(() => {
    loadData();
  }, [loadData]);

  // 自動リフレッシュ
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

### キャンセルパターン

コンポーネントがアンマウントされた後のstate更新を防ぐ:

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

## useInput活用パターン

### 基本パターン

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

### 条件付きハンドリング

```typescript
function MyComponent({ disabled, mode }: Props) {
  useInput((input, key) => {
    // 無効化時は何もしない
    if (disabled) return;

    // モードによって処理を分岐
    if (mode === "edit") {
      // 編集モードの処理
    } else {
      // 通常モードの処理
    }
  });
}
```

### グローバルショートカット

```typescript
function App() {
  const [filterMode, setFilterMode] = useState(false);

  // グローバルショートカット（フィルターモード以外で有効）
  useInput((input, key) => {
    if (filterMode) return;  // フィルター入力中は無効

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

### スピナーアニメーション

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

// 使用
function LoadingIndicator({ loading }: { loading: boolean }) {
  const frame = useSpinner(loading);

  if (!loading) return null;

  return <Text>{frame} Loading...</Text>;
}
```
