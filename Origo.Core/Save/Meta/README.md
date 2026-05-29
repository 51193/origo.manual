# Meta

> [↑ 回到 Save](../README.md)

## 概述

存档展示元数据（meta.map）的构建与合并系统。展示元数据与业务数据（progress.json、session.json）分离，仅用于存档选择界面展示（如关卡名、游玩时间、截图路径）。通过可插拔的贡献者模式收集元数据键值对。

## 包含文件

| 文件 | 职责 |
|------|------|
| `ISaveMetaContributor.cs` | 元数据贡献者接口 |
| `DelegateSaveMetaContributor.cs` | 委托适配的贡献者实现 |
| `SaveMetaBuildContext.cs` | 单次存档时的只读构建上下文 |
| `SaveMetaDataEntry.cs` | 存档槽条目模型（SaveId + MetaData 字典） |
| `SaveMetaMapCodec.cs` | meta.map 文件的编解码 |
| `SaveMetaMerger.cs` | 合并贡献者 + 入参覆写的合并逻辑 |

## 模块详解

### 元数据贡献流程

1. **注册贡献者**：实现 `ISaveMetaContributor`，通过 `SaveContext` 注册
2. **构建上下文**：存档时创建 `SaveMetaBuildContext`（含 saveId、levelId、黑板、场景访问）
3. **收集**：`SaveMetaMerger.Merge()` 按注册顺序调用每个贡献者的 `Contribute()`，同名键后者覆盖前者
4. **覆写**：调用方提供的 `customMeta` 参数再次键级覆盖
5. **序列化**：最终字典通过 `SaveMetaMapCodec.Serialize()` 转为 key:value 文本，写入 `meta.map`

### ISaveMetaContributor

单方法接口：
```csharp
void Contribute(in SaveMetaBuildContext context, IDictionary<string, string> target);
```

`target` 为可变字典，贡献者向其中添加键值对。`context` 为只读结构体（struct），通过 `in` 传递避免拷贝。

### SaveMetaMapCodec

- **Parse**：委托给 `KeyValueFileParser.Parse`（见 DataSource 模块）
- **Serialize**：按键字典序输出 `"key: value"` 行

### SaveMetaMerger

静态工具类。合并逻辑：非空 contributors → 按序贡献 → 覆写键（跳过空值键）→ null 或无键时返回 null。

## 设计决策

### 为什么展示元数据与业务数据分离

业务数据（progress.json / session.json）包含完整的时序状态和类型信息，体积大；展示元数据仅包含存档选择界面需要的少量摘要。分离后读取存档列表时只需读取 meta.map（KB 级），无需解析完整的 progress.json（可能 MB 级）。

### 为什么贡献者是按序同名覆盖而非去重

不同贡献者可能对同一键有不同视角（如"play_time"，一个贡献者从流程黑板读取，另一个可能从会话黑板）。后注册者的值可能更准确。按序覆盖提供可预测的优先级模型。

### 为什么 SaveMetaBuildContext 是 readonly struct

构建上下文在存档调用树中频繁传递。struct 避免堆分配，readonly 确保贡献者无法修改上下文（防止副作用扩散）。`in` 参数按引用传递避免拷贝。

---
[↑ 回到 Save](../README.md)
