# Strategy

> [↑ 回到 Snd](../README.md)

## 概述

SND 策略系统的完整实现。策略是实体行为逻辑的载体，遵循"无状态共享"模型：策略实例在池中全局共享，不持有实例字段，所有可变状态存储在实体的 Data 中。

## 包含文件

| 文件 | 职责 |
|------|------|
| `BaseStrategy.cs` | 所有策略的抽象根基类 |
| `EntityStrategyBase.cs` | 实体策略基类：8 个生命周期虚方法钩子 |
| `SndStrategyPool.cs` | 策略池：注册、实例化、引用计数、无状态校验 |
| `SndStrategyManager.cs` | 单实体策略管理器：增删策略 + 协调生命周期回调 |
| `StrategyIndexAttribute.cs` | 策略索引声明特性：`[StrategyIndex("core.health")]` |

## 模块详解

### 策略继承体系

```
BaseStrategy
├── EntityStrategyBase    (实体策略: Process, AfterSpawn, AfterLoad, AfterAdd, BeforeRemove, BeforeSave, BeforeQuit, BeforeDead)
└── StateMachineStrategyBase  (状态机策略: OnPushRuntime, OnPushAfterLoad, OnPopRuntime, OnPopBeforeQuit)
```

### SndStrategyPool

策略的全局注册表和实例池：

- **Register**：`Register(Type strategyType, Func<BaseStrategy> factory)` —— 注册类型前通过反射校验无状态性（检查实例字段和可写属性）
- **GetStrategy<TBase>(index)**：若池中已有则复用（引用计数 +1），否则通过工厂创建
- **ReleaseStrategy(index)**：引用计数 -1，归零时从池中移除（但工厂保留，下次可再创建）
- **GetPriority(index)**：返回策略在实体上的执行优先级（默认 6205）
- **策略排序**：同实体上的多个策略按优先级升序排列，同优先级按插入顺序

### SndStrategyManager

每个 `SndEntity` 持有一个 manager 实例：

- **策略列表**：`List<StrategyEntry>`（Index + EntityStrategyBase），按优先级升序插入
- **Process**：基于快照迭代，允许 Process 中增删策略
- **生命周期钩子触发**：全部基于 `ToArray()` 快照迭代——因为钩子内可增删策略
- **Release**：逐个 Release 池引用，然后清空列表
- **错误回滚**：Recover（恢复策略列表）中若某策略注册失败，回滚 Release 已恢复的

### StrategyIndexAttribute

```csharp
[StrategyIndex("my_game.player_control", Priority = 100)]
public class PlayerControlStrategy : EntityStrategyBase { ... }
```

- `Index`：必填，策略在池中的唯一索引键
- `Priority`：可选，默认 6205，决定同实体上多策略的执行顺序

## 设计决策

### 为什么策略强制无状态（注册期校验）

策略实例在多个实体间共享，若持有实例字段（如 `int _hp`），多实体间会互相污染。注册期通过反射检查 `BaseStrategy` 到具体类型之间的所有层级，拒绝声明实例字段或可写属性的策略，从源头阻止此错误。

### 为什么使用引用计数而非单一实例

同一策略可能被多个实体同时引用（如 `core.health` 策略在每个实体上都活跃）。引用计数确保只要有一个实体持有，策略就不被回收。归零时才释放，下次访问重新创建或复用。

### 为什么策略执行顺序按优先级排序

复杂实体可能有多层策略（如渲染策略需要在物理策略之后执行）。优先级提供确定性的执行顺序，避免依赖隐式的注册顺序。默认值 6205 让未声明优先级的策略居中，有足够空间向上（高优先级先执行）和向下调整。

### 为什么生命周期钩子全都基于快照迭代

钩子回调中经常需要增删策略（例如 `BeforeDead` 中移除自身策略）。直接迭代修改中的列表会导致异常。快照确保了每次钩子调用的列表稳定性，以少量分配换取安全。

---
[↑ 回到 Snd](../README.md)
