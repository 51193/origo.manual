# Console

> [↑ 回到 Runtime](../README.md)

## 模块能力

Origo 的运行时控制台命令系统。提供命令解析（位置参数 + 命名参数）、命令路由（命令名 → 处理器）以及输入输出通道（队列 + 发布-订阅）。

## 子模块

| 子模块 | 能力 | 详情 |
|--------|------|------|
| [CommandHandlers](CommandHandlers/README.md) | 10 个内置命令处理器 | help / bb_get / bb_set / bb_keys / spawn / find_entity / clear_entities / snd_count / entity_get_data / entity_set_data |

## 本层核心文件

| 文件 | 职责 |
|------|------|
| `OrigoConsole.cs` | 控制台门面：持有 Router + Input + OutputChannel + Parser |
| `ConsoleCommandRouter.cs` | 命令路由：命令名 → IConsoleCommandHandler 注册与查找 |
| `ConsoleCommandParser.cs` | 命令解析：字符串 → CommandInvocation（位置参数 + 命名参数）|
| `ConsoleCommandHandlerBase.cs` | 命令处理器基类：Name/HelpText/参数范围校验/执行 |
| `CommandInvocation.cs` | 命令调用模型：CommandName + PositionalArgs + NamedArgs |
| `IConsoleCommandHandler.cs` | 命令处理器接口：Name + HelpText + TryExecute |
| `ConsoleInputQueue.cs` | 线程安全的命令输入队列：Enqueue/TryDequeue |
| `ConsoleOutputChannel.cs` | IConsoleOutputChannel 实现：订阅/发布 |

## 命令生命周期

```
外部输入 (Godot 控制台 / TCP 桥接)
    │
    ▼
ConsoleInputQueue.Enqueue(line)
    │
    ▼
OrigoConsole.ProcessPending()
    ├── TryDequeueCommand → line
    ├── ConsoleCommandParser.Parse(line)
    │   └── CommandInvocation { Name, PositionalArgs, NamedArgs }
    ├── ConsoleCommandRouter.TryExecute(invocation, outputChannel)
    │   └── handler.TryExecute(invocation, outputChannel)
    └── outputChannel.Publish(result)
```

## 设计原则

- **命名参数支持**：除位置参数外，支持 `key=value` 命名参数（如 `spawn name=x template=y`）。两种模式不可混用
- **参数校验前置**：`ConsoleCommandHandlerBase.TryExecute` 在执行前校验参数数量，失败时返回清晰错误
- **线程安全输入**：`ConsoleInputQueue` 使用 `lock` 保护，支持 TCP 桥接线程并发入队

---
[↑ 回到 Runtime](../README.md)
