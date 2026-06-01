# Scene (Abstractions)

> [↑ 回到 Abstractions](../README.md) · [↔ 实现: Snd/Scene](../../Snd/Scene/README.md)

## 概述

定义 Core 层编排 SND 场景的抽象能力。`ISndSceneAccess` 提供最小化的串行化/恢复操作，`ISndSceneHost` 在其基础上补充实体生成、查询、帧更新、实体销毁和场景清空能力。

## 包含文件

| 文件 | 职责 |
|------|------|
| `ISndSceneAccess.cs` | 最小场景访问：序列化元数据列表 / 从元数据恢复 |
| `ISndSceneHost.cs` | 场景宿主（继承 ISndSceneAccess）：生成实体 / 查询实体 / 帧更新 / 实体销毁 / 场景清空 |

## 接口详细

### ISndSceneAccess

| 成员 | 说明 |
|------|------|
| `SerializeMetaList()` | 导出当前场景全部实体元数据列表 |
| `LoadFromMetaList(metaList)` | 从元数据列表恢复场景 |

### ISndSceneHost : ISndSceneAccess

| 自有成员 | 说明 |
|------|------|
| `Spawn(SndMetaData)` | 从元数据创建实体（不校验重名） |
| `GetEntities()` | 枚举所有存活实体 |
| `FindByName(name)` | 按名查找实体 |
| `ProcessAll(delta)` | 对所有存活实体执行帧更新 |
| `RequestKillEntity(name)` | 立即将指定实体标记为待销毁（帧末统一执行）。若实体不存在或已标记则抛异常 |
| `DeadByName(name)` | 按名称销毁单个实体（立即执行，从集合移除并触发 BeforeDead）。仅由框架统一 Kill 步骤调用 |
| `ClearAll()` | 清空场景全部实体（触发 Quit 流程）。仅由框架在生命周期切换时内部调用 |

## 设计决策

### 为什么分离 ISndSceneAccess 和 ISndSceneHost

状态机上下文只需要 `ISndSceneAccess` 的序列化/恢复能力，不需要实体生成和查询。ISP 分离让依赖更精确：存档系统通过 `ISndSceneAccess` 操作场景，会话管理通过 `ISndSceneHost` 管理实体生命周期。两者对外暴露不同消费场景。

### 为什么 ClearAll 定义在 ISndSceneHost 而非 ISndSceneAccess

`ClearAll()` 触发 Quit 流程，语义为场景卸载。它是框架生命周期切换时的内部操作，不应暴露给业务代码和状态机上下文。`ISndSceneHost` 作为会话语义接口承载此操作，`ISndSceneAccess` 仅提供序列化/恢复能力。

### 为什么 Spawn 不做重名校验

重名校验是上层业务规则（通过 `SndRuntime.Spawn` 执行），不在接口层强制。接口保持最小语义，将校验职责留给编排层。参见 [SndRuntime](../../Snd/Scene/README.md)。

### 为什么 Kill 分为 RequestKillEntity（标记）和 DeadByName（执行）

`RequestKillEntity` 立即标记实体为待销毁（`IsPendingKill = true`），但不立即物理移除。这允许同帧内后续操作通过 `IsPendingKill` 判断实体存活状态，避免延迟 Kill 导致的重复操作。物理销毁在帧末由 `KillPendingEntities()` 统一执行（业务队列之后、系统队列之前）。`DeadByName` 仅由框架统一 Kill 步骤调用，业务代码应使用 `RequestKillEntity` 或 `RequestKillAll`。

---
[↑ 回到 Abstractions](../README.md)
