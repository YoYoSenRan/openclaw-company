---
name: openspec-ff-change
description: 快速前进 OpenSpec 工件创建。当用户想要快速创建实现所需的所有工件而无需逐步处理每个工件时使用。
license: MIT
compatibility: 需要 openspec CLI。
metadata:
  author: openspec
  version: "1.0"
  generatedBy: "1.1.1"
---

快速前进工件创建 - 一次性生成开始实现所需的所有内容。

**输入**：用户的请求应包含变更名称（kebab-case 格式）或者用户想要构建的内容描述。

**步骤**

1. **如果没有明确的输入，询问想要构建什么**

   使用 **AskUserQuestion 工具**（开放式，无预设选项）询问：
   > "你想要处理什么变更？描述你想要构建或修复的内容。"

   根据他们的描述，派生一个 kebab-case 名称（例如 "add user authentication" → `add-user-auth`）。

   **重要**：在不了解用户想要构建什么之前，不要继续。

2. **创建变更目录**
   ```bash
   openspec new change "<name>"
   ```
   这将在 `openspec/changes/<name>/` 创建一个脚手架变更。

3. **获取工件构建顺序**
   ```bash
   openspec status --change "<name>" --json
   ```
   解析 JSON 以获取：
   - `applyRequires`：实现前需要的工件 ID 数组（例如 `["tasks"]`）
   - `artifacts`：所有工件列表及其状态和依赖关系

4. **按顺序创建工件直到可以应用**

   使用 **TodoWrite 工具** 跟踪工件的进度。

   按依赖顺序循环处理工件（没有待处理依赖的工件优先）：

   a. **对于每个 `ready`（依赖已满足）的工件**：
      - 获取指令：
        ```bash
        openspec instructions <artifact-id> --change "<name>" --json
        ```
      - 指令 JSON 包括：
        - `context`：项目背景（对你的约束 - 不要包含在输出中）
        - `rules`：工件特定的规则（对你的约束 - 不要包含在输出中）
        - `template`：输出文件使用的结构
        - `instruction`：此工件类型的模式特定指导
        - `outputPath`：工件写入位置
        - `dependencies`：需要读取以获取上下文的已完成工件
      - 读取任何已完成的依赖文件以获取上下文
      - 使用 `template` 作为结构创建工件文件
      - 将 `context` 和 `rules` 作为约束应用 - 但不要将它们复制到文件中
      - 显示简短进度："✓ 已创建 <artifact-id>"

   b. **继续直到所有 `applyRequires` 工件完成**
      - 创建每个工件后，重新运行 `openspec status --change "<name>" --json`
      - 检查 `applyRequires` 中的每个工件 ID 在工件数组中是否有 `status: "done"`
      - 当所有 `applyRequires` 工件完成时停止

   c. **如果工件需要用户输入**（上下文不明确）：
      - 使用 **AskUserQuestion 工具** 澄清
      - 然后继续创建

5. **显示最终状态**
   ```bash
   openspec status --change "<name>"
   ```

**输出**

完成所有工件后，总结：
- 变更名称和位置
- 创建的工件列表及简要描述
- 准备就绪的内容："所有工件已创建！准备实现。"
- 提示："运行 `/opsx:apply` 或让我实现以开始处理任务。"

**工件创建指南**

- 遵循 `openspec instructions` 中每个工件类型的 `instruction` 字段
- 模式定义了每个工件应包含的内容 - 遵循它
- 在创建新工件之前读取依赖工件以获取上下文
- 使用 `template` 作为输出文件的结构 - 填充其各个部分
- **重要**：`context` 和 `rules` 是对你的约束，不是文件的内容
  - 不要将 `<context>`、`<rules>`、`<project_context>` 块复制到工件中
  - 这些指导你编写什么，但不应出现在输出中

**边界规则**
- 创建实现所需的所有工件（由模式的 `apply.requires` 定义）
- 在创建新工件之前始终读取依赖工件
- 如果上下文严重不明确，询问用户 - 但优先做出合理决策以保持动力
- 如果该名称的变更已存在，建议继续该变更
- 在继续下一个之前验证每个工件文件在写入后是否存在

