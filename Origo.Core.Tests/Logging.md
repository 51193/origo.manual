# 日志 测试

> [↑ 回到 Origo.Core.Tests](README.md)
> [↔ 被测模块: Origo.Core/Logging](../Origo.Core/Logging/README.md)

## 被测行为概览

验证 LogMessageBuilder 的结构化构建：纯消息、带时间戳（SetElapsedMs）、
带前缀（AddPrefix）、带后缀（AddSuffix）、null/空白键跳过、组合使用。

## 测试文件清单

| 文件 | 验证侧重点 |
|------|-----------|
| `LogMessageBuilderTests.cs` | LogMessageBuilder 结构化构建 |

## 测试详情

### 正确路径

| 测试方法 | 验证的行为 | 文档出处 |
|---------|-----------|---------|
| `Build_PlainMessage` | Build("hello") 返回 "hello" | Logging |
| `SetElapsedMs_IncludesTimestamp` | SetElapsedMs(12.345) → "[+12.34ms] test" | Logging |
| `AddPrefix_IncludesPrefix` | AddPrefix("ctx","val") → "ctx=val \| test" | Logging |
| `AddSuffix_IncludesSuffix` | AddSuffix("key","val") → "test \| key=val" | Logging |
| `CombinedPrefixSuffix` | timestamp + prefix + suffix 组合正确 | Logging |

### 边界路径

| 测试方法 | 边界条件 | 预期行为 |
|---------|---------|---------|
| `AddPrefix_NullKey_Skipped` | null 前缀 key | 跳过 |
| `AddPrefix_NullValue_Skipped` | null 前缀 value | 跳过 |
| `AddSuffix_WhitespaceKey_Skipped` | 空白后缀 key | 跳过 |

## 已知覆盖缺口

无——此模块仅一个简单工具类，测试覆盖已完整。

---

[↑ 回到 Origo.Core.Tests](README.md)
