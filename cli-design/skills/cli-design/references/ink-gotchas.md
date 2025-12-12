# Ink.js 注意点・よくある問題

## 目次

1. [アイコン/絵文字の幅問題](#1-アイコン絵文字の幅問題)
2. [Ctrl+Cハンドリング](#2-ctrlcハンドリング)
3. [useInput競合](#3-useinput競合)
4. [Enter二度押し問題](#4-enter二度押し問題)
5. [日本語カーソル移動](#5-日本語カーソル移動)
6. [レイアウト崩れ](#6-レイアウト崩れ)
7. [console.log干渉](#7-consolelog干渉)
8. [Inkの色型エラー](#8-inkの色型エラー)

## 1. アイコン/絵文字の幅問題

### 問題

`string-width` v8以降、Variation Selector (VS15/VS16) の扱いが変わり、絵文字の幅計算がずれる。
ターミナルでは絵文字が2文字分の幅を取ることがあるが、`stringWidth()`は1を返すことがある。

### 解決策: WIDTH_OVERRIDESパターン

```typescript
import stringWidth from "string-width";

const iconWidthOverrides: Record<string, number> = {
  "⚡": 1,
  "✨": 1,
  "🐛": 1,
  "🔥": 1,
  "🚀": 1,
  "📌": 1,
  "🟢": 1,
  "🟠": 1,
  "👉": 1,
  "💾": 1,
  "📤": 1,
  "🔃": 1,
  "✅": 1,
  "⚠️": 1,
  "🔗": 1,
  "💻": 1,
  "☁️": 1,
};

const getIconWidth = (icon: string): number => {
  const baseWidth = stringWidth(icon);
  const override = iconWidthOverrides[icon];
  return override !== undefined ? Math.max(baseWidth, override) : baseWidth;
};

// 固定幅カラムへのパディング
const COLUMN_WIDTH = 2;
const padIcon = (icon: string): string => {
  const width = getIconWidth(icon);
  const padding = Math.max(0, COLUMN_WIDTH - width);
  return icon + " ".repeat(padding);
};
```

### 影響箇所

- ブランチリスト表示
- ステータスアイコン表示
- プログレスバー
- すべての固定幅カラム

## 2. Ctrl+Cハンドリング

### 問題

Inkはデフォルトで`Ctrl+C`を処理してアプリを終了させるが、これが2回呼ばれたり、
カスタム終了処理と競合することがある。

### 解決策: exitOnCtrlC: false + useInput + sigint

```typescript
import { render, useApp, useInput } from "ink";
import process from "node:process";

function App({ onExit }: { onExit: () => void }) {
  const { exit } = useApp();

  useInput((input, key) => {
    if (key.ctrl && input === "c") {
      onExit();
      exit();
    }
  });

  // SIGINT も処理（Ctrl+Cの別シグナル）
  useEffect(() => {
    const handler = () => {
      onExit();
      exit();
    };
    process.on("SIGINT", handler);
    return () => process.off("SIGINT", handler);
  }, [exit, onExit]);

  return <Box>...</Box>;
}

// render時に exitOnCtrlC を無効化
render(<App onExit={cleanup} />, { exitOnCtrlC: false });
```

## 3. useInput競合

### 問題

複数の`useInput`フックが存在すると、**すべてのハンドラーが呼び出される**。
親コンポーネントと子コンポーネントの両方で同じキーを処理してしまう。

### 解決策

**パターン1: disabled prop**

```typescript
useInput((input, key) => {
  if (disabled) return;  // 無効化時は何もしない
  // キー処理...
});
```

**パターン2: モードフラグ**

```typescript
const [filterMode, setFilterMode] = useState(false);

useInput((input, key) => {
  if (filterMode) return;  // フィルター入力中はグローバルショートカット無効
  if (input === "c") onCleanupCommand?.();
});
```

**パターン3: blockKeys prop（Inputコンポーネント）**

```typescript
// Inputコンポーネント内
useInput((input) => {
  if (blockKeys && blockKeys.includes(input)) {
    return;  // キーを消費して親へ伝播させない
  }
});

// 使用時
<Input blockKeys={["c", "r", "f"]} ... />
```

## 4. Enter二度押し問題

### 問題

Selectコンポーネントで選択後、次の画面でもEnterが反応してしまう。
初回のEnterイベントが伝播している。

### 解決策: 初回受付バッファリング

```typescript
const [ready, setReady] = useState(false);

useEffect(() => {
  // 最初のレンダリング後にreadyをtrueに
  const timer = setTimeout(() => setReady(true), 50);
  return () => clearTimeout(timer);
}, []);

useInput((input, key) => {
  if (!ready) return;  // 初期化完了まで入力を無視
  if (key.return) {
    onSelect(selectedItem);
  }
});
```

## 5. 日本語カーソル移動

### 問題

日本語文字は表示幅2だが、文字数は1。
カーソル位置を文字数で計算すると、日本語入力時にカーソルがずれる。

### 解決策: 表示幅ベースの位置計算

```typescript
// 文字の表示幅を取得
function getCharWidth(char: string): number {
  const code = char.codePointAt(0);
  if (!code) return 1;

  // CJK文字、絵文字などは幅2
  if (
    (code >= 0x1100 && code <= 0x115F) ||  // ハングル
    (code >= 0x2E80 && code <= 0x9FFF) ||  // CJK
    (code >= 0xAC00 && code <= 0xD7AF) ||  // ハングル
    (code >= 0xF900 && code <= 0xFAFF) ||  // CJK互換
    (code >= 0xFE10 && code <= 0xFE1F) ||  // 縦書きフォーム
    (code >= 0x1F300 && code <= 0x1F9FF)   // 絵文字
  ) {
    return 2;
  }
  return 1;
}

// 文字列の表示幅を計算
function getDisplayWidth(str: string): number {
  return [...str].reduce((width, char) => width + getCharWidth(char), 0);
}

// カーソル位置（文字数）から表示幅へ変換
function toDisplayColumn(text: string, charPosition: number): number {
  return getDisplayWidth(text.slice(0, charPosition));
}
```

## 6. レイアウト崩れ

### 問題

- 長い行がターミナル幅を超えて折り返される
- タイムスタンプなどが次の行に押し出される
- 列の位置がずれる

### 解決策

**安全マージンを設ける**

```typescript
const { stdout } = useStdout();
const columns = Math.max(20, (stdout?.columns ?? 80) - 1);  // 1文字分のマージン

// または90%幅（Gemini CLIパターン）
const safeColumns = Math.floor(columns * 0.9);
```

**固定幅カラムを使用**

```typescript
const COLUMN_WIDTH = 2;
const padToWidth = (content: string, width: number): string => {
  const actualWidth = stringWidth(content);
  const padding = Math.max(0, width - actualWidth);
  return content + " ".repeat(padding);
};
```

**文字列を切り詰め**

```typescript
function truncateToWidth(text: string, maxWidth: number): string {
  let width = 0;
  let result = "";
  for (const char of text) {
    const charWidth = getCharWidth(char);
    if (width + charWidth > maxWidth) {
      return result + "…";
    }
    width += charWidth;
    result += char;
  }
  return result;
}
```

## 7. console.log干渉

### 問題

`console.log`の出力がInkのレンダリングと競合し、UIが乱れる。
Inkは定期的に画面を再描画するため、ログ出力が上書きされる。

### 解決策

**構造化ログを別ストリームに出力**

```typescript
import pino from "pino";

// ログは stderr に出力
const logger = pino({
  transport: {
    target: "pino-pretty",
    options: { destination: 2 }  // stderr
  }
});
```

**条件付きログ**

```typescript
if (process.env.DEBUG) {
  console.error("Debug:", data);  // stderr を使用
}
```

## 8. Inkの色型エラー

### 問題

TypeScriptで`<Text color="cyan">`などを使うと、色の型が厳密でエラーになることがある。

### 解決策

```typescript
// 型アサーションを使用
<Text color={"cyan" as const}>...</Text>

// または変数で定義
const selectedColor = "cyan" as const;
<Text color={selectedColor}>...</Text>

// chalk を使う方法も
import chalk from "chalk";
<Text>{chalk.cyan("Selected")}</Text>
```
