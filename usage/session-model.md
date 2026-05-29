# 会话模型

> [↑ 回到 usage](README.md)

## 概述

Origo 支持**前台会话**和**后台会话**并存的模型，两种会话通过相同的 `ISessionRun` 接口表达，差异仅在于注入的 `ISndSceneHost` 和 `IsFrontSession` 标志。

## 前台 vs 后台

| 属性 | 前台会话 | 后台会话 |
|------|---------|---------|
| 键名 | `__foreground__`（固定） | 用户自定义（如 `"dungeon"`） |
| 数量 | 至多一个 | 可多个 |
| 场景宿主 | GodotSndManager（引擎渲染） | FullMemorySndSceneHost（无渲染） |
| Strategy 访问 | 前台 ISndContext | 隔离的 SessionSndContext |
| 状态机 | 会话级 StateMachineContainer | 会话级 StateMachineContainer |
| 黑板 | 独立 SessionBlackboard | 独立 SessionBlackboard |

## 接口

### ISessionRun

```csharp
public interface ISessionRun : IDisposable
{
    IBlackboard SessionBlackboard { get; }
    ISndSceneHost SceneHost { get; }       // 支持完整实体操作
    string LevelId { get; }
    bool IsFrontSession { get; }
    StateMachineContainer GetSessionStateMachines();
}
```

### ISessionManager

```csharp
public interface ISessionManager
{
    const string ForegroundKey = "__foreground__";
    ISessionRun? ForegroundSession { get; }
    IReadOnlyCollection<string> Keys { get; }
    ISessionRun? TryGet(string key);
    bool Contains(string key);
    ISessionRun CreateBackgroundSession(string key, string levelId, bool syncProcess = false);
    void DestroySession(string key);
    void ProcessAllSessions(double delta, bool includeForeground = false);
}
```

## 创建后台会话

```csharp
var bgSession = sessionManager.CreateBackgroundSession(
    key: "dungeon",
    levelId: "dungeon_level",
    syncProcess: true);  // 参与 Process 帧更新

// 后台会话拥有独立的实体、黑板和状态机
bgSession.SceneHost.Spawn(dungeonEntityMeta);
bgSession.SessionBlackboard.Set("explored", true);
```

## 会话拓扑

多个并发会话之间的关系通过 `SessionTopology` 编解码。编码格式（存储在 Progress 黑板中）：

```
foreground:{level_id};background:{key}={level_id},{key}={level_id}
```

例如：`foreground:town;background:dungeon=dungeon_level,farm=farm_level`

读档恢复时，`SessionTopologyCodec` 解析此字符串，重建所有后台会话。

## 状态机上下文

状态机策略钩子通过 `IStateMachineContext` 访问黑板：

```csharp
public interface IStateMachineContext
{
    IBlackboard SystemBlackboard { get; }     // 系统级
    IBlackboard? ProgressBlackboard { get; }  // 流程级
    IBlackboard? SessionBlackboard { get; }   // 当前会话级
    ISndSceneAccess SceneAccess { get; }      // 当前会话场景
    void EnqueueBusinessDeferred(Action action);
}
```

`SessionStateMachineContext` 是会话级适配器，确保前台/后台会话的状态机钩子拿到各自的 `SessionBlackboard` 和 `SceneAccess`，前后台无语义分差。

## 生命周期

```
创建:
  SessionManager.CreateBackgroundSession(key, levelId)
    → 创建 SessionRun
    → 注入 FullMemorySndSceneHost
    → 挂载到 _sessions 字典
    → 可选：从存档恢复 SessionBlackboard + 状态机 + 实体

运行:
  SessionManager.ProcessAllSessions(delta)
    → 遍历所有 syncProcess=true 的后台会话
    → 调用 session.SceneHost.ProcessAll(delta)

销毁:
  SessionManager.DestroySession(key)
    → Dispose SessionRun
    → 从字典移除

退出:
  ProgressRun dispose
    → 按序销毁所有后台会话
    → 最后销毁前台会话
```

## 设计原则

- 前后台共享同一接口，无类型分叉
- 管理层全权管理生命周期，会话层仅暴露内部状态
- 后台会话通过 `FullMemorySndSceneHost` 获得完整策略能力，但不渲染

## 相关文档

- [持久化流程](persistence-flow.md) — 会话数据的存档/恢复
- [状态机](state-machine.md) — 会话级状态机

---
[↑ 回到 usage](README.md)
