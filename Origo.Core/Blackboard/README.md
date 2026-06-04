# Blackboard

> [↑ 回到 Origo.Core](../README.md) · [↔ 抽象: Abstractions/Blackboard](../Abstractions/Blackboard/README.md) · [相关测试: Blackboard](../../Origo.Core.Tests/Blackboard.md)

## 概述

`IBlackboard` 接口的默认内存实现。使用 `Dictionary<string, TypedData>` 作为底层存储，以 `StringComparer.Ordinal` 进行键比对，确保键名的大小写敏感精确匹配。

## 包含文件

| 文件 | 职责 |
|------|------|
| `Blackboard.cs` | `IBlackboard` 的 `sealed` 实现，键值通过 `TypedData` 保留类型 |

## 实现细节

- **键校验**：`Set` 和 `TryGet` 均对空/null 键做防御性检查，抛出 `ArgumentException`
- **Set**：将值包装为 `new TypedData(typeof(T), value)` 存入字典，泛型类型在编译期捕获
- **TryGet**：先查找 `TypedData`，再通过模式匹配 `typedData.Data is T value` 校验运行时类型
- **SerializeAll/DeserializeAll**：全量导出/导入，不涉及增量合并。`DeserializeAll` 先清空再填充，替换语义

## 设计决策

### 为什么是 sealed

`Blackboard` 是纯数据容器，不应通过继承扩展行为。需要定制黑板行为的场景应通过组合（包装新的 `IBlackboard` 实现）或装饰器模式实现，而非继承。

### 为什么使用 Ordinal 比较

键名是代码中硬编码的常量（如 `"core.player.health"`），不涉及文化相关排序。`Ordinal` 提供最快的字符串比对性能。

### 为什么 DeserializeAll 是替换而非合并

存档恢复是"从已知状态恢复"的语义，不是"增量更新"。部分键残留会导致旧存档数据污染新会话，替换语义更安全。

---
[↑ 回到 Origo.Core](../README.md)
