# 04. 领域模型与状态机

## Scope
定义核心实体、关系、状态机与状态迁移规则，作为前后端一致实现基线。

## Non-goals
- 不定义数据库物理表结构（仅逻辑模型）。
- 不定义所有统计报表模型。

## Out-of-scope
- 不覆盖权限系统完整 RBAC 模型。

## 核心实体

### AgentEmployee（员工/Agent）
- `agentId: string`
- `name: string`
- `role: 'pm' | 'dev' | 'qa' | 'design' | 'ops' | 'support'`
- `status: AgentStatus`
- `location: { x: number; y: number; zoneId: string }`
- `stats: { energy: number; focus: number; stress: number; skill: number }`
- `currentTaskId?: string`
- `queueTaskIds: string[]`
- `lastSeenAt: number`

### Task（任务）
- `taskId: string`
- `title: string`
- `type: 'feature' | 'bugfix' | 'incident' | 'review' | 'ops'`
- `priority: 1 | 2 | 3 | 4 | 5`
- `difficulty: 1..100`
- `estimateMinutes: number`
- `progress: 0..100`
- `ownerAgentId?: string`
- `status: TaskStatus`
- `dependencyTaskIds: string[]`
- `createdAt: number`
- `deadlineAt?: number`

### DispatchOrder（调度指令）
- `orderId: string`
- `requestId: string`
- `operatorId: string`
- `action: 'assign' | 'pause' | 'resume' | 'retry' | 'handoff' | 'cancel'`
- `targetAgentId: string`
- `taskId?: string`
- `payload?: Record<string, unknown>`
- `issuedAt: number`

### EventRecord（事件记录）
- `eventId: string`
- `seq: number`
- `eventType: string`
- `agentId?: string`
- `taskId?: string`
- `timestamp: number`
- `payload: object`

### Alert（告警）
- `alertId: string`
- `level: 'info' | 'warning' | 'error'`
- `code: string`
- `agentId?: string`
- `taskId?: string`
- `message: string`
- `createdAt: number`
- `resolvedAt?: number`

## 关系模型
1. 一个 Agent 同时最多处理一个 `currentTask`。
2. 一个 Task 同时最多有一个 owner。
3. 一个 Task 可依赖多个 Task。
4. 一个 DispatchOrder 只作用于一个目标 Agent。

## Agent 状态机

### 状态定义
- `Offline`：不可用。
- `Idle`：空闲待命。
- `Briefing`：接收任务说明（电话/面交/会议）。
- `Thinking`：分析任务方案。
- `Executing`：执行中。
- `Reviewing`：自检或互审。
- `Blocked`：依赖或资源阻塞。
- `Error`：执行异常。
- `Paused`：被人工暂停。

### 状态迁移（主路径）
1. `Offline -> Idle`（上线）
2. `Idle -> Briefing`（收到任务）
3. `Briefing -> Thinking`（任务接受）
4. `Thinking -> Executing`（开始执行）
5. `Executing -> Reviewing`（提交审查）
6. `Reviewing -> Idle`（任务完成）

### 异常迁移
1. `Executing -> Blocked`（依赖不可用）
2. `Blocked -> Executing`（依赖恢复）
3. `Executing -> Error`（执行失败）
4. `Error -> Thinking`（重试）
5. `* -> Paused`（人为暂停）
6. `Paused -> previous`（恢复到暂停前状态）

#### Paused 恢复规则
- 进入 `Paused` 时必须记录 `pausedFromState` 与 `pausedAt`。
- `Paused -> previous` 的目标状态取 `pausedFromState`。
- 若 `pausedFromState` 已无效（例如任务已取消），强制恢复到 `Idle` 并生成告警。

## Task 状态机
- `Backlog`
- `Assigned`
- `InProgress`
- `Blocked`
- `InReview`
- `Done`
- `Cancelled`

迁移规则：
1. `Backlog -> Assigned`：分配负责人。
2. `Assigned -> InProgress`：开始执行。
3. `InProgress -> Blocked`：遇到阻塞。
4. `Blocked -> InProgress`：阻塞解除。
5. `InProgress -> InReview`：进入审核。
6. `InReview -> Done`：通过。
7. `InReview -> InProgress`：驳回返工。
8. `* -> Cancelled`：人工取消。

## 经营数值驱动规则
1. 执行速度系数：
   `speedFactor = f(energy, focus, stress, skill)`
2. 错误概率系数：
   `errorRate = baseError + stressFactor - skillFactor`
3. 阻塞概率：
   与依赖复杂度、外部服务健康度相关。

建议默认参数：
- `energy` 每 10 分钟下降 1~3。
- `stress` 在连续中断时上升。
- 休息行为可恢复 `energy` 与 `focus`。

### 取值边界（统一）
- `energy/focus/stress/skill` 范围固定为 `0..100`。
- 所有计算结果必须 clamp 到 `0..100`。

### 默认公式（v1）
- `speedFactor = clamp(0.4, 1.6, 0.6 + 0.004*energy + 0.003*focus - 0.003*stress + 0.002*skill)`
- `errorRate = clamp(0.01, 0.60, 0.08 + 0.0025*stress - 0.0018*skill - 0.001*focus)`
- `blockRate = clamp(0.01, 0.50, baseBlock + depComplexity*0.02 + externalHealthPenalty)`

### 数值恢复机制
- 休息状态每分钟：`energy +1.2`, `focus +0.8`, `stress -1.0`。
- 连续工作每分钟：`energy -0.6`, `focus -0.3`, `stress +0.5`。
- 加班模式额外惩罚：`stress +0.4/min`。

### 技能与任务难度关系
- 定义差值：`delta = skill - difficulty`。
- `delta >= 20`：速度 +15%，错误率 -25%。
- `-20 < delta < 20`：按默认公式计算。
- `delta <= -20`：速度 -20%，错误率 +30%，阻塞概率 +15%。

## 动画触发映射（状态 -> 演出）
1. `Briefing`：电话/对话泡泡动画。
2. `Thinking`：灯泡或思考粒子。
3. `Executing`：键盘敲击循环。
4. `Blocked`：橙色告警 + 抖动。
5. `Error`：红色提示 + 短暂停顿。
6. `Reviewing`：文件传递/检查动作。

## 一致性约束
1. 同一 Agent 在同一时刻只能处于一个状态。
2. `currentTaskId` 必须与 Task.ownerAgentId 对应一致。
3. 事件 `seq` 不可回退。
4. `requestId` 必须全局唯一（短时间窗口）。

## 冲突处理规则
1. 多命令并发：按 `issuedAt` 与优先级判定。
2. 抢占命令：高优先级可中断低优先任务。
3. 无效命令：返回 `INVALID_TRANSITION`。

### 并发仲裁细则
1. 若 `issuedAt` 不同，时间早者先执行。
2. 若 `issuedAt` 相同，按动作优先级：`cancel > handoff > pause > assign > resume > retry`。
3. 若时间与优先级都相同，按 `requestId` 字典序稳定排序，确保结果可重现。

## 状态快照结构（用于回放与重连）
- `snapshotVersion`
- `timestamp`
- `agents[]`
- `tasks[]`
- `activeAlerts[]`
- `metrics`

## 测试建议
1. 状态迁移单测：覆盖主路径与异常路径。
2. 并发命令冲突测试：同 Agent 多请求场景。
3. 快照恢复测试：断连后状态重建。
4. 数值模型稳定性测试：长时间模拟漂移检查。
