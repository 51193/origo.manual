# Strategy

> [↑ 回到 Snd](../README.md)

## 概述

SND 策略系统的完整实现。策略是实体行为逻辑的载体，遵循"无状态共享"模型：策略实例在池中全局共享，不持有实例字段，所有可变状态存储在实体的 Data 中。

策略分为三类：被动实体策略（帧驱动和生命周期钩子）、主动策略（外部按索引调用）、状态机策略（字符串栈状态管理）。三者共享 `BaseStrategy` + `SndStrategyPool` 基础设施，容器和管理器完全独立。

## 包含文件

| 文件 | 职责 |
|------|------|
| `BaseStrategy.cs` | 所有策略的抽象根基类 |
| `EntityStrategyBase.cs` | 实体策略基类：8 个生命周期虚方法钩子 |
| `ActiveStrategyBase.cs` | 主动策略基类：`Invoke(entity, ctx, input)` — 外部按索引主动调用 |
| `ActiveStrategyExtensions.cs` | `ISndEntity` 扩展方法：泛型 `InvokeStrategy<TInput, TOutput>` 消除 JSON 序列化样板；`EnsureStrategy` 惰性策略挂载 + 幂等守卫 |
| `ActiveStrategyManager.cs` | 单实体主动策略管理器：Dictionary 容器 + 增删 + 序列化 |
| `SndStrategyPool.cs` | 策略池：注册、实例化、引用计数、无状态校验 |
| `SndStrategyManager.cs` | 单实体被动策略管理器：策略容器的增删 + 生命周期钩子协调 |
| `StrategyIndexAttribute.cs` | 策略索引声明特性：`[StrategyIndex("core.health")]` |

## 模块详解

### 策略继承体系

```
BaseStrategy
├── EntityStrategyBase         (被动: Process, AfterSpawn, AfterLoad, AfterAdd, BeforeRemove, BeforeSave, BeforeQuit, BeforeDead)
├── ActiveStrategyBase         (主动: Invoke)
└── StateMachineStrategyBase   (状态机: OnPushRuntime, OnPushAfterLoad, OnPopRuntime, OnPopBeforeQuit)
```

### SndStrategyPool

策略的全局注册表和实例池：

- **Register**：`Register(Type strategyType, Func<BaseStrategy> factory)` —— 注册类型前通过反射校验无状态性（检查实例字段和可写属性）
- **GetStrategy<TBase>(index)**：若池中已有则复用（引用计数 +1），否则通过工厂创建
- **ReleaseStrategy(index)**：引用计数 -1，归零时从池中移除（但工厂保留，下次可再创建）
- **GetPriority(index)**：返回策略在实体上的执行优先级（默认 6205）
- **策略排序**：仅被动实体策略按优先级升序排列，同优先级按插入顺序

### SndStrategyManager

每个 `SndEntity` 持有一个 manager 实例，管理被动实体策略。对外暴露分阶段操作方法（均为 `internal`），由 `SndEntity` 通过 `IEntityLifecycle` 调用：

| 方法 | 说明 |
|------|------|
| `RecoverStrategiesOnly(indices)` | 从策略池按索引获取策略并排序插入（释放旧策略，不触发钩子） |
| `ReleaseStrategiesOnly()` | 释放全部策略引用并清空列表（不触发钩子） |
| `TriggerAfterSpawn(entity, ctx)` | 快照迭代触发 AfterSpawn |
| `TriggerAfterLoad(entity, ctx)` | 快照迭代触发 AfterLoad |
| `TriggerBeforeSave(entity, ctx)` | 快照迭代触发 BeforeSave |
| `TriggerBeforeQuit(entity, ctx)` | 快照迭代触发 BeforeQuit |
| `TriggerBeforeDead(entity, ctx)` | 快照迭代触发 BeforeDead |
| `GetStrategyIndices()` | 返回当前持有的所有策略索引 |
| `Process(entity, delta, ctx)` | 帧更新（快照迭代） |
| `Add(entity, index, ctx)` | 动态添加策略（触发 AfterAdd） |
| `Remove(entity, index, ctx)` | 动态移除策略（触发 BeforeRemove） |

- **Recover**：从池获取时进行类型过滤，仅保留 `EntityStrategyBase` 子类，其余立即 `ReleaseStrategy`
- **生命周期钩子触发**：全部基于 `ToArray()` 快照迭代——因为钩子内可增删策略。五个触发器方法（`TriggerAfterSpawn/Load/Save/Quit/Dead`）统一委托给 `TriggerAll`，消除复制粘贴重复

### ActiveStrategyManager

每个 `SndEntity` 持有一个 manager 实例，管理主动策略：

- **容器**：`Dictionary<string, ActiveStrategyBase>` — O(1) 按索引查找，不参与每帧遍历
- **Recover**：从 metadata 批量恢复，进行类型过滤（不触发钩子）
- **ReleaseAll**：逐个 `ReleaseStrategy` 并清空容器（不触发钩子）
- **Add / Remove**：动态增删主动策略
- **Invoke**：按索引查找策略实例，调用 `Invoke(entity, ctx, input)` 并返回结果
- **序列化**：`SerializeIndices()` 返回当前持有的全部索引

### ActiveStrategyExtensions

`ISndEntity` 的扩展方法，提供类型安全的泛型 ActiveStrategy 调用和惰性策略挂载：

```csharp
// 泛型调用（输入和输出强类型）
var result = entity.InvokeStrategy<SearchInput, PathResult>("traversability.find_path", input);

// 无输入泛型调用
var list = entity.InvokeStrategy<List<FoodEntry>>("food.get_registry");

// 惰性策略挂载（带幂等守卫）
entity.EnsureStrategy("character.path_impl", "character.pathfind.astar");
```

- `InvokeStrategy<TInput, TOutput>` / `InvokeStrategy<TOutput>`：透明处理 JSON 序列化/反序列化，消除调用侧样板
- `EnsureStrategy(string dataKey, string strategyIndex)`：检查实体 dataKey 是否已有值，若无则写入并挂载策略。用于惰性策略层初始化，幂等安全可重复调用

原始 `entity.InvokeStrategy(string, object?)` 接口保持不变，泛型扩展方法作为可选便利层。

### 实体生命周期中的策略顺序

实体策略的生命周期钩子在两阶段批处理中的顺序：

```
Phase 1 (RecoverForLifecycle):
  1. Recover Data
  2. Recover Nodes
  3. Recover EntityStrategy (RecoverStrategiesOnly)
  4. Recover ActiveStrategy

Phase 2 (Trigger hooks — **internal**):
  5. 触发实体策略钩子 (TriggerAfterSpawn/AfterLoad/etc.，由 IEntityLifecycle 的 internal 方法驱动)

Phase 3 (Teardown):
  6. Release ActiveStrategy (ReleaseAll)
  7. Release EntityStrategy (ReleaseStrategiesOnly)
  8. Teardown Nodes + Data (TeardownOnly)
```

ActiveStrategy 在 Phase 1 恢复，Phase 2 期间可被 `InvokeStrategy` 安全调用。

### StrategyMetaData 拆分

实体元数据中的策略索引按类型分开存储：

```
StrategyMetaData
├── EntityIndices: List<string>   (被动实体策略)
└── ActiveIndices: List<string>   (主动策略)
```

序列化后 JSON 格式：
```json
{ "entity_indices": ["patrol", "idle"], "active_indices": ["query.hp"] }
```

两者在 RecoverForLifecycle 时分别恢复，互不交叉。

### StrategyIndexAttribute

```csharp
[StrategyIndex("my_game.player_control", Priority = 100)]
public class PlayerControlStrategy : EntityStrategyBase { ... }
```

- `Index`：必填，策略在池中的唯一索引键
- `Priority`：可选，默认 6205，决定同实体上多被动策略的执行顺序；主动策略不参与排序

## 设计决策

### 为什么策略强制无状态（注册期校验）

策略实例在多个实体间共享，若持有实例字段（如 `int _hp`），多实体间会互相污染。注册期通过反射检查 `BaseStrategy` 到具体类型之间的所有层级，拒绝声明实例字段或可写属性的策略，从源头阻止此错误。

此约束的一个副作用：**测试策略无法使用实例字段作为事件接收器**，必须使用静态字段（`static List<string>?`）在各策略实例间共享事件收集。使用静态字段的测试类必须通过 `[Collection]` 属性串行化执行，或通过 `[assembly: CollectionBehavior(DisableTestParallelization = true)]` 全局禁用并行，以防止并行测试间的竞态。详见 `Origo.Core.Tests/Architecture.md`。

### 为什么使用引用计数而非单一实例

同一策略可能被多个实体同时引用（如 `core.health` 策略在每个实体上都活跃）。引用计数确保只要有一个实体持有，策略就不被回收。归零时才释放，下次访问重新创建或复用。

### 为什么被动和主动策略容器分离

主动策略只需按索引查找（O(1) Dictionary），不参与每帧遍历。被动策略需要按优先级排序迭代（List）。容器分离避免遍历时的类型检查和无关数据，也给序列化提供清晰的分组边界。

### 为什么生命周期钩子全都基于快照迭代

钩子回调中经常需要增删策略（例如 `BeforeDead` 中移除自身策略）。直接迭代修改中的列表会导致异常。快照确保了每次钩子调用的列表稳定性，以少量分配换取安全。

### 为什么 ActiveStrategy 在实体策略钩子之前恢复

ActiveStrategy 在 `RecoverForLifecycle` (Phase 1) 中恢复，早于 `FireAfterSpawnHooks` / `FireAfterLoadHooks` (Phase 2)。这确保实体策略钩子中可以通过 `InvokeStrategy` 调用自身的 ActiveStrategy，也可以调用其他已恢复实体的 ActiveStrategy——实现加载顺序无关的跨实体互操作。

---

[↑ 回到 Snd](../README.md)
