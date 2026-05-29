# 策略测试

> [↑ 回到 usage](README.md)

## 概述

`StrategyTestScenario` 是 Core 层提供的策略隔离测试框架。无需启动完整运行时，可在单元测试中孤立地测试单个策略的所有生命周期行为。

## 三阶段模式

### Phase 1: 配置

```csharp
var harness = StrategyTestScenario
    .For<HealthStrategy>("core.health")  // 策略类型 + 索引
    .WithEntityName("player")
    .WithData("hp", 100)
    .WithData("max_hp", 100)
    .WithSystemConfig("difficulty", "hard")
    .Build();  // 创建 Harness（自动触发 AfterSpawn）
```

配置方法：

| 方法 | 说明 |
|------|------|
| `WithEntityName(string)` | 设置实体名（默认 `__test_entity__`） |
| `WithData<TValue>(key, value)` | 向实体注入初始数据 |
| `WithSystemConfig<TValue>(key, value)` | 向系统黑板写入配置 |
| `WithProgressConfig<TValue>(key, value)` | 向流程黑板写入配置 |
| `WithSessionConfig<TValue>(key, value)` | 向会话黑板写入配置 |
| `WithTemplate(key, SndMetaData)` | 注册模板 |

### Phase 2: 模拟

```csharp
harness.RunFrame();          // 1 帧（Process + flush deferred actions）
harness.RunFrames(60);       // 60 帧
harness.TriggerAfterLoad();  // 手动触发生命周期钩子
harness.TriggerBeforeSave();
harness.TriggerBeforeQuit();
```

钩子触发方法：`TriggerAfterSpawn` / `TriggerAfterLoad` / `TriggerAfterAdd` / `TriggerBeforeRemove` / `TriggerBeforeSave` / `TriggerBeforeQuit` / `TriggerBeforeDead`

### Phase 3: 检查

```csharp
// 数据访问
int hp = harness.GetEntityData<int>("hp");
var (found, lives) = harness.TryGetEntityData<int>("lives");

// 黑板访问
harness.SystemBlackboard.TryGet("difficulty", out string diff);
harness.SessionBlackboard.TryGet("score", out int score);

// 副作用检查
harness.SaveRequests.Count;       // 请求保存次数
harness.LoadRequests.Count;       // 请求加载次数
harness.LevelSwitchRequests;      // 请求关卡切换列表
harness.DeferredActionCount;      // 延迟动作执行数
harness.ConsoleCommands;          // 控制台命令记录
harness.ConsoleOutput;            // 控制台输出记录
```

## 完整示例

### 伤害策略测试

```csharp
[Test]
public void DamageStrategy_ReducesHp_EachFrame()
{
    var harness = StrategyTestScenario
        .For<DamageTickStrategy>("core.damage_tick")
        .WithData("hp", 100f)
        .Build();

    harness.RunFrame();

    var hp = harness.GetEntityData<float>("hp");
    Assert.That(hp, Is.LessThan(100f));
}
```

### 存档请求测试

```csharp
[Test]
public void HealthStrategy_RequestsSave_WhenHpReachesZero()
{
    var harness = StrategyTestScenario
        .For<DeathCheckStrategy>("core.death_check")
        .WithData("hp", 5f)
        .Build();

    harness.RunFrames(10);

    Assert.That(harness.SaveRequests.Count, Is.GreaterThan(0));
}
```

### 延迟动作测试

```csharp
[Test]
public void SpawnStrategy_EnqueuesDeferredAction()
{
    var harness = StrategyTestScenario
        .For<SpawnEnemyStrategy>("core.spawn_enemy")
        .Build();

    harness.RunFrame();

    // 延迟动作在 RunFrame 中自动执行
    Assert.That(harness.DeferredActionCount, Is.GreaterThan(0));
}
```

## 限制

### 支持的能力

- 实体数据读写 (SetData / GetData / TryGetData)
- 三级黑板访问 (System / Progress / Session)
- 所有 8 个生命周期钩子
- Process 帧更新
- 延迟动作 (BusinessDeferred)
- 持久化请求记录 (Save/Load/LevelSwitch)
- 控制台命令输入输出
- 模板注册

### 不支持的能力

- 实体节点访问 (`GetNode`) — 测试实体不支持节点
- 后台会话创建 — 需要完整 `SndContext` 集成测试
- 多实体交互 — 每个 Harness 对应一个实体
- 引擎类型操作 — 如 `Vector2` 的位置计算（需要注册 Godot 转换器）

## 多策略优先级测试

需要使用 `SndStrategyPool` 直接注册多个策略，测试优先级排序和交互：

```csharp
var pool = new SndStrategyPool(logger);
pool.Register<StrategyA>(() => new StrategyA());
pool.Register<StrategyB>(() => new StrategyB());
// ... 使用 pool 测试策略顺序
```

## 相关文档

- [SND 实体模型](snd-entity-model.md) — 策略编写
- [Agent Reference](agent-reference.md) — 完整接口签名和测试模式

---
[↑ 回到 usage](README.md)
