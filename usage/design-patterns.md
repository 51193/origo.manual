# 设计模式

> [↑ 回到 usage](README.md)

本文档收录使用 Origo 框架开发时的通用设计模式。这些模式从实际项目中提炼，适用于任何基于 SND 模型的游戏。

---

## 命名约定

### 策略索引

格式：`{域}.{模块}.{功能}`

```
camera.main.participant
map.generator
character.pathfind.astar
ui.main_menu
```

- 使用小写蛇形（`snake_case`）
- 域表示功能领域（`camera`、`map`、`character`、`ui`、`item`）
- 模块为可选的子分类
- 功能描述策略行为

### 数据键

格式：`{域}.{键名}`

```
character.hp
camera.zoom_level
map.grid_size
```

- 实体专属数据使用短键名（如 `hp`、`speed`）
- 跨实体共享配置带命名空间前缀（如 `camera.stack.pair`）
- 避免不同策略使用相同键名导致冲突

---

## 策略与节点交互

策略通过 `entity.GetNode("name")?.Native` 获取引擎节点。核心规则：

1. **永远判空**：后台会话中 `GetNode()` 返回 `NullNodeHandle`，`?.Native` 为 null
2. **节点操作封装到 helper 类**：策略不直接操作场景树（`AddChild`、`QueueFree` 等）
3. **通过节点名映射访问**：使用 Origo 模板定义的节点名，不遍历场景子树

```csharp
public override void AfterSpawn(ISndEntity entity, ISndContext ctx)
{
    var node = entity.GetNode("root")?.Native;
    if (node is Node3D node3d)
    {
        var pos = GridToWorld(entity.GetData<int>("grid_x"), entity.GetData<int>("grid_z"));
        node3d.GlobalPosition = pos;
    }
}

public override void AfterLoad(ISndEntity entity, ISndContext ctx)
{
    var node = entity.GetNode("root")?.Native;
    if (node is Node3D node3d)
    {
        var pos = GridToWorld(entity.GetData<int>("grid_x"), entity.GetData<int>("grid_z"));
        node3d.GlobalPosition = pos;
    }
}
```

`AfterSpawn` 和 `AfterLoad` 中都必须判空并执行相同的节点初始化逻辑，因为后台会话创建的实体在前台恢复时才首次获得真实节点。

---

## 自销毁初始化模式

一次性初始化策略在完成设置后自行移除，避免空闲开销：

```csharp
[StrategyIndex("game.entity.init")]
public class EntityInitStrategy : EntityStrategyBase
{
    public override void AfterSpawn(ISndEntity entity, ISndContext ctx)
    {
        // 展开原型属性
        LoadArchetypeAttributes(entity, ctx);

        // 动态添加运行时策略
        entity.AddStrategy("game.entity.perception");
        entity.AddStrategy("game.entity.scheduling");

        // 自销毁
        entity.RemoveStrategy("game.entity.init");
    }
}
```

适用场景：
- 实体首次创建时需要复杂的初始化逻辑（展开属性、动态构建策略组合）
- 初始化逻辑只需执行一次，不需要每帧 Process
- 模板中只声明此初始化策略，运行时策略由它动态构建

---

## Manager 实体 + ActiveStrategy 服务模式

用实体充当全局服务，通过 ActiveStrategy 暴露查询/变更接口：

```csharp
// Manager 策略：维护数据，无 Process 循环
[StrategyIndex("food.manager")]
public class FoodManagerStrategy : EntityStrategyBase
{
    public override void AfterSpawn(ISndEntity entity, ISndContext ctx)
    {
        entity.AddActiveStrategy("food.register");
        entity.AddActiveStrategy("food.unregister");
        entity.AddActiveStrategy("food.find_nearest");
    }
}

// ActiveStrategy：提供查询服务
[StrategyIndex("food.find_nearest")]
public class FoodFindNearestStrategy : ActiveStrategyBase
{
    public override object? Execute(ISndEntity entity, object? input, ISndContext ctx)
    {
        // 从 entity Data 中读取食物注册表，计算最近食物
        // 返回结果
    }
}
```

特征：
- Manager 实体作为纯数据容器 + 按需 ActiveStrategy，无 Process 循环
- 其他实体通过 `managerEntity.InvokeStrategy("food.find_nearest", position)` 调用服务
- Manager 状态参与持久化（存档/恢复后服务状态自动恢复）

---

## 可替换实现模式（`*_impl` 键）

使用 data 键存储策略索引，运行时切换行为实现：

```csharp
// 调度器读取 impl 键，动态选择策略
[StrategyIndex("game.scheduling")]
public class SchedulingStrategy : EntityStrategyBase
{
    public override void AfterSpawn(ISndEntity entity, ISndContext ctx)
    {
        // 设置默认实现
        entity.SetData("pathfind_impl", "game.pathfind.astar");
        entity.SetData("move_impl", "game.move.walk");
    }

    private static void EnsureImpl(ISndEntity entity, string implKey)
    {
        var (found, index) = entity.TryGetData<string>(implKey);
        if (found && !string.IsNullOrEmpty(index))
            entity.AddStrategy(index);
    }
}
```

适用场景：
- 同一功能有多种实现（如 A\* 寻路 vs 直线移动）
- 需要运行时切换行为（如地面移动 → 飞行移动）
- 避免在模板中固化实现选择

---

## 实体间通信

### InvokeStrategy：同步请求/响应

```csharp
var target = ctx.FindEntity("TraversabilityManager");
var path = target.InvokeStrategy<GridPos[], List<GridPos>>(
    "traversability.find_path", new[] { start, end });
```

### Subscribe：异步数据变更通知

```csharp
entity.Subscribe("intent", (target, observer, oldVal, newVal) =>
{
    if (newVal.TryGetString(out var intent) && intent == "idle")
        ResetSchedule(target);
});
```

### 规则

- **禁止持有实体直接引用**（实体可能随时被销毁）
- 通过 `ctx.FindEntity(name)` 按需查找
- 通过 `InvokeStrategy` 实现同步请求/响应
- 通过 `Subscribe` / `ObserveData` 实现异步通知

---

## UI/节点逻辑分离

策略保持纯净逻辑，将 Godot 节点操作委托给 `internal static` helper 类：

```csharp
// 策略：决策逻辑
[StrategyIndex("ui.main_menu")]
public class MainMenuStrategy : EntityStrategyBase
{
    public override void AfterLoad(ISndEntity entity, ISndContext ctx)
    {
        var container = entity.GetNode("root")?.Native;
        if (container is Control ctrl)
            MenuBuilder.BuildMainMenu(ctrl, ctx);
    }
}

// Helper：纯 UI 操作
internal static class MenuBuilder
{
    internal static void BuildMainMenu(Control container, ISndContext ctx)
    {
        // Godot 节点操作...
    }
}
```

好处：
- 策略逻辑可通过 `StrategyTestScenario` 独立测试
- Helper 类可跨策略复用
- 关注点分离：策略管「何时做」，helper 管「怎么做」

---

## 优先级分层执行

用不同 `Priority` 值划分策略的执行层次（数值越小越先执行）：

```
P4   感知层      读取环境/自身状态 → 产出意图
P5   调度层      根据意图 → 拆解行动计划
P6   行动层      执行具体行为 → 报告完成/失败
P20  寻路层      读取目标 → 计算路径
P30  移动层      读取下一步 → 执行位移
P35  检测层      逐帧检测条件 → 触发结果
```

设计要点：
- 同一帧内，低 Priority 先执行，产出的数据在同帧被高 Priority 策略消费
- 持续运行的子系统（寻路、移动）与决策系统（感知、调度）使用不同优先级段
- 不同层之间通过 data 键通信，无直接耦合

---

## 模板最佳实践

### 模板必须包含策略所需的全部 data 键

策略运行时通过 `GetData<T>("key")` 读取数据。如果模板未声明该键，运行时抛出异常。

```json
{
  "strategy": { "indices": ["game.camera.zoom"] },
  "data": {
    "pairs": {
      "camera.zoom_level": { "type": "Single", "data": 0.5 },
      "camera.height": { "type": "Single", "data": 20.0 }
    }
  }
}
```

### 两级模板设计

- **第一级：SND 模板** — 定义策略拓扑（挂载哪些策略、绑定哪些节点）
- **第二级：Archetype 原型文件** — 定义具体属性数值（`.map` 格式的扁平键值对）

SND 模板只包含 `archetype` 键指向原型文件名，策略在 `AfterSpawn` 中加载原型属性展开到 entity Data。

```json
{
  "strategy": { "indices": ["game.entity.init"] },
  "data": {
    "pairs": {
      "archetype": { "type": "String", "data": "warrior" }
    }
  }
}
```

好处：
- 同一模板、不同原型 = 不同属性的实体变体（无需每个变体独立模板）
- 原型文件使用 `.map` 格式（扁平键值对），编辑轻便

### 只声明初始策略

模板 `strategy.indices` 只写初始化策略（如 `game.entity.init`），运行时策略由初始化策略动态添加。避免模板固化实现选择、每次新增策略都要改模板。

### 后台会话节点为空

后台会话中创建的实体没有引擎节点（`GetNode()` 返回 `NullNodeHandle`）。策略的 `AfterSpawn` 和 `AfterLoad` 中访问节点必须判空：

```csharp
var node = entity.GetNode("root")?.Native;
if (node is Node3D n)
{
    // 有节点时的操作（前台会话）
}
// 无节点时静默跳过（后台会话）
```

---

## `type` 字段使用 Origo 注册名

模板 JSON 中 `data.pairs` 的 `"type"` 字段必须使用 Origo 的注册短名：

| 实际类型 | 正确写法 | 错误写法 |
|----------|----------|----------|
| `int` | `Int32` | `int`, `Integer` |
| `float` | `Single` | `Float32`, `float` |
| `string` | `String` | `string` |
| `bool` | `Boolean` | `bool`, `Bool` |
| `double` | `Double` | `double` |
| `long` | `Int64` | `long`, `Long` |

适配层注册类型（如 Godot 的 `Vector3`、`Transform3D`）使用其 .NET 类型短名。

---

## 下一个文档

- [策略生命周期](strategy-lifecycle.md) — 闭环配对与资源管理
- [SND 实体模型](snd-entity-model.md) — 模型基础

---
[↑ 回到 usage](README.md)
