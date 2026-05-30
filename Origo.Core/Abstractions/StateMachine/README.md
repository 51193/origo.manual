# StateMachine (Abstractions)

> [↑ 回到 Abstractions](../README.md) · [↔ 实现: StateMachine](../../StateMachine/README.md)

## 概述

定义字符串栈状态机体系。状态机本身只存储字符串，Push/Pop 的具体语义由关联的策略钩子实现。同时定义了状态机策略钩子所需的运行时上下文接口 `IStateMachineContext`，以及其会话级适配器 `SessionStateMachineContext`。

## 包含文件

| 文件 | 职责 |
|------|------|
| `IStateMachine.cs` | 字符串栈状态机接口：Push/Pop/Peek/Snapshot + 持久化 |
| `IStateMachineContext.cs` | 状态机策略钩子运行时上下文：黑板访问 + 场景访问 + 延迟队列 |
| `SessionStateMachineContext.cs` | internal：会话级适配器，将 SessionBlackboard/SceneAccess 绑定到当前会话 |

## IStateMachine 成员

| 方法 | 说明 |
|------|------|
| `MachineKey` | 状态机唯一标识 |
| `PushStrategyIndex` / `PopStrategyIndex` | 关联的 Push/Pop 策略索引 |
| `Push(value)` | 压栈 |
| `TryPopRuntime(out popped)` | 运行时出栈，触发 pop 策略的 BeforeRemove |
| `TryPopOnQuit(out popped)` | 退出逐级出栈，触发 pop 策略的 BeforeQuit |
| `Peek()` | 查看栈顶 |
| `Snapshot()` | 栈底到栈顶快照 |
| `FlushAfterLoad()` | 读档后按入栈顺序重放 Push 策略的 AfterLoad |
| `RestoreStackWithoutHooks(list)` | 从存档恢复栈内容，不触发策略钩子 |

## IStateMachineContext 成员

> 继承自 [ISndBlackboardAccess](../Snd/README.md) 和 [ISndDeferredActions](../Snd/README.md) 的成员不再重复列出。

| 成员 | 说明 |
|------|------|
| `SystemBlackboard` | 系统级黑板（跨进程），继承自 [ISndBlackboardAccess](../Snd/README.md) |
| `ProgressBlackboard` | 进度级黑板；无活动流程时为 null，继承自 [ISndBlackboardAccess](../Snd/README.md) |
| `EnqueueBusinessDeferred(action)` | 将业务逻辑延迟动作入队，继承自 [ISndDeferredActions](../Snd/README.md) |
| `FlushDeferredActionsForCurrentFrame()` | 冲刷延迟队列，继承自 [ISndDeferredActions](../Snd/README.md) |
| `GetPendingPersistenceRequestCount()` | 待持久化请求数，继承自 [ISndDeferredActions](../Snd/README.md) |
| `SessionBlackboard` | 会话级黑板；无活动会话时为 null（自有） |
| `SceneAccess` | 当前会话 SND 场景访问（自有） |

## 设计决策

### 为什么状态机只存储字符串而非状态对象

将状态逻辑委托给 `StateMachineStrategyBase`（见 [StateMachine 实现](../../StateMachine/README.md)），状态机本身只维护一个标识符栈。这保持了状态机的轻量和无状态特性，所有业务逻辑集中在策略中，便于测试和复用。

### 为什么分离 TryPopRuntime 和 TryPopOnQuit

两种出栈触发不同的策略钩子语义。运行时出栈触发 `BeforeRemove`（正常状态流转），退出逐级出栈触发 `BeforeQuit`（状态机销毁时的清理）。如果合并为一个方法，调用方需传递额外参数区分语义，增加误用风险。

### 为什么 SessionStateMachineContext 是 internal

外部代码只需要 `IStateMachineContext` 接口。`SessionStateMachineContext` 是会话层的实现细节，其构造函数参数（全局上下文 + 会话黑板 + 场景访问）由 `SessionManager` 在构造 `SessionRun` 时注入，不应被外部直接实例化。

---
[↑ 回到 Abstractions](../README.md)
