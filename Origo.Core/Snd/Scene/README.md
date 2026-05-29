# Scene

> [↑ 回到 Snd](../README.md)

## 概述

SND 场景宿主实现层。提供 `ISndSceneHost` 的三种实现：完整内存宿主（用于后台会话）、轻量内存宿主（用于测试）、引擎无关的 NullNodeFactory。`SndRuntime` 作为面向上层的门面，组合 `SndWorld` 和 `ISndSceneHost`。

## 包含文件

| 文件 | 职责 |
|------|------|
| `SndRuntime.cs` | SND 运行时门面：组合 World + SceneHost，提供 Spawn/序列化/查询 |
| `FullMemorySndSceneHost.cs` | 完整内存场景宿主，创建真实 SndEntity，具备完整策略生命周期 |
| `MemorySndSceneHost.cs` | 轻量内存场景宿主，使用简单 MemorySndEntity（无策略/节点）|
| `ISndContextAttachableSceneHost.cs` | 接口：允许运行时绑定 ISndContext |
| `NullNodeFactory.cs` | 内存级节点工厂，创建无操作句柄 |

## 模块详解

### SndRuntime

`SndRuntime` 是调用方面向 SND 系统的主要入口：

```
SndRuntime = SndWorld (策略池 + 配置) + ISndSceneHost (实体宿主)
```

- `Spawn(meta)`：校验重名（通过 `SceneHost.FindByName`），然后委托给宿主
- `SpawnMany()`：批量生成
- `SerializeMetaList()` / `ClearAll()` / `GetEntities()` / `FindByName()`：透传给 SceneHost

### FullMemorySndSceneHost

后台会话的默认场景宿主。关键特性：
- 通过 `SndWorld.CreateEntity` 创建完整 `SndEntity`（非简单内存实体）
- 需延迟绑定 `SndWorld` 和 `ISndContext`（配合 `OrigoRuntime` 两阶段构造）
- **Spawn/Load**：创建实体后存储到 `MemoryEntityEntry` 列表
- **QuitAll**：反向迭代退出（LIFO 语义）
- **DeadByName**：按名销毁实体（触发 BeforeDead 钩子）
- **ProcessAll**：基于快照迭代所有实体

### MemorySndSceneHost

轻量实现，直接使用内嵌的 `MemorySndEntity` 类。这个实体不支持节点访问、策略和订阅，仅支持基础键值数据存取。用于单元测试和 `LevelBuilder` 离线构建。

### NullNodeFactory / NullNodeHandle

用于 `FullMemorySndSceneHost`。`Create()` 返回不绑定任何引擎节点的句柄，所有操作（`Free`、`SetVisible`）为空操作。Core 层后台会话不需要实际渲染节点。

## 设计决策

### 为什么需要两个场景宿主

`FullMemorySndSceneHost` 提供完整策略生命周期但需要 `SndWorld` 和 `ISndContext` 的上游依赖；`MemorySndSceneHost` 零依赖、完全自治但不能运行策略。前者用于后台会话，后者用于测试（测试中通常只测数据流转而无需策略执行）。

### 为什么 FullMemorySndSceneHost 延迟绑定 World/Context

`OrigoRuntime` 的两阶段构造（先创建宿主，后注入运行时依赖）需要宿主支持延迟绑定。在构造函数中提供这些依赖会形成循环：`SndWorld` 的创建需要 `OrigoRuntime`，而宿主的创建又在 `SndWorld` 之前。

### 为什么 SndWorld.CreateEntity 通过 SndWorld 创建实体

实体构造需要 `SndStrategyPool`（来自 `SndWorld`），`SndContext`（来自 `OrigoRuntime`）。将这些外部依赖的传递封装在 `CreateEntity` 中，避免在 `FullMemorySndSceneHost` 内部重构实体创建逻辑。

---
[↑ 回到 Snd](../README.md)
