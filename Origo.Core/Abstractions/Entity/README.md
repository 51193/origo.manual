# Entity (Abstractions)

> [↑ 回到 Abstractions](../README.md) · [↔ 实现: Snd/Entity](../../Snd/Entity/README.md)

## 概述

定义 SND 实体的抽象接口体系。所有接口遵循接口隔离原则（ISP），将实体的数据、节点、策略三种能力拆分为独立接口，再由 `ISndEntity` 组合统一入口。

## 包含文件

| 文件 | 职责 |
|------|------|
| `ISndDataAccess.cs` | 数据存取与变更订阅 |
| `ISndNodeAccess.cs` | 节点查询与枚举 |
| `ISndStrategyAccess.cs` | 策略动态添加/移除 |
| `ISndActiveStrategyAccess.cs` | 主动策略：添加/移除/调用 |
| `ISndEntity.cs` | 组合接口：继承上述四个接口 + `Name` 属性 |

## 接口详细

### ISndDataAccess

| 成员 | 说明 |
|------|------|
| `SetData<T>(name, value)` | 写入命名数据 |
| `GetData<T>(name)` | 直接读取；键不存在时抛异常 |
| `TryGetData<T>(name)` | 安全读取，返回 `(found, value?)` |
| `Subscribe(name, callback, filter?)` | 订阅数据变更通知 |
| `Unsubscribe(name, callback)` | 取消订阅 |

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

### ISndEntity

`ISndEntity : ISndDataAccess, ISndNodeAccess, ISndStrategyAccess, ISndActiveStrategyAccess`

`ISndEntity` 组合后唯一自有的成员：`string Name { get; }`——稳定的实体标识名。

## 设计决策

### 为什么拆分为四个访问接口

策略代码通常只需访问具体某一方面的实体能力（例如只读写数据而不管理节点）。ISP 拆分让策略声明依赖时更精确，也便于测试时只 mock 需要的部分。

### 为什么主动策略独立为 ISndActiveStrategyAccess

主动策略与被动实体策略共享 `BaseStrategy` 和 `SndStrategyPool` 基础设施，但容器完全独立：主动策略使用 Dictionary 实现 O(1) 按索引查找，不参与帧遍历；被动策略使用排序列表，每帧按优先级迭代。接口分离避免消费者耦合到不必要的策略类型。

### 为什么 TryGetData 使用 found/value 元组

参考 [Blackboard 的相同设计决策](../Blackboard/README.md#为什么是泛型tryget而非object)。

---
[↑ 回到 Abstractions](../README.md)
