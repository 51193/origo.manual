# Metadata

> [↑ 回到 Snd](../README.md)

## 概述

SND 实体元数据的数据模型。这些模型是纯数据容器（POCO），不包含业务逻辑，仅作为 Core 层各子系统之间传递实体信息的标准契约。所有类型公开（`public sealed`），用于序列化/反序列化、网络传输、存档存储。

## 包含文件

| 文件 | 职责 |
|------|------|
| `TypedData.cs` | 带类型信息的单条数据值 |
| `DataMetaData.cs` | 实体数据字典容器 |
| `NodeMetaData.cs` | 节点元数据（逻辑名 → 资源 ID 映射） |
| `StrategyMetaData.cs` | 策略索引列表 |
| `SndMetaData.cs` | 实体元数据聚合：Name + Node + Strategy + Data |

## 模型详解

### TypedData

```csharp
public sealed class TypedData {
    public Type DataType { get; }
    public object? Data { get; }
}
```

SND 系统的核心类型保留机制。在存储时捕获泛型 `T` 的类型信息（`typeof(T)`），在反序列化时通过 `TypeStringMapping` 还原为 `Type`，确保 `int` 不会在 JSON 往返后变成 `double`。

### SndMetaData

聚合实体全部元数据：

| 字段 | 类型 | 说明 |
|------|------|------|
| `Name` | `string` | 实体唯一标识名 |
| `NodeMetaData` | `NodeMetaData?` | 节点映射；后台实体可为 null |
| `StrategyMetaData` | `StrategyMetaData?` | 策略索引列表 |
| `DataMetaData` | `DataMetaData?` | 数据字典；默认为空容器 |

提供 `DeepClone()` 方法：浅复制值引用（不递归复制对象图），但新建字典/列表容器。

### DataMetaData

仅包装一个 `Dictionary<string, TypedData>`。每个键对应一个带类型的实体数据点。

### NodeMetaData

包装 `Dictionary<string, string>`，存储逻辑节点名到资源标识符的映射。具体资源 ID 的语义由适配层定义（如 Godot 中为 `res://` 路径）。

### StrategyMetaData

包装 `List<string>`，存储该实体关联的策略索引列表（如 `["core.health", "gameplay.movement"]`）。

## 设计决策

### 为什么元数据是 public sealed 而非 record

这些类型设计为可序列化的纯数据容器。`sealed` 确保无继承扩展（违反协议一致性的风险），`public` 是因为它们需要跨程序集消费（适配层、测试）。不使用 record 是因为这类值相等语义并非必需，且 record 的 `with` 表达式在当前场景不提供价值。

### 为什么 DeepClone 不递归复制对象图

Entity 的 Data 值可能是复杂引用类型（如字符串、数组、嵌套对象）。全量递归深拷贝成本高且可能错误复制引擎内部引用（通过 `INodeHandle.Native` 等）。当前浅复制语义与 JSON 往返序列化行为一致——如果 JSON 序列化不会复制，DeepClone 也不复制。

### 为什么 TypedData.DataType 是 Type 而非 string

直接用 `System.Type` 允许在运行时做类型检查和转换，无需通过 `TypeStringMapping` 的额外查找。从 `string` 恢复 `Type` 的代价只在反序列化时支付一次。

---
[↑ 回到 Snd](../README.md)
