# トラブルシューティング

## unity-mcp-server接続問題

### タイムアウト発生時

1. Unityエディターをアクティブウィンドウにする
2. 5秒待機後に再試行
3. コンパイル中・アセットインポート中は待機

### PlayModeテスト不安定

ドメインリロードにより接続が切断される：

```
# 対策
1. PlayMode開始後、十分な待機時間を置く
2. 切断検知後、3秒以上のインターバル
3. 最大10回リトライ
4. 強制停止は避ける
```

## コンパイルエラー

### 名前空間エラー

```csharp
// 正しい名前空間
namespace Xyla              // 通常クラス
namespace Xyla.Editor       // Editorフォルダ内

// 禁止（サブ名前空間）
namespace Xyla.AI.Core      // NG
namespace Xyla.AI.Persona   // NG
```

### 静的解析エラー

- **SR0001**: Update内GetComponent → Awake/Startでキャッシュ
- **SR0002**: Find/FindObjectOfType → DI使用
- **SR0003**: GetComponent後nullチェック → 削除
- **SR0004**: [Inject]後nullチェック → 削除

## VContainer問題

### 全Injectがnull

1. LifetimeScope有効か確認
2. Auto Run有効か確認
3. Configure()で登録しているか確認

### MonoBehaviour未注入

```csharp
// 対策1: Auto Inject Game Objectsに追加
// 対策2: builder登録
builder.RegisterComponentInHierarchy<MyComponent>();
```

## シーン保存問題

LifetimeScope無効化後は必ず有効化してシーン保存。
