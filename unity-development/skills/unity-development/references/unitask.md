# UniTask運用ガイド

## 基本原則

UniTaskはUnity向け高性能非同期ライブラリ。コルーチンの代わりに使用する。

```csharp
using Cysharp.Threading.Tasks;
using System.Threading;
```

## 基本パターン

### 非同期メソッド

```csharp
// 戻り値なし
public async UniTaskVoid StartAsync()
{
    await UniTask.Delay(1000);
}

// 戻り値あり
public async UniTask<int> LoadDataAsync()
{
    await UniTask.Delay(100);
    return 42;
}
```

### CancellationToken

```csharp
// MonoBehaviourでの使用
public class MyComponent : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        // destroyCancellationTokenでオブジェクト破棄時に自動キャンセル
        await DoWorkAsync(destroyCancellationToken);
    }

    async UniTask DoWorkAsync(CancellationToken token)
    {
        await UniTask.Delay(1000, cancellationToken: token);
    }
}
```

### フレーム待機

```csharp
await UniTask.Yield();                    // 1フレーム待機
await UniTask.NextFrame();                // 次フレーム待機
await UniTask.WaitForEndOfFrame();        // フレーム終了待機
await UniTask.WaitForFixedUpdate();       // FixedUpdate待機
```

### 時間待機

```csharp
await UniTask.Delay(1000);                          // 1秒（ミリ秒）
await UniTask.Delay(TimeSpan.FromSeconds(1));       // 1秒
await UniTask.DelayFrame(60);                       // 60フレーム
```

## 並列・直列実行

### 並列実行

```csharp
// すべて完了まで待機
await UniTask.WhenAll(
    LoadAssetAsync(),
    LoadDataAsync(),
    InitializeAsync()
);

// いずれか完了で継続
var result = await UniTask.WhenAny(task1, task2);
```

### 直列実行

```csharp
await InitializeAsync();
await LoadDataAsync();
await StartGameAsync();
```

## Unity API連携

### シーンロード

```csharp
await SceneManager.LoadSceneAsync("BattleScene");
```

### アセットロード

```csharp
var asset = await Resources.LoadAsync<GameObject>("Prefabs/Player");
```

### Addressables

```csharp
var handle = Addressables.LoadAssetAsync<GameObject>("Player");
await handle.ToUniTask();
```

## 禁止パターン

### async void禁止

```csharp
// 禁止: 例外がキャッチされない
async void BadMethod() { }

// 正しい: UniTaskVoid使用
async UniTaskVoid GoodMethod() { }
```

### Update内での動的取得禁止

```csharp
// 禁止: Update内でUniTask開始
void Update()
{
    DoAsync().Forget(); // 毎フレーム新しいTask生成
}

// 正しい: フラグで制御
private bool _isRunning;
void Update()
{
    if (!_isRunning && shouldStart)
    {
        _isRunning = true;
        DoAsync().Forget();
    }
}
```

## テストでの使用

```csharp
[Test]
public async Task TestAsyncMethod()
{
    var result = await MyService.LoadAsync();
    Assert.AreEqual(expected, result);
}
```
