# SND 实体模型

> [↑ 回到 usage](README.md)

## 模型

```
Strategy + Node + Data
```

SND 模型将游戏实体拆分为三个正交维度：

| 维度 | 用途 | 存储位置 | 可变性 |
|------|------|---------|--------|
| **Strategy** | 行为逻辑 | 策略池（全局复用） | 不可变（无状态共享） |
| **Node** | 表现层映射 | 引擎场景树 | 由引擎管理 |
| **Data** | 可变状态 | SndDataManager (TypedData) | 可变（唯一权威来源） |

## 策略系统

### 根基类

```
BaseStrategy (抽象，所有策略的基类)
├── EntityStrategyBase  → 8 个实体生命周期钩子
└── StateMachineStrategyBase → 4 个状态机钩子
```

### 实体策略生命周期钩子

钩子按执行顺序排列：

| 钩子 | 触发时机 | 典型用途 |
|------|---------|---------|
| `AfterSpawn(entity, ctx)` | 新实体生成后 | 初始化属性（hp, position 等） |
| `AfterLoad(entity, ctx)` | 从存档恢复后 | 恢复后重新注册事件等 |
| `AfterAdd(entity, ctx)` | 策略动态添加到实体 | 动态添加状态逻辑 |
| `Process(entity, delta, ctx)` | 每帧 | 持续逻辑（移动、计时等） |
| `BeforeRemove(entity, ctx)` | 策略被移除前 | 清理订阅/资源 |
| `BeforeSave(entity, ctx)` | 序列化存档前 | 刷新待写入数据 |
| `BeforeQuit(entity, ctx)` | 实体正常退出 | 保存进度、清理 |
| `BeforeDead(entity, ctx)` | 实体销毁前 | 死亡特效、掉落等 |

### 编写策略

```csharp
[StrategyIndex("my_game.damage_tick", Priority = 100)]
public class DamageTickStrategy : EntityStrategyBase
{
    public override void Process(ISndEntity entity, double delta, ISndContext ctx)
    {
        var (found, hp) = entity.TryGetData<float>("hp");
        if (!found) return;

        entity.SetData("hp", hp - 5f * (float)delta);

        if (hp <= 0)
            ctx.RequestSaveGame("player_died");
    }
}
```

### 策略池规则

- **无状态强制**：策略类不得声明实例字段或可写属性（注册时反射校验）
- **注册方式**：`[StrategyIndex("xxx.yyy")]` 特性 + 程序集扫描
- **索引命名**：点分命名空间 + 小写蛇形分段（如 `core.player.health`）
- **优先级**：`Priority` 属性决定同实体上多策略的执行序（默认 6205，越小越先执行）
- **引用计数**：同一策略被多实体引用时计数 +1，全部释放后才回收

### 策略中禁止的行为

- 持有 `IDisposable` 字段或非托管资源
- 缓存跨帧可变上下文对象
- 声明实例字段存储运行时数据（应存入实体 Data）

## 实体数据

### TypedData

Core 的核心类型保留机制：

```csharp
public sealed class TypedData {
    public Type DataType { get; }  // 如 typeof(int)
    public object? Data { get; }   // 如 42
}
```

- **存储时**：`new TypedData(typeof(T), value)` 在泛型 SetData 中自动捕获类型
- **读取时**：`typedData.Data is T value` 模式匹配校验运行时类型
- **序列化时**：`TypedDataConverter` 将类型转字符串，通过 `TypeStringMapping` 双向映射

### 安全读取

```csharp
// ❌ 错误：值类型的 default 无法区分"不存在"和"值为0"
int hp = entity.GetData<int>("hp");

// ✅ 正确：先判断 found 再取值
var (found, hp) = entity.TryGetData<int>("hp");
if (found) { /* 使用 hp */ }
```

### 数据观察者

```csharp
entity.Subscribe("hp", (target, oldValue, newValue) =>
{
    if (newValue is int hp && hp <= 0)
        ctx.RequestSaveGame("entity_died");
});
```

订阅在键对应的数据变更时触发。可在回调中增删策略。

## 策略执行顺序

同实体上的多个策略按 `Priority` 升序执行，同优先级按添加顺序：

```
Priority: 10  →  Strategy A  (先执行)
Priority: 50  →  Strategy B
Priority: 100 →  Strategy C
Priority: 6205 (默认) → Strategy D
```

所有钩子（Process / AfterSpawn / etc.）均遵循此顺序。

## 实体元数据

实体的序列化格式：

```json
{
  "name": "player",
  "node": {
    "pairs": {
      "main": "res://scenes/player.tscn"
    }
  },
  "strategy": {
    "indices": ["my_game.health", "my_game.movement"]
  },
  "data": {
    "pairs": {
      "hp": { "type": "Int32", "data": 100 },
      "pos": { "type": "Vector2", "data": { "x": 10, "y": 20 } }
    }
  }
}
```

## 下一个文档

- [会话模型](session-model.md) — 前台/后台会话
- [持久化流程](persistence-flow.md) — 存档/读档
- [策略测试](strategy-testing.md) — StrategyTestScenario

---
[↑ 回到 usage](README.md)
