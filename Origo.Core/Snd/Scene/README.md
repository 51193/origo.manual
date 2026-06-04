# Scene

> [↑ 回到 Snd](../README.md)

## 概述

SND 场景宿主实现层。提供 `ISndSceneHost` 的三种实现：完整内存宿主（用于后台会话）、轻量内存宿主（用于测试）、引擎无关的 NullNodeFactory。`SndRuntime` 作为面向上层的门面，组合 `SndWorld` 和 `ISndSceneHost`，并负责实体策略生命周期钩子的两阶段批处理编排。

## 包含文件

| 文件 | 职责 |
|------|------|
| `SndRuntime.cs` | SND 运行时门面：组合 World + SceneHost，编排 Spawn/Kill/Save/Quit 的批处理钩子 |
| `FullMemorySndSceneHost.cs` | 完整内存场景宿主，创建真实 SndEntity，具备完整策略生命周期 |
| `MemorySndSceneHost.cs` | 轻量内存场景宿主，使用简单 MemorySndEntity（无策略/节点）|
| `ISndContextAttachableSceneHost.cs` | 接口：允许运行时绑定 ISndContext |
| `NullNodeFactory.cs` | 内存级节点工厂，创建无操作句柄 |

## 模块详解

### SndRuntime — 生命周期编排

`SndRuntime` 不仅组合 `SndWorld` + `ISndSceneHost`，还负责实体策略钩子的两阶段批处理：

```
SndRuntime = SndWorld (策略池 + 配置) + ISndSceneHost (实体宿主) + 钩子编排
```

**批处理方法**：

| 方法 | 说明 |
|------|------|
| `Spawn(meta)` | 重名校验后委托 `SceneHost.Spawn(meta)`（宿主内部完成创建 + AfterSpawn） |
| `SpawnMany(metaList)` | 重名校验后委托 `SceneHost.SpawnMany(metaList)`（宿主内部完成批量创建 + AfterSpawn） |
| `KillPendingEntities()` | 收集 IsPendingKill → 全部 `FireBeforeDeadHooks()` → 全部 `SceneHost.TeardownEntity(name)` |
| `ClearAll()` | 全部 `FireBeforeQuitHooks()` + `ReleaseStrategiesOnly()` + `TeardownOnly()` → `SceneHost.RemoveAllEntities()` |
| `BuildMetaList()` | 透传 `SceneHost.BuildMetaList()`（无 BeforeSave） |

Spawn/SpawnMany 不再自行编排实体钩子触发，而是委托给 `SceneHost.Spawn` / `SceneHost.SpawnMany`（宿主内部完成创建和 AfterSpawn），SndRuntime 仅负责重名校验。

### FullMemorySndSceneHost

后台会话的默认场景宿主。关键特性：
- 通过 `SndWorld.CreateEntity` 创建完整 `SndEntity`（非简单内存实体）
- 需延迟绑定 `SndWorld` 和 `ISndContext`（配合 `OrigoRuntime` 两阶段构造）
- **实体容器管理**：只负责创建、查找、移除实体，不触发任何策略钩子
- **Spawn / SpawnMany**：创建实体，恢复数据/策略/节点（通过 `RecoverForLifecycle`），然后触发 AfterSpawn 钩子。`Spawn` 为单个实体，`SpawnMany` 批量创建所有实体后统一触发 AfterSpawn。
- **RecoverFromMetaList**：仅恢复实体数据/策略/节点（不触发 AfterLoad 钩子），用于存档加载场景。先通过 `entity.Name = metaData.Name` 设置实体名，再将实体注册到内部集合，最后调用 `RecoverForLifecycle(meta)` 恢复数据/策略/节点（不触发钩子）。因此钩子执行前，`FindByName` 可查找所有已注册实体。
- **TeardownEntity**：仅拆解实体资源（释放策略 + 拆卸节点/数据），不触发钩子
- **RemoveAllEntities**：仅清空内部集合
- **ProcessAll**：基于快照迭代所有实体

### MemorySndSceneHost

轻量实现，直接使用内嵌的 `MemorySndEntity` 类。这个实体不支持节点访问、策略和订阅，仅支持基础键值数据存取。用于单元测试和 `LevelBuilder` 离线构建。

### NullNodeFactory / NullNodeHandle

用于 `FullMemorySndSceneHost`。`Create()` 返回不绑定任何引擎节点的句柄，所有操作（`Free`、`SetVisible`）为空操作。Core 层后台会话不需要实际渲染节点。

## 设计决策

### 为什么场景宿主不触发策略钩子

策略生命周期钩子的触发由 `SndRuntime` 和 `SessionRun` 统一编排。场景宿主仅负责实体容器管理。这种职责分离确保：

- Godot 适配层（`GodotSndManager`）不参与策略生命周期管理
- 批量操作可以在"全部创建/恢复"阶段和"全部触发钩子"阶段之间进行
- 钩子触发期间，所有实体已完全恢复并注册到查找集合，实现加载顺序无关的跨实体互操作

### 为什么需要两个场景宿主

`FullMemorySndSceneHost` 提供完整策略生命周期但需要 `SndWorld` 和 `ISndContext` 的上游依赖；`MemorySndSceneHost` 零依赖、完全自治但不能运行策略。前者用于后台会话，后者用于测试（测试中通常只测数据流转而无需策略执行）。

### 为什么 FullMemorySndSceneHost 延迟绑定 World/Context

`OrigoRuntime` 的两阶段构造（先创建宿主，后注入运行时依赖）需要宿主支持延迟绑定。在构造函数中提供这些依赖会形成循环：`SndWorld` 的创建需要 `OrigoRuntime`，而宿主的创建又在 `SndWorld` 之前。

### 为什么实体在钩子触发前先登记到查找集合

策略钩子可能需要在创建期间引用兄弟实体（例如通过 `FindByName` 查找依赖实体）。先登记后触发钩子保证了所有实体在整个生命周期内始终可被检索。批处理模式下，所有实体先全部登记，再统一触发钩子，进一步加强了这一保证。

---

[↑ 回到 Snd](../README.md)
