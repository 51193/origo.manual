# Agent 强制工作流

> **任何对 Origo 框架源代码（`origo` 仓库）的变更，必须完成以下四步闭环。缺少任何一步视为未完成。**

---

## 四步闭环

| 步骤 | 目标仓库 | 说明 |
|------|----------|------|
| 1. 代码变更 | `origo` | 实现功能/修复/重构 |
| 2. Changelog 对齐 | `origo/CHANGELOG.md` | 将面向用户的显著变更写入 `[Unreleased]` 区块 |
| 3. 文档严格同步 | `origo.manual` | 镜像目录结构、接口列表、设计决策、使用文档 |
| 4. 测试文件补齐 | `origo` | 新增 public API 必须有行为测试，修复的 bug 必须有回归测试 |

**禁止只完成部分步骤。** 如果某步骤确实不适用（如纯内部重构不影响 public API），必须在提交消息中显式说明跳过原因。

---

## Changelog 编写规范

格式基于 [Keep a Changelog 1.1.0](https://keepachangelog.com/zh-CN/1.1.0/)，遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

### 文件位置

`origo/CHANGELOG.md`

### 结构

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [Unreleased]

### Added
### Changed
### Deprecated
### Removed
### Fixed
### Security

---

## [x.y.z] - YYYY-MM-DD
...
```

> **Nightly 包（如 `0.0.8-nightly.20260612`）不是正式发布版本，不产生独立的版本区块。** 所有变更（无论所属 nightly）均归入 `[Unreleased]`，仅无后缀的正式语义化版本号（`x.y.z`）才产生 `## [x.y.z] - YYYY-MM-DD` 区块。

### 变动分类

| 类别 | 含义 |
|------|------|
| `Added` | 新添加的功能 |
| `Changed` | 对现有功能的变更 |
| `Deprecated` | 已不建议使用、即将移除的功能 |
| `Removed` | 已经移除的功能 |
| `Fixed` | 对 bug 的修复 |
| `Security` | 对安全性的改进 |

### 关键约束

1. **基线是上一个正式发布版本的 tag。** 编写 changelog 时，必须对比上一个正式版本到当前状态的差异，提炼面向用户的显著变更。Nightly tag（如 `v0.0.8-nightly.20260612`）不作为基线。
2. **Nightly 不视为版本。** 带 `-nightly`、`-alpha`、`-preview` 等预发布后缀的包号是快照标识，不是语义化版本。这些包对应的变更一律留在 `[Unreleased]` 中。仅无后缀的正式语义化版本（如 `0.0.7`）才产生带日期标题的版本区块。Nightly 间存在的功能反复（引入又删除）应在日常迭代中清理，不体现在任何版本区块中。
3. **禁止记录版本内的来回变动。** 如果一个功能在本版本周期内引入又删除、或引入后又修复了自身引入的 bug，这些来回变动不应出现在 changelog 中。最终状态才是记录对象。
4. **禁止记录版本内自身引入问题的修复。** 例如：nightly-0608 引入了一个 bug，nightly-0610 修复了它——如果两者都在同一个正式版本周期内，则 changelog 中既不记录该 bug 的引入，也不记录其修复。
5. **面向用户撰写。** 每条记录描述行为变化对使用者的影响，而非内部实现细节。
6. **日常变更写入 `[Unreleased]`。** Nightly 每日构建，变更持续累积在 `[Unreleased]` 区块中。当积累足够更新发布正式版本时，将 `[Unreleased]` 内容移入带版本号和日期的新区块。

### 编写流程

1. 确定上一个正式版本 tag（如 `0.0.7`）
2. 对比该 tag 到当前 HEAD 的所有变更
3. 按分类归纳面向用户的显著变更
4. 过滤掉版本内来回变动（引入又撤销、引入又修复的噪声）
5. 将最终结果写入 `[Unreleased]` 对应分类下

### 发版流程

1. 确定新版本号（正式语义化版本，如 `0.1.0`，不可带 `-nightly` 等预发布后缀）
2. 将 `[Unreleased]` 中的内容移入新版本区块 `## [x.y.z] - YYYY-MM-DD`
3. 清空 `[Unreleased]`（保留空分类标题或删除空分类均可）
4. 更新 `Directory.Build.props` 中的 `<Version>`
5. 提交并打 tag（tag 名称 `vx.y.z`，与版本号对应）

---

## 文档同步规则

文档仓库 `origo.manual` 是源代码的结构镜像。以下情况必须同步更新文档：

| 源代码变更 | 文档操作 |
|------------|----------|
| 新增/删除/重命名目录 | 在 `origo.manual` 中镜像相同操作 |
| 新增 public 接口/方法 | 更新对应叶子 README 的接口列表 |
| 删除/重命名 public 接口 | 更新对应叶子 README，删除旧条目 |
| 设计决策变更 | 更新对应 README 的设计决策章节 |
| 新增配置键/命令 | 更新相关 README 和 usage 文档 |
| 模块间依赖关系变化 | 更新模块 README 的链接 |
| 新增使用场景/模式 | 补充 usage/ 下对应文档 |

### 无需同步的情况

- 纯内部实现细节变更（不影响公开 API 或设计意图）
- 代码重构（不改变模块职责和接口）
- 性能优化（不改变外部行为语义）

### 同步检查清单

完成代码变更后，逐项检查：

- [ ] 目录结构是否镜像（新增/删除/重命名）？
- [ ] 叶子 README 的接口/文件清单是否准确？
- [ ] 中间层 README 的子模块索引是否完整？
- [ ] 所有链接是否有效（无 404）？
- [ ] 设计决策章节是否反映当前设计意图？
- [ ] usage/ 文档是否覆盖新增的使用场景？

---

## 测试要求

| 变更类型 | 测试要求 |
|----------|----------|
| 新增 public API | 必须有对应的行为测试 |
| Bug 修复 | 必须有回归测试（先红后绿） |
| 行为变更 | 更新已有测试以反映新行为 |
| 重构 | 现有测试必须全部通过，无需新增 |

### 测试风格

- 参照同模块已有测试的命名、组织和断言风格
- 测试运行命令：`bash scripts/ci.sh`（在 `origo` 仓库根目录）
- 测试项目：`Origo.Core.Tests`、`Origo.GodotAdapter.Tests`、`Origo.ConsoleBridge.Tests`

---

## 文档编写规范

详见 [META.md](META.md)。核心要点：

- 自底向上结构（叶子→中间→模块根→项目根）
- 每个 README 必须有父链接（↑）和子模块链接
- 禁止孤立叶子，文档树严格连通
- 禁止演进标记（"新增"、"旧版"、"已废弃"等暗示版本历史的字样）
- 文档是现状快照，直接陈述当前职责和理由

---

## 仓库布局约定

两个仓库作为兄弟目录 checkout：

```
<workspace>/
├── origo/          # 源代码仓库
└── origo.manual/   # 文档仓库（本仓库）
```

| 仓库 | 相对路径 | 用途 |
|------|----------|------|
| 源代码 | `../origo` | 框架实现、测试、CHANGELOG |
| 文档 | `../origo.manual` | 结构镜像文档、使用指南、元指令 |
