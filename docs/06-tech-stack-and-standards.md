# 06. 技术栈与工程标准

## Scope
定义项目统一技术栈、版本基线、目录规范、编码规范与质量门禁。

## Non-goals
- 不强制规定每个内部函数命名细节。
- 不定义团队协作制度（工时、排班）。

## Out-of-scope
- 不包含完整 CI 脚本内容。

## 技术栈定版（v1）

### 前端
1. `React 19.2`
2. `TypeScript 5.x`
3. `Vite 7`
4. `PixiJS 8.x`
5. `@pixi/react 8.x`
6. `Zustand`
7. `XState v5`
8. `TanStack Query`（可选，用于 HTTP 查询缓存）

### 中间层（BFF）
1. `Node.js >= 22.12`
2. `Fastify 5.x`
3. `@fastify/websocket`
4. `ws`（作为上游 Gateway 客户端库）
5. `zod`（运行时输入校验）
6. `pino`（结构化日志）

### 数据与缓存（可选按阶段启用）
1. `Redis`：事件缓存、去重、回放索引。
2. `SQLite/Postgres`：任务与配置持久化。

### 美术与音频工具链
1. `Aseprite`：像素帧动画。
2. `Tiled`：地图编辑与导出。
3. `Spine`（可选）：关键角色骨骼动画。
4. `howler.js`：音效播放管理。

## 版本兼容基线
1. Node 统一 `>=22.12`。
2. 前后端统一使用 ESM。
3. 浏览器目标遵循 Vite 7 默认 baseline。
4. 禁止混用多个实时协议风格（默认只用 WS JSON 包络）。

## 前端版本降级策略（防守路径）
1. 当前默认：`React 19.2 + Vite 7 + PixiJS 8 + @pixi/react 8`。
2. 若 Phase 0 Spike 失败，优先降级为：`React 19 + Vite 6 + PixiJS 8`。
3. 若仍不稳定，再降级为：`React 18 + Vite 6 + PixiJS 7`。
4. 任一降级必须同步更新 `09-risk-and-adr.md` 与 `08-roadmap-and-milestones.md`。

## 依赖策略
1. 所有依赖必须锁定到明确主版本范围。
2. 新增依赖需说明用途、替代方案与退出成本。
3. 对协议关键路径依赖（WS、校验、日志）优先选成熟库。

## Monorepo 目录建议
```text
openclaw-company/
  apps/
    web/
    bff/
  packages/
    protocol/
    sim/
    ui-kit/
  docs/
  plans/
  tests/
```

## Monorepo 工具决策
1. 首期采用 `pnpm workspaces`（默认方案）。
2. 不在 Phase 0 引入 Turborepo/Nx，避免前期复杂度。
3. 当包数量 > 10 且 CI 构建瓶颈明显时，再评估 Turborepo 增量缓存。

## 测试框架决策
1. 单元/集成：`Vitest`（与 Vite 原生集成）。
2. E2E：`Playwright`（桌面 Chrome 优先，后续扩展多浏览器）。
3. 协议契约：`Vitest + schema fixtures`。

## 样式方案决策
1. HUD 与管理面板采用 `CSS Modules + CSS Variables`。
2. Pixi 场景视觉由纹理与着色逻辑控制，不依赖 CSS-in-JS。
3. 不引入 Tailwind 作为默认方案，避免与场景样式体系割裂。

## 协议代码生成策略
1. `packages/protocol` 作为唯一类型源（SSOT）。
2. 若 OpenClaw 上游 schema 变更，先更新映射定义再生成本地类型。
3. 生成物仅限类型与校验器，禁止覆盖业务逻辑层代码。

## IDE 与仓库基线
1. 推荐 IDE：VS Code。
2. 必装插件：ESLint、Prettier、TypeScript、EditorConfig。
3. 提供统一 `.editorconfig` 与 `.vscode/settings.json`（后续补仓库文件）。

## Git 分支策略
1. 采用 trunk-based（`main` + 短生命周期 `feature/*`）。
2. feature 分支建议 1~3 天内合并，避免长期漂移。
3. 所有合并必须通过 lint/type-check/核心测试门禁。

## 前端分层规范
1. `scene/`：Pixi 场景、实体渲染。
2. `hud/`：状态面板、右侧详情、指挥控件。
3. `state/`：Zustand store + XState machine。
4. `services/`：WS 客户端、API 客户端。
5. `features/`：按业务域拆分（dispatch、replay、alerts）。

## BFF 分层规范
1. `gateway/`：OpenClaw 连接与映射。
2. `realtime/`：客户端 WS 会话与广播。
3. `commands/`：命令处理器与幂等。
4. `simulation/`：经营数值计算。
5. `storage/`：缓存与持久化适配。

## 命名规范
- 类型：`PascalCase`
- 变量/函数：`camelCase`
- 常量：`UPPER_SNAKE_CASE`
- 事件名：`dot.case`（如 `agent.status.changed`）
- 文件名：`kebab-case`

## TypeScript 规范
1. `strict: true`
2. 禁止 `any`（仅允许极少数边界层注释说明）
3. 所有协议对象必须有显式类型定义。
4. 对外输入必须使用 zod 校验并返回结构化错误。

## 代码风格
- ESLint + Prettier 统一格式。
- 单函数建议不超过 80~120 行。
- 复杂逻辑允许短注释说明业务意图。

## 状态管理规范
1. UI 局部状态与业务状态分离。
2. Agent 生命周期逻辑统一收敛到 XState。
3. 全局派发、筛选、视图设置使用 Zustand。

## 实时与性能规范
1. 事件处理必须可批处理（batch）。
2. 高频位置更新可抽样渲染（例如 100ms）。
3. UI 只订阅必要状态，避免全树重渲染。
4. 大量文本优先 BitmapText 或缓存样式。

## 错误处理规范
1. 用户可见错误必须有可操作提示。
2. 系统错误必须带 `errorCode` 与 `requestId`。
3. 不可恢复错误必须自动触发降级模式。

## 日志与观测规范
1. 统一结构化日志字段：`ts`, `level`, `requestId`, `agentId`, `action`, `durationMs`。
2. 前端关键交互也记录埋点（命令结果、耗时）。
3. 关键指标采集：连接数、事件吞吐、失败率、重连次数。

## 测试标准
1. 单元测试：状态机、协议校验、命令处理。
2. 集成测试：BFF 与 Gateway Mock 联调。
3. E2E（阶段性）：派单、阻塞处理、回放。

## 发布门禁
1. lint 通过。
2. type-check 通过。
3. 核心测试通过。
4. 文档引用更新完成。

## 禁止事项
1. 禁止前端直接持有 Gateway 主 token。
2. 禁止绕过协议层直接写 UI 状态。
3. 禁止未评审引入重型依赖。
