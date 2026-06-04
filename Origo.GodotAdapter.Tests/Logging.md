# 日志 测试（适配层）

> [↑ 回到 Origo.GodotAdapter.Tests](README.md)
> [↔ 被测模块: Origo.GodotAdapter/Logging](../../Origo.GodotAdapter/Logging/README.md)

## 被测行为概览

验证 GodotLogger 的委托注入模式：通过 `Action<LogLevel, string, string>` 委托代理日志输出、
null handler 时不抛异常（启动期间的灵活性）。

## 测试文件清单

| 文件 | 验证侧重点 |
|------|-----------|
| `GodotLoggerTests.cs` | GodotLogger 委托注入和 null handler 安全 |

### 正确路径

| 测试方法 | 验证的行为 | 文档出处 |
|---------|-----------|---------|
| `Log_WithHandler_InvokesHandlerWithCorrectLevelTagAndMessage` | Log(Warning, "Tag", "msg") → handler 收到正确参数 | GodotAdapter Logging |
| `Log_WithNullHandler_DoesNotThrow` | 构造时不传 handler，Log 调用不抛异常 | GodotAdapter Logging |

---

[↑ 回到 Origo.GodotAdapter.Tests](README.md)
