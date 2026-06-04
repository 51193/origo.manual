# SND 场景 测试

> [↑ 回到 Origo.Core.Tests](README.md)
> [↔ 被测模块: Origo.Core/Snd/Scene](../../Origo.Core/Snd/Scene/README.md)
> [↔ 被测行为: usage/snd-entity-model](../../usage/snd-entity-model.md)

## 被测行为概览

验证 SND 场景宿主层的实现：MemorySndSceneHost 的实体生命周期管理、
NullNodeFactory 的无操作行为。

## 测试文件清单

| 文件 | 验证侧重点 |
|------|-----------|
| `MemorySndSceneHostTests.cs` | MemorySndSceneHost 的 SpawnEntity/FindByName/BuildMetaList/RecoverFromMetaList/RemoveAllEntities |
| `NullNodeFactoryTests.cs` | NullNodeFactory 返回 NullNodeHandle，不抛异常 |

## 测试详情

### 正确路径

| 测试方法 | 验证的行为 | 文档出处 |
|---------|-----------|---------|
| MemorySndSceneHost.SpawnEntity 创建实体并加入列表 | SpawnEntity 后实体可被 FindByName 找到 | ISndSceneHost |
| BuildMetaList 返回当前实体元数据 | 序列化快照反映当前状态 | ISndSceneHost |
| RecoverFromMetaList 恢复实体列表 | 反序列化后 GetEntities 包含实体 | ISndSceneHost |
| RemoveAllEntities 清空所有实体 | RemoveAllEntities 后 GetEntities 为空 | ISndSceneHost |
| NullNodeFactory.Create 返回 NullNodeHandle | 无渲染模式下工厂返回空节点 | INodeFactory |

### 边界路径

| 测试方法 | 边界条件 | 预期行为 |
|---------|---------|---------|
| FindByName 不存在时返回 null | 查询不存在的实体 | 返回 null |

## 已知覆盖缺口

| 缺口描述 | 影响 | 文档依据 |
|---------|------|---------|
| 并发 SpawnEntity/RemoveAllEntities 的线程安全 | 场景宿主是否承诺线程安全 | ISndSceneHost |

---

[↑ 回到 Origo.Core.Tests](README.md)
