---
name: "OPSX: New"
description: 使用实验性工件工作流开始新变更
category: Workflow
tags: [workflow, artifacts, experimental]
---

使用实验性工件驱动方法开始新变更。

**输入**：`/opsx:new` 后的参数是变更名称（kebab-case 格式），或者用户想要构建的内容描述。

**步骤**

1. **如果未提供输入，询问想要构建什么**

   使用 **AskUserQuestion 工具**（开放式，无预设选项）询问：
   > "你想要处理什么变更？描述你想要构建或修复的内容。"

   根据他们的描述，派生一个 kebab-case 名称（例如 "add user authentication" → `add-user-auth`）。

   **重要**：在不了解用户想要构建什么之前，不要继续。

2. **确定工作流模式**

   使用默认模式（省略 `--schema`），除非用户明确请求不同的工作流。

   **仅在用户提到以下情况时使用不同模式：**
   - 特定模式名称 → 使用 `--schema <name>`
   - "显示工作流"或"什么工作流"→ 运行 `openspec schemas --json` 并让他们选择

   **否则**：省略 `--schema` 以使用默认值。

3. **创建变更目录**
   ```bash
   openspec new change "<name>"
   ```
   仅当用户请求特定工作流时添加 `--schema <name>`。
   这将在 `openspec/changes/<name>/` 创建一个带有选定模式的脚手架变更。

4. **显示工件状态**
   ```bash
   openspec status --change "<name>"
   ```
   这显示需要创建哪些工件以及哪些已准备就绪（依赖已满足）。

5. **获取第一个工件的指令**
   第一个工件取决于模式。检查状态输出以找到第一个 `status: "ready"` 的工件。
   ```bash
   openspec instructions <first-artifact-id> --change "<name>"
   ```
   这输出创建第一个工件的模板和上下文。

6. **停止并等待用户指导**

**输出**

完成步骤后，总结：
- 变更名称和位置
- 正在使用的模式/工作流及其工件序列
- 当前状态（0/N 工件已完成）
- 第一个工件的模板
- 提示："准备创建第一个工件？运行 `/opsx:continue` 或直接描述这个变更是关于什么的，我会起草它。"

**边界规则**
- 尚不创建任何工件 - 只显示指令
- 不要超越显示第一个工件模板
- 如果名称无效（不是 kebab-case），请求有效名称
- 如果该名称的变更已存在，建议使用 `/opsx:continue` 代替
- 如果使用非默认工作流，传递 --schema

