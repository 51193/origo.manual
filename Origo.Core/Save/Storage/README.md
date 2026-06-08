# Storage

> [↑ 回到 Save](../README.md)

## 概述

存档存储层的完整实现。负责文件 I/O（读写、目录管理、快照）、路径布局策略、Payload 构造。所有文件操作通过 `IFileSystem` + `IDataSourceIoGateway` 进行，不直接调用 `File.*` API。

## 包含文件

| 文件 | 职责 |
|------|------|
| `ISaveStorageService.cs` | 存档读写服务公开接口（跨程序集） |
| `ISavePathPolicy.cs` | 存档路径策略接口（可替换布局） |
| `DefaultSaveStorageService.cs` | ISaveStorageService 默认实现，内部委托给 SaveFileHandle + SavePayloadWriter/Reader |
| `DefaultSavePathPolicy.cs` | ISavePathPolicy 默认实现，委托给 SavePathLayout |
| `SaveFileHandle.cs` | 统一 I/O 上下文：封装 IFileSystem + IDataSourceIoGateway + saveRootPath + ISavePathPolicy，合并原 SavePathResolver 的路径工具方法和 SaveStorageGatewayFactory 的网关创建 |
| `SavePathLayout.cs` | 标准路径布局常量与方法（current/、save_*、level_*） |
| `SavePayloadWriter.cs` | 存档写入编排（两阶段写入 + marker 管理） |
| `SavePayloadReader.cs` | 存档读取编排（严格读取 + 完整性校验） |
| `SaveGamePayloadFactory.cs` | 构造 SaveGamePayload（业务数据聚合） |

## 文件布局

```
{saveRoot}/
├── current/                          # 活动存档目录
│   ├── .write_in_progress            # 写入中断标记
│   ├── .payload.sha                  # Payload SHA-256 摘要（幂等去重）
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

1. **幂等检查**：若目标快照 `save_{id}/.payload.sha` 存在且 hash 相同，直接返回（跳过写入）
2. **写入标记**：在 `current/` 下创建 `.write_in_progress`
3. **写入 payload**：写入 progress.json、各关卡三件套、meta.map、`.payload.sha`
4. **清除第一阶段标记**：`WriteToCurrent` 完成后删除 marker
5. **重新创建标记**：为快照阶段重建 marker
6. **快照**：复制 `current/` 到 `save_{id}.tmp/` → 删除旧的 `save_{id}/` → 重命名 `.tmp` 为正式目录
7. **清除标记**：删除 marker

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

### 为什么 SaveFileHandle 统一封装 I/O 依赖

SavePayloadReader/SavePayloadWriter/SaveStorageFacade 的所有方法原本需要逐层传递 `(IFileSystem, IDataSourceIoGateway, string saveRootPath, ISavePathPolicy)` 四件套，导致每个公共方法产生三级重载链（~36 个签名派生 11 个实际实现）。引入 `SaveFileHandle` 后，这四个依赖封装为单一参数对象，方法与实现一对一映射（~36 → ~16 个签名），`DefaultSaveStorageService` 内部从 4 个字段简化为 1 个字段。`SaveFileHandle` 同时合并了原 `SavePathResolver` 的路径工具方法和 `SaveStorageGatewayFactory` 的网关创建逻辑，消除了独立辅助类的碎片化。

### 为什么通过 SHA 摘要实现幂等去重

同一游戏状态多次写入同一存档槽时，SHA-256 摘要比较可避免无用 I/O。
`WriteSavePayloadToCurrentThenSnapshot` 入口处先比对目标快照的 `.payload.sha` 与待写入 payload 的 hash：

- **hash 相同** → 记录 INFO 日志 "idempotent save skip"，直接返回，不执行任何文件操作
- **hash 不同或 .sha 不存在** → 正常走两阶段写入流程

`ComputePayloadHash` 对各组件节点树分别计算 SHA 后拼合，确保：
- Progress 黑板变更 → hash 不同 → 重写
- 任何关卡数据变更 → hash 不同 → 重写
- CustomMeta 键值变更 → hash 不同 → 重写
- 仅 SaveId 不同（写入不同槽）→ 总是写入（新槽位无 .sha 可比较）

### 为什么 DataSourceNode 计算 Canonical Hash 而非序列化后 Hash

`DataSourceNode.ComputeSha256Hash()` 递归生成确定性字符串表示后做 SHA-256。
与通过 codec 序列化的方式不同，canonical 字符串不依赖 codec 版本和缩进配置，
键已按字典序排序，确保同一数据树始终产生相同 hash。

---
[↑ 回到 Save](../README.md)
