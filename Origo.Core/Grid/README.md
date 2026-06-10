# Grid

> [↑ 回到 Origo.Core](../README.md)

## 概述

通用网格坐标系工具。提供网格坐标（整数索引）与世界坐标（浮点，以世界原点为中心）的双向转换能力。任何基于方形网格的游戏（如战棋、Roguelike、沙盒）都可使用。

## 包含文件

| 文件 | 职责 |
|------|------|
| `GridCoordinateSystem.cs` | GridToWorld / WorldToGrid 坐标转换，支持可变网格大小和单元格尺寸 |

## 实现详解

### GridCoordinateSystem

全部为静态方法，无状态：

- **GridToWorld(int gridCoord, float cellSize, int gridSize)**：将网格整型坐标转为世界中心坐标（单元格正中）。单轴转换，调用方分别传入 X 和 Z
- **WorldToGrid(float worldCoord, float cellSize, int gridSize, out bool outOfBounds)**：将世界坐标转换为网格整型索引（向下取整），通过 outOfBounds 报告越界

公式（单轴）：`worldCoord = gridCoord * cellSize - (gridSize * cellSize) / 2 + cellSize * 0.5f`，即网格原点在世界的-bounds/2偏移，每个单元格在其尺寸的中心。

## 设计决策

### 为什么拆为单轴而非双轴接口

单轴转换（`GridToWorld(int, float, int)`）比双轴（`GridToWorld(int, int, int, float, out float, out float)`）更灵活：不同轴的cellSize可以不同（如非正方形网格），且不需要out参数，返回值为单一float可直接参与Godot Vector3构造。

### 为什么不放在Snd命名空间

坐标转换是通用几何工具，不依赖SND实体模型。独立命名空间使其可用于任何游戏逻辑（包括非Snd系统）。

---

[↑ 回到 Origo.Core](../README.md)
