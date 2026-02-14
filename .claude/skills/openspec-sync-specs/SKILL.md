---
name: openspec-sync-specs
description: 将增量规格从变更同步到主规格。当用户想要用增量规格中的更改更新主规格而无需归档变更时使用。
license: MIT
compatibility: 需要 openspec CLI。
metadata:
  author: openspec
  version: "1.0"
  generatedBy: "1.1.1"
---

将增量规格从变更同步到主规格。

这是一个**代理驱动**的操作 - 你将读取增量规格并直接编辑主规格以应用更改。这允许智能合并（例如，添加场景而无需复制整个需求）。

**输入**：可选择指定一个变更名称。如果省略，检查是否可以从对话上下文中推断。如果模糊或不明确，你必须提示用户选择可用的变更。

**步骤**

1. **如果未提供变更名称，提示选择**

   运行 `openspec list --json` 获取可用变更。使用 **AskUserQuestion 工具** 让用户选择。

   显示具有增量规格（在 `specs/` 目录下）的变更。

   **重要**：不要猜测或自动选择变更。始终让用户选择。

2. **查找增量规格**

   在 `openspec/changes/<name>/specs/*/spec.md` 中查找增量规格文件。

   每个增量规格文件包含如下部分：
   - `## ADDED Requirements` - 要添加的新需求
   - `## MODIFIED Requirements` - 对现有需求的更改
   - `## REMOVED Requirements` - 要删除的需求
   - `## RENAMED Requirements` - 要重命名的需求（FROM:/TO: 格式）

   如果未找到增量规格，通知用户并停止。

3. **对于每个增量规格，将更改应用到主规格**

   对于 `openspec/changes/<name>/specs/<capability>/spec.md` 下的每个具有增量规格的能力：

   a. **读取增量规格** 以了解预期的更改

   b. **读取主规格** 在 `openspec/specs/<capability>/spec.md`（可能尚不存在）

   c. **智能地应用更改**：

      **ADDED Requirements：**
      - 如果需求在主规格中不存在 → 添加它
      - 如果需求已存在 → 更新它以匹配（视为隐式 MODIFIED）

      **MODIFIED Requirements：**
      - 在主规格中找到需求
      - 应用更改 - 这可以是：
        - 添加新场景（不需要复制现有场景）
        - 修改现有场景
        - 更改需求描述
      - 保留增量中未提及的场景/内容

      **REMOVED Requirements：**
      - 从主规格中删除整个需求块

      **RENAMED Requirements：**
      - 找到 FROM 需求，重命名为 TO

   d. **创建新的主规格** 如果能力尚不存在：
      - 创建 `openspec/specs/<capability>/spec.md`
      - 添加 Purpose 部分（可以简短，标记为 TBD）
      - 添加 Requirements 部分及 ADDED 需求

4. **显示摘要**

   应用所有更改后，总结：
   - 更新了哪些能力
   - 做了哪些更改（需求添加/修改/删除/重命名）

**增量规格格式参考**

```markdown
## ADDED Requirements

### Requirement: New Feature
系统应该做一些新的事情。

#### Scenario: Basic case
- **WHEN** 用户执行 X
- **THEN** 系统执行 Y

## MODIFIED Requirements

### Requirement: Existing Feature
#### Scenario: New scenario to add
- **WHEN** 用户执行 A
- **THEN** 系统执行 B

## REMOVED Requirements

### Requirement: Deprecated Feature

## RENAMED Requirements

- FROM: `### Requirement: Old Name`
- TO: `### Requirement: New Name`
```

**关键原则：智能合并**

与程序化合并不同，你可以应用**部分更新**：
- 要添加场景，只需在 MODIFIED 下包含该场景 - 不需要复制现有场景
- 增量代表*意图*，而不是整体替换
- 使用你的判断明智地合并更改

**成功时的输出**

```
## 规格已同步：<change-name>

已更新的主规格：

**<capability-1>**：
- 已添加需求："New Feature"
- 已修改需求："Existing Feature"（添加了 1 个场景）

**<capability-2>**：
- 已创建新规格文件
- 已添加需求："Another Feature"

主规格现已更新。变更仍处于活动状态 - 实现完成后归档。
```

**边界规则**
- 在进行更改之前读取增量和主规格
- 保留增量中未提及的现有内容
- 如果某些内容不清楚，请求澄清
- 随时显示你正在更改的内容
- 操作应该是幂等的 - 运行两次应该给出相同的结果

