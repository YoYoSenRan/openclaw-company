# OpenClaw 可视化经营指挥项目计划（2026-02-14）

## 1. 目标与边界

### 目标
- 建立一套可执行的项目基线文档，覆盖产品、架构、协议、状态机、技术标准与研发流程。
- 确保后续开发围绕“模拟经营 + 指挥中心 + OpenClaw 实时连接”展开，不偏离方向。
- 给出技术可行性结论与替代路径，降低早期选型风险。

### 非目标
- 本次不实现业务代码。
- 本次不产出美术资源文件（角色图集、Spine 资源、地图素材）。
- 本次不搭建 CI/CD 实例，仅定义流程与标准。

## 2. 可行性核验（官方文档）
- [x] OpenClaw Gateway WS 协议与 HTTP 工具调用确认
- [x] React 19.2 / Vite 7 版本与 Node 版本约束确认
- [x] PixiJS v8 与 @pixi/react v8 适配确认
- [x] Zustand / XState / WebSocket 协议层 / Tiled JSON / EasyStar 可行性确认
- [x] 风险点识别（Fastify Socket.IO 插件版本支持边界、Spine 许可）

## 3. 文档产出清单
- [x] docs/README.md
- [x] docs/00-project-charter.md
- [x] docs/01-product-design.md
- [x] docs/02-visual-and-animation-spec.md
- [x] docs/03-system-architecture.md
- [x] docs/04-domain-model-and-state-machine.md
- [x] docs/05-event-protocol.md
- [x] docs/06-tech-stack-and-standards.md
- [x] docs/07-development-workflow.md
- [x] docs/08-roadmap-and-milestones.md
- [x] docs/09-risk-and-adr.md

## 4. 执行里程碑
- [x] Milestone A: 完成项目总纲、产品与视觉规范（00/01/02）
- [x] Milestone B: 完成系统架构、领域模型、状态机与协议（03/04/05）
- [x] Milestone C: 完成技术标准、开发流程、路线图与风险（06/07/08/09）
- [x] Milestone D: 文档一致性检查与交付说明

## 5. 交付标准（Definition of Done）
- 每份文档都包含 Scope / Non-goals / Out-of-scope。
- 架构、状态机、事件协议均提供可实现级别的字段与时序描述。
- 文档之间存在交叉引用，形成单一事实来源（SSOT）。
- 给出技术路线可行性结论和已知风险的规避策略。

## 6. 补充文档（第二轮完善）
- [x] docs/10-runtime-config-and-env.md
- [x] docs/11-openclaw-method-matrix.md
- [x] docs/12-test-matrix.md

## 7. Note
- 若后续发现 OpenClaw 协议字段变化，以 docs/05-event-protocol.md 的“版本对齐策略”为准进行增量修订。
