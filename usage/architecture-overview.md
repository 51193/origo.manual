# 架构总览

> [↑ 回到 usage](README.md)

## 项目设计

Origo 是一个**轻量、平台无关的 C# 游戏框架**，采用 SND（Strategy + Node + Data）实体模型。项目由两层组成：

- **Origo.Core**：平台无关核心，所有游戏逻辑、持久化、实体模型在此层
- **Origo.GodotAdapter**：Godot 4 适配层，将 Core 的抽象接口与 Godot 引擎 API 桥接

## 四层运行时

```
SystemRun (OrigoRuntime)
  ↓
ProgressRun (ProgressRuntime)
  ↓
SessionManager (SessionManagerRuntime)
  ↓
SessionRun (foreground + background)
```

### 能力传递规则

- 上层向下层注入依赖，下层不得反向引用上层
- 每层通过结构化参数对象构造，不使用零散参数
- 每层维护本层聚合容器（`*Runtime`），集中持有引用

### 各层职责

| 层级 | 容器 | 持有资源 |
|------|------|---------|
| System | `SystemRuntime` | SystemBlackboard, SndWorld, ILogger, IScheduler |
| Progress | `ProgressRuntime` | ProgressBlackboard, SaveContext, SessionManager |
| Session | `SessionManagerRuntime` | ISessionManager → 管理所有 SessionRun |
| Instance | `SessionRun` | SessionBlackboard, ISndSceneHost, StateMachineContainer |

## SND 实体模型

```
Strategy + Node + Data
```

| 组件 | 说明 | 位置 |
|------|------|------|
| **Strategy** | 行为逻辑，无实例可变状态 | 策略池（全局共享） |
| **Node** | 表现层映射，由引擎实现驱动 | 适配层（GodotNodeHandle） |
| **Data** | 实体可变状态的唯一权威来源 | SndDataManager (TypedData) |

### 策略池约束

- 策略通过 `[StrategyIndex("xxx")]` 显式注册
- 索引缺失、空值、类型不匹配必须抛错
- 策略注册时通过反射校验无状态性（拒绝实例字段/可写属性）
- 策略按优先级排序，同优先级按插入顺序

## 会话模型

- 前台/后台均通过 `ISessionRun` 表达，无两套并行类型
- 前台唯一键为 `__foreground__`
- 差异仅在 `IsFrontSession` 与注入的 `ISndSceneHost`
- 业务逻辑不得按会话实现类型分支

## 持久化

### 文件布局

```
{saveRoot}/
├── current/                          # 活动存档（临时）
│   ├── .write_in_progress
│   ├── progress.json
│   ├── progress_state_machines.json
│   ├── meta.map
│   └── level_{id}/
│       ├── snd_scene.json
│       ├── session.json
│       └── session_state_machines.json
└── save_{id}/                        # 快照存档（持久）
    └── ... (同上)
```

### 两阶段写入

1. 写入 `current/`（先建标记 → 写全部文件 → 删标记）
2. 快照 `current/` → `save_{id}/`（atomic rename via temp directory）

### 严格读取

- 关卡三件套全部缺失 → 视为尚无存档
- 部分存在 → 视为损坏，抛异常
- `current/` 下有 `.write_in_progress` → 抛异常（中断写入）

## 并发模型

Origo 采用单线程帧循环模型：
- 生命周期动作通过延迟队列（`ActionScheduler` / `ConcurrentActionQueue`）顺序执行
- 单帧视为逻辑原子边界
- 跨帧语义落入可恢复状态（黑板、状态机、实体 Data）

## I/O 边界

Core 层所有文件操作通过 `IFileSystem` 和 `IDataSourceIoGateway`，禁止直接 `File.*` API：

```
业务模块 → DataSourceNode → IDataSourceIoGateway → IFileSystem → 文件系统
```

后缀路由、编解码策略与 I/O 错误语义集中在 Gateway 一侧统一治理。

## 项目结构

```
Origo.Core/           # 平台无关核心 (~90 个 .cs 文件)
├── Abstractions/     # 公共接口（Blackboard/Entity/FSM/...）
├── Blackboard/       # 黑板实现
├── DataSource/       # JSON/Map 编解码 + 类型转换
├── Random/           # 随机数 + 噪声
├── Runtime/          # 运行时四层生命周期 + 控制台
├── Save/             # 持久化存储
├── Scheduling/       # 延迟队列
├── Serialization/    # 类型 ↔ 字符串映射
├── Snd/              # SND 实体系统（策略 + 数据 + 节点）
├── StateMachine/     # 字符串栈状态机
└── Testing/          # 策略测试框架

Origo.GodotAdapter/   # Godot 4 适配层 (~17 个 .cs 文件)
├── Bootstrap/        # 启动编排
├── Console/          # Godot 命令
├── FileSystem/       # Godot 文件系统
├── Logging/          # Godot 日志
├── Serialization/    # Godot 类型序列化
└── Snd/              # Godot 实体 + 管理器

Origo.ConsoleBridge/  # TCP 远程控制台（~2 个 .cs 文件）
```

## 测试策略

- Core 侧：引擎依赖泄露、生命周期边界、持久化契约、策略池约束
- Adapter 侧：宿主装配、路径策略、序列化注册在 Godot 环境下可运行
- `Origo.Core` 总行覆盖率 ≥ 90%（Coverlet）
- 测试目录结构与生产代码镜像

---
[↑ 回到 usage](README.md)
