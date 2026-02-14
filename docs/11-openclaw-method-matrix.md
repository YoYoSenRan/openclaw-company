# 11. OpenClaw 方法映射矩阵

## Scope
定义本项目命令/事件与 OpenClaw Gateway 接口的映射关系，作为联调与实现约束。

## Non-goals
- 不替代 OpenClaw 官方协议文档。
- 不承诺覆盖 Gateway 全量方法。

## Out-of-scope
- 不包含移动端 node 专用命令的完整矩阵。

## 使用说明
1. 本矩阵优先记录“已由官方文档明确的方法”。
2. 未确认的方法一律标记 `TBD`，禁止在实现中猜测。
3. 每次升级 OpenClaw 版本后必须复核本矩阵。

## 字段说明
- `Project Action`：项目内动作名（见 `05-event-protocol.md`）。
- `Gateway Method/Event`：OpenClaw 侧方法或事件。
- `Direction`：`C->BFF->GW`（命令）或 `GW->BFF->C`（事件）。
- `Status`：`verified` / `proposed` / `tbd`。
- `Notes`：参数、限制、风险。

## 已验证映射（官方文档可证）

| Project Action | Gateway Method/Event | Direction | Status | Notes |
|---|---|---|---|---|
| `client.connect` | `connect` | C->BFF->GW | verified | Gateway 首帧必须 connect；需携带 role/scope/auth。 |
| `chat.history.query` | `chat.history` | C->BFF->GW | verified | WebChat 文档明确使用。 |
| `chat.message.send` | `chat.send` | C->BFF->GW | verified | WebChat 文档明确使用。 |
| `chat.message.inject` | `chat.inject` | C->BFF->GW | verified | 追加 assistant note，不触发 agent run。 |
| `chat.abort` | `chat.abort` | C->BFF->GW | verified | macOS WebChat 文档出现。 |
| `presence.sync` | `presence` (event domain) | GW->BFF->C | verified | Architecture/WebChat 提及 presence 事件。 |
| `tick.sync` | `tick` (event domain) | GW->BFF->C | verified | Architecture/WebChat 提及 tick 事件。 |
| `health.sync` | `health` (method/event domain) | 双向 | verified | Architecture/WebChat 提及 health。 |
| `approval.resolve` | `exec.approval.resolve` | C->BFF->GW | verified | 需 `operator.approvals` scope。 |
| `skills.catalog.sync` | `skills.bins` | C->BFF->GW | verified | node helper method。 |
| `device.token.rotate` | `device.token.rotate` | C->BFF->GW | verified | 需 `operator.pairing` scope。 |
| `device.token.revoke` | `device.token.revoke` | C->BFF->GW | verified | 需 `operator.pairing` scope。 |
| `tool.invoke` | `POST /tools/invoke` | C->BFF->GW | verified | HTTP 补充通道，受 auth + policy 控制。 |

## 项目调度动作映射（待契约确认）

| Project Action | Gateway Method/Event | Direction | Status | Notes |
|---|---|---|---|---|
| `agent.assign` | `agent` (TBD action payload) | C->BFF->GW | proposed | Architecture 提及 `agent` 请求域，需以 schema 确认参数。 |
| `agent.pause` | `agent` (TBD action payload) | C->BFF->GW | proposed | 同上。 |
| `agent.resume` | `agent` (TBD action payload) | C->BFF->GW | proposed | 同上。 |
| `agent.retry` | `agent` (TBD action payload) | C->BFF->GW | proposed | 同上。 |
| `agent.handoff` | `agent` (TBD action payload) | C->BFF->GW | proposed | 同上。 |
| `task.create` | `TBD` | C->BFF->GW | tbd | 可能由 BFF 内部任务域处理，不直接调用 GW。 |
| `replay.query` | `TBD` | C->BFF->GW | tbd | 建议主要由 BFF 自有存储实现。 |

## 事件域映射建议

| Gateway 事件域 | 项目事件 | Status | Notes |
|---|---|---|---|
| `agent` | `agent.status.changed` | proposed | 需将网关 payload 标准化到项目事件结构。 |
| `presence` | `agent.presence.updated` | proposed | 需要补 `online/lastSeenAt` 派生逻辑。 |
| `chat` | `task.message.appended` | proposed | 仅在任务上下文启用映射。 |
| `tick` | `system.tick` | proposed | 用于 UI 心跳与时间轴推进。 |
| `health` | `system.health.updated` | proposed | 用于告警与降级判断。 |

## 契约校验流程（必须）
1. 获取当前 OpenClaw 版本对应 protocol schema。
2. 对 `proposed/tbd` 项逐一验证方法名与参数。
3. 通过后将状态改为 `verified` 并记录日期。
4. 同步更新 `05-event-protocol.md` 示例与 `packages/protocol` 类型。

## 变更记录模板
- 日期：
- OpenClaw 版本：
- 变更项：
- 影响范围：
- 回归结论：

## 当前状态（2026-02-14）
- `verified`：基础连接、chat、approval、skills、token、tools-invoke。
- `proposed/tbd`：agent 调度细节与任务域方法。

## 风险提示
- 在 `proposed/tbd` 未转为 `verified` 前，禁止直接编码硬写 Gateway 方法名。
- 必须先完成契约校验再进入联调开发。
