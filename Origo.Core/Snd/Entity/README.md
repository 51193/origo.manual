# Entity

> [↑ 回到 Snd](../README.md) · [↔ 抽象: Abstractions/Entity](../../Abstractions/Entity/README.md)

## 概述

SND 实体模型的具体实现。`SndEntity` 是运行时实体聚合根，组合了 `SndDataManager`（数据）、`SndNodeManager`（节点）、`SndStrategyManager`（策略）三个内部管理器，实现了 `ISndEntity`、`IEntityLifecycle` 和 `ISndEntityRawSubscription` 接口。

策略生命周期钩子不再由实体自身方法直接触发，而是通过 `IEntityLifecycle` 接口暴露分阶段方法，由框架层的 `SndRuntime` 和 `SessionRun` 统一编排批发钩子调用。

`SndEntity` 也是统一观察模式的核心：所有订阅（自身数据、自身生命周期、跨实体数据、跨实体生命周期）均通过 `SubscribeDataInternal` / `SubscribeLifecycleInternal` 统一内部链路，并在 `TeardownOnly` 中自动清理全部传出订阅。

## 包含文件

| 文件 | 职责 |
|------|------|
| `SndEntity.cs` | 实体聚合根：组合数据/节点/策略管理器，实现 `ISndEntity` + `IEntityLifecycle` + `ISndEntityRawSubscription`，持有传出订阅追踪和生命周期观察者列表 |
| `SndDataManager.cs` | 实体数据字典管理 + 观察者变更通知 |
| `SndNodeManager.cs` | 实体节点管理：从元数据恢复到节点创建/查询/释放 |
| `DataObserverManager.cs` | 通用观察者订阅/通知基础设施（键 → 回调列表）|
| `ISndEntityRawSubscription.cs` | **`internal`** — 原始订阅操作接口。供 `SndEntity` 在内部链路中绕过公开 API 的追踪逻辑直接操作目标实体的 `SndDataManager` 和生命周期观察者列表 |

## 模块详解

### SndEntity（聚合根）

**构造函数**要求注入 `INodeFactory`、`SndStrategyPool`、`SndMappings`、`ISndContext`、`ILogger`。不暴露无参构造。

**统一订阅模型**：

所有公开订阅方法最终汇入两个内部方法：

| 内部方法 | 对应公开方法 | 接收 |
|----------|-------------|------|
| `SubscribeDataInternal(target, name, callback, filter)` | `Subscribe`（自身）、`ObserveData`（跨实体） | `Action<ISndEntity, ISndEntity, TypedData, TypedData>` |
| `SubscribeLifecycleInternal(target, callback)` | `SubscribeLifecycle`（自身）、`ObserveLifecycle`（跨实体） | `Action<ISndEntity, ISndEntity, EntityLifecycleEvent>` |

内部方法执行三个步骤：
1. 将用户回调包装为内部签名（注入 `self` 作为 observer 参数），调用目标实体的原始订阅方法（`ISndEntityRawSubscription`）
2. 将订阅记录加入观察方实体的传出追踪列表（`_outgoingDataSubs` / `_outgoingLifecycleSubs`）

**传出追踪**：两个列表分别记录数据订阅（`OutgoingDataSub { Target, DataName, OriginalCallback, WrappedCallback }`）和生命周期订阅（`OutgoingLifecycleSub { Target, OriginalCallback, WrappedCallback }`）。Unobserve 时通过 `OriginalCallback` 委托引用匹配，使用 `WrappedCallback` 在目标上执行退订。

**生命周期观察者**：`List<Action<ISndEntity, EntityLifecycleEvent>>` 存储所有对本实体的生命周期订阅（传入方向）。在 `FireXxxHooks` 时通过 `NotifyLifecycleObservers` 以快照迭代通知。

**`IEntityLifecycle` 分阶段方法**：

这些方法供框架层批量编排使用，业务代码不应直接调用：

| 方法 | 阶段 | 说明 |
|------|------|------|
| `RecoverForLifecycle(meta)` | Phase 1: 恢复 | 恢复 Name + Data + Node + EntityStrategy + ActiveStrategy，不触发任何钩子 |
| `FireAfterSpawnHooks()` | Phase 2: 钩子 | 先通知生命周期观察者 AfterSpawn，再按优先级触发策略 AfterSpawn |
| `FireAfterLoadHooks()` | Phase 2: 钩子 | 先通知生命周期观察者 AfterLoad，再按优先级触发策略 AfterLoad |
| `FireBeforeSaveHooks()` | Phase 2: 钩子 | 先通知生命周期观察者 BeforeSave，再按优先级触发策略 BeforeSave |
| `FireBeforeQuitHooks()` | Phase 2: 钩子 | 先通知生命周期观察者 BeforeQuit，再按优先级触发策略 BeforeQuit |
| `FireBeforeDeadHooks()` | Phase 2: 钩子 | 先通知生命周期观察者 BeforeDead，再按优先级触发策略 BeforeDead |
| `ReleaseStrategiesOnly()` | Phase 3: 拆卸 | 释放 EntityStrategy + ActiveStrategy 引用（不触发钩子） |
| `TeardownOnly()` | Phase 3: 拆卸 | 1. 遍历传出数据订阅 → 全部退订  2. 遍历传出生命周期订阅 → 全部退订  3. 清空生命周期观察者  4. 释放 Node + Data 资源 |
| `BuildMetaData()` | 序列化 | 构建元数据（不触发 BeforeSave） |

`Process(delta)` 保持不变：按优先级 + 快照迭代触发策略 Process。

`IsPendingKill` 标记由 `RequestKillEntity()` 立即设置。BeforeDead 钩子由 `SndRuntime.KillPendingEntities()` 批量触发，`RemoveEntity()` 仅做拆解。

> **注意**：`CreateEntity` 是场景宿主（`ISndSceneHost`）的方法，不在实体自身上。`ISndSceneHost.CreateEntity` 创建实体并通过 `RecoverForLifecycle` 恢复数据/策略/节点，但不触发 AfterSpawn 钩子。AfterSpawn 钩子由 `SndRuntime.Spawn` / `SndRuntime.SpawnMany` 在创建完成后统一触发。

### SndDataManager

- **存储**：`Dictionary<string, TypedData>`
- **SetData**：比较旧值，相同时跳过通知（避免无意义事件）
- **GetData vs TryGetData**：前者 KeyNotFound 时抛 `InvalidOperationException`，后者安全
- **Subscribe/Unsubscribe**：接收 `Action<ISndEntity, TypedData, TypedData>`，内部包装为 `Action<TypedData, TypedData>` 适配 `DataObserverManager`。订阅信息存入 `_subscriptionMap` 用于退订匹配。两阶段包装：上层 `SndEntity.SubscribeDataInternal` 将用户回调（含 observer 参数）包装为 `(target, old, new)`，`SndDataManager.Subscribe` 再包装为 `(old, new)` 供 `DataObserverManager` 使用
- **Recover / Release / SerializeMeta**：存档恢复/清理/序列化

### SndNodeManager

- 实现 `INodeHost`（internal 接口）
- `Recover`：先 Release 旧节点，再按元数据逐个通过 `INodeFactory.Create` 创建新节点。创建失败时回滚 Release 全部
- `Release`：逐个 `node.Free()` 然后清空
- 节点资源 ID 通过 `SndMappings.ResolveSceneAlias` 解析（支持别名）

### DataObserverManager

独立于引擎的观察者基础设施：
- 每个数据键维护一个 `List<Subscription>`
- 每个 Subscription 包含 `Callback(Action<TypedData, TypedData>)` + 可选的 `Filter(Func<TypedData, TypedData, bool>)`
- `NotifyObservers` 通过 `ToArray()` 快照迭代，允许回调中修改订阅列表
- `Unsubscribe` 通过委托引用比对移除

## 设计决策

### 为什么分离 IEntityLifecycle

策略生命周期钩子（AfterSpawn/AfterLoad/BeforeSave/BeforeQuit/BeforeDead）的触发时机由框架层控制，不应该直接暴露在 `ISndEntity`（面向业务代码）上。`IEntityLifecycle` 接口暴露分阶段方法给 `SndRuntime` 和 `SessionRun`，实现批量编排的同时保持业务代码接口简洁。

参见 [IEntityLifecycle](../../Abstractions/Entity/README.md)。

### 为什么 SndEntity 是聚合根而非组合暴露子管理器

外部策略代码通过 `ISndEntity` 接口操作实体（SetData/GetData/AddStrategy），不感知内部管理器。聚合根封装确保实体内部状态一致性。

### 为什么节点恢复失败时回滚全部已创建节点

`SndNodeManager.Recover` 中若第 N 个节点创建失败，前 N-1 个已创建的节点处于半初始状态，无法安全使用。回滚释放确保不残留不完整状态。

### 为什么 DataObserverManager 使用快照迭代通知回调

通知回调中可能触发 Subscribe/Unsubscribe/SetData（从而再次 NotifyObservers）。若直接在列表上 foreach 同时修改，会导致 `Collection was modified` 异常。`ToArray()` 快照以少量分配换取安全。

### 为什么自身订阅和跨实体观察走统一内部链路

`Subscribe` 等价于 `ObserveData(this, ...)`，`SubscribeLifecycle` 等价于 `ObserveLifecycle(this, ...)`。统一链路确保无论是自观察还是跨实体观察，均经过相同的包装、追踪和 Teardown 清理流程。避免两条代码路径分歧导致的遗漏。

### 为什么 TeardownOnly 自动清理传出订阅

实体死亡时，所有对外的订阅关系应当终止。在 `TeardownOnly` 中统一遍历传出订阅列表进行退订，确保：
- 观察方死亡后，被观察实体的 `DataObserverManager` 和生命周期观察者列表不再残留已失效的回调
- 策略代码无需在 `BeforeDead` / `BeforeRemove` 中手动退订

### 为什么传出追踪存储 (OriginalCallback, WrappedCallback) 对

与 `SndDataManager._subscriptionMap` 的 `SubscriptionPair` 设计一致：OriginalCallback 用于 Unobserve 时的委托引用匹配，WrappedCallback 用于在目标实体上执行退订。Unobserve 通过 `OriginalCallback` 匹配同一方法引用，自然强制了方法引用模式。

### 为什么 ISndEntityRawSubscription 是 internal 接口

公开 API（`Subscribe`、`ObserveData` 等）带有传出追踪逻辑。在内部链路中调用目标实体的原始订阅方法时，必须绕过追踪（否则会导致目标实体误将此次调用作为自身的传出订阅）。`ISndEntityRawSubscription` 提供"只订阅不追踪"的内部通道，仅 `SndEntity` 和 `GodotSndEntity` 实现。

---

[↑ 回到 Snd](../README.md)
