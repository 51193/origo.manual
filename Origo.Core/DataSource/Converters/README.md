# Converters

> [↑ 回到 DataSource](../README.md)

## 概述

`DataSourceConverter<T>` 的注册实现集合。负责将 `DataSourceNode` 与 CLR 类型（基础类型、数组、领域类型）互相转换。所有转换器均为 `internal`，由 `DataSourceConverterRegistry` 统一管理和调度。

## 包含文件

| 文件 | 职责 |
|------|------|
| `PrimitiveConverters.cs` | 14 种基础类型转换器（string, byte, int, float, bool 等） |
| `ArrayConverters.cs` | 14 种基础类型数组转换器（byte[], int[], float[] 等） |
| `DomainConverters.cs` | 领域类型转换器（SndMetaData, Blackboard, StateMachine 等） |
| `TypedDataConverter.cs` | TypedData ↔ DataSourceNode，携带类型元数据 |

## 转换器一览

### PrimitiveConverters

`string`, `byte`, `sbyte`, `short`, `ushort`, `int`, `uint`, `long`, `ulong`, `float`, `double`, `decimal`, `char`, `bool`

读写模式统一：Read 调用 `DataSourceNode.AsXxx()`，Write 调用 `DataSourceNode.CreateXxx()`。

### ArrayConverters

每个基础类型的数组对应一个转换器。Read 遍历 `node.Elements` 逐个转换，Write 构建 `DataSourceNode.CreateArray()` 并填充元素。

### DomainConverters

| 转换器 | 处理类型 |
|------|------|
| `NodeMetaDataConverter` | 节点元数据（pairs 字典） |
| `StrategyMetaDataConverter` | 策略索引列表 |
| `DataMetaDataConverter` | 实体数据（依赖 TypedDataConverter） |
| `SndMetaDataConverter` | SND 实体元数据（组合上述三个） |
| `SndMetaDataListConverter` | 实体元数据列表 |
| `BlackboardDataConverter` | 黑板全部数据字典 |
| `StringDictionaryConverter` | 字符串字典 |
| `StateMachineContainerPayloadConverter` | 状态机组序列化 |

### TypedDataConverter

特殊转换器，携带类型元数据。读取时从 `"type"` 字段获取 CLR 类型名，通过 `TypeStringMapping` 解析为 `Type`，再用注册表中的对应转换器读取 `"data"` 字段。这是序列化系统保持类型信息的核心机制。

## 设计决策

### 为什么每种基础类型独立一个转换器

泛型转换器（如单个 `PrimitiveConverter<T>`）需要在运行时通过反射实例化不同 T 的版本，违反零反射约束。每个具体类型显式实现避免反射，且在注册表中可静态枚举。

### 为什么数组转换器独立于基础类型转换器

数组是复合类型，其 Read/Write 需要遍历语义（foreach over `Elements`），与标量类型（直接 AsXxx）差异显著。合并会导致转换器内部出现类型分支，违反单一职责。

### 为什么 DomainConverters 共享同一个文件

领域类型转换器之间存在层级依赖（如 `SndMetaDataConverter` 依赖 `DataMetaDataConverter`），放在同一文件有助于保持依赖关系的可见性，且每个转换器体量较小（30-60 行）。

---
[↑ 回到 DataSource](../README.md)
