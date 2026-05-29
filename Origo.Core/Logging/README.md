# Logging

> [↑ 回到 Origo.Core](../README.md) · [↔ 抽象: Abstractions/Logging](../Abstractions/Logging/README.md)

## 概述

`ILogger` 接口的实现层。提供两个实现：生产可用的结构化日志构建器 `LogMessageBuilder`，以及用于测试或静默场景的 `NullLogger`。

## 包含文件

| 文件 | 职责 |
|------|------|
| `LogMessageBuilder.cs` | 流式构建结构化日志消息（前缀/后缀上下文 + 耗时） |
| `NullLogger.cs` | 无输出日志实现，单例模式 |

## 实现详解

### LogMessageBuilder

流式 API，支持链式调用：

```
builder.SetElapsedMs(12.3).AddPrefix("entity", "player").AddSuffix("hp", 100).Build("damage applied")
// 输出: [+12.3ms] entity=player | damage applied | hp=100
```

- **前缀上下文**：消息前插入的元数据（如实体名、会话 ID）
- **后缀上下文**：消息后插入的元数据（如结果值、状态）
- **耗时**：可选，以 `[+Xms]` 格式前置
- **空值过滤**：key 非空且 value 非 null 才添加

### NullLogger

单例实现，`Log` 方法为空操作体。用于测试或不需要日志的上下文，避免 null 检查。

## 设计决策

### 为什么 LogMessageBuilder 是 Builder 模式而非直接拼接

结构化日志格式（前缀 | 正文 | 后缀）在不同上下文中前缀/后缀不同，但正文相同。Builder 让调用方可以预设上下文（如会话信息），逐步追加数据点，最后一次性 Build。避免重复传递相同的上下文参数。

### 为什么不直接在 ILogger 提供结构化方法

`ILogger` 保持最小接口（仅 `Log(level, tag, message)`），以适配不同的日志后端（Godot 的 `GD.Print`、文件 appender、网络日志）。结构化构建是 Core 层的增值服务，由 `LogMessageBuilder` 负责，最终产出纯字符串。

### 为什么 NullLogger 使用单例

`NullLogger` 无任何可变状态，创建多个实例是资源浪费。私有构造函数 + 静态 Instance 属性确保全局唯一。

---
[↑ 回到 Origo.Core](../README.md)
