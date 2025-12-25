# コンポーネント設計パターン

## 目次

1. [ディレクトリ構造](#ディレクトリ構造)
2. [コンポーネント分類](#コンポーネント分類)
3. [3層レイアウト](#3層レイアウト)
4. [React.memo最適化](#reactmemo最適化)
5. [制御/非制御モード](#制御非制御モード)

## ディレクトリ構造

```
src/cli/ui/
├── components/
│   ├── App.tsx              # ルートコンポーネント（スクリーン管理）
│   ├── common/              # 汎用コンポーネント
│   │   ├── Select.tsx       # 選択リスト
│   │   ├── Input.tsx        # テキスト入力
│   │   ├── Confirm.tsx      # Yes/No確認
│   │   ├── LoadingIndicator.tsx
│   │   └── ErrorBoundary.tsx
│   ├── parts/               # UI部品
│   │   ├── Header.tsx
│   │   ├── Footer.tsx
│   │   ├── ProgressBar.tsx
│   │   ├── Stats.tsx
│   │   └── ScrollableList.tsx
│   └── screens/             # 画面コンポーネント
│       ├── BranchListScreen.tsx
│       ├── ModelSelectorScreen.tsx
│       └── BranchCreatorScreen.tsx
├── hooks/                   # カスタムフック
├── utils/                   # ユーティリティ関数
├── types.ts                 # 型定義
└── __tests__/               # テスト
```

## コンポーネント分類

### Screen（画面）

完全な画面を表すコンポーネント。

**責務:**
- `useInput`でキーボード入力を処理
- Header/Content/Footerの3層レイアウト
- 画面固有のビジネスロジック

**命名:** `<概念>Screen.tsx`

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

### Part（部品）

再利用可能なUI部品。状態を持たない純粋コンポーネント。

**責務:**
- 表示のみ
- `React.memo`で最適化
- propsからすべてのデータを受け取る

**命名:** `<機能>.tsx`

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

### Common（共通）

基本的な入力コンポーネント。

**責務:**
- 制御/非制御両モードをサポート
- 汎用的なインターフェース
- 他のプロジェクトでも再利用可能

**命名:** `<基本型>.tsx`

## 3層レイアウト

標準的な画面レイアウトパターン。

```typescript
export function StandardScreen({ children }: PropsWithChildren) {
  const { rows } = useTerminalSize();

  return (
    <Box flexDirection="column" height={rows}>
      {/* Layer 1: Header - 固定高さ */}
      <Header title="App Title" />

      {/* Layer 2: Content - flexGrow で残りの高さを埋める */}
      <Box flexDirection="column" flexGrow={1}>
        {children}
      </Box>

      {/* Layer 3: Footer - 固定高さ */}
      <Footer actions={footerActions} />
    </Box>
  );
}
```

### 動的高さ計算

```typescript
const { rows } = useTerminalSize();

// 固定部分の行数を計算
const headerLines = 2;    // ボーダー + タイトル
const filterLines = 1;    // フィルター入力
const statsLines = 1;     // 統計情報
const footerLines = 1;    // フッター

const fixedLines = headerLines + filterLines + statsLines + footerLines;
const contentHeight = rows - fixedLines;
const listLimit = Math.max(5, contentHeight);  // 最低5行は確保

return (
  <Select
    items={items}
    limit={listLimit}
    onSelect={handleSelect}
  />
);
```

## React.memo最適化

### 基本パターン

```typescript
export const Header = React.memo(function Header(props: HeaderProps) {
  return <Box>...</Box>;
});
```

### カスタム比較関数

> **重要**: カスタム比較関数を使用する場合、コールバックprops（`onSelect`など）の参照が安定していることが前提です。親コンポーネントで`useCallback`を使用してコールバックをメモ化してください。

配列の参照ではなくコンテンツで比較する場合:

```typescript
function arePropsEqual<T extends SelectItem>(
  prevProps: SelectProps<T>,
  nextProps: SelectProps<T>,
): boolean {
  // 単純な値の比較
  if (
    prevProps.limit !== nextProps.limit ||
    prevProps.disabled !== nextProps.disabled ||
    prevProps.onSelect !== nextProps.onSelect
  ) {
    return false;
  }

  // 配列の長さを比較
  if (prevProps.items.length !== nextProps.items.length) {
    return false;
  }

  // 配列の内容を比較
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

## 制御/非制御モード

Selectなどのコンポーネントで、親から状態を制御できるようにする。

```typescript
interface SelectProps<T> {
  items: T[];
  onSelect: (item: T) => void;
  // 非制御モード用
  initialIndex?: number;
  // 制御モード用
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
  // 内部状態（非制御モード用）
  const [internalSelectedIndex, setInternalSelectedIndex] =
    useState(initialIndex);

  // 制御モードか非制御モードかを判定
  const isControlled = externalSelectedIndex !== undefined;
  const selectedIndex = isControlled
    ? externalSelectedIndex
    : internalSelectedIndex;

  // 統一された更新関数
  const updateSelectedIndex = (
    value: number | ((prev: number) => number)
  ) => {
    const newIndex = typeof value === "function"
      ? value(selectedIndex)
      : value;

    // 非制御モードの場合は内部状態を更新
    if (!isControlled) {
      setInternalSelectedIndex(newIndex);
    }

    // コールバックがあれば呼び出し
    if (onSelectedIndexChange) {
      onSelectedIndexChange(newIndex);
    }
  };

  // ...
}
```

## ループなしナビゲーション

リストの端で止まる（ループしない）パターン:

```typescript
useInput((input, key) => {
  if (key.upArrow || input === "k") {
    // 上に移動（0で止まる）
    updateSelectedIndex((current) => Math.max(0, current - 1));
  } else if (key.downArrow || input === "j") {
    // 下に移動（最後で止まる）
    updateSelectedIndex((current) =>
      Math.min(items.length - 1, current + 1)
    );
  }
});
```

## ErrorBoundaryパターン

> **注**: ErrorBoundaryはReactの制約上、クラスコンポーネントでのみ実装可能です。これは「関数コンポーネント + フック」の原則の唯一の例外です。

クラスコンポーネントで実装（フックでは不可）:

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
