# Entity (Abstractions)

> [↑ 回到 Abstractions](../README.md) · [↔ 实现: Snd/Entity](../../Snd/Entity/README.md)

## 概述

定义 SND 实体的抽象接口体系。所有接口遵循接口隔离原则（ISP），将实体的数据、节点、策略、生命周期、观察五种能力拆分为独立接口，再由 `ISndEntity` 组合统一入口。`IEntityLifecycle` 单独定义，供框架层进行批处理生命周期编排。**此接口为 `internal`，用户代码不可访问。**

## 包含文件

| 文件 | 职责 |
|------|------|
| `ISndDataAccess.cs` | 数据存取与变更订阅 |
| `ISndNodeAccess.cs` | 节点查询与枚举 |
| `ISndStrategyAccess.cs` | 策略动态添加/移除 |
| `ISndActiveStrategyAccess.cs` | 主动策略：添加/移除/调用 |
| `ISndEntityLifecycleAccess.cs` | 实体生命周期事件订阅 |
| `ISndObservation.cs` | 跨实体观察：数据 + 生命周期的统一观察入口 |
| `ISndEntity.cs` | 组合接口：继承上述六个接口 + `Name` 属性 + `IsPendingKill` |
| `IEntityLifecycle.cs` | **`internal`** — 框架内部生命周期接口：分阶段恢复/钩子/拆卸方法 |
| `EntityLifecycleEvent.cs` | 实体生命周期事件枚举 |

## 接口详细

### ISndDataAccess

| 成员 | 说明 |
|------|------|
| `SetData<T>(name, value)` | 写入命名数据 |
| `GetData<T>(name)` | 直接读取；键不存在时抛异常 |
| `TryGetData<T>(name)` | 安全读取，返回 `(found, value?)` |
| `Subscribe(name, callback, filter?)` | 订阅本实体数据变更。等价于 `ObserveData(this, ...)`，走统一内部链路。回调签名 `(target, observer, oldValue, newValue)` |
| `Unsubscribe(name, callback)` | 取消数据订阅。`callback` 必须与 `Subscribe` 调用时的委托实例相同（方法引用），lambda 表达式每次编译产生不同实例，会导致退订失败 |

### ISndNodeAccess

| 成员 | 说明 |
|------|------|
| `GetNode(name)` | 按名称获取节点句柄 |
| `GetNodeNames()` | 枚举所有已挂载节点名称 |

### ISndStrategyAccess

| 成员 | 说明 |
|------|------|
| `AddStrategy(index)` | 向实体动态添加策略 |
| `RemoveStrategy(index)` | 从实体移除策略 |

### ISndActiveStrategyAccess

| 成员 | 说明 |
|------|------|
| `AddActiveStrategy(index)` | 向实体动态添加主动策略 |
| `RemoveActiveStrategy(index)` | 从实体移除主动策略 |
| `InvokeStrategy(strategyIndex, input?)` | 主动调用策略并返回结果 |

### ISndEntityLifecycleAccess

| 成员 | 说明 |
|------|------|
| `SubscribeLifecycle(callback)` | 订阅本实体的生命周期事件。回调签名 `(target, observer, lifecycleEvent)` |
| `UnsubscribeLifecycle(callback)` | 取消生命周期订阅。`callback` 必须与订阅时相同的委托实例 |

### ISndObservation

实体观察的统一入口。自身订阅（`Subscribe` / `SubscribeLifecycle`）等价于 `ObserveData(this, ...)` / `ObserveLifecycle(this, ...)`，走统一内部链路。所有订阅均在观察方实体 Teardown 时自动清理。

| 成员 | 说明 |
|------|------|
| `ObserveData(target, dataName, callback, filter?)` | 观察目标实体的数据变更。回调签名 `(target, observer, oldValue, newValue)` |
| `UnobserveData(target, dataName, callback)` | 取消对目标实体的数据观察。`callback` 必须与 `ObserveData` 相同的委托实例 |
| `ObserveLifecycle(target, callback)` | 观察目标实体的生命周期事件。回调签名 `(target, observer, lifecycleEvent)` |
| `UnobserveLifecycle(target, callback)` | 取消对目标实体的生命周期观察。`callback` 必须与 `ObserveLifecycle` 相同的委托实例 |

### EntityLifecycleEvent

```
enum EntityLifecycleEvent {
    AfterSpawn, AfterLoad, BeforeSave, BeforeQuit, BeforeDead
}
```

暴露策略代码关心的五个实体生命周期事件。`AfterAdd` / `BeforeRemove` 是策略级别事件，不在此枚举中。

### ISndEntity

`ISndEntity : ISndDataAccess, ISndNodeAccess, ISndStrategyAccess, ISndActiveStrategyAccess, ISndEntityLifecycleAccess, ISndObservation`

组合后自有的成员：

| 成员 | 说明 |
|------|------|
| `Name { get; }` | 稳定的实体标识名 |
| `IsPendingKill { get; }` | 标记为待销毁状态。框架在帧末统一执行销毁（业务延迟队列之后、系统延迟队列之前）。策略应在操作实体前通过此标志位判断实体是否仍然存活 |

### IEntityLifecycle（`internal`）

**框架内部使用的接口**，供 `SndRuntime` 和 `SessionRun` 进行两阶段批处理编排。**此接口为 `internal`，业务代码无法访问。**

| 方法 | 阶段 | 说明 |
|------|------|------|
| `RecoverForLifecycle(meta)` | Phase 1 | 恢复 Name + Data + Node + EntityStrategy + ActiveStrategy，不触发任何钩子 |
| `FireAfterSpawnHooks()` | Phase 2 | 先通知生命周期观察者 AfterSpawn，再触发策略 AfterSpawn |
| `FireAfterLoadHooks()` | Phase 2 | 先通知生命周期观察者 AfterLoad，再触发策略 AfterLoad |
| `FireBeforeSaveHooks()` | Phase 2 | 先通知生命周期观察者 BeforeSave，再触发策略 BeforeSave |
| `FireBeforeQuitHooks()` | Phase 2 | 先通知生命周期观察者 BeforeQuit，再触发策略 BeforeQuit |
| `FireBeforeDeadHooks()` | Phase 2 | 先通知生命周期观察者 BeforeDead，再触发策略 BeforeDead |
| `ReleaseStrategiesOnly()` | Phase 3 | 释放 EntityStrategy + ActiveStrategy 引用 |
| `TeardownOnly()` | Phase 3 | 清理全部传出订阅、清空生命周期观察者、释放 Node + Data 资源 |
| `BuildMetaData()` | 序列化 | 构建元数据（不触发 BeforeSave） |

实现者：`SndEntity`（Core 内存实体）、`GodotSndEntity`（Godot 适配层实体）。

## 设计决策

### 为什么拆分为六个访问接口

策略代码通常只需访问具体某一方面的实体能力（例如只读写数据而不管理节点）。ISP 拆分让策略声明依赖时更精确，也便于测试时只 mock 需要的部分。

### 为什么主动策略独立为 ISndActiveStrategyAccess

主动策略与被动实体策略共享 `BaseStrategy` 和 `SndStrategyPool` 基础设施，但容器完全独立：主动策略使用 Dictionary 实现 O(1) 按索引查找，不参与帧遍历；被动策略使用排序列表，每帧按优先级迭代。接口分离避免消费者耦合到不必要的策略类型。

### 为什么 IEntityLifecycle 单独定义

策略生命周期钩子的触发由框架层控制。将 `RecoverForLifecycle`、`FireXxxHooks`、`ReleaseStrategiesOnly`、`TeardownOnly` 放入独立接口可以：
- 确保 `ISndEntity`（面向业务代码）不暴露生命周期编排能力
- 允许 `SndRuntime` 和 `SessionRun` 通过 `IEntityLifecycle` 进行统一批处理
- Godot 适配层的 `GodotSndEntity` 也能实现此接口，委托给内部 `SndEntity`

### 为什么 TryGetData 使用 found/value 元组

参考 [Blackboard 的相同设计决策](../Blackboard/README.md#为什么是泛型tryget而非object)。

### 为什么订阅回调签名包含 observer 参数

策略实例通过 `SndStrategyPool` 在多个实体间共享，禁止持有实例字段存储实体引用。回调需要知道观察方实体是谁才能修改其数据，因此统一将 `observer` 作为回调的第二参数传入。自订阅时 `target == observer`。

### 为什么 Subscribe 与 ObserveData 走统一内部链路

自身数据订阅（`Subscribe`）等价于 `ObserveData(this, ...)`，自身生命周期订阅（`SubscribeLifecycle`）等价于 `ObserveLifecycle(this, ...)`。统一链路保证所有订阅——无论自观察还是跨实体观察——均通过相同的传出追踪和 Teardown 自动清理机制处理，避免代码路径分歧带来的遗漏和 bug。

### 为什么 Observe / Unobserve 要求相同的委托实例

内部匹配依赖委托引用比对（`==`）。方法引用（method group）在 C# 中每次编译生成相同的委托实例，可以正确匹配。lambda 表达式每次出现都是新的委托实例，导致退订匹配失败。此约束由 API 签名自然强制。

### 为什么生命周期观察者通知在策略钩子之前

外部观察者先收到通知（例如 `BeforeDead`），然后被观察实体的策略钩子才执行。这确保观察者可以在目标实体拆卸前做出反应（如设置数据标志），且被观察实体仍处于完整状态（`FindByName` 可查找，策略未释放）。

---

[↑ 回到 Abstractions](../README.md)
