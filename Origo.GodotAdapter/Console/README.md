# Console

> [↑ 回到 Origo.GodotAdapter](../README.md) · [↔ Core: Runtime/Console](../../Origo.Core/Runtime/Console/README.md)

## 概述

Godot 适配层的控制台命令扩展。一个适配层的命令处理器基类（提供 `OrigoRuntime` 引用），以及一个 Godot 特有的 `press_button` 命令——模拟按下 Godot Node 中的 Button 节点。

## 包含文件

| 文件 | 职责 |
|------|------|
| `CommandHandlerBase.cs` | 适配层命令处理器基类，持有 `OrigoRuntime` 引用，校验参数数量 |
| `PressButtonCommandHandler.cs` | Godot 特有命令：按 entity + path 查找并发射 Button.Pressed 信号 |

## 模块详解

### CommandHandlerBase

继承自 Core 的 `IConsoleCommandHandler`，提供与 Core `ConsoleCommandHandlerBase` 相同的参数校验逻辑。区别在于此基类直接持有 `OrigoRuntime` 引用（而非子类注入），简化 Godot 侧的命令实现。

### PressButtonCommandHandler

```
press_button <entity> <path>
```

流程：
1. `Runtime.Snd.FindByName(entity)` 找到实体
2. 检查实体是否为 `GodotSndEntity` 类型
3. 通过 `godotEntity.GetNodeOrNull<Button>(path)` 查找 Button 节点
4. `button.EmitSignal(BaseButton.SignalName.Pressed)` 模拟按下

## 设计决策

### 为什么适配层还需要自己的 CommandHandlerBase

Core 的 `ConsoleCommandHandlerBase` 要求子类持有对 `OrigoRuntime` 的引用。适配层基类提供一致的 `Runtime` 属性访问方式，避免每个 Godot 命令处理器重复相同的注入模式。

### 为什么 PressButton 需要 Godot 实体类型检查

`Runtime.Snd.FindByName` 返回 `ISndEntity` 抽象接口，但 `GetNodeOrNull<Button>` 是 Godot.Node 的方法。运行时检查确保类型安全——如果实体是纯内存实体（如 `MemorySndEntity`），提前用清晰错误信息告知而非 NullReferenceException。

---
[↑ 回到 Origo.GodotAdapter](../README.md)
