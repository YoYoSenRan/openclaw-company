# 03. 系统架构设计

## Scope
定义系统的整体分层、组件职责、数据流、实时链路、部署拓扑与可行性结论。

## Non-goals
- 不定义最终云原生生产部署细节（K8s、Service Mesh）。
- 不覆盖所有安全合规模块实现细节。

## Out-of-scope
- 本文不写具体代码。
- 本文不替代 API 字段级协议（见 `05-event-protocol.md`）。

## 架构目标
1. 实时：状态推送低延迟、可持续同步。
2. 可控：所有关键操作可审计、可回放。
3. 可扩展：可从 20 Agent 扩展到 200 Agent（分片/分房间）。
4. 可维护：前端和 OpenClaw 协议解耦，降低变更冲击。

## 逻辑分层

### L1. Client（Web App）
- 技术：React + PixiJS
- 职责：场景渲染、交互输入、局部状态同步、演出驱动。

### L2. BFF/Realtime Hub（本项目中间层）
- 技术：Node.js + Fastify + @fastify/websocket
- 职责：
  1. 对前端提供统一实时事件流。
  2. 连接 OpenClaw Gateway 并做协议映射。
  3. 聚合事件、计算衍生状态（热力图、风险分）。
  4. 执行权限校验、节流、幂等控制。

### L3. OpenClaw Gateway
- 技术：OpenClaw 官方 Gateway（WS 控制平面）
- 职责：真实 Agent、会话、通道、工具调用、系统事件。

### L4. Data/Infra
- Redis（可选）：事件缓存、回放索引。
- SQLite/Postgres（可选）：任务与配置持久化。
- 对象存储（可选）：回放快照与统计归档。

## 关键链路

### 链路 A：状态上行（Gateway -> BFF -> Web）
1. Gateway 推送事件（agent/chat/presence/tick/health）。
2. BFF 标准化事件并写入短时缓存。
3. BFF 通过 WS 广播增量事件给前端。
4. 前端更新状态机并触发画面动画。

### 链路 B：命令下行（Web -> BFF -> Gateway）
1. 用户在 UI 下发命令（assign/pause/retry/handoff）。
2. BFF 做鉴权、参数校验、幂等检查。
3. BFF 调用 Gateway（WS 方法或 /tools/invoke）。
4. 回执与执行结果事件返回前端。

### 链路 C：回放查询
1. 前端请求回放时间窗。
2. BFF 从缓存/持久层拉取事件序列。
3. 返回可回放帧并附状态快照。

## 组件拓扑（单机开发）
1. Browser（React + Pixi）
2. BFF（Fastify + WS）
3. OpenClaw Gateway（local）
4. Redis（可选）

通信：
- Browser <-> BFF: WebSocket + HTTP
- BFF <-> OpenClaw: WebSocket（主）+ HTTP（补充）

## 状态一致性策略
1. 事件溯源优先：前端状态以事件流为准。
2. 定时快照校正：每 30~60 秒发送全量摘要，纠偏漂移。
3. 幂等命令：带 `requestId`，重复提交只生效一次。
4. 顺序控制：事件使用全局 `seq` + 可选 `agentSeq`（单 Agent 顺序）双重校验乱序。

### 缺口恢复流程（强制）
1. 客户端发现 `seq/agentSeq` 缺口后立即冻结增量应用（进入 `DegradedSync`）。
2. 发送 `state.resync` 请求并附当前已知 `lastSeq`。
3. BFF 返回“快照 + 缺失事件窗口”。
4. 客户端先加载快照，再按序重放缺失事件，最后恢复实时流。
5. 若 2 次恢复失败，触发全量重连并上报告警。

### 冲突解决策略
1. 客户端命令可做乐观 UI，但状态以服务器确认事件为准。
2. 若乐观状态与服务端结果冲突，以服务端事件覆盖并记录 `conflict_reconciled` 日志。
3. 同一实体冲突处理优先级：`server_event > optimistic_local > stale_local`。

## 错误与降级策略
1. Gateway 断连：前端进入只读模式，显示 reconnect 倒计时。
2. 高频告警风暴：BFF 聚合相同告警，降低 UI 抖动。
3. 局部异常：单 Agent 熔断，不影响全局渲染。
4. 资源压力：自动降级动画与特效，优先保状态可读性。

### Gateway 重连策略
1. 使用指数退避：`1s -> 2s -> 4s -> 8s -> 15s`（上限 15s）。
2. 连续失败超过 10 次后进入 `gateway_degraded`，仅保留本地回放与只读 UI。
3. 每次重连成功必须触发一次 `state.resync`。

### 慢消费保护
1. 每个客户端维护独立发送队列（上限默认 2000 条）。
2. 超过上限时执行“聚合替换策略”：位置类高频事件仅保留最新一条。
3. 队列持续积压超过 10s，降级该客户端为“低频推送模式”。

### 广播模型
1. 默认按 `workspaceId` 分组广播（类似房间）。
2. 单 Agent 详情事件仅发给订阅该 Agent 的客户端。
3. 全局公告/告警发到 workspace 级别广播组。

## 性能目标
- 50 Agent 并发事件下前端渲染保持 60 FPS（目标）。
- BFF 广播延迟 P95 < 120ms（同机房）。
- 指令回执 P95 < 300ms。

## 回放容量估算（基线）
- 估算模型：`agents * eventRatePerSec * durationSec * avgEventBytes`。
- 基线假设：`50 * 4 * 900 * 220B ≈ 39.6MB/15min`（未压缩）。
- 启用压缩与聚合后目标：15 分钟窗口控制在 `<15MB`。
- 若回放窗口扩大到 60 分钟，建议启用 Redis + 分段归档。

## Redis 启用时机（决策标准）
1. 满足以下任一条件即从内存缓存升级 Redis：
   - 并发客户端 > 30。
   - 回放窗口 > 15 分钟且稳定运行 > 8 小时。
   - BFF 重启后要求保留短时可回放数据。
2. 未达到条件时可先用内存缓存，降低复杂度。

## 安全边界
1. 前端不直接暴露 Gateway token。
2. 所有敏感调用经 BFF 转发与白名单校验。
3. 操作命令全量审计（操作人、对象、时间、结果）。
4. 支持会话级只读令牌（用于观战屏）。

## 技术路线可行性结论（已核验）

### 结论
路线可行，且与 OpenClaw 架构天然匹配。

### 依据（官方）
1. OpenClaw Gateway 为统一 WS 控制平面，首帧 `connect`，事件/请求/响应结构清晰。
2. OpenClaw 提供 `POST /tools/invoke`，可作为命令补充通道。
3. React 19.2、Vite 7 已发布并可用于现代前端工程。
4. PixiJS v8 与 `@pixi/react` v8 支持当前方案。
5. Fastify 官方 `@fastify/websocket` 可稳定提供 WS 路由能力。

### 关键兼容约束
1. Node 版本统一为 `>=22.12`：同时满足 OpenClaw（>=22）与 Vite 7 要求。
2. 若采用 `fastify-socket.io`，需注意其声明主要支持 Fastify 4.x；
   本项目默认不采用该插件，避免版本耦合。

## 模块划分建议

### `apps/web`
- 场景渲染、交互、状态管理、HUD、回放 UI。

### `apps/bff`
- WS 网关、协议映射、命令转发、审计日志、缓存聚合。

### `packages/protocol`
- 共享类型定义（事件、命令、错误码）。

### `packages/sim`
- 经营模拟计算（精力/压力/效率模型）。

### `packages/ui-kit`（可选）
- 统一 HUD 组件、状态图标、面板控件。

## 可观测性
- Metrics：连接数、事件吞吐、命令成功率、平均时延。
- Logging：结构化日志（requestId、agentId、eventType）。
- Tracing（可选）：关键链路端到端耗时。

## 验收检查
- 断网/重连后状态可自动恢复。
- 命令幂等可验证。
- 回放数据与实时结果一致。
- 50 Agent 压测下无明显卡顿或状态丢失。
