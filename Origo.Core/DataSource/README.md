# DataSource

> [↑ 回到 Origo.Core](../README.md)

## 模块能力

Origo 的数据源抽象层——Core 与外部格式（JSON、.map）之间的编解码桥梁。提供统一的 `DataSourceNode` 树形数据模型、按文件后缀自动路由编解码器的 I/O Gateway、以及 CLR 类型与节点数据之间的双向转换器注册表。

## 子模块

| 子模块 | 能力 | 详情 |
|--------|------|------|
| [Codec](Codec/README.md) | 格式编解码 | `JsonDataSourceCodec`（延迟展开）+ `MapDataSourceCodec`（key:value） |
| [Converters](Converters/README.md) | 类型转换 | 14 种基础类型 + 14 种数组 + 8 种领域类型 + TypedData |

## 本层核心文件

| 文件 | 职责 |
|------|------|
| `DataSourceNode.cs` | 树形数据节点：Object/Array/String/Number/Boolean/Null + 延迟展开（Lazy）|
| `DataSourceNodeKind.cs` | 节点类型枚举 |
| `DataSourceCodecKind.cs` | 编解码格式枚举（Json / Map） |
| `IDataSourceCodec.cs` | 编解码器接口：Decode/Encode |
| `IDataSourceIoGateway.cs` | I/O 网关接口：按后缀路由编解码器后读写文件（Core 与文件的唯一接触点）|
| `DataSourceIoGateway.cs` | I/O 网关实现：后缀 → CodecKind 映射 + 读写 |
| `DataSourceIoOptions.cs` | I/O 选项：是否缩进输出 |
| `DataSourceFactory.cs` | 工厂：创建默认 Registry + IoGateway |
| `DataSourceConverter.cs` | 泛型转换器基类：`Read(DataSourceNode)` / `Write(T)` |
| `DataSourceConverterRegistry.cs` | 转换器注册表：按 Type 查找 Converter + 泛型 Read/Write |
| `KeyValueFileParser.cs` | key:value 格式解析器（用于 .map 文件） |
| `MemoryFileSystem.cs` | 测试用内存文件系统实现 `IFileSystem` |

## 数据流

```
外部文件 (.json / .map)
    │
    ▼
IDataSourceIoGateway (后缀路由 → Codec)
    │
    ▼
DataSourceNode (树形数据)
    │
    ▼
DataSourceConverterRegistry (类型转换)
    │
    ▼
CLR 对象 (TypedData / SndMetaData / etc.)
```

## 设计原则

- **IDataSourceIoGateway 硬边界**：Core 中所有文件 I/O 必须经过 Gateway，禁止直接 `File.*` API
- **延迟展开**：JSON 大型节点在访问时才展开子节点，避免全量解析
- **零反射**：所有转换器显式注册，不使用反射自动发现

---
[↑ 回到 Origo.Core](../README.md)
