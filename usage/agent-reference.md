# Agent Reference

> [↑ 回到 usage](README.md)

## 概述

面向 AI Agent 开发者的完整运行时参考。包含核心接口签名、生命周期时间线、策略编写模板和测试模式。

## 核心接口

### ISndEntity

```csharp
public interface ISndEntity : ISndDataAccess, ISndNodeAccess, ISndStrategyAccess,
    ISndActiveStrategyAccess, ISndEntityLifecycleAccess, ISndObservation
{
    string Name { get; }
    bool IsPendingKill { get; }
}

public interface ISndDataAccess
{
    void SetData<T>(string name, T value);
    T GetData<T>(string name);
    (bool found, T? value) TryGetData<T>(string name);
    void Subscribe(string name, Action<ISndEntity, ISndEntity, TypedData, TypedData> callback,
        Func<ISndEntity, ISndEntity, TypedData, TypedData, bool>? filter = null);
    void Unsubscribe(string name, Action<ISndEntity, ISndEntity, TypedData, TypedData> callback);
}

public interface ISndEntityLifecycleAccess
{
    void SubscribeLifecycle(Action<ISndEntity, ISndEntity, EntityLifecycleEvent> callback);
    void UnsubscribeLifecycle(Action<ISndEntity, ISndEntity, EntityLifecycleEvent> callback);
}

public interface ISndObservation
{
    void ObserveData(ISndEntity target, string dataName,
        Action<ISndEntity, ISndEntity, TypedData, TypedData> callback,
        Func<ISndEntity, ISndEntity, TypedData, TypedData, bool>? filter = null);
    void UnobserveData(ISndEntity target, string dataName,
        Action<ISndEntity, ISndEntity, TypedData, TypedData> callback);
    void ObserveLifecycle(ISndEntity target,
        Action<ISndEntity, ISndEntity, EntityLifecycleEvent> callback);
    void UnobserveLifecycle(ISndEntity target,
        Action<ISndEntity, ISndEntity, EntityLifecycleEvent> callback);
}

public interface ISndNodeAccess
{
    INodeHandle GetNode(string name);
    IReadOnlyCollection<string> GetNodeNames();
}

public interface ISndStrategyAccess
{
    void AddStrategy(string index);
    void RemoveStrategy(string index);
}

public interface ISndActiveStrategyAccess
{
    void AddActiveStrategy(string index);
    void RemoveActiveStrategy(string index);
    object? InvokeStrategy(string strategyIndex, object? input = null);
}

public enum EntityLifecycleEvent
{
    AfterSpawn, AfterLoad, BeforeSave, BeforeQuit, BeforeDead
}
```

### IBlackboard

```csharp
public interface IBlackboard
{
    void Set<T>(string key, T value);
    (bool found, T value) TryGet<T>(string key);
    void Clear();
    IReadOnlyCollection<string> GetKeys();
    IReadOnlyDictionary<string, TypedData> SerializeAll();
    void DeserializeAll(IReadOnlyDictionary<string, TypedData> data);
}
```

### ISndContext

ISndContext 是策略钩子接收的统一门面接口，组合了 9 个角色接口。命名空间 `Origo.Core.Snd`。

```csharp
public interface ISndContext : ISndBlackboardAccess, ISndSessionAccess, ISndDeferredActions,
    ISndTemplateAccess, ISndConsoleAccess, ISndStateMachineAccess, ISndSaveOperations,
    ISndLifecycleOperations, ISndEntityOperations
{
}

// === 角色接口概览 ===

// 黑板访问
public interface ISndBlackboardAccess {
    IBlackboard SystemBlackboard { get; }
    IBlackboard? ProgressBlackboard { get; }
}

// 会话管理
public interface ISndSessionAccess {
    ISessionManager SessionManager { get; }
    ISessionRun? CurrentSession { get; }
    bool IsFrontSession { get; }
}

// 延迟动作
public interface ISndDeferredActions {
    void EnqueueBusinessDeferred(Action action);
    void FlushDeferredActionsForCurrentFrame();
    int GetPendingPersistenceRequestCount();
}

// 模板
public interface ISndTemplateAccess {
    SndMetaData CloneTemplate(string templateKey, string? overrideName = null);
}

// 控制台
public interface ISndConsoleAccess {
    bool TrySubmitConsoleCommand(string commandLine);
    void ProcessConsolePending();
    long SubscribeConsoleOutput(Action<string> onLine);
    void UnsubscribeConsoleOutput(long subscriptionId);
}

// 状态机
public interface ISndStateMachineAccess {
    StateMachineContainer? GetProgressStateMachines();
}

// 存档操作
public interface ISndSaveOperations {
    IReadOnlyList<string> ListSaves();
    void RequestLoadGame(string saveId);
    void RequestSaveGame(string newSaveId);
    string RequestSaveGameAuto(string? newSaveId = null);
    void SetContinueTarget(string saveId);
    void RequestSwitchForegroundLevel(string newLevelId);
}

// 生命周期入口
public interface ISndLifecycleOperations {
    bool HasContinueData();
    bool RequestContinueGame();
    void RequestLoadInitialSave();
    void RequestLoadMainMenuEntrySave();
}

// 实体操作
public interface ISndEntityOperations {
    void RequestKillAll();
    void RequestKillEntity(string entityName);
}
```

### ISessionManager / ISessionRun

命名空间 `Origo.Core.Abstractions.Lifecycle`。

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

```csharp
public interface ISessionRun : IDisposable
{
    IBlackboard SessionBlackboard { get; }
    ISndSceneHost SceneHost { get; }
    string LevelId { get; }
    bool IsFrontSession { get; }
    IStateMachineContainer GetSessionStateMachines();
}
```

### IStateMachine / IStateMachineContext / IStateMachineContainer

```csharp
public interface IStateMachine
{
    string MachineKey { get; }
    string PushStrategyIndex { get; }
    string PopStrategyIndex { get; }
    void Push(string value);
    bool TryPopRuntime(out string? popped);
    bool TryPopOnQuit(out string? popped);
    (bool found, string? top) Peek();
    IReadOnlyList<string> Snapshot();
    void FlushAfterLoad();
    void RestoreStackWithoutHooks(IReadOnlyList<string> stackBottomToTop);
}
```

```csharp
public interface IStateMachineContext
{
    IBlackboard SystemBlackboard { get; }
    IBlackboard? ProgressBlackboard { get; }
    IBlackboard? SessionBlackboard { get; }
    ISndSceneAccess SceneAccess { get; }
    void EnqueueBusinessDeferred(Action action);
}
```

```csharp
public interface IStateMachineContainer
{
    IStateMachine CreateOrGet(string machineKey, string pushStrategyIndex, string popStrategyIndex);
    bool TryGet(string machineKey, out IStateMachine? machine);
    void Remove(string machineKey);
    void Clear();
}
```

## 初始化时间线

```
OrigoAutoHost._Ready()
│
├── 1. 创建 IFileSystem (GodotFileSystem)
├── 2. 创建 ILogger (GodotLogger)
├── 3. 创建 GodotSndManager
├── 4. 注册 TypeStringMapping + Converters (BCL + Godot types)
├── 5. 创建 PersistentBlackboard → LoadFromDisk
├── 6. 创建 ConsoleInputQueue + ConsoleOutputChannel
├── 7. 创建 OrigoRuntime
│   ├── SndWorld (策略池 + 转换器注册表)
│   ├── SystemRun (持有 SystemBlackboard)
│   └── OrigoConsole (命令路由)
│
├── 8. BindRuntimeDependencies (World + Logger to SndManager)
│
└── OrigoDefaultEntry._Ready() [覆写]
    ├── 9. 注册适配层命令处理器
    ├── 10. OrigoAutoInitializer.DiscoverAndRegisterStrategies (反射扫描)
    ├── 11. SndContext 创建 (注入 Runtime + FileSystem + saveRoot + config)
    ├── 12. SndManager.BindContext(context)
    ├── 13. LoadSceneAliases + LoadTemplates
    └── 14. RequestLoadMainMenuEntrySave → FlushDeferredActions
```

## 完整策略示例

```csharp
using Origo.Core.Abstractions.Entity;
using Origo.Core.Snd.Metadata;
using Origo.Core.Snd.Strategy;

[StrategyIndex("example.simple_health", Priority = 6205)]
public class SimpleHealthStrategy : EntityStrategyBase
{
    public override void AfterSpawn(ISndEntity entity, ISndContext ctx)
    {
        entity.SetData("hp", 100);
        entity.SetData("max_hp", 100);

        entity.Subscribe("hp", OnHpChanged,
            filter: (t, obs, old, @new) => @new.TryGetInt32(out var n) && n <= 0);
    }

    public override void Process(ISndEntity entity, double delta, ISndContext ctx)
    {
        var (found, hp) = entity.TryGetData<int>("hp");
        if (!found) return;

        entity.SetData("hp", hp - 1);
    }

    private static void OnHpChanged(ISndEntity target, ISndEntity observer,
        TypedData oldValue, TypedData newValue)
    {
    }
}
```

## 跨实体观察示例

```csharp
[StrategyIndex("enemy.watcher")]
public sealed class EnemyWatcherStrategy : EntityStrategyBase
{
    private static void OnBossHpChanged(ISndEntity target, ISndEntity observer,
        TypedData oldVal, TypedData newVal)
    {
        observer.SetData("boss_hp", newVal.AsInt32());
    }

    private static void OnBossLifecycle(ISndEntity target, ISndEntity observer,
        EntityLifecycleEvent evt)
    {
        if (evt == EntityLifecycleEvent.BeforeDead)
            observer.SetData("boss_alive", false);
    }

    public override void AfterLoad(ISndEntity entity, ISndContext ctx)
    {
        var boss = ctx.CurrentSession?.SceneHost.FindByName("boss");
        if (boss is null) return;

        entity.ObserveData(boss, "hp", OnBossHpChanged);
        entity.ObserveLifecycle(boss, OnBossLifecycle);
    }

    // 提前取消观察（如不再需要）。框架在 entity Teardown 时自动清理传出订阅，通常无需手动调用。
    public override void BeforeRemove(ISndEntity entity, ISndContext ctx)
    {
        var boss = ctx.CurrentSession?.SceneHost.FindByName("boss");
        if (boss is null) return;

        entity.UnobserveData(boss, "hp", OnBossHpChanged);
        entity.UnobserveLifecycle(boss, OnBossLifecycle);
    }
}
```

## 测试模式

```csharp
// 完整策略测试模板
[Test]
public void MyStrategy_AfterSpawn_InitializesData()
{
    var harness = StrategyTestScenario
        .For<MyStrategy>("test.strategy")
        .WithEntityName("test_entity")
        .WithData("hp", 100)
        .Build();

    var hp = harness.GetEntityData<int>("hp");
    Assert.That(hp, Is.EqualTo(100));
}

[Test]
public void MyStrategy_Process_UpdatesState()
{
    var harness = StrategyTestScenario
        .For<MyStrategy>("test.strategy")
        .WithData("counter", 0)
        .Build();

    harness.RunFrame();

    var counter = harness.GetEntityData<int>("counter");
    Assert.That(counter, Is.GreaterThan(0));
}

// 架构守卫测试
[Test]
public void Core_ContainsNoGodotReferences()
{
    var coreAssembly = typeof(SndEntity).Assembly;
    var adapterAssembly = typeof(GodotSndManager).Assembly;

    foreach (var type in coreAssembly.GetTypes())
    {
        Assert.That(type.Namespace, Does.Not.Contain("Godot"));
    }
}
```

## 常见陷阱

1. **策略中不存实例状态** — 所有可变数据存入实体 Data
2. **TryGetData 先判断 found** — 值类型 default(T) 不可靠
3. **AfterLoad 是恢复钩子** — 不是 AfterSpawn 的替代
4. **BeforeSave 中修改数据会被写入存档** — 这是设计意图，用于刷新
5. **延迟动作在 Process 后自动执行** — RunFrame 中已经处理
6. **后台会话无渲染节点** — 不做依赖节点的操作
7. **Subscribe / Unsubscribe 须传相同委托实例** — 方法引用（method group）保证一致性。lambda 每次编译产生不同实例，会导致退订失败
8. **ObserveData / ObserveLifecycle 也须传相同委托实例** — 同理。策略中建议定义为 `static` 方法并用方法引用
9. **观察方 Teardown 自动清理传出订阅** — 策略通常无需手动调用 Unobserve，框架在实体死亡时统一处理

## 相关文档

- [SND 实体模型](snd-entity-model.md) — 策略编写详细指南
- [策略测试](strategy-testing.md) — StrategyTestScenario 详细用法

---
[↑ 回到 usage](README.md)
