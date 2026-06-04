# 状态机 测试

> [↑ 回到 Origo.Core.Tests](README.md)
> [↔ 被测模块: Origo.Core/StateMachine](../../Origo.Core/StateMachine/README.md)
> [↔ 被测行为: usage/state-machine](../../usage/state-machine.md)

## 被测行为概览

验证 StackStateMachine 的字符串栈操作：Push/PopRuntime/PopOnQuit 触发对应策略钩子、
Snapshot/RestoreStackWithoutHooks/FlushAfterLoad 两阶段恢复、
StateMachineContainer 的 CreateOrGet/TryGet/Remove/Clear 容器操作、
PopAllRuntime/PopAllOnQuit 批量操作、序列化/反序列化往返。

## 测试文件清单

| 文件 | 验证侧重点 |
|------|-----------|
| `StateMachineStrategyBaseTests.cs` | 默认钩子不调度动作 |
| `RandomAndStateMachineTests.StringStack.cs` | StringStack 核心操作：Push/Pop/恢复/FlushAfterLoad |
| `RandomAndStateMachineTests.Container.cs` | Container：CreateOrGet/序列化/反序列化/批量Pop |
| `RandomAndStateMachineTests.SessionAndAdapter.cs` | 状态机在会话和适配层中的集成 |
| `RandomAndStateMachineTests.Random.cs` | 状态机与随机数集成 |
| `RandomAndStateMachineTests.TestStrategies.cs` | 测试辅助策略定义 |

## StateMachineStrategyBaseTests 测试详情

### 边界路径

| 测试方法 | 边界条件 | 预期行为 |
|---------|---------|---------|
| `DefaultHooks_DoNotScheduleActions` | 全部 4 个默认钩子调用 | EnqueueCount = 0 |

## StringStack 测试详情

### 正确路径

| 测试方法 | 验证的行为 | 文档出处 |
|---------|-----------|---------|
| `StringStackStateMachine_Snapshot_RestoreStackWithoutHooks_RoundTrip` | Snapshot → Restore 后栈一致 | state-machine: 读档恢复（两阶段） |
| `PushPopRuntime_AfterAddAndBeforeRemove_OrderAndContext` | Push→Pop 触发正确钩子，上下文 BeforeTop/AfterTop 正确 | state-machine: TryPopRuntime |
| `PushPopOnQuit_AfterAddAndBeforeQuit_OrderAndContext` | PopOnQuit 触发 BeforeQuit 钩子 | state-machine: TryPopOnQuit |
| `FlushAfterLoad_CallsAfterLoadInPushOrder` | RestoreWithoutHooks→FlushAfterLoad 按入栈顺序重放 | state-machine: 读档恢复 |

### 错误路径

| 测试方法 | 触发的错误 | 预期行为 |
|---------|-----------|---------|
| `StringStackStateMachine_Throws_WhenStrategyNotRegistered` | 使用未注册的策略索引创建状态机 | InvalidOperationException |

## Container 测试详情

### 正确路径

| 测试方法 | 验证的行为 | 文档出处 |
|---------|-----------|---------|
| `PopAllRuntime_InvokesBeforeRemoveTopToBottom` | PopAllRuntime 按寄存器 LIFO 弹空 | state-machine: 容器操作 |
| `PopAllOnQuit_InvokesBeforeQuitTopToBottom` | PopAllOnQuit 触发 BeforeQuit | state-machine: 容器操作 |
| `PopAllOnQuit_TraversesMachinesInInsertionOrder` | 多状态机按插入顺序遍历 | state-machine |
| `SerializeDeserialize_RoundTrip` | 序列化→反序列化后状态机栈一致 | state-machine: 序列化格式 |
| `DeserializeWithoutHooks_SwapsAtomically` | 无钩子恢复后旧状态被替换 | state-machine |
| `CreateOrGet_IdempotentForSameKeyAndIndices` | 同 key+同索引 CreateOrGet 返回同实例 | state-machine |
| `FlushAllAfterLoad_NotifiesPushStrategy` | FlushAllAfterLoad 重放 AfterLoad | state-machine |

### 错误路径

| 测试方法 | 触发的错误 | 预期行为 |
|---------|-----------|---------|
| `CreateOrGet_ConflictingIndices_Throws` | 同 key 不同索引的 CreateOrGet | InvalidOperationException |
| `DeserializeFromNode_DuplicateMachineKey_Throws` | 反序列化含重复 key | InvalidOperationException |
| `DeserializeFromNode_ThrowsOnNullNode` | null 节点反序列化 | ArgumentNullException |

### 边界路径

| 测试方法 | 边界条件 | 预期行为 |
|---------|---------|---------|
| 空栈 TryPeek/TryPop 返回 false | 空状态机 | false |

## 测试辅助策略

| 策略类 | 定义位置 | 用途 |
|--------|---------|------|
| `SmPushStrategy` | RandomAndStateMachineTests.TestStrategies.cs | Push 钩子：记录 "push:runtime:null→a" 格式事件 |
| `SmPopStrategy` | RandomAndStateMachineTests.TestStrategies.cs | Pop 钩子：记录 PopRemove/PopQuit 事件 |
| `SmPopOrderProbeStrategy` | RandomAndStateMachineTests.TestStrategies.cs | 记录 MachineKey，验证多状态机遍历顺序 |
| `SwapTestPushStrategy` | RandomAndStateMachineTests.Container.cs | 空 Push 钩子 |
| `SwapTestPopStrategy` | RandomAndStateMachineTests.Container.cs | 空 Pop 钩子 |

## 已知覆盖缺口

| 缺口描述 | 影响 | 文档依据 |
|---------|------|---------|
| RestoreStackWithoutHooks 后 Peek 返回栈顶 | 恢复后栈状态验证 | state-machine |
| Push 空字符串 | 空字符串状态值 | IStateMachine |
| Pop 空栈后连续 Push 的行为 | 栈为空后的正常操作 | IStateMachine |
| RestoreStackWithoutHooks 传入 null 或空列表 | 防御性校验 | IStateMachine |

---

[↑ 回到 Origo.Core.Tests](README.md)
