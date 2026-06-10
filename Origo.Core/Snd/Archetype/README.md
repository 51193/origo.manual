# Archetype

> [↑ 回到 Snd](../README.md)

## 概述

`.map` 原型文件解析器。解析 key:value 扁平对象格式的 archetype 定义文件，将字符串值推断为最合适的类型（int/float/bool/string），并写入实体数据字典。

## 包含文件

| 文件 | 职责 |
|------|------|
| `SndArchetypeLoader.cs` | TryLoad（从 .map 文件读取键值对）+ ApplyAttributes（类型推断并写入实体数据）|

## 实现详解

### SndArchetypeLoader

- **TryLoad(ISndFileAccess fileAccess, string path)**：通过框架文件抽象读取 .map 文件，要求根节点为 Object，返回 `Dictionary<string, string>`。文件不存在或格式不正确时返回 false
- **ApplyAttributes(ISndEntity entity, Dictionary<string, string> attributes)**：遍历属性字典，按 int → float → bool → string 的顺序尝试解析每个值，优先匹配数值更高的类型

类型推断规则：
1. `int.TryParse` 成功 → 写入为 `int`
2. `float.TryParse`（`InvariantCulture`）成功 → 写入为 `float`
3. `bool.TryParse` 成功 → 写入为 `bool`
4. 上述均失败 → 写入为 `string`

## 设计决策

### 为什么使用严格的类型推断顺序而非类型注解

`.map` 格式设计起源于简洁的 key:value 文本文件，不包含类型元信息。int → float → bool → string 的回退顺序确保 `"100"` 存为整数、`"3.14"` 存为浮点、`"true"` 存为布尔，符合游戏数据最常用场景。如需精确类型控制，使用 JSON 模板（`snd_templates`）。

### 为什么作为独立 Snd 子模块而非放在 DataSource

虽然逻辑上涉及文件解析，但其输出是针对 `ISndEntity` 的数据写入，直接服务于 SND 原型加载工作流。DataSource 是通用数据层，不应包含 SND 绑定。

---

[↑ 回到 Snd](../README.md)
