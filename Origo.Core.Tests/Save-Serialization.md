# 持久化：序列化 测试

> [↑ 回到 Origo.Core.Tests](README.md)
> [↔ 被测模块: Origo.Core/Save/Serialization](../../Origo.Core/Save/Serialization/README.md)
> [↔ 被测行为: usage/persistence-flow](../../usage/persistence-flow.md)

## 被测行为概览

验证存档 Payload 的序列化编排：Blackboard 序列化/反序列化往返、SND 场景实体列表序列化、
SaveContext 在 ProgressRun 流程中的协调行为。

## 测试文件清单

| 文件 | 验证侧重点 |
|------|-----------|
| `BlackboardSerializerTests.cs` | Blackboard 序列化/反序列化：TypedData 类型保留、多键值往返 |
| `SndSceneSerializerTests.cs` | SND 场景序列化：实体元数据列表 ↔ DataSourceNode 往返 |
| `SaveContextTests.cs` | SaveContext：Payload 构建/写入编排与 Session 数据收集 |

## BlackboardSerializerTests 测试详情

### 正确路径

| 测试方法 | 验证的行为 | 文档出处 |
|---------|-----------|---------|
| `Serialize_ThenDeserialize_PreservesAllKeys` | 序列化后反序列化恢复全部键值，类型不丢失 | Blackboard Abstraction |

### 边界路径

| 测试方法 | 边界条件 | 预期行为 |
|---------|---------|---------|
| `Serialize_EmptyBlackboard_ProducesEmptyDict` | 空白板序列化 | 空字典 |

## SndSceneSerializerTests 测试详情

### 正确路径

| 测试方法 | 验证的行为 | 文档出处 |
|---------|-----------|---------|
| `Serialize_ThenDeserialize_PreservesEntityNames` | 场景实体列表序列化后恢复实体名 | Snd/Scene |
| `Deserialize_IntoHost_RestoresEntities` | 反序列化后 SceneHost 恢复实体列表 | Snd/Scene |

## SaveContextTests 测试详情

### 正确路径

| 测试方法 | 验证的行为 | 文档出处 |
|---------|-----------|---------|
| 测试 SaveContext 在 ProgressRun 流程中构建 Payload 并写入 | SaveContext 编排黑板的 Collection + 写入 | Save/README.md |

## 测试辅助策略

| 策略类 | 定义位置 | 用途 |
|--------|---------|------|
| 无（本组测试直接使用 TestSndSceneHost 和 TestFileSystem） | — | — |

## 已知覆盖缺口

| 缺口描述 | 影响 | 文档依据 |
|---------|------|---------|
| SaveContext 在序列化失败时的回滚行为 | 部分写入的清理 | persistence-flow |
| BeforeSave 钩子触发后数据未正确刷新到 Payload | 序列化前钩子未执行 | snd-entity-model |

---

[↑ 回到 Origo.Core.Tests](README.md)
