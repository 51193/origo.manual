# Random

> [↑ 回到 Origo.Core](../README.md) · [↔ Addons: FastNoiseLite](../Addons/FastNoiseLite/README.md)

## 概述

游戏随机数的统一入口。提供两组独立能力：基于 XorShift128+ 的伪随机数序列生成，以及基于 FastNoiseLite 的二维噪声图生成。两者均支持可复现：给定相同种子，输出相同序列/地图。

## 包含文件

| 文件 | 职责 |
|------|------|
| `RandomNumberGenerator.cs` | XorShift128+ 伪随机数生成器，状态由调用方显式维护 |
| `NoiseMapGenerator.cs` | Simplex + Worley 混合噪声图生成，行优先一维数组输出 |

## 实现详解

### RandomNumberGenerator

- **算法**：XorShift128+（高性能、周期 2^128-1）
- **状态模型**：返回 `(value, nextS0, nextS1)` 元组，调用方持有状态。无隐式全局状态
- **种子**：字符串输入 → FNV-1a 64位散列 → 状态对
- **提供 NextUInt64 / NextInt64 / NextInt32** 三种精度的随机数

### NoiseMapGenerator

- **算法**：OpenSimplex2 (70%) + Worley Cellular (30%) 混合
- **混合比**：`SimplexWeight=0.7f, WorleyWeight=0.3f`
- **输出**：行优先 `float[size*size]` 数组，值域 `[0, 1]`
- **归一化**：`(value + 1) * 0.5`，通过 `Math.Clamp` 确保安全
- **默认参数**：seed=1337, frequency=0.01

## 设计决策

### 为什么随机状态由调用方显式维护

全局随机状态在多实体、多会话、并行存档场景中难以追溯和复现。显式状态让每个实体/会话持有独立状态，存档时序列化状态，恢复后继续产出一致序列。

### 为什么 XorShift128+ 而非 System.Random

`System.Random` 不保证跨 .NET 版本的一致性，且无法序列化内部状态。XorShift128+ 仅 16 字节状态，可完整序列化/恢复，且跨平台行为一致。

### 为什么噪声比为 70/30

Simplex 提供平滑的宏观地形结构，Worley 提供微观的细胞状细节。70/30 的比例在视觉上产生"大结构 + 小细节"的效果，经过实验选定。

### 为什么不提供 3D 噪声

2D 噪声覆盖当前游戏需求（地形、高度图）。3D 噪声 API 与 2D 不同（需要 z 参数），且计算量大 1-2 个数量级。若无明确需求，不提前引入。

---
[↑ 回到 Origo.Core](../README.md)
