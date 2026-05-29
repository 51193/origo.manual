# Storage

> [↑ 回到 Save](../README.md)

## 概述

存档存储层的完整实现。负责文件 I/O（读写、目录管理、快照）、路径布局策略、Payload 构造。所有文件操作通过 `IFileSystem` + `IDataSourceIoGateway` 进行，不直接调用 `File.*` API。

## 包含文件

| 文件 | 职责 |
|------|------|
| `ISaveStorageService.cs` | 存档读写服务公开接口（跨程序集） |
| `ISavePathPolicy.cs` | 存档路径策略接口（可替换布局） |
| `DefaultSaveStorageService.cs` | ISaveStorageService 默认实现 |
| `DefaultSavePathPolicy.cs` | ISavePathPolicy 默认实现，委托给 SavePathLayout |
| `SavePathLayout.cs` | 标准路径布局常量与方法（current/、save_*、level_*） |
| `SavePathResolver.cs` | 路径工具（父目录确保、相对路径提取、遍历防护） |
| `SavePayloadWriter.cs` | 存档写入编排（两阶段写入 + marker 管理） |
| `SavePayloadReader.cs` | 存档读取编排（严格读取 + 完整性校验） |
| `SaveStorageFacade.cs` | 存档 I/O 门面：编排 Reader/Writer + 快照 |
| `SaveGamePayloadFactory.cs` | 构造 SaveGamePayload（业务数据聚合） |
| `SaveStorageGatewayFactory.cs` | 创建 IDataSourceIoGateway 实例的工厂 |

## 文件布局

```
{saveRoot}/
├── current/                          # 活动存档目录
│   ├── .write_in_progress            # 写入中断标记
│   ├── progress.json                # 流程黑板
│   ├── progress_state_machines.json  # 流程级状态机
│   ├── meta.map                     # 展示元数据
│   └── level_{id}/                  # 各关卡存档
│       ├── snd_scene.json           # 三件套
│       ├── session.json
│       └── session_state_machines.json
├── save_001/                        # 快照存档槽
│   └── ... (同上结构)
└── save_002/
    └── ...
```

## 两阶段写入流程

1. **写入标记**：在 `current/` 下创建 `.write_in_progress`
2. **写入 payload**：写入 progress.json、各关卡三件套、meta.map
3. **清除第一阶段标记**：`WriteToCurrent` 完成后删除 marker
4. **重新创建标记**：为快照阶段重建 marker
5. **快照**：复制 `current/` 到 `save_{id}.tmp/` → 删除旧的 `save_{id}/` → 重命名 `.tmp` 为正式目录
6. **清除标记**：删除 marker

## 严格读取规则

- **current/ 下有 `.write_in_progress`** → 拒绝读取，抛异常（上次写入中断）
- **关卡三件套不全** → 拒绝读取（部分存在 = 损坏）
- **progress.json 缺失** → 拒绝读取

## 设计决策

### 为什么两阶段写入

`current/` 先写入确保数据落地，快照阶段将已验证的完整数据复制到持久槽。如果快照阶段失败，`current/` 仍保留完整数据（但 marker 残留，下次读取会拒绝并通过日志告知）。不会出现"快照槽有数据但 current/ 因崩溃丢失"的情况。

### 为什么不使用临时文件 + Rename 原子写入单个文件

存档涉及多个文件（progress.json + N 个关卡的 3 个文件）。单个文件的原子 rename 无法保证多文件一致性。`.write_in_progress` marker 是整个存档目录的事务标记。

### 为什么 ISavePathPolicy 可替换

不同平台（桌面、移动、云存档）可能需要不同路径布局。将布局策略注入 `DefaultSaveStorageService` 而非硬编码，允许在适配层注入平台特定策略。

### 为什么 SaveStorageFacade 是静态门面

保存逻辑是纯数据变换 + I/O 操作，无实例状态。静态方法减少依赖注入的工厂复杂度。但内部 Reader/Writer 通过参数接收所有依赖（fileSystem、dataSourceIo、pathPolicy），保持可测试性。

---
[↑ 回到 Save](../README.md)
