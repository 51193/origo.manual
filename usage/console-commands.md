# 控制台命令

> [↑ 回到 usage](README.md)

## 概述

Origo 内置的控制台命令系统。支持通过 Godot 控制台或 TCP 桥接（`Origo.ConsoleBridge`，端口 9876）执行命令。命令支持位置参数和命名参数。

## 可用命令

| 命令 | 参数 | 说明 |
|------|------|------|
| `help` | 无 | 列出所有可用命令及帮助信息 |
| `bb_get` | `<layer> <key>` | 读取黑板键值。`layer`: system |
| `bb_set` | `<layer> <key> <value>` | 向黑板写入值（自动推断类型） |
| `bb_keys` | `<layer>` | 列出黑板层的全部键 |
| `spawn` | `<name> <template>` | 按模板生成实体 |
| `spawn` | `name=<n> template=<t>` | 按模板生成实体（命名参数） |
| `find_entity` | `<name>` | 查找实体并显示节点信息 |
| `request_clear_entities` | 无 | 请求清理所有已生成实体（帧末延迟执行） |
| `snd_count` | 无 | 显示当前实体数量 |
| `press_button` | `<entity> <path>` | 模拟按下 Godot Node 中的 Button（适配层命令） |
| `entity_get_data` | `<entity> <key>` | 读取实体数据的值及类型 |
| `entity_set_data` | `<entity> <key> <value>` | 设置实体数据（自动推断类型，保留已有键的类型） |

## 命令详细

### bb_get

```
> bb_get system core.player.health
[system] core.player.health = 100 (type: Int32)
```

当前仅支持 `system` 层。

### bb_set

```
> bb_set system debug_mode true
[system] debug_mode = true
```

类型推断规则：整数 → Int32、浮点数 → Single、true/false → Boolean、其余 → String。

### spawn

```
> spawn player template_basic
Spawned 'player' from template 'template_basic'.

> spawn name=enemy template=template_enemy
Spawned 'enemy' from template 'template_enemy'.
```

不支持位置参数和命名参数混用。

### find_entity

```
> find_entity player
Entity 'player' found. Nodes: [sprite, collider, shadow]
```

### request_clear_entities

```
> request_clear_entities
Requested clear of 12 entities (deferred to end of frame).
```

清空操作使用 `EnqueueBusinessDeferred` 延迟至帧末执行，与其他生命周期操作（`RequestSaveGame`、`RequestSwitchForegroundLevel`）保持一致。不可在 Godot signal handler 中同步调用底层 `ClearAll()`，应使用本命令或 `ISndContext.RequestClearEntities()`。

### press_button（适配层）

```
> press_button main_menu_ui StartButton
Pressed button 'StartButton' on entity 'main_menu_ui'.
```

通过 `GodotSndEntity.GetNodeOrNull<Button>(path)` 查找 Button 节点并发射 `Pressed` 信号。

### entity_get_data

```
> entity_get_data player health
health = 100 (type: Int32)
```

通过 `TryGetData<object>` 读取任意类型的实体数据，显示值和运行时类型。键不存在时返回 `Key '...' not found`。

### entity_set_data

```
> entity_set_data player health 50
[player] health = 50
```

类型推断规则同 `bb_set`（int/float/bool/string）。若键已存在，则**保留已有键的类型**再写入（如已有 `float` 类型的 `hunger` 键，执行 `entity_set_data player hunger 15` 会写为 `Single(15)` 而非 `Int32(15)`）。

## 添加自定义命令

### Core 层命令

继承 `ConsoleCommandHandlerBase`：

```csharp
[StrategyIndex("my_game.my_command")]
internal sealed class MyCommandHandler : ConsoleCommandHandlerBase
{
    private readonly OrigoRuntime _runtime;
    public MyCommandHandler(OrigoRuntime runtime) { _runtime = runtime; }

    public override string Name => "my_command";
    public override string HelpText => "my_command <arg> — 说明";
    public override int MinPositionalArgs => 1;
    public override int MaxPositionalArgs => 1;

    protected override bool ExecuteCore(
        CommandInvocation invocation,
        IConsoleOutputChannel output,
        out string? error)
    {
        var arg = invocation.PositionalArgs[0];
        output.Publish($"Executed with arg: {arg}");
        error = null;
        return true;
    }
}
```

注册：`runtime.Console.RegisterHandler(new MyCommandHandler(runtime));`

### 适配层命令

继承 `Origo.GodotAdapter.Console.CommandHandlerBase`：

```csharp
internal sealed class MyGodotCommand : CommandHandlerBase
{
    public MyGodotCommand(OrigoRuntime runtime) : base(runtime) { }
    public override string Name => "my_godot_cmd";
    public override string HelpText => "my_godot_cmd — does something";
    public override int MinPositionalArgs => 0;
    public override int MaxPositionalArgs => 0;

    protected override bool ExecuteCore(
        CommandInvocation invocation,
        IConsoleOutputChannel output,
        out string? error)
    {
        // 访问 Godot 特有 API
        output.Publish("Done.");
        error = null;
        return true;
    }
}
```

## TCP 远程控制台

```
# 启动桥接服务器
var server = new ConsoleBridgeServer(consoleInput, consoleOutput);
server.Start();  // 监听 localhost:9876

# 客户端连接
nc localhost 9876
> snd_count
Snd count: 42.
```

单连接模式：同时只允许一个 TCP 客户端连接。控制台输出通过 publish-subscribe 推送到所有监听者（包括桥接客户端）。

## 相关文档

- [快速开始](quick-start.md) — 如何接入 Origo
- [Origo.ConsoleBridge](../Origo.ConsoleBridge/README.md) — 桥接实现细节

---
[↑ 回到 usage](README.md)
