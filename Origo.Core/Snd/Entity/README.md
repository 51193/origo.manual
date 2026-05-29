# Entity

> [↑ 回到 Snd](../README.md) · [↔ 抽象: Abstractions/Entity](../../Abstractions/Entity/README.md)

## 概述

SND 实体模型的具体实现。`SndEntity` 是运行时实体聚合根，组合了 `SndDataManager`（数据）、`SndNodeManager`（节点）、`SndStrategyManager`（策略）三个内部管理器，完整实现了 `ISndEntity` 接口。所有生命周期钩子通过策略管理器协调调用。

## 包含文件

| 文件 | 职责 |
|------|------|
| `SndEntity.cs` | 实体聚合根：组合数据/节点/策略管理器，实现全部生命周期方法 |
| `SndDataManager.cs` | 实体数据字典管理 + 观察者变更通知 |
| `SndNodeManager.cs` | 实体节点管理：从元数据恢复到节点创建/查询/释放 |
| `DataObserverManager.cs` | 通用观察者订阅/通知基础设施（键 → 回调列表）|

## 模块详解

### SndEntity（聚合根）

**构造函数**要求注入 `INodeFactory`、`SndStrategyPool`、`SndMappings`、`ISndContext`、`ILogger`。不暴露无参构造。

**生命周期方法**：

| 方法 | 触发时机 | 策略钩子序列 |
|------|---------|------------|
| `Spawn(meta)` | 新实体创建 | Recover → AfterSpawn（快照迭代）|
| `Load(meta)` | 从存档恢复 | Recover → AfterLoad（快照迭代）|
| `Quit()` | 正常退出 | BeforeQuit → Teardown |
| `Dead()` | 死亡销毁 | BeforeDead → Teardown |
| `Process(delta)` | 每帧更新 | Process（按优先级 + 快照迭代）|

`SerializeMetaData()` 收集全部三层元数据（节点 + 策略 + 数据），合并为 `SndMetaData`。

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

### 为什么 SndEntity 是聚合根而非组合暴露子管理器

外部策略代码通过 `ISndEntity` 接口操作实体（SetData/GetData/AddStrategy），不感知内部管理器。聚合根封装确保实体内部状态一致性：例如 `Quit()` 需要先触发策略 BeforeQuit 钩子，再释放节点和数据，这个顺序若暴露为独立 API 容易误用。

### 为什么节点恢复失败时回滚全部已创建节点

`SndNodeManager.Recover` 中若第 N 个节点创建失败，前 N-1 个已创建的节点处于半初始状态，无法安全使用。回滚释放确保不残留不完整状态。

### 为什么 DataObserverManager 使用快照迭代通知回调

通知回调中可能触发 Subscribe/Unsubscribe/SetData（从而再次 NotifyObservers）。若直接在列表上 foreach 同时修改，会导致 `Collection was modified` 异常。`ToArray()` 快照以少量分配换取安全。

---
[↑ 回到 Snd](../README.md)
