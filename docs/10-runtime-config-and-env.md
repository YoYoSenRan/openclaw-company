# 10. 运行配置与环境变量

## Scope
定义本项目在本地开发、测试与预发布环境中的运行配置、环境变量规范与安全边界。

## Non-goals
- 不定义云平台的完整 IaC 模板。
- 不定义 OpenClaw 本体的全部配置项。

## Out-of-scope
- 不包含生产环境密钥托管系统实现细节。

## 配置分层
1. `apps/web`：前端运行与渲染配置。
2. `apps/bff`：协议、鉴权、缓存、日志配置。
3. `infra`（可选）：Redis/DB 等外部依赖配置。

## 环境文件约定
1. Web：`apps/web/.env.local`、`apps/web/.env.production`
2. BFF：`apps/bff/.env.local`、`apps/bff/.env.production`
3. 不允许提交真实密钥到仓库。
4. 仓库内提供 `.env.example` 作为模板。

## Web 环境变量

### 必需
- `VITE_BFF_WS_URL`
  - 示例：`ws://127.0.0.1:3001/realtime`
  - 说明：前端连接 BFF 的实时地址。

- `VITE_BFF_HTTP_URL`
  - 示例：`http://127.0.0.1:3001`
  - 说明：前端访问 BFF HTTP 接口。

### 可选
- `VITE_ENABLE_REPLAY`
  - 默认：`true`
  - 说明：是否启用回放 UI。

- `VITE_ENABLE_AUDIO`
  - 默认：`true`
  - 说明：是否启用音效。

- `VITE_RENDER_QUALITY`
  - 可选：`high|medium|low`
  - 默认：`high`
  - 说明：渲染质量分级。

## BFF 环境变量

### 必需
- `PORT`
  - 默认：`3001`
  - 说明：BFF 对外服务端口。

- `OPENCLAW_GATEWAY_WS_URL`
  - 示例：`ws://127.0.0.1:18789`
  - 说明：连接 OpenClaw Gateway 的 WS 地址。

- `OPENCLAW_GATEWAY_TOKEN`
  - 说明：Gateway 鉴权 token（如启用 token auth）。

### 推荐
- `BFF_JWT_SECRET`
  - 说明：前端访问令牌签名密钥。

- `BFF_READONLY_TOKEN`
  - 说明：观战屏只读令牌。

- `LOG_LEVEL`
  - 可选：`debug|info|warn|error`
  - 默认：`info`

- `REQUEST_TIMEOUT_MS`
  - 默认：`5000`
  - 说明：对上游调用超时阈值。

### 可选
- `REDIS_URL`
  - 示例：`redis://127.0.0.1:6379`
  - 说明：用于事件缓存与幂等去重。

- `REPLAY_RETENTION_MINUTES`
  - 默认：`30`
  - 说明：回放数据保留时长。

- `METRICS_ENABLED`
  - 默认：`true`
  - 说明：是否暴露指标端点。

- `ALLOW_ORIGIN`
  - 默认：`http://127.0.0.1:5173`
  - 说明：CORS 白名单。

## OpenClaw 对接配置
1. 优先通过 `OPENCLAW_GATEWAY_WS_URL` 建立长连接。
2. 若启用 `/tools/invoke` 补充通道，建议复用同 host/port。
3. OpenClaw 协议版本变更时，先更新 `05-event-protocol.md` 再改实现。

## 默认端口建议
- Web dev server：`5173`
- BFF：`3001`
- OpenClaw Gateway：`18789`
- OpenClaw canvas host（若启用）：`18793`
- Redis：`6379`

## 启动顺序建议
1. 启动 OpenClaw Gateway。
2. 启动 BFF 并确认上游连接成功。
3. 启动 Web 前端。
4. 打开健康检查页验证链路。

## 健康检查建议
- BFF 提供 `GET /healthz`：
  - `gatewayConnected`
  - `uptimeSec`
  - `lastEventTs`
  - `version`

## 安全规范
1. `OPENCLAW_GATEWAY_TOKEN` 仅存在于 BFF 环境。
2. 前端只接收短时访问令牌，不持有主密钥。
3. 所有配置项变更必须可审计。
4. 生产环境强制 HTTPS/WSS。

## 配置校验
1. BFF 启动时对必填环境变量做 schema 校验（zod）。
2. 缺失必填项时直接 fail-fast。
3. 输出清晰错误信息（不打印敏感值）。

## 示例 `.env.local`（BFF）
```env
PORT=3001
OPENCLAW_GATEWAY_WS_URL=ws://127.0.0.1:18789
OPENCLAW_GATEWAY_TOKEN=replace_me
BFF_JWT_SECRET=replace_me
LOG_LEVEL=info
REQUEST_TIMEOUT_MS=5000
REDIS_URL=redis://127.0.0.1:6379
REPLAY_RETENTION_MINUTES=30
METRICS_ENABLED=true
ALLOW_ORIGIN=http://127.0.0.1:5173
```

## 示例 `.env.local`（Web）
```env
VITE_BFF_WS_URL=ws://127.0.0.1:3001/realtime
VITE_BFF_HTTP_URL=http://127.0.0.1:3001
VITE_ENABLE_REPLAY=true
VITE_ENABLE_AUDIO=true
VITE_RENDER_QUALITY=high
```

## 验收清单
- `.env.example` 与文档一致。
- 启动失败时能明确提示缺失配置项。
- 密钥未泄露到前端构建产物。
- 本地默认配置可一键跑通主链路。
