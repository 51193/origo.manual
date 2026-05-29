# Lifecycle

> [↑ 回到 Runtime](../README.md)

## 概述

运行时四层生命周期的实现层。定义了从系统级到会话级的完整启动、运行、退出流程。所有类型通过结构化参数对象构造，上层依赖单向向下传递。

## 包含文件

| 文件 | 职责 |
|------|------|
| `SystemParameters.cs` | 系统层构造参数 |
| `SystemRuntime.cs` | 系统级运行时容器：持有 SystemRun、SystemBlackboard、SndWorld |
| `SystemRun.cs` | 系统层启动：创建 SndWorld → 构造 ProgressRuntime |
| `ProgressParameters.cs` | 流程层构造参数 |
| `ProgressRuntime.cs` | 流程级运行时容器：持有 ProgressRun、ProgressBlackboard、SaveContext |
| `ProgressRun.cs` | 流程层主逻辑：关卡切换、读写存档、会话生命周期编排 |
| `ProgressRun.Persistence.cs` | 流程层持久化分支（partial class）|
| `ProgressRun.SessionLoading.cs` | 流程层会话加载分支（partial class）|
| `SessionParameters.cs` | 会话层构造参数 |
| `SessionManager.cs` | 会话管理器完整实现 |
| `SessionManagerRuntime.cs` | 会话管理器运行时容器 |
| `SessionRun.cs` | 会话级别运行时实现 |
| `SessionTopologyCodec.cs` | 会话拓扑编解码（前台+后台会话关系）|
| `ISessionManager.cs` | 会话管理器接口（public）|
| `ISessionRun.cs` | 会话运行时接口（public）|
| `EmptySessionManager.cs` | 无操作会话管理器（测试/空场景）|
| `RunStateScope.cs` | 运行时状态作用域工具 |

## 四层容器模型

```
SystemRuntime
├── SystemRun
│   └── ProgressRuntime
│       └── ProgressRun
│           └── SessionManagerRuntime (via SessionManager)
│               └── SessionRun (foreground + background)
```

每层容器持有本层的核心对象引用和公共访问入口：
- `SystemRuntime` 持有 `SystemBlackboard`、`SndWorld`、`IScheduler`
- `ProgressRuntime` 持有 `ProgressBlackboard`、`SaveContext`、`SessionManager`
- `SessionManagerRuntime` 持有 `ISessionManager`
- `SessionRun` 持有 `SessionBlackboard`、`ISndSceneHost`、`StateMachineContainer`

## 关键生命周期流程

### 启动

1. `OrigoAutoHost` 创建 `SystemParameters` → `SystemRun.Create()`
2. `SystemRun` 创建 `SndWorld`（策略池 + 配置）→ `ProgressRuntime`
3. 加载初始关卡时：`ProgressRun` 创建 `SessionManager` → `SessionRun`（前台 + 后台）

### 运行

- 每帧：`IScheduler.Tick()` → 执行延迟队列 → `SessionRun.ProcessAllSessions()`
- 控制台命令路由到 `OrigoConsole.ProcessPending()`

### 持久化

- `ProgressRun.RequestSaveGame` → `SaveContext.SaveGame(...)` → `SavePayloadWriter.WriteToCurrent()` → snapshot
- `ProgressRun.RequestLoadGame` → `SavePayloadReader.ReadFromCurrent/Snapshot` → 恢复黑板 + 场景

### 退出

- `Dispose` 级联：SessionRun → SessionManager → ProgressRun → SystemRun
- 退出时自动触发持久化（auto-persist 语义）

## 设计决策

### 为什么 ProgressRun 使用 partial class 拆分持久化和会话加载

`ProgressRun` 原本体量较大，拆分将持久化逻辑（两阶段写入、严格读取）和会话加载逻辑（拓扑编解码、后台会话创建）分离为独立文件，保持主文件聚焦核心编排流程。

### 为什么前台会话键固定为 `__foreground__`

前/后台会话共享同一接口 `ISessionRun`，差异仅在于注入的 `ISndSceneHost` 和键名。固定键名消除了"查找前台"的逻辑分支——直接从 SessionManager 中按常量键取值。

### 为什么运行时容器按层分离

每层容器（`SystemRuntime`、`ProgressRuntime` 等）仅暴露本层和下层的能力，上层无法访问下层的实现细节。例如策略只能通过 `ISessionRun` 操作会话，无法访问 `ProgressRun` 内部。

---
[↑ 回到 Runtime](../README.md)
