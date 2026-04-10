---
feature_ids: [F017]
topics: [oauth-proxy, token-management, research]
doc_kind: research
created: 2026-04-11
researcher: 宪宪/Opus-46
status: closed
decision: "D5=B+(读为主+兜底刷新, eventual consistency); D6=C'(WS tunnel) — CVO 批准 2026-04-11"
---

# F017 RT1-RT3: OAuth Token 调研报告

## RT1: Claude CLI OAuth Token 存储位置

### 结论

| 平台 | 存储位置 | 读取方式 |
|---|---|---|
| **macOS** | Keychain, service = `Claude Code-credentials` | `security find-generic-password -s 'Claude Code-credentials' -w` |
| **Linux** | `~/.claude/.credentials.json` | 直接读文件 |
| **Windows** | `~/.claude/.credentials.json` | 直接读文件 |
| **环境变量覆盖** | `CLAUDE_CODE_OAUTH_TOKEN` | 优先级最高，覆盖上述所有 |

### 凭据结构（本机验证）

```json
{
  "claudeAiOauth": {
    "accessToken": "sk-ant-oat01-...",     // Bearer token，直接用于 API 调用
    "refreshToken": "sk-ant-ort01-...",    // 长期 token，用于换新 accessToken
    "expiresAt": 1775851578536,            // Unix ms 时间戳
    "scopes": [
      "user:file_upload",
      "user:inference",
      "user:mcp_servers",
      "user:profile",
      "user:sessions:claude_code"
    ],
    "subscriptionType": "max",             // 订阅类型
    "rateLimitTier": "default_claude_max_5x"  // 速率档位
  }
}
```

### Token 前缀约定

- Access token: `sk-ant-oat01-*`（oat = OAuth Access Token）
- Refresh token: `sk-ant-ort01-*`（ort = OAuth Refresh Token）

### Bridge 实现要点

- **macOS**: 用 `security` 命令或 Keychain API 读取
- **Linux**: 直接读 JSON 文件
- **环境变量兜底**: 先检查 `CLAUDE_CODE_OAUTH_TOKEN`
- `subscriptionType` 和 `rateLimitTier` 可用于 bridge 能力声明

### 证据等级

- Keychain service name `Claude Code-credentials`：**empirical/local verification**（本机 Keychain 实测，非官方 contract）
- 凭据 JSON 结构：本机验证 + 多个开源项目一致
- 生产 `CLIENT_ID` / `TOKEN_URL`：从本机 CLI bundle `cli.js` 提取（最高可信度）

### 来源

- 本机 Keychain 验证
- 本机 Claude Code 生产 bundle (`cli.js`) — `TOKEN_URL` / `CLIENT_ID` 硬编码
- [Claude Code Auth Docs](https://code.claude.com/docs/en/authentication)
- [opencode-claude-auth](https://github.com/griffinmartin/opencode-claude-auth)
- [Claude Code #21765](https://github.com/anthropics/claude-code/issues/21765)

---

## RT2: Token Refresh 端点格式

### 结论

**刷新端点**: `POST https://platform.claude.com/v1/oauth/token`

> 来源：本机 Claude Code 生产 bundle (`cli.js`) 中硬编码的 `TOKEN_URL`。网上多个来源报告不同 URL（`console.anthropic.com/v1/oauth/token`、`claude.ai/v1/oauth/token`），可能是旧版或重定向，以 CLI 实际 bundle 为准。

### 刷新请求

```http
POST https://platform.claude.com/v1/oauth/token
Content-Type: application/json

{
  "grant_type": "refresh_token",
  "refresh_token": "sk-ant-ort01-...",
  "client_id": "9d1c250a-e61b-44d9-88ed-5944d1962f5e"
}
```

- `client_id` 是 Claude Code CLI 的官方 OAuth client ID
- 可能随 Claude Code 版本更新变化

### 刷新响应

```json
{
  "token_type": "Bearer",
  "access_token": "sk-ant-oat01-...",
  "expires_in": 28800,
  "refresh_token": "sk-ant-ort01-...",
  "scope": "user:inference user:profile ..."
}
```

### Refresh Token Rotation（RFC 9700）

**关键约束**：Anthropic 使用 refresh token rotation——每次刷新时旧 refresh token 失效，返回新的 refresh token。

**影响**：
- 同一 refresh token **不能被两个进程同时使用**——refresh 发生时旧 refresh token 的 lineage 断裂
- 已签发的 access token **在 `expiresAt` 之前仍然有效**——冲突发生在下一次 refresh 边界，不是立即中断正在运行的请求
- 真实的破坏场景：bridge 刷新 → owner 的 refresh token lineage 断裂 → owner 下次 refresh 失败 → owner 被迫重新 `claude login`（反之亦然）

### 已知问题

1. **并发刷新冲突**（Claude Code v2.1.81 修复）：一个 session 刷新会使其他 session 的 token 失效
2. **刷新后 scope 丢失**（[#34785](https://github.com/anthropics/claude-code/issues/34785)）：刷新后可能丢失部分 scope
3. **Token endpoint 限流**（[#18329](https://github.com/anomalyco/opencode/issues/18329)）：刷新请求可能被 429

### Bridge 实现要点

**两种策略（需决策，二选一，不混合）**：

| 策略 | 描述 | 优点 | 缺点 |
|---|---|---|---|
| **A: Bridge 独占刷新** | Bridge 管理完整 refresh 生命周期，owner 不同时使用 Claude Code | 简单可靠，cc-gateway 验证过 | owner 使用受限 |
| **B: Bridge 只读、永不刷新** | Bridge 每次请求前从 Keychain 读最新 access token，**永不触碰 refresh token**；token 过期 → fail-closed → 标 unhealthy | owner 可同时使用 | 依赖 owner 的 Claude Code 保持运行来管刷新；bridge 可用性受 owner 在线状态约束 |

**关键约束**：不存在安全的"混合模式"——bridge 一旦自行刷新，就会断裂 owner 的 refresh token lineage，等于退化为策略 A 的副作用但没有策略 A 的可控性。

**cc-gateway 的做法**：策略 A（独占刷新），启动时读取 token，到期前 5 分钟主动刷新。

**推荐**：策略 A（与 cc-gateway 一致），bridge 启动时明确告知 owner "bridge 运行期间请勿在本机使用 Claude Code CLI"。需要 CVO 决策。

### 来源

- [Claude Code Auth Docs](https://code.claude.com/docs/en/authentication)
- [cc-gateway](https://github.com/motiful/cc-gateway)（5min buffer 提前刷新）
- [opencode-claude-auth](https://github.com/griffinmartin/opencode-claude-auth)（lazy refresh within 60s of expiry）
- [mitmproxy 抓包分析](https://www.alif.web.id/posts/claude-oauth-api-key)
- [Claude Code #34785](https://github.com/anthropics/claude-code/issues/34785)（scope 丢失）

---

## RT3: Token 过期周期验证

### 结论

| Token 类型 | 有效期 | 依据 |
|---|---|---|
| **Access token** | **8 小时**（`expires_in: 28800`） | mitmproxy 抓包 + 本机验证 |
| **Refresh token** | 无固定过期（长期有效） | 社区报告 + 官方文档 |

### 本机验证

```
当前时间:  2026-04-11 02:30
Token 到期: 2026-04-11 04:06
剩余: 1.59 小时（已使用 ~6.4 小时，符合 8 小时窗口）
```

### 历史变化

- 早期报告 1-2 小时过期 → 实为并发 session 冲突 bug（v2.1.81 修复）
- 当前稳定版本: 8 小时

### Refresh Token 失效条件

1. **使用后自动失效**（rotation, RFC 9700）
2. **Anthropic 服务端撤销**（logout / 安全事件）
3. **凭据存储损坏**

### Bridge 刷新策略参数

| 参数 | 推荐值 | 依据 |
|---|---|---|
| 提前刷新 buffer | 5 分钟 | cc-gateway 采用 |
| 刷新重试间隔 | 30 秒 | 避免 429 |
| 刷新失败→unhealthy 阈值 | 连续 3 次失败 | 容忍短暂网络波动 |
| 重启后重新读取 | 立即 | 防止用过期 token 启动 |

### 来源

- 本机 Keychain expiresAt 验证
- [mitmproxy 抓包](https://www.alif.web.id/posts/claude-oauth-api-key)（`expires_in: 28800`）
- [Claude Code #36911](https://github.com/anthropics/claude-code/issues/36911)（过期频率讨论）
- [better-ccflare](https://github.com/tombii/better-ccflare)（30min buffer）
- [opencode-claude-auth](https://github.com/griffinmartin/opencode-claude-auth)（60s lazy refresh）

---

## 补充发现：设备指纹归一化（cc-gateway 方案）

从 cc-gateway 调研中获得的额外信息，直接服务于 Phase 1C：

### cc-gateway 归一化策略

| 维度 | 处理方式 |
|---|---|
| `<env>` block | **整体替换**（swap, not patch）——把完整 40+ 字段的 env 对象替换为 canonical profile |
| `User-Agent` | 标准化为 canonical Claude Code 版本 |
| `x-anthropic-billing-header` | **完全去除**（fingerprint hash，可通过 `CLAUDE_CODE_ATTRIBUTION_HEADER=false` 配置） |
| `device_id` | 重写为 canonical ID |
| `email` | 重写为 canonical value |

### 对我们的启示

**应进入 spec 的**（证据充分）：
1. **`<env>` 替换用整体替换，不用正则 patch**——cc-gateway 验证过，且更安全
2. **`x-anthropic-billing-header` 需要丢弃**——本机 CLI bundle 确认此 header 存在，是 fingerprint hash，spec 需补入"丢弃"分类

**待验证，暂不进 spec**（证据不足）：
3. **`device_id` / `email` 归一化**——cc-gateway README 提及，但未在本机 CLI 请求中验证其存在位置和传输方式，保留为 Phase 1C 待验证项

---

## 风险与开放问题

### R1: Refresh Token Rotation 与 Owner 并发使用冲突（HIGH）

Bridge 刷新 token → owner 的 refresh token lineage 断裂 → owner 下次 refresh 失败 → owner 被迫重新 `claude login`。注意：已签发的 access token 在 `expiresAt` 之前仍然有效，冲突在下一次 refresh 边界才爆发。

**需要 CVO 决策**（策略 A vs B，见 RT2 §Bridge 实现要点）：
- 策略 A（推荐）：bridge 独占刷新，owner 不并发使用本地 Claude Code
- 策略 B：bridge 只读不刷新，fail-closed 标 unhealthy，依赖 owner 在线管刷新

### R2: Client ID 可能变化（LOW）

`9d1c250a-e61b-44d9-88ed-5944d1962f5e` 是当前 Claude Code 的 client ID，可能随版本更新。需要做成可配置项。

### R3: Anthropic TOS 合规（MEDIUM）

Anthropic 2026 年文档明确表示 OAuth tokens from Free/Pro/Max 不应用于第三方工具。我们的使用场景是团队内部共享，但需要知晓风险。

### R4: 刷新后 Scope 丢失（LOW）

[#34785](https://github.com/anthropics/claude-code/issues/34785) 报告刷新后可能丢失 scope。Bridge 应在刷新后验证 scope 包含 `user:inference`。
