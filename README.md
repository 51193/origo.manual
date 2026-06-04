# Origo Manual

Origo 框架的完整文档手册。采用**自底向上**的结构——从源代码目录逐级向上汇总，确保任何问题都能通过目录的多级索引找到目标位置，无需从源代码从头读起。

## 设计原则

Origo 框架遵循以下核心设计约束，所有模块实现和接口设计均以此为准：

| 原则 | 说明 |
|------|------|
| **平台无关** | Origo.Core 零引擎依赖，所有游戏逻辑、持久化、实体模型仅使用 `System.*` 类型 |
| **适配层隔离** | 引擎集成仅通过 `Origo.GodotAdapter` 实现 Core 抽象接口，适配层不得触发策略钩子、管理策略生命周期、冲刷延迟管线、持有 Core 编排状态 |
| **接口隔离（ISP）** | `ISndContext` 拆分为 9 个窄角色接口，`ISessionRun` 返回抽象 `IStateMachineContainer` 而非具体类型 |
| **依赖方向单向** | Abstractions → Core 实现 → Adapter，反向依赖严格禁止 |
| **public 白名单** | 不为"可能未来有用"提前公开接口；每个 public 接口必须有明确的跨程序集消费者 |
| **显式失败优先** | 接口契约被违反时抛异常而非静默降级；存档/读档严格校验完整性 |
| **策略一等公民** | 游戏由策略驱动，`ISndContext` 作为上帝对象向策略暴露全部能力，不限制策略可访问的框架功能 |
| **单线程帧模型** | 一帧 = 一个逻辑原子边界；延迟动作通过队列顺序执行 |

## 如何使用本手册

```
Root (this file)
  ├── 我需要了解"如何使用 Origo"
  │   └── usage/README.md → 按场景选择文档
  │
  ├── 我需要了解"某个模块的能力和设计决策"
  │   ├── Origo.Core/README.md → 子系统一览 → 进入具体子模块
  │   │   └── Snd/README.md → Entity/README.md → ...
  │   ├── Origo.GodotAdapter/README.md → 适配层子模块
  │   └── Origo.ConsoleBridge/README.md → TCP 桥接
  │
  ├── 我需要了解"测试覆盖了什么能力"
  │   ├── Origo.Core.Tests/README.md → 按 22 个能力查看 Core 测试
  │   ├── Origo.GodotAdapter.Tests/README.md → 适配层 6 个能力测试
  │   └── Origo.ConsoleBridge.Tests/README.md → TCP 桥接测试
  │
  └── 我需要了解"这个手册本身怎么维护"
      └── META.md
```

每个目录下的 `README.md` 包含：
- **子模块链接**（向下导航）
- **父模块链接**（向上导航，`↑` 标记）
- **相关模块链接**（横向关联，`↔` 标记）

## 项目模块索引

| 模块 | 位置 | 说明 |
|------|------|------|
| **Origo.Core** | [README](Origo.Core/README.md) | 平台无关核心：SND 实体系统、运行时、持久化、状态机 |
| **Origo.GodotAdapter** | [README](Origo.GodotAdapter/README.md) | Godot 4 适配层：文件系统、日志、序列化、启动 |
| **Origo.ConsoleBridge** | [README](Origo.ConsoleBridge/README.md) | TCP 远程控制台桥接（端口 9876） |
| **使用文档** | [README](usage/README.md) | 从快速入门到深度参考的使用指南 |
| **测试: Core** | [README](Origo.Core.Tests/README.md) | Core 层 22 个能力的行为测试文档 |
| **测试: GodotAdapter** | [README](Origo.GodotAdapter.Tests/README.md) | 适配层 6 个能力的行为测试文档 |
| **测试: ConsoleBridge** | [README](Origo.ConsoleBridge.Tests/README.md) | TCP 桥接服务器行为测试文档 |
| **手册元指令** | [META.md](META.md) | 本手册的编写与维护规范 |

## Origo.Core 子系统

| 子系统 | 职责 |
|--------|------|
| [Abstractions](Origo.Core/Abstractions/README.md) | 10 组公共接口（IBlackboard、IFileSystem、ISndEntity、ISessionManager、IStateMachineContainer...） |
| [Snd](Origo.Core/Snd/README.md) | SND 实体系统（Strategy + Node + Data） |
| [Runtime](Origo.Core/Runtime/README.md) | 四层运行时生命周期 + 控制台 |
| [Save](Origo.Core/Save/README.md) | 持久化（两阶段写入 + 严格读取） |
| [DataSource](Origo.Core/DataSource/README.md) | 数据源抽象层（JSON/Map 编解码 + 类型转换） |
| [StateMachine](Origo.Core/StateMachine/README.md) | 字符串栈状态机 |
| [Scheduling](Origo.Core/Scheduling/README.md) | 延迟动作调度 |
| [Blackboard](Origo.Core/Blackboard/README.md) | 内存黑板实现 |
| [Random](Origo.Core/Random/README.md) | 随机数 + 噪声图 |
| [Serialization](Origo.Core/Serialization/README.md) | 类型 ↔ 字符串映射 |
| [Logging](Origo.Core/Logging/README.md) | 日志构建器 + NullLogger |
| [Addons](Origo.Core/Addons/README.md) | FastNoiseLite 噪声库 |
| [Testing](Origo.Core/Testing/README.md) | StrategyTestScenario 测试框架 |

## 快速导航

| 我想... | 去这里 |
|---------|--------|
| 快速接入 Origo | [usage/quick-start](usage/quick-start.md) |
| 理解整体架构 | [usage/architecture-overview](usage/architecture-overview.md) |
| 编写游戏策略 | [usage/snd-entity-model](usage/snd-entity-model.md) |
| 测试策略 | [usage/strategy-testing](usage/strategy-testing.md) |
| 使用存档系统 | [usage/persistence-flow](usage/persistence-flow.md) |
| 使用状态机 | [usage/state-machine](usage/state-machine.md) |
| 使用控制台命令 | [usage/console-commands](usage/console-commands.md) |
| 查看接口签名 | [usage/agent-reference](usage/agent-reference.md) |
| 理解 Core 模块实现 | [Origo.Core/](Origo.Core/README.md) |
| 理解 Godot 适配 | [Origo.GodotAdapter/](Origo.GodotAdapter/README.md) |

## 版本

本文档与 Origo 代码库同步。代码目录结构变更时，手册应同步更新目录镜像和索引。

手册维护规则详见 [META.md](META.md)。
