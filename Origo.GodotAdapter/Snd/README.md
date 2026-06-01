# Snd

> [↑ 回到 Origo.GodotAdapter](../README.md) · [↔ Core: Snd](../../Origo.Core/Snd/README.md)

## 概述

SND 实体体系在 Godot 引擎中的具体实现。将 Core 的抽象 `ISndEntity` / `INodeHandle` / `INodeFactory` / `ISndSceneHost` 与 Godot 的 `Node` / `PackedScene` 生命周期对接。

## 包含文件

| 文件 | 职责 |
|------|------|
| `GodotSndManager.cs` | Godot 场景宿主：管理 GodotSndEntity 集合，实现 ISndSceneHost，驱动 Process 帧循环 |
| `GodotSndEntity.cs` | Godot 实体：将 Core SndEntity 绑定到 Godot Node 生命周期，委托所有 ISndEntity 调用 |
| `GodotPackedSceneNodeFactory.cs` | INodeFactory 实现：通过 PackedScene.Instantiate 创建 Godot Node |
| `GodotNodeHandle.cs` | INodeHandle 实现：包装 Godot.Node，提供 Free / SetVisible |

## 模块详解

### GodotSndManager

适配层的核心入口节点（`[GlobalClass]`），直接挂载在 Godot 场景树中：

- **实现 ISndSceneHost**：Spawn / LoadFromMetaList / ClearAll（框架内部生命周期操作）/ RequestKillEntity / DeadByName / GetEntities / FindByName / ProcessAll。`ClearAll()` 使用 `Free()`（即时释放）而非 `QueueFree()`，因 Core 保证在安全的生命周期时机调用。
- **实现 ISndContextAttachableSceneHost**：支持运行时切换上下文
- **_Process(delta)**：Godot 帧循环入口，快照迭代实体 Process，刷新延迟队列
- **回滚机制**：`LoadFromMetaList` 中若某实体加载失败，回滚释放所有已创建的实体
- **EntityView**：惰性创建 `IReadOnlyList<ISndEntity>` 包装，缓存引用避免重复分配

### GodotSndEntity

Core `SndEntity` 的 Godot 包装器（`[GlobalClass]`）：

- **延迟初始化**：`_entity` 在首次访问时通过 `SndWorld.CreateEntity` 创建
- **Lifecycle 分离**：`QuitFromManager` / `DeadFromManager` 仅由 GodotSndManager 调用，确保先从管理列表移除再释放
- **StableName**：独立存储实体稳定名（Godot Node 的 Name 可能因重名自动修改后缀）
- **GetNodeFromSnd<TNode>**：Godot 特有扩展——从 SND 节点系统按名查找并强转为 Godot 具体类型

### GodotPackedSceneNodeFactory

- **Create**：`ResourceLoader.Load<PackedScene>(resourceId)` → `Instantiate<Node>()` → `parent.AddChild(node)` → 返回 GodotNodeHandle
- resourceId 通过 `SndMappings.ResolveSceneAlias` 解析（支持别名），因此可以是原始 `res://` 路径或别名

### GodotNodeHandle

- **Free()** → `_node.Free()`（Godot 即时释放）
- **SetVisible(bool)** → 根据节点类型（`CanvasItem` 或 `Node3D`）设置对应的 Visible 属性

## 设计决策

### 为什么 GodotSndEntity 使用延迟创建 Core Entity

Core `SndEntity` 需要 `INodeFactory` 注入，而 `INodeFactory` 需要 GodotSndEntity 自身作为父节点。构造顺序问题——GodotSndEntity 先创建，再以自身为参数构造 `INodeFactory`，然后通过 factory 创建 Core Entity。延迟创建解决这个循环依赖。

### 为什么 StableName 独立于 Godot Node.Name

Godot 场景树中如果存在同名节点，Godot 会自动在 Name 后追加 `@2`、`@3` 等后缀。SND 的实体查找依赖稳定名称，不能使用被 Godot 篡改的 Name。`StableName` 在 Spawn/Load 时设置后不再被 Godot 影响。

### 为什么 LoadFromMetaList 使用回滚机制

如果加载 100 个实体时第 50 个失败，前 49 个已创建的实体处于不完整状态（可能已触发 AfterLoad 钩子但场景不完整）。回滚全部释放防止残留损坏的实体污染后续操作。

### 为什么 QuitFromManager 必须先移出列表再释放

如果在 `_entities` 遍历中直接释放节点，Godot 的节点树变化可能导致后续迭代跳过实体或重复处理。先移除，后释放，保证列表迭代安全。

---
[↑ 回到 Origo.GodotAdapter](../README.md)
