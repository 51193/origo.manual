# FileSystem (Abstractions)

> [↑ 回到 Abstractions](../README.md) · [↔ 实现: GodotAdapter/FileSystem](../../../Origo.GodotAdapter/FileSystem/README.md)

## 概述

定义平台无关的文件系统抽象接口 `IFileSystem`。Core 层对底层文件系统（引擎虚拟路径、OS 本地路径）无感知，所有路径操作由实现负责，以正确处理平台特定的路径语义（如 Godot 的 `res://`、`user://` 前缀）。

## 包含文件

| 文件 | 职责 |
|------|------|
| `IFileSystem.cs` | 文件系统完整抽象：读写、遍历、目录管理、路径拼接 |

## 接口成员

| 方法 | 说明 |
|------|------|
| `Exists(path)` | 检查文件是否存在 |
| `DirectoryExists(path)` | 检查目录是否存在 |
| `ReadAllText(path)` | 读取文件全部文本 |
| `WriteAllText(path, content, overwrite)` | 写入文本；可控制覆盖行为 |
| `Copy(src, dst, overwrite)` | 复制文件或目录 |
| `EnumerateFiles(dir, pattern, recursive)` | 枚举文件，支持搜索模式和递归 |
| `CreateDirectory(path)` | 创建目录（含父目录） |
| `Delete(path)` | 删除文件；不存在则忽略 |
| `DeleteDirectory(path)` | 递归删除目录；不存在则忽略 |
| `Rename(src, dst)` | 原子重命名/移动 |
| `CombinePath(base, relative)` | 平台正确的路径拼接 |
| `GetParentDirectory(path)` | 获取父目录路径 |

## 设计决策

### 为什么 IFileSystem 包含路径操作

像 `CombinePath` 和 `GetParentDirectory` 这类方法看似是纯字符串操作，但在 Godot 中 `res://` 和 `user://` 的语义不同于普通路径。将路径操作纳入接口，让实现层可以正确处理虚拟路径前缀，避免 Core 层做错误的字符串拼接假设。

### 为什么用显式的 overwrite 参数

路径操作的安全语义应明确可读。隐式覆盖或静默失败会让调用方难以判断写操作的真实结果。显式 `overwrite` 参数强制调用方做明确选择。

### 为什么 Rename 的覆盖行为由实现决定

不同平台（Godot 虚拟文件系统 vs OS 本地文件系统）对同名目标的重命名行为不同。Core 层不应预设行为，由适配层根据平台语义实现最安全的策略。

### 为什么策略不直接使用 IFileSystem

`IFileSystem` 仅暴露原始文本读写，不提供解析能力。策略通过 `ISndFileAccess`（`ISndContext` 的第 10 个角色接口）访问文件——此接口的文件内容操作均经过 `IDataSourceIoGateway` 边界，自动处理后缀路由（`.json` / `.map` 等）并返回已解析的 `DataSourceNode` 树或强类型对象。这确保：

- 所有文件内容读写统一经过 `IDataSourceIoGateway`（框架的硬性 I/O 边界）
- 策略无需自行解析原始 JSON/Map 文本
- 编码/解码策略集中管理，更换引擎时无需修改策略代码

---
[↑ 回到 Abstractions](../README.md)
