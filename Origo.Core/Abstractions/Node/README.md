# Node (Abstractions)

> [↑ 回到 Abstractions](../README.md) · [↔ 实现: GodotAdapter/Snd](../../../Origo.GodotAdapter/Snd/README.md)

## 概述

定义抽象引擎节点操作接口体系。Core 层通过 `INodeHandle` 触发基础节点行为（可见性、释放），通过 `INodeFactory` 创建节点实例，通过 `INodeHost`（internal）管理节点的恢复与导出。所有接口均不暴露具体引擎类型。

## 包含文件

| 文件 | 职责 |
|------|------|
| `INodeFactory.cs` | 按资源标识创建节点实例 |
| `INodeHandle.cs` | 抽象节点句柄：Name / Native / Free / SetVisible |
| `INodeHost.cs` | internal：节点容器行为——恢复、回收、导出元数据 |

## 接口详细

### INodeFactory

| 成员 | 说明 |
|------|------|
| `Create(logicalName, resourceId)` | 创建节点并挂载到宿主，返回句柄 |

### INodeHandle

| 成员 | 说明 |
|------|------|
| `Name` | 节点逻辑名 |
| `Native` | 平台原生对象引用（`object`） |
| `Free()` | 释放节点资源 |
| `SetVisible(bool)` | 控制节点可见性 |

### INodeHost (internal)

| 成员 | 说明 |
|------|------|
| `GetNode(name)` | 按名获取节点句柄 |
| `GetNodeNames()` | 枚举已挂载节点名称 |
| `Recover(NodeMetaData)` | 从元数据恢复节点 |
| `Release()` | 回收全部节点 |
| `SerializeMetaData()` | 导出当前节点元数据 |

## 设计决策

### 为什么 INodeHost 是 internal

`INodeHost` 是 SND 实体内部管理节点的契约，不是对外公开的能力。外部策略代码通过 `ISndEntity`（组合了 `ISndNodeAccess`）访问节点，不需要感知节点容器的恢复/回收生命周期。internal 可见性防止策略代码绕过实体直接操作节点池。

### 为什么 INodeHandle.Native 是 object

`object` 是 C# 中最通用的类型引用，不引入对 Godot 或其他引擎的依赖。适配层实现时可以安全地将 `Godot.Node` 赋值给 `Native`，Core 层代码不直接访问它。

---
[↑ 回到 Abstractions](../README.md)
