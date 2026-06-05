# SourceGeneration

> [↑ 回到 Origo.Core](../README.md)

## 概述

编译期代码生成器，在 Origo.Core 构建时运行。通过读取程序集级别的类型注册属性，为 `TypedData` 部分结构体生成类型特化的成员方法、工厂类和序列化桥接代码。

## 包含文件

| 文件 | 职责 |
|------|------|
| `TypedDataGenerator.cs` | Roslyn `IIncrementalGenerator`，生成 `TypedData` 的 partial 成员 |

## 生成内容

Source Generator 读取 `[assembly: SndInlineTypes(typeof(int), typeof(float), ...)]` 属性，为 `TypedData` 结构体生成以下成员：

| 生成类别 | 生成内容 | 说明 |
|---------|---------|------|
| **种类常量** | `KindMap` 静态类 | 每种注册类型的 `const byte` 判别值（如 `KindMap.Int32 = 5`） |
| **类型映射表** | `TypedDataTypeMap` 静态类 | `kind` ↔ `System.Type` 双向映射，用于序列化边界和运行时类型查询 |
| **强类型构造** | `explicit operator TypedData(...)` | 每种注册类型的显式转换运算符，将值内联写入结构体字段，零装箱 |
| **强类型读取** | `AsXxx()` / `TryGetXxx()` | 类型安全的访问器方法，通过判别值分发，无 `is T` 模式匹配开销 |
| **泛型工厂** | `TypedDataFactory<T>` 静态类 | `Create(T)` / `TryExtract(TypedData, out T)`，使用 `typeof(T) == typeof(int)` if-else 链（JIT 编译时完整常量折叠） |
| **序列化桥接** | `TypedDataObjectConverter` 静态类 | `ToObject(TypedData)` / `FromObject(byte, object)`，在 TypedData 和 `object?` 之间转换（用于序列化边界） |
| **静态构造器** | `static TypedData()` | 填充 `KindTypeMap` 数组，将 `kind` 值映射到 `System.Type` |

### 类型内联策略

| 类型 | 存储方式 | 说明 |
|------|---------|------|
| 值类型 ≤ 8 字节（`int`、`float`、`double`、`long`、`bool`、`byte` 等） | `_inlineBits : long` 字段内联 | 零堆分配，零装箱 |
| 引用类型（`string`） | `_ref : object?` 字段存储 | 无额外包装 |
| 未注册类型 | `_ref : object?` 兜底 | 通过 `FromObject(Type, object?)` 构造，性能等同原 `class` 方案 |

> 判别枚举值 `0` 固定为 `Null` 哨兵值（`default(TypedData)`），以允许值类型在 `Dictionary` 中安全存储未初始化状态。

## 注册机制

类型通过程序集级属性注册：

```csharp
[assembly: SndInlineTypes(
    typeof(byte), typeof(int), typeof(float), typeof(double),
    typeof(bool), typeof(char), typeof(string)
)]
```

`SndInlineTypesAttribute` 定义在 `Origo.Core.Snd.Metadata` 命名空间下，接受 `params Type[]` 参数。同一程序集可多次声明，所有声明合并为完整的类型集合。

由于 Source Generator 是 per-project 的，下游适配层（如 `Origo.GodotAdapter`）如需内联自己的类型，需使用各自程序集中的 `[assembly: SndInlineTypes]` 属性。未注册的类型走 `_ref` 兜底路径，性能等同原有方案。

## 设计决策

### 为什么使用 Source Generator 而非手写

TypedData 需要为每种注册类型生成大量样板代码（构造器、访问器、工厂方法、序列化桥接）。手写维护这些代码容易出错且类型间不一致。Source Generator 确保所有注册类型自动生成一致且最优的实现。

### 为什么使用 `typeof(T) == typeof(int)` if-else 链

`TypedDataFactory<T>` 的泛型分发使用 `typeof` 比较链，而非接口注入或虚方法调用。原因：每个泛型方法的 JIT 特化版本在编译时进行常量折叠，每个 `T` 的特化版本只保留一条路径，零运行时分支开销。

### 为什么生成 `explicit operator` 而非 `implicit`

显式转换要求调用方明确表达转换意图（`(TypedData)42`），避免隐式转换在参数传递中意外发生（如 `NotifyObservers("hp", 100, 50)` 中 `int` 意外转为 `TypedData`）。同时与 `TypedDataFactory<T>` 的显式构造语义一致。

---

[↑ 回到 Origo.Core](../README.md)
