# 12. 测试矩阵与验收基线

## Scope
定义各阶段必须执行的测试矩阵、覆盖目标与通过标准，确保“可观察+可操控+可回放”主路径稳定。

## Non-goals
- 不替代具体测试代码实现。
- 不定义外部渠道（WhatsApp/Telegram 等）全量测试。

## Out-of-scope
- 不包含性能压测平台搭建细节。

## 测试分层
1. 单元测试（状态机、协议、算法）。
2. 集成测试（Web/BFF/Gateway Mock）。
3. 场景测试（派单、阻塞、回放）。
4. 非功能测试（性能、稳定性、恢复）。

## 优先级定义
- P0：主链路不可用风险。
- P1：核心体验明显退化。
- P2：边界行为异常。

## 核心测试矩阵

| ID | 类别 | 用例 | 优先级 | 阶段 | 通过标准 |
|---|---|---|---|---|---|
| T-001 | 连接 | Web 与 BFF 建连握手 | P0 | Phase 0 | `client.hello`/`server.hello` 成功，心跳稳定。 |
| T-002 | 连接 | BFF 与 Gateway 建连 | P0 | Phase 0 | Gateway 鉴权通过，事件可接收。 |
| T-003 | 协议 | `requestId` 幂等去重 | P0 | Phase 1 | 重发不产生重复副作用。 |
| T-004 | 协议 | `seq/agentSeq` 缺口重同步 | P0 | Phase 1 | 自动触发 `state.resync`，状态恢复一致。 |
| T-005 | 渲染 | 20/50 Agent 同屏渲染 | P1 | Phase 1 | 1080p 下可交互，无明显卡顿。 |
| T-006 | 交互 | 电话派单流程 | P0 | Phase 2 | 状态流转正确，回执完整。 |
| T-007 | 交互 | 面交派单流程 | P0 | Phase 2 | 寻路到位、动画与状态一致。 |
| T-008 | 交互 | 批量 pause/resume | P1 | Phase 2 | 批量操作成功率 >= 99%。 |
| T-009 | 故障 | Blocked -> Recover | P0 | Phase 2 | 告警、恢复、重试链路完整。 |
| T-010 | 经营 | energy/focus/stress/skill 对产出的影响 | P1 | Phase 3 | 数值变化可解释且稳定。 |
| T-011 | 回放 | 15 分钟时间窗回放 | P0 | Phase 4 | 回放结果与实时记录一致。 |
| T-012 | 恢复 | Gateway 短时断线重连 | P0 | Phase 4 | 自动重连，最多丢失可恢复窗口内事件。 |
| T-013 | 安全 | 前端无 Gateway 主 token 泄露 | P0 | 全阶段 | 构建产物与网络请求无主密钥。 |
| T-014 | 稳定性 | 连续运行 8 小时 | P1 | Phase 4 | 无致命崩溃，核心功能可用。 |

## 性能基线
1. 指令回执 P95 < 300ms（局域网）。
2. BFF 广播延迟 P95 < 120ms。
3. 50 Agent 场景目标 60 FPS（允许短时波动）。
4. 重连恢复时间 < 5s（目标）。

## 自动化测试建议目录
```text
tests/
  unit/
    state-machine/
    protocol/
    simulation/
  integration/
    realtime/
    gateway-mock/
  scenario/
    dispatch/
    blocking/
    replay/
  non-functional/
    perf/
    soak/
```

## 网关 Mock 约束
1. Mock 需模拟 `connect/req/res/event` 基本帧。
2. 支持注入乱序、丢包、延迟、断连场景。
3. 对关键事件域（agent/presence/chat/tick/health）可控回放。

## 每阶段最小通过门槛

### Phase 0
- T-001, T-002 通过。

### Phase 1
- T-003, T-004, T-005 通过。

### Phase 2
- T-006, T-007, T-008, T-009 通过。

### Phase 3
- T-010 通过。

### Phase 4
- T-011, T-012, T-014 通过。

## 缺陷分级与放行规则
1. 存在 P0 未关闭：禁止发布。
2. P1 可带条件放行，但需明确回退方案。
3. P2 进入后续迭代处理。

## 测试报告模板
- 版本：
- 测试时间：
- 测试范围：
- 通过率：
- 失败清单：
- 风险说明：
- 放行建议：

## 临时文件清理要求
- 测试后必须清理 `.tmp/.log/test.db`。
- 若需保留，加入 `.gitignore` 并说明原因。

## 验收结论输出标准
1. 给出“可上线/不可上线”明确结论。
2. 给出阻塞项（若存在）。
3. 给出下一步修复建议与责任归属。
