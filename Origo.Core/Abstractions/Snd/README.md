# Snd (Abstractions)

> [↑ 回到 Abstractions](../README.md) · [↔ 实现: Snd](../../Snd/README.md)

## 概述

ISndContext 的角色接口拆分。将原先 30 成员的巨型接口按职责分解为 8 个窄接口，遵循接口隔离原则（ISP）。ISndContext 本身作为组合接口继承全部角色。

## 包含文件

| 文件 | 职责 |
|------|------|
| `ISndBlackboardAccess.cs` | 系统级 + 流程级黑板访问（2 成员） |
| `ISndSessionAccess.cs` | 会话管理器 + 当前会话 + 前台判定（3 成员） |
| `ISndDeferredActions.cs` | 延迟动作队列：入队 + 冲刷 + 计数（3 成员） |
| `ISndTemplateAccess.cs` | 模板克隆（1 成员） |
| `ISndConsoleAccess.cs` | 控制台命令提交/处理/输出订阅（4 成员） |
| `ISndStateMachineAccess.cs` | 流程级状态机容器访问（1 成员） |
| `ISndSaveOperations.cs` | 存档列表/读/写 + 关卡切换 + 实体清空/单体销毁（8 成员） |
| `ISndLifecycleOperations.cs` | Continue/Initial/MainMenu 生命周期入口（4 成员） |

## ISndContext 组合

```
ISndContext : ISndBlackboardAccess + ISndSessionAccess + ISndDeferredActions
            + ISndTemplateAccess + ISndConsoleAccess + ISndStateMachineAccess
            + ISndSaveOperations + ISndLifecycleOperations
```

ISndContext 自身不声明任何成员——所有成员均来自继承的角色接口。

## 与 IStateMachineContext 的关系

`IStateMachineContext` 继承 `ISndBlackboardAccess` 和 `ISndDeferredActions` 两个共享角色接口，避免了两个接口中的成员重复定义：

```
IStateMachineContext : ISndBlackboardAccess + ISndDeferredActions
                     + SessionBlackboard + SceneAccess
```

## 设计决策

### 为什么拆分 ISndContext

原先 30 成员的 ISndContext 虽文档注明为"统一业务门面"，但为单一接口，大型接口违反 ISP。拆分后每个消费者可按需依赖窄接口：

- 仅需黑板访问的代码可依赖 `ISndBlackboardAccess`
- 仅需延迟队列的代码可依赖 `ISndDeferredActions`
- 仅需存档操作的代码可依赖 `ISndSaveOperations`
- 等等

策略钩子（`EntityStrategyBase` 的 8 个虚方法）保持 `ISndContext ctx` 全量参数——零 breaking change，策略作者仍可访问全部能力。

### 为什么 ISndContext 仍作为组合接口存在

策略钩子需要全量能力（如 `Process` 中也允许 `RequestSaveGame`）。作为组合接口保持调用方兼容性，拆分仅在类型层级表达职责边界。

### 为什么 IStateMachineContext 也继承了角色接口

`IStateMachineContext` 原有 5 个成员，其中 `SystemBlackboard`、`ProgressBlackboard`、`EnqueueBusinessDeferred` 与 `ISndContext` 中的语义完全一致。通过继承 `ISndBlackboardAccess` + `ISndDeferredActions`，消除了跨接口的重复定义，同时保持 IStateMachineContext 的独立语义（SessionBlackboard + SceneAccess 为状态机特有）。

---
[↑ 回到 Abstractions](../README.md)
