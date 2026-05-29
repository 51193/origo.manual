# Scene (Abstractions)

> [↑ 回到 Abstractions](../README.md) · [↔ 实现: Snd/Scene](../../Snd/Scene/README.md)

## 概述

定义 Core 层编排 SND 场景的抽象能力。`ISndSceneAccess` 提供最小化的串行化/恢复/清空操作，`ISndSceneHost` 在其基础上补充实体生成、查询和帧更新能力。

## 包含文件

| 文件 | 职责 |
|------|------|
| `ISndSceneAccess.cs` | 最小场景访问：序列化元数据列表 / 从元数据恢复 / 清空 |
| `ISndSceneHost.cs` | 场景宿主（继承 ISndSceneAccess）：生成实体 / 查询实体 / 帧更新 |

## 接口详细

### ISndSceneAccess

| 成员 | 说明 |
|------|------|
| `SerializeMetaList()` | 导出当前场景全部实体元数据列表 |
| `LoadFromMetaList(metaList)` | 从元数据列表恢复场景 |
| `ClearAll()` | 清空场景全部实体 |

### ISndSceneHost : ISndSceneAccess

| 新增成员 | 说明 |
|------|------|
| `Spawn(SndMetaData)` | 从元数据创建实体（不校验重名） |
| `GetEntities()` | 枚举所有存活实体 |
| `FindByName(name)` | 按名查找实体 |
| `ProcessAll(delta)` | 对所有存活实体执行帧更新 |

## 设计决策

### 为什么分离 ISndSceneAccess 和 ISndSceneHost

状态机上下文只需要 `ISndSceneAccess` 的序列化/恢复能力，不需要实体生成和查询。ISP 分离让依赖更精确：存档系统通过 `ISndSceneAccess` 操作场景，会话管理通过 `ISndSceneHost` 管理实体生命周期。两者对外暴露不同消费场景。

### 为什么 Spawn 不做重名校验

重名校验是上层业务规则（通过 `SndRuntime.Spawn` 执行），不在接口层强制。接口保持最小语义，将校验职责留给编排层。参见 [SndRuntime](../../Snd/Scene/README.md)。

---
[↑ 回到 Abstractions](../README.md)
