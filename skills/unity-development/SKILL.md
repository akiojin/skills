---
name: unity-development
description: Unity C#コードの探索・編集・テスト実行のためのスキル。unity-mcp-serverツールを使用したシンボルベースのコード操作、VContainer DI、UniTask非同期処理、Fail-Fast原則に従った実装を支援。Unityスクリプト修正、コンポーネント追加、シーン操作、PlayModeテスト実行時に使用。
---

# Unity Development

Unity C#コードの効率的な探索・編集・テスト実行を支援するスキル。

## 使用タイミング

- Unity C#スクリプトの修正・作成
- GameObjectやコンポーネントの操作
- シーン操作・プレハブ管理
- Unityテスト（EditMode/PlayMode）の実行
- コンパイルエラーの解決
- VContainer DIの設定
- UniTask非同期処理の実装

## 必須ルール

### 1. unity-mcp-serverツールの使用

**Read/Edit/Writeは使用禁止。必ずunity-mcp-serverツールを使用する。**

```
# コード探索
mcp__unity-mcp-server__script_symbols_get     # ファイル構造把握
mcp__unity-mcp-server__script_symbol_find     # シンボル検索
mcp__unity-mcp-server__script_read            # 部分読み取り
mcp__unity-mcp-server__script_refs_find       # 参照検索

# コード編集
mcp__unity-mcp-server__script_edit_structured # シンボル単位編集
mcp__unity-mcp-server__script_edit_snippet    # 部分編集
mcp__unity-mcp-server__script_create_class    # クラス作成

# 変更後は必ず実行
mcp__unity-mcp-server__system_refresh_assets  # コンパイル実行
```

### 2. Fail-Fast原則

**以下のコードは絶対に生成しない：**

```csharp
// 禁止パターン
if (component != null) { ... }     // GetComponent後のnullチェック
if (gameObject != null) { ... }    // Find後のnullチェック
if (service != null) { ... }       // [Inject]後のnullチェック
```

**正しいコード：**

```csharp
// 直接使用（存在前提）
GetComponent<Rigidbody>().velocity = Vector3.zero;
GameService.Initialize();
target.position = Vector3.zero;
```

### 3. Update内GetComponent禁止

```csharp
// 禁止
void Update() {
    GetComponent<Rigidbody>().velocity = input; // 毎フレームGC発生
}

// 正しい
private Rigidbody _rb;
void Awake() { _rb = GetComponent<Rigidbody>(); }
void Update() { _rb.velocity = input; }
```

## ワークフロー

### コード修正

1. **構造確認**: `script_symbols_get` でファイル構造を把握
2. **影響範囲**: `script_refs_find` で参照を確認
3. **編集実行**: `script_edit_structured` または `script_edit_snippet`
4. **コンパイル**: `system_refresh_assets` で変更を反映
5. **エラー確認**: `compilation_get_state` でエラーチェック

### テスト実行

```
# EditModeテスト
mcp__unity-mcp-server__test_run(testMode="EditMode")

# PlayModeテスト（接続不安定に注意）
mcp__unity-mcp-server__test_run(testMode="PlayMode")
# ドメインリロードで3秒以上待機、最大10回リトライ
```

### シーン・GameObject操作

```
# 階層取得
mcp__unity-mcp-server__gameobject_get_hierarchy

# コンポーネント操作
mcp__unity-mcp-server__component_add
mcp__unity-mcp-server__component_modify
mcp__unity-mcp-server__component_list
```

## パス規約

- 自作スクリプト: `Assets/@Xyla/Scripts/`
- テスト: `Assets/@Xyla/Scripts/Tests/`
- 名前空間: `namespace Xyla`（通常）、`namespace Xyla.Editor`（エディター）

## UniTask

コルーチンの代わりにUniTaskを使用。`async void`は禁止、`UniTaskVoid`を使用。

```csharp
using Cysharp.Threading.Tasks;

// destroyCancellationTokenで安全なキャンセル
async UniTaskVoid Start()
{
    await DoWorkAsync(destroyCancellationToken);
}
```

詳細は [references/unitask.md](references/unitask.md) 参照。

## VContainer

詳細は [references/vcontainer.md](references/vcontainer.md) 参照。

## トラブルシューティング

詳細は [references/troubleshooting.md](references/troubleshooting.md) 参照。
