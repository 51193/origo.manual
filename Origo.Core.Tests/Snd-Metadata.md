# SND 元数据 测试

> [↑ 回到 Origo.Core.Tests](README.md)
> [↔ 被测模块: Origo.Core/Snd/Metadata](../../Origo.Core/Snd/Metadata/README.md)
> [↔ 被测行为: usage/snd-entity-model](../../usage/snd-entity-model.md)

## 被测行为概览

验证 SND 元数据的核心类型：
- **TypedData**：类型+值的保留容器，支持值类型/引用类型/null/数组/struct
- **SndMetaData**：实体元数据的深拷贝，Node/Strategy/Data 三大模块全部正确复制

## 测试文件清单

| 文件 | 验证侧重点 |
|------|-----------|
| `TypedDataTests.cs` | TypedData 构造、类型保留、值/引用类型行为、独立实例 |
| `SndMetaDataTests.cs` | SndMetaData 默认值、DeepClone 深拷贝、修改不影响原对象 |

## TypedDataTests 测试详情

### 正确路径

| 测试方法 | 验证的行为 | 文档出处 |
|---------|-----------|---------|
| `Constructor_StoresTypeAndValue` | `new TypedData(typeof(int), 42)` 正确存储 Type 和 Data | snd-entity-model: TypedData |
| `NullValue_IsAllowed` | `new TypedData(typeof(string), null)` 允许 null 值 | — |
| `WithIntValue_PreservesExactType` | int 类型保留为 typeof(int) | snd-entity-model: TypedData |
| `WithFloatValue_PreservesExactType` | float 类型保留为 typeof(float) | snd-entity-model: TypedData |
| `WithDoubleValue_PreservesExactType` | double 类型保留 | snd-entity-model: TypedData |
| `WithBoolValue_PreservesExactType` | bool 类型保留 | snd-entity-model: TypedData |
| `WithStringValue_PreservesExactType` | string 类型保留 | snd-entity-model: TypedData |
| `WithStructValue_PreservesExactType` | Guid 类型保留 | snd-entity-model: TypedData |
| `WithDateTimeValue_PreservesExactType` | DateTime 类型保留 | snd-entity-model: TypedData |
| `WithBoxedInt_KeepsRuntimeType` | 装箱 int 保持 typeof(int) | snd-entity-model: TypedData |
| `WithArrayType_PreservesExactType` | int[] 类型保留 | snd-entity-model: TypedData |
| `WithReferenceType_PreservesIdentity` | 引用类型保持同一对象引用 | snd-entity-model: TypedData |
| `WithNullValueForReferenceType_PreservesTypeInfo` | null 值保留 List<int> 类型信息 | snd-entity-model: TypedData |

### 边界路径

| 测试方法 | 边界条件 | 预期行为 |
|---------|---------|---------|
| `TwoInstances_SameTypeAndSameValue_HaveDifferentReferences` | 相同值的两个 TypedData 是不同引用 | 不共享状态 |
| `TwoInstances_DifferentType_HaveDifferentReferences` | 不同值的两个 TypedData 是不同引用 | 不共享状态 |

## SndMetaDataTests 测试详情

### 正确路径

| 测试方法 | 验证的行为 | 文档出处 |
|---------|-----------|---------|
| `DefaultValues` | Name 为空串、NodeMetaData/StrategyMetaData 为 null、DataMetaData 非 null | snd-entity-model: 实体元数据 |
| `DeepClone_CopiesName` | DeepClone 复制 Name | SndMetaData |
| `DeepClone_CopiesNodeMetaData` | NodeMetaData 深复制（Pairs 独立） | SndMetaData |
| `DeepClone_CopiesStrategyMetaData` | StrategyMetaData 深复制（EntityIndices 独立） | SndMetaData |
| `DeepClone_CopiesDataMetaData` | DataMetaData 深复制（Pairs 独立，TypedData 值保留） | SndMetaData |
| `DeepClone_NullNodeMetaData_RemainsNull` | null NodeMetaData 克隆后仍为 null | SndMetaData |
| `DeepClone_ModifyCloneDoesNotAffectOriginal` | 修改克隆不影响原对象（Name + EntityIndices） | SndMetaData |
| `WithActiveStrategyIndices_DeepClones` | ActiveIndices 正确深复制 | SndMetaData |

### 边界路径

| 测试方法 | 边界条件 | 预期行为 |
|---------|---------|---------|
| `DeepClone_EmptyNodePairs_CopiesCorrectly` | 空 NodeMetaData.Pairs 复制后仍为空 | 不抛异常 |
| `DeepClone_EmptyDataPairs_CopiesCorrectly` | 空 DataMetaData.Pairs 复制后仍为空 | 不抛异常 |

## 已知覆盖缺口

| 缺口描述 | 影响 | 文档依据 |
|---------|------|---------|
| TypedData 自定义 Equality（若实现）的测试 | TypedData 是纯数据载体，是否需要值相等语义 | TypedData |
| SndMetaData 非常大量策略索引时的性能 | 极端数据量 | — |

---

[↑ 回到 Origo.Core.Tests](README.md)
