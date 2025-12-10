# VContainer運用ガイド

## 基本設定

### LifetimeScope配置

各シーンに`LifetimeScope`を1つ配置し、`Auto Run`を有効化：

```csharp
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // サービス登録
        builder.Register<IGameService, GameService>(Lifetime.Singleton);

        // MonoBehaviour登録
        builder.RegisterComponentInHierarchy<PlayerController>();
    }
}
```

### Root LifetimeScope

1. ルートLifetimeScopeをプレハブ化
2. `VContainerSettings`の「Root Lifetime Scope」に登録
3. Project Settings > Player > Preloaded Assetsに設定

## MonoBehaviourへの注入

### 方法1: Auto Inject Game Objects

LifetimeScopeの`Auto Inject Game Objects`リストに対象を追加。

### 方法2: builder登録

```csharp
// シーン上のコンポーネント
builder.RegisterComponentInHierarchy<PlayerController>();

// プレハブから生成
builder.RegisterComponentInNewPrefab(playerPrefab, Lifetime.Scoped);
```

### 方法3: IObjectResolver.Instantiate

```csharp
[Inject]
private IObjectResolver _resolver;

void SpawnEnemy()
{
    var enemy = _resolver.Instantiate(enemyPrefab);
}
```

## 注意事項

- **Auto Run無効化禁止**: 無効だとコンテナ未構築、全Injectがnull
- **ヒエラルキー移動不要**: VContainerはシーン全体を探索
- **[Inject]後のnullチェック禁止**: Fail-Fast原則
