# Snd

> [↑ 回到 Origo.Core](../README.md)

## 模块能力

SND（Strategy + Node + Data）实体系统的完整实现。这是 Origo 的核心业务模型——所有游戏实体通过策略表达行为逻辑，通过数据存储可变状态，通过节点映射引擎表现层。

## 子模块

| 子模块 | 能力 | 详情 |
|--------|------|------|
| [Entity](Entity/README.md) | 运行时实体聚合根 | SndEntity + 数据/节点/策略三个内部管理器 |
| [Metadata](Metadata/README.md) | 实体元数据模型 | TypedData / SndMetaData / NodeMetaData / StrategyMetaData / DataMetaData |
| [Scene](Scene/README.md) | 场景宿主与运行时门面 | SndRuntime + FullMemorySndSceneHost + MemorySndSceneHost |
| [Strategy](Strategy/README.md) | 策略系统核心 | BaseStrategy → EntityStrategyBase \| ActiveStrategyBase。策略池、双管理器（被动/主动） |

## 本层核心文件

| 文件 | 职责 |
|------|------|
| `ISndContext.cs` | SND 上下文组合接口：继承 8 个角色接口（[详见 Abstractions/Snd](../Abstractions/Snd/README.md)） |
| `StrategyMetaData.cs` | 策略元数据：`EntityIndices`（被动）和 `ActiveIndices`（主动）分离存储 |
| `SndContext.cs` | 默认 ISndContext 实现（全局/流程级）|
| `SndContextParameters.cs` | SndContext 构造参数对象 |
| `NullSndContext.cs` | 测试用空上下文实现 |
| `SessionSndContext.cs` | 会话级上下文适配器 |
| `SndWorld.cs` | SND 世界：策略池 + 类型映射 + 转换器注册表 + 模板/别名 |
| `SndDefaults.cs` | SND 系统默认值常量 |
| `SndMappings.cs` | 场景别名解析 + 模板注册与解析 |
| `SndTemplateResolver.cs` | 模板解析器：支持 JSON 数组和 .map 简写两种模板格式 |
| `LevelBuilder.cs` | 离线关卡构建工具 |

## 实体模型

```
SndEntity (聚合根)
├── SndDataManager
│   ├── DataObserverManager (订阅/通知)
│   └── Dictionary<string, TypedData> (数据存储)
├── SndNodeManager : INodeHost
│   ├── Dictionary<string, INodeHandle> (节点存储)
│   └── INodeFactory (节点创建，由适配层注入)
├── SndStrategyManager (被动策略)
│   ├── List<StrategyEntry> (按优先级排序，每帧遍历)
│   └── SndStrategyPool (全局策略池引用)
└── ActiveStrategyManager (主动策略)
    ├── Dictionary<string, ActiveStrategyBase> (O(1) 按索引查找)
    └── SndStrategyPool (共享同一池实例)
```

## 策略生命周期钩子（顺序）

1. **AfterSpawn** — 实体新生成后
2. **AfterLoad** — 实体从存档恢复后
3. **AfterAdd** — 策略动态添加到实体后
4. **Process** — 每帧执行（按优先级）
5. **BeforeRemove** — 策略从实体移除前
6. **BeforeSave** — 序列化存档前
7. **BeforeQuit** — 实体正常退出前
8. **BeforeDead** — 实体销毁前

## 核心原则

- **策略无状态**：策略实例共享，可变状态在实体 Data 中
- **节点解耦**：Core 不持有引擎节点引用，通过 `INodeHandle` 抽象操作
- **元数据驱动**：实体的创建/恢复/序列化全部通过 `SndMetaData` 中介

---
[↑ 回到 Origo.Core](../README.md)
