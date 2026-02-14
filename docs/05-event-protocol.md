# 05. 事件协议（Event Protocol）

## Scope
定义 Web 客户端与 BFF 的实时协议，以及 BFF 与 OpenClaw Gateway 的对接映射策略。

## Non-goals
- 不覆盖 OpenClaw 全部原生方法。
- 不定义二进制媒体传输协议。

## Out-of-scope
- 不定义移动端私有协议。

## 版本与兼容策略
- 协议名：`openclaw-company-realtime`
- 当前版本：`v1`
- 兼容规则：
  1. 仅新增字段时不破坏兼容（向后兼容）。
  2. 删除字段或变更语义必须升主版本。
  3. 客户端握手需声明 `supportedVersions`。

## 传输层
- Browser <-> BFF：WebSocket（JSON 文本帧）
- BFF <-> Gateway：WebSocket（OpenClaw 原生）+ HTTP `/tools/invoke`（可选）

## 通用消息包络

### Request Envelope
```json
{
  "kind": "req",
  "requestId": "req_01J...",
  "action": "agent.assign",
  "ts": 1760000000000,
  "payload": {}
}
```

### Response Envelope
```json
{
  "kind": "res",
  "requestId": "req_01J...",
  "ok": true,
  "ts": 1760000000123,
  "payload": {}
}
```

### Event Envelope
```json
{
  "kind": "event",
  "eventId": "evt_01J...",
  "eventType": "agent.status.changed",
  "seq": 1024,
  "agentSeq": 88,
  "ts": 1760000000456,
  "payload": {}
}
```

### Batch Envelope（批处理）
```json
{
  "kind": "batch",
  "batchId": "bat_01J...",
  "ts": 1760000000500,
  "events": [
    {
      "kind": "event",
      "eventId": "evt_01J_a",
      "eventType": "agent.position.updated",
      "seq": 1025,
      "agentSeq": 89,
      "ts": 1760000000490,
      "payload": {}
    }
  ]
}
```
- 批处理只允许包含 `event`，禁止混入 `req/res`。
- 单批建议上限：`<=200` 事件或 `<=256KB`。

## 握手协议

### client.hello（客户端 -> BFF）
```json
{
  "kind": "req",
  "requestId": "req_hello_001",
  "action": "client.hello",
  "payload": {
    "clientId": "web_admin_001",
    "supportedVersions": ["v1"],
    "authToken": "optional_view_token",
    "resumeFromSeq": 1000
  }
}
```

### server.hello（BFF -> 客户端）
```json
{
  "kind": "res",
  "requestId": "req_hello_001",
  "ok": true,
  "payload": {
    "protocolVersion": "v1",
    "serverTime": 1760000001000,
    "sessionId": "sess_abc",
    "heartbeatMs": 15000
  }
}
```

### 重连恢复流程（`resumeFromSeq`）
1. 客户端握手携带 `resumeFromSeq`。
2. BFF 判断是否可从缓存补齐缺失区间：
   - 可补齐：先下发 `replay.frame` 缺失段，再恢复实时流。
   - 不可补齐：先下发 `state.snapshot`，再切换实时流。
3. 客户端必须在恢复完成前暂停本地乐观状态提交。

`state.snapshot` 示例：
```json
{
  "kind": "event",
  "eventType": "state.snapshot",
  "seq": 5000,
  "ts": 1760000002000,
  "payload": {
    "snapshotVersion": 12,
    "agents": [],
    "tasks": [],
    "activeAlerts": []
  }
}
```

## 心跳协议
- 客户端每 `heartbeatMs` 发送 `client.ping`。
- BFF 返回 `server.pong`。
- 连续 3 次超时视为断连。

## 超时与限流基线
- 命令默认超时：`5000ms`（可按 action 覆写）。
- 握手超时：`3000ms`。
- 心跳周期：`15000ms`（默认）。
- 心跳失效判定：连续 3 个周期未收到 pong。
- 默认限流：单客户端 `60 req/min`，突发桶 `20`。
- 超限响应：`RATE_LIMITED` + `retryAfterMs`。

## 核心事件定义

### 1) `agent.status.changed`
用途：员工状态变化。

payload:
```json
{
  "agentId": "agent_dev_01",
  "from": "Thinking",
  "to": "Executing",
  "reason": "task_started",
  "taskId": "task_1001"
}
```

### 2) `agent.position.updated`
用途：角色移动。

payload:
```json
{
  "agentId": "agent_pm_01",
  "x": 42,
  "y": 16,
  "zoneId": "workspace_a",
  "speed": 1.25
}
```

### 3) `task.updated`
用途：任务状态或进度变化。

payload:
```json
{
  "taskId": "task_1001",
  "status": "InProgress",
  "progress": 45,
  "ownerAgentId": "agent_dev_01",
  "etaMinutes": 30
}
```

### 4) `dispatch.created`
用途：派单动作发生。

payload:
```json
{
  "orderId": "order_2001",
  "action": "assign",
  "operatorId": "operator_01",
  "targetAgentId": "agent_dev_01",
  "taskId": "task_1001",
  "channel": "phone"
}
```

### 5) `alert.raised`
用途：阻塞/错误告警。

payload:
```json
{
  "alertId": "alert_3001",
  "level": "warning",
  "code": "DEPENDENCY_BLOCKED",
  "agentId": "agent_dev_01",
  "taskId": "task_1001",
  "message": "Waiting for API response"
}
```

### 6) `replay.frame`
用途：回放帧。

payload:
```json
{
  "frameSeq": 445,
  "frameTs": 1760000005000,
  "events": [],
  "snapshotRef": "snapshot_10"
}
```

### 7) `agent.presence.updated`
用途：在线状态与心跳更新。

payload:
```json
{
  "agentId": "agent_dev_01",
  "online": true,
  "lastSeenAt": 1760000005000,
  "source": "gateway.presence"
}
```

### 8) `task.message.appended`
用途：任务上下文消息追加（例如来自 chat/event 的任务语义消息）。

payload:
```json
{
  "taskId": "task_1001",
  "messageId": "msg_9001",
  "role": "assistant",
  "content": "API dependency is ready.",
  "ts": 1760000005500
}
```

## 客户端命令定义

### `agent.assign`
```json
{
  "kind": "req",
  "requestId": "req_assign_001",
  "action": "agent.assign",
  "payload": {
    "agentId": "agent_dev_01",
    "taskId": "task_1001",
    "dispatchChannel": "face_to_face"
  }
}
```

### `agent.pause`
```json
{
  "kind": "req",
  "requestId": "req_pause_001",
  "action": "agent.pause",
  "payload": {
    "agentId": "agent_dev_01",
    "reason": "priority_preemption"
  }
}
```

### `agent.resume`
```json
{
  "kind": "req",
  "requestId": "req_resume_001",
  "action": "agent.resume",
  "payload": {
    "agentId": "agent_dev_01"
  }
}
```

### `agent.retry`
```json
{
  "kind": "req",
  "requestId": "req_retry_001",
  "action": "agent.retry",
  "payload": {
    "agentId": "agent_dev_01",
    "taskId": "task_1001",
    "reason": "manual_retry"
  }
}
```

### `agent.handoff`
```json
{
  "kind": "req",
  "requestId": "req_handoff_001",
  "action": "agent.handoff",
  "payload": {
    "fromAgentId": "agent_dev_01",
    "toAgentId": "agent_dev_02",
    "taskId": "task_1001",
    "reason": "specialized_skill_required"
  }
}
```

### `task.create`
```json
{
  "kind": "req",
  "requestId": "req_task_create_001",
  "action": "task.create",
  "payload": {
    "title": "Integrate payment API",
    "type": "feature",
    "priority": 2,
    "estimateMinutes": 180,
    "ownerAgentId": "agent_dev_01"
  }
}
```

### `replay.query`
```json
{
  "kind": "req",
  "requestId": "req_replay_001",
  "action": "replay.query",
  "payload": {
    "fromTs": 1760000000000,
    "toTs": 1760000900000,
    "limit": 2000
  }
}
```

## 响应与回执语义
- `ok=true`：命令已被接受并进入执行流程（不等于业务已完成）。
- 业务最终结果通过后续 event 回传。
- 若命令被拒绝，返回 `ok=false` + 错误码。
- 若客户端在超时窗口内未收到回执，允许按相同 `requestId` 重试。

## 错误码规范

### 通用
- `UNAUTHORIZED`
- `FORBIDDEN`
- `INVALID_PAYLOAD`
- `RATE_LIMITED`
- `INTERNAL_ERROR`

### 业务
- `AGENT_NOT_FOUND`
- `TASK_NOT_FOUND`
- `INVALID_TRANSITION`
- `AGENT_BUSY`
- `DEPENDENCY_BLOCKED`
- `GATEWAY_UNAVAILABLE`

错误响应示例：
```json
{
  "kind": "res",
  "requestId": "req_assign_001",
  "ok": false,
  "error": {
    "code": "INVALID_TRANSITION",
    "message": "Agent cannot transition from Offline to Executing"
  }
}
```

## 幂等与重试
1. 每个命令必须携带唯一 `requestId`。
2. BFF 维护短时去重缓存（建议 2~5 分钟）。
3. 网络超时后客户端可重发同 `requestId`。
4. BFF 返回同一结果，不重复执行副作用。

## 顺序与一致性
1. 事件含全局 `seq`（跨全局顺序）。
2. 与单 Agent 强相关的事件可附加 `agentSeq`（单 Agent 顺序）。
3. 客户端检测到 `seq` 或 `agentSeq` 缺口时触发 `state.resync`。
4. BFF 返回最新快照 + 缺失事件重放。

## Gateway 映射策略

### 上行映射（Gateway -> BFF）
- Gateway `agent` 相关事件 -> `agent.status.changed`
- Gateway `presence` 相关事件 -> `agent.presence.updated`
- Gateway `chat` 相关事件 -> `task.message.appended`（按场景映射）

#### 字段映射表（v1 基线）
| Gateway 侧字段 | 项目侧字段 | 说明 |
|---|---|---|
| `event` | `eventType` | 事件域名称映射 |
| `seq` | `seq` | 全局顺序号直传 |
| `payload.deviceId` | `agentId` | 通过映射表将设备/会话归一到 agentId |
| `payload.ts` | `ts` | 时间戳统一毫秒 |
| `payload.status` | `to` / `status` | 依据事件类型映射到状态字段 |
| `payload.message` | `content` / `message` | 按任务消息或告警消息场景映射 |

### 下行映射（BFF -> Gateway）
- 指令优先通过 Gateway WS 方法调用。
- 必要时通过 `POST /tools/invoke` 补充执行。
- 对应 OpenClaw 方法名与参数以目标版本网关 schema 为准（实现前需做一次契约校验）。

## 安全策略
1. 浏览器不持有 Gateway 主凭据。
2. BFF 鉴权后再执行任何控制命令。
3. 高风险动作可要求二次确认（可扩展）。

## 监控指标
- 每秒事件吞吐（events/sec）
- 命令成功率
- 指令端到端时延
- 丢包/重同步频率

## 协议演进流程
1. 提交协议变更提案。
2. 更新本文件版本与差异说明。
3. 同步更新 `packages/protocol` 类型。
4. 完成兼容性测试再发布。
