---
name: cli-design
description: |
  Ink.jsを使用したターミナルCLI UIの設計・実装ガイド。
  以下の場合に使用:
  (1) Ink.jsコンポーネントの新規作成・改修
  (2) useInput/useApp等のInk固有フックの実装
  (3) 絵文字/アイコンの幅管理問題への対応（string-width対策）
  (4) ターミナルサイズ対応・レスポンシブレイアウト
  (5) Enter/キー入力の問題解決
  (6) 日本語カーソル移動問題への対応
  (7) Ctrl+Cハンドリング
  (8) CLI UIのテスト設計（ink-testing-library）
  (9) パフォーマンス最適化（React.memo/カスタム比較関数）
---

# CLI Design

Ink.jsを用いたターミナルUI設計のガイド。

## クイックスタート

### 新規コンポーネント作成

1. コンポーネント種別を決定: Screen / Part / Common
2. [component-patterns.md](references/component-patterns.md) から類似パターンを参照
3. 型定義を `types.ts` に追加
4. コンポーネント実装
5. テスト作成

### よくある問題の解決

| 問題 | 参照 |
|------|------|
| 絵文字の幅がずれる | [ink-gotchas.md#アイコン絵文字の幅問題](references/ink-gotchas.md) |
| Ctrl+Cが2回呼ばれる | [ink-gotchas.md#ctrlcハンドリング](references/ink-gotchas.md) |
| useInputが競合する | [ink-gotchas.md#useinput競合](references/ink-gotchas.md) |
| 日本語でカーソルがずれる | [ink-gotchas.md#日本語カーソル移動](references/ink-gotchas.md) |
| レイアウトが崩れる | [ink-gotchas.md#レイアウト崩れ](references/ink-gotchas.md) |

## ディレクトリ規約

```
src/cli/ui/
├── components/
│   ├── App.tsx              # ルートコンポーネント
│   ├── common/              # 汎用コンポーネント（Select, Input, Confirm）
│   ├── parts/               # UI部品（Header, Footer, ProgressBar）
│   └── screens/             # 画面コンポーネント
├── hooks/                   # カスタムフック
├── utils/                   # ユーティリティ関数
└── types.ts                 # 型定義
```

## コンポーネント分類

### Screen（画面）

- 完全な画面を表す
- `useInput`でキーボード入力を処理
- Header/Content/Footerの3層レイアウト

### Part（部品）

- 再利用可能なUI部品
- `React.memo`で最適化
- 状態を持たない純粋コンポーネント

### Common（共通）

- 基本的な入力コンポーネント
- 制御/非制御両モードをサポート

## 重要パターン

### 1. アイコン幅オーバーライド

string-width v8で絵文字の幅計算がずれる問題の対策:

```typescript
const iconWidthOverrides: Record<string, number> = {
  "⚡": 1, "✨": 1, "🐛": 1, "🔥": 1, "🚀": 1,
  "🟢": 1, "🟠": 1, "✅": 1, "⚠️": 1,
};

const getIconWidth = (icon: string): number => {
  const baseWidth = stringWidth(icon);
  const override = iconWidthOverrides[icon];
  return override !== undefined ? Math.max(baseWidth, override) : baseWidth;
};
```

### 2. useInput競合回避

複数のuseInputフックが存在する場合、すべてのハンドラーが呼び出される:

```typescript
useInput((input, key) => {
  if (disabled) return;  // 無効化時は何もしない
  // キー処理...
});
```

### 3. Ctrl+Cハンドリング

```typescript
render(<App />, { exitOnCtrlC: false });

// コンポーネント内
useInput((input, key) => {
  if (key.ctrl && input === "c") {
    cleanup();
    exit();
  }
});
```

### 4. 動的高さ計算

```typescript
const { rows } = useTerminalSize();
const fixedLines = headerLines + footerLines;
const contentHeight = rows - fixedLines;
const listLimit = Math.max(5, contentHeight);
```

### 5. React.memo + カスタム比較

```typescript
function arePropsEqual<T>(prev: Props<T>, next: Props<T>): boolean {
  if (prev.items.length !== next.items.length) return false;
  for (let i = 0; i < prev.items.length; i++) {
    if (prev.items[i].value !== next.items[i].value) return false;
  }
  return true;
}

export const Select = React.memo(SelectComponent, arePropsEqual);
```

## 詳細リファレンス

- **Ink.js注意点・よくある問題**: [ink-gotchas.md](references/ink-gotchas.md)
- **コンポーネント設計パターン**: [component-patterns.md](references/component-patterns.md)
- **カスタムフック設計ガイド**: [hooks-guide.md](references/hooks-guide.md)
- **テストパターン**: [testing-patterns.md](references/testing-patterns.md)
