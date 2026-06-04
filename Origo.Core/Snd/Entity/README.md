# Entity

> [↑ 回到 Snd](../README.md) · [↔ 抽象: Abstractions/Entity](../../Abstractions/Entity/README.md)

## 概述

SND 实体模型的具体实现。`SndEntity` 是运行时实体聚合根，组合了 `SndDataManager`（数据）、`SndNodeManager`（节点）、`SndStrategyManager`（策略）三个内部管理器，实现了 `ISndEntity` 和 `IEntityLifecycle` 接口。

策略生命周期钩子不再由实体自身方法直接触发，而是通过 `IEntityLifecycle` 接口暴露分阶段方法，由框架层的 `SndRuntime` 和 `SessionRun` 统一编排批发钩子调用。

## 包含文件

| 文件 | 职责 |
|------|------|
| `SndEntity.cs` | 实体聚合根：组合数据/节点/策略管理器，实现 `ISndEntity` + `IEntityLifecycle` |
| `SndDataManager.cs` | 实体数据字典管理 + 观察者变更通知 |
| `SndNodeManager.cs` | 实体节点管理：从元数据恢复到节点创建/查询/释放 |
| `DataObserverManager.cs` | 通用观察者订阅/通知基础设施（键 → 回调列表）|

## 模块详解

### SndEntity（聚合根）

**构造函数**要求注入 `INodeFactory`、`SndStrategyPool`、`SndMappings`、`ISndContext`、`ILogger`。不暴露无参构造。

**`IEntityLifecycle` 分阶段方法**：

这些方法供框架层批量编排使用，业务代码不应直接调用：

| 方法 | 阶段 | 说明 |
|------|------|------|
| `RecoverForLifecycle(meta)` | Phase 1: 恢复 | 恢复 Name + Data + Node + EntityStrategy + ActiveStrategy，不触发任何钩子 |
| `FireAfterSpawnHooks()` | Phase 2: 钩子 | 按优先级触发实体策略的 AfterSpawn |
| `FireAfterLoadHooks()` | Phase 2: 钩子 | 按优先级触发实体策略的 AfterLoad |
| `FireBeforeSaveHooks()` | Phase 2: 钩子 | 按优先级触发实体策略的 BeforeSave |
| `FireBeforeQuitHooks()` | Phase 2: 钩子 | 按优先级触发实体策略的 BeforeQuit |
| `FireBeforeDeadHooks()` | Phase 2: 钩子 | 按优先级触发实体策略的 BeforeDead |
| `ReleaseStrategiesOnly()` | Phase 3: 拆卸 | 释放 EntityStrategy + ActiveStrategy 引用（不触发钩子） |
| `TeardownOnly()` | Phase 3: 拆卸 | 释放 Node + Data 资源（不触发钩子） |
| `BuildMetaData()` | 序列化 | 构建元数据（不触发 BeforeSave） |

`Process(delta)` 保持不变：按优先级 + 快照迭代触发策略 Process。

`IsPendingKill` 标记由 `RequestKillEntity()` 立即设置。BeforeDead 钩子由 `SndRuntime.KillPendingEntities()` 批量触发，`TeardownEntity()` 仅做拆解。

> **注意**：`Spawn` / `SpawnMany` 是场景宿主（`ISndSceneHost`）的方法，不在实体自身上。`ISndSceneHost.Spawn` 创建实体并通过 `RecoverForLifecycle` + `FireAfterSpawnHooks` 触发 AfterSpawn；`ISndSceneHost.SpawnMany` 批量创建后统一触发 AfterSpawn。

### SndDataManager

- **存储**：`Dictionary<string, TypedData>`
- **SetData**：比较旧值，相同时跳过通知（避免无意义事件）
- **GetData vs TryGetData**：前者 KeyNotFound 时抛 `InvalidOperationException`，后者安全
- **Subscribe/Unsubscribe**：外部回调签名 `Action<ISndEntity, object?, object?>`，内部包装为 `Action<object?, object?>` 适配 `DataObserverManager`
- **Recover / Release / SerializeMeta**：存档恢复/清理/序列化

### SndNodeManager

- 实现 `INodeHost`（internal 接口）
- `Recover`：先 Release 旧节点，再按元数据逐个通过 `INodeFactory.Create` 创建新节点。创建失败时回滚 Release 全部
- `Release`：逐个 `node.Free()` 然后清空
- 节点资源 ID 通过 `SndMappings.ResolveSceneAlias` 解析（支持别名）

### DataObserverManager

独立于引擎的观察者基础设施：
- 每个数据键维护一个 `List<Subscription>`
- 每个 Subscription 包含 `Callback(Action<object?,object?>)` + 可选的 `Filter(Func<object?,object?,bool>)`
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

---

[↑ 回到 Snd](../README.md)
