# Origo.GodotAdapter

> [↑ 回到 Origo.manual](../../README.md)

## 模块概述

**Origo.GodotAdapter** 是 Origo 框架的 Godot 4 适配层。负责将 Core 层的平台无关抽象与 Godot 引擎的具体 API 对接，包括文件系统（通过 `FileAccess`/`DirAccess`）、日志输出（通过 `GD.Print`）、节点生命周期（通过 `Node`/`PackedScene`）以及引擎类型序列化（`Vector2`、`Transform3D` 等 14 种类型）。

## 子系统一览

| 子系统 | 能力 | 详情 |
|--------|------|------|
| [Bootstrap](Bootstrap/README.md) | 启动编排 | OrigoAutoHost → OrigoDefaultEntry → Runtime 创建 + 策略发现 + Context 绑定 |
| [Console](Console/README.md) | Godot 控制台命令 | PressButton 命令 + 适配层 CommandHandlerBase |
| [FileSystem](FileSystem/README.md) | Godot 文件系统 | IFileSystem 实现：FileAccess/DirAccess + res:// 和 user:// 支持 |
| [Logging](Logging/README.md) | Godot 日志 | ILogger 实现：委托注入 GD.Print/PushWarning/PushError |
| [Serialization](Serialization/README.md) | Godot 类型序列化 | 14 种上帝类型 → DataSourceNode 转换器 |
| [Snd](Snd/README.md) | Godot SND 实体 | ISndSceneHost 实现：GodotSndManager + GodotSndEntity + PackedSceneNodeFactory |

## 启动流程

```
OrigoDefaultEntry._Ready()
  ├── base._Ready()                          // OrigoAutoHost
  │   └── CreateRuntime()
  │       ├── GodotFileSystem
  │       ├── GodotSndManager
  │       ├── GodotJsonConverterRegistry 注册
  │       └── OrigoRuntime
  ├── 自动发现策略 (reflection scan, skip Godot assemblies)
  ├── SndContext 创建 (注入 Runtime + FileSystem + saveRoot + config)
  ├── SndManager.BindContext(sndContext)
  ├── LoadSceneAliases / LoadTemplates
  └── RequestLoadMainMenuEntrySave
```

## 架构约束

- **不承载核心业务规则**：所有业务逻辑在 Core 层，适配层仅做"翻译"
- **不反向依赖**：Core 绝不引用 GodotAdapter 的任何类型
- **Godot 类型仅在适配层出现**：`Godot.Vector*`、`Godot.Node` 等不会出现在 Core 层

## 与 Core 的桥接

| Core 接口 | Adapter 实现 | 文件 |
|-----------|------------|------|
| `IFileSystem` | `GodotFileSystem` | [FileSystem/](FileSystem/README.md) |
| `ILogger` | `GodotLogger` | [Logging/](Logging/README.md) |
| `ISndSceneHost` | `GodotSndManager` | [Snd/](Snd/README.md) |
| `INodeFactory` | `GodotPackedSceneNodeFactory` | [Snd/](Snd/README.md) |
| `INodeHandle` | `GodotNodeHandle` | [Snd/](Snd/README.md) |
| `IConsoleCommandHandler` | `CommandHandlerBase` + 子类 | [Console/](Console/README.md) |

---
[↑ 回到 Origo.manual](../../README.md)
