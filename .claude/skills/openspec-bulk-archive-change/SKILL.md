---
name: openspec-bulk-archive-change
description: 一次性归档多个已完成的变更。当归档多个并行变更时使用。
license: MIT
compatibility: 需要 openspec CLI。
metadata:
  author: openspec
  version: "1.0"
  generatedBy: "1.1.1"
---

在单个操作中归档多个已完成的变更。

此技能允许你批量归档变更，通过检查代码库来确定实际实现了什么，从而智能地处理规格冲突。

**输入**：无需输入（会提示选择）

**步骤**

1. **获取活动变更**

   运行 `openspec list --json` 获取所有活动变更。

   如果没有活动变更，通知用户并停止。

2. **提示选择变更**

   使用 **AskUserQuestion 工具** 的多选功能让用户选择变更：
   - 显示每个变更及其模式
   - 包含"所有变更"选项
   - 允许任意数量的选择（1+ 即可，2+ 是典型用例）

   **重要**：不要自动选择。始终让用户选择。

3. **批量验证 - 收集所有选定变更的状态**

   对于每个选定的变更，收集：

   a. **工件状态** - 运行 `openspec status --change "<name>" --json`
      - 解析 `schemaName` 和 `artifacts` 列表
      - 记录哪些工件是 `done` 还是其他状态

   b. **任务完成情况** - 读取 `openspec/changes/<name>/tasks.md`
      - 统计 `- [ ]`（未完成）和 `- [x]`（已完成）
      - 如果不存在任务文件，记为"无任务"

   c. **增量规格** - 检查 `openspec/changes/<name>/specs/` 目录
      - 列出存在哪些能力规格
      - 对于每个规格，提取需求名称（匹配 `### Requirement: <name>` 的行）

4. **检测规格冲突**

   构建 `capability -> [changes that touch it]` 的映射：

   ```
   auth -> [change-a, change-b]  <- 冲突（2+ 个变更）
   api  -> [change-c]            <- 正常（仅 1 个变更）
   ```

   当 2+ 个选定的变更对同一能力有增量规格时，就存在冲突。

5. **代理式解决冲突**

   **对于每个冲突**，调查代码库：

   a. **读取增量规格**，从每个冲突变更中了解每个变更声称要添加/修改的内容

   b. **搜索代码库**，寻找实现证据：
      - 查找实现每个增量规格需求的代码
      - 检查相关文件、函数或测试

   c. **确定解决方案**：
      - 如果只有一个变更实际实现 -> 同步该变更的规格
      - 如果两者都已实现 -> 按时间顺序应用（旧的先，新的覆盖）
      - 如果都未实现 -> 跳过规格同步，警告用户

   d. **记录解决方案**，针对每个冲突：
      - 要应用哪个变更的规格
      - 按什么顺序（如果两者都应用）
      - 理由（在代码库中发现了什么）

6. **显示合并状态表**

   显示汇总所有变更的表格：

   ```
   | 变更               | 工件 | 任务 | 规格   | 冲突 | 状态 |
   |-------------------|------|------|--------|------|------|
   | schema-management | 完成 | 5/5  | 2 增量 | 无   | 就绪 |
   | project-config    | 完成 | 3/3  | 1 增量 | 无   | 就绪 |
   | add-oauth         | 完成 | 4/4  | 1 增量 | auth (!) | 就绪*|
   | add-verify-skill  | 剩1  | 2/5  | 无     | 无   | 警告 |
   ```

   对于冲突，显示解决方案：
   ```
   * 冲突解决方案：
     - auth 规格：将先应用 add-oauth 然后 add-jwt（都已实现，按时间顺序）
   ```

   对于未完成的变更，显示警告：
   ```
   警告：
   - add-verify-skill：1 个未完成的工件，3 个未完成的任务
   ```

7. **确认批量操作**

   使用 **AskUserQuestion 工具** 进行单一确认：

   - "归档 N 个变更？"并根据状态提供选项
   - 选项可能包括：
     - "归档所有 N 个变更"
     - "仅归档 N 个就绪的变更（跳过未完成的）"
     - "取消"

   如果有未完成的变更，明确说明它们将带有警告被归档。

8. **为每个确认的变更执行归档**

   按确定的顺序处理变更（遵守冲突解决方案）：

   a. **同步规格**，如果存在增量规格：
      - 使用 openspec-sync-specs 方法（代理驱动的智能合并）
      - 对于冲突，按解决的顺序应用
      - 跟踪是否完成同步

   b. **执行归档**：
      ```bash
      mkdir -p openspec/changes/archive
      mv openspec/changes/<name> openspec/changes/archive/YYYY-MM-DD-<name>
      ```

   c. **跟踪结果**，针对每个变更：
      - 成功：成功归档
      - 失败：归档期间出错（记录错误）
      - 跳过：用户选择不归档（如适用）

9. **显示摘要**

   显示最终结果：

   ```
   ## 批量归档完成

   已归档 3 个变更：
   - schema-management-cli -> archive/2026-01-19-schema-management-cli/
   - project-config -> archive/2026-01-19-project-config/
   - add-oauth -> archive/2026-01-19-add-oauth/

   跳过 1 个变更：
   - add-verify-skill（用户选择不归档未完成的）

   规格同步摘要：
   - 4 个增量规格已同步到主规格
   - 1 个冲突已解决（auth：按时间顺序应用了两者）
   ```

   如果有任何失败：
   ```
   失败 1 个变更：
   - some-change：归档目录已存在
   ```

**冲突解决示例**

示例 1：只有一个已实现
```
冲突：specs/auth/spec.md 被 [add-oauth, add-jwt] 修改

检查 add-oauth：
- 增量添加"OAuth Provider Integration"需求
- 搜索代码库... 找到 src/auth/oauth.ts 实现 OAuth 流程

检查 add-jwt：
- 增量添加"JWT Token Handling"需求
- 搜索代码库... 未找到 JWT 实现

解决方案：仅 add-oauth 已实现。将仅同步 add-oauth 规格。
```

示例 2：两者都已实现
```
冲突：specs/api/spec.md 被 [add-rest-api, add-graphql] 修改

检查 add-rest-api（创建于 2026-01-10）：
- 增量添加"REST Endpoints"需求
- 搜索代码库... 找到 src/api/rest.ts

检查 add-graphql（创建于 2026-01-15）：
- 增量添加"GraphQL Schema"需求
- 搜索代码库... 找到 src/api/graphql.ts

解决方案：两者都已实现。将先应用 add-rest-api 规格，
然后应用 add-graphql 规格（按时间顺序，较新的优先）。
```

**成功时的输出**

```
## 批量归档完成

已归档 N 个变更：
- <change-1> -> archive/YYYY-MM-DD-<change-1>/
- <change-2> -> archive/YYYY-MM-DD-<change-2>/

规格同步摘要：
- N 个增量规格已同步到主规格
- 无冲突（或：M 个冲突已解决）
```

**部分成功时的输出**

```
## 批量归档完成（部分）

已归档 N 个变更：
- <change-1> -> archive/YYYY-MM-DD-<change-1>/

跳过 M 个变更：
- <change-2>（用户选择不归档未完成的）

失败 K 个变更：
- <change-3>：归档目录已存在
```

**没有变更时的输出**

```
## 没有要归档的变更

未找到活动变更。使用 `/opsx:new` 创建新变更。
```

**边界规则**
- 允许任意数量的变更（1+ 即可，2+ 是典型用例）
- 始终提示选择，从不自动选择
- 及早检测规格冲突并通过检查代码库解决
- 当两个变更都已实现时，按时间顺序应用规格
- 仅当实现缺失时跳过规格同步（警告用户）
- 确认前显示每个变更的清晰状态
- 对整个批次使用单一确认
- 跟踪并报告所有结果（成功/跳过/失败）
- 移动到归档时保留 .openspec.yaml
- 归档目录目标使用当前日期：YYYY-MM-DD-<name>
- 如果归档目标存在，该变更失败但继续处理其他变更

