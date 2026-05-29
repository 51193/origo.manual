# Runtime (Abstractions)

> [↑ 回到 Abstractions](../README.md) · [↔ 实现: Scheduling](../../Scheduling/README.md)

## 概述

定义抽象调度接口 `IScheduler`，由宿主环境（Godot 主循环播放、测试驱动）驱动帧或周期的执行。Core 层通过该接口将动作放入延迟队列，后续按帧/周期批量执行。

## 包含文件

| 文件 | 职责 |
|------|------|
| `IScheduler.cs` | 调度核心接口：Enqueue / Tick / Clear |

## 接口成员

| 方法 | 说明 |
|------|------|
| `Enqueue(Action)` | 将动作安排在本帧或之后的某个阶段执行 |
| `Tick()` | 执行已排队动作，返回本周期执行的动作数量 |
| `Clear()` | 清空尚未执行的动作队列 |

## 设计决策

### 为什么 IScheduler 放在 Abstractions/Runtime 而非 Scheduling

`IScheduler` 是对 Core 层可见的抽象契约（`public`），而 `Scheduling/` 下的 `ActionScheduler` 是该契约的具体实现。Abstractions 是 Core 的公共抽象层，Scheduling 是具体模块实现。接口与实现分层分离符合项目架构约定。

### 为什么不提供取消单个动作的能力

在单线程帧循环模型中，帧内排队的动作一般是一次性的轻量事务，不需要取消。如果需要条件执行，应由策略在 Enqueue 前自行判断，或在 Action 内部做早期退出。引入取消机制会显著增加队列实现复杂度，但不解决实际业务问题。

---
[↑ 回到 Abstractions](../README.md)
