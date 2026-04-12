---
feature_ids: [F017]
topics: [fingerprint-normalization, competitor-analysis, oauth-proxy, risk-control]
doc_kind: discussion
created: 2026-04-12
participants: [codex, CVO]
status: draft
---

# F017 竞品调研：设备指纹实现与管理机制（Phase 1C）

## 1. 调研目标

回答两件事：

1. 竞品如何实现设备指纹（headers + `<env>`）归一化。
2. 竞品如何管理指纹（采集、缓存、fallback、刷新、失败保护）。

本次只采用一手源码/官方仓库文档，不采用二手转载。

## 2. 样本与证据

### 样本 A: `motiful/cc-gateway`

- 仓库: <https://github.com/motiful/cc-gateway>
- 本地核查 commit: `447fad1`
- 关键证据:
  - `config.example.yaml` 里显式区分 `env` 与 `prompt_env`，并要求一致。
  - `src/rewriter.ts` 对 prompt 中 `Platform/Shell/OS Version/Working directory` 做替换。
  - `src/rewriter.ts` 重写 `user-agent`，剥离 `x-anthropic-billing-header`。
  - `src/oauth.ts` 有自动刷新调度（过期前刷新 + 失败 30s 重试）。

### 样本 B: `CaddyGlow/ccproxy-api`

- 仓库: <https://github.com/CaddyGlow/ccproxy-api>
- 本地核查 commit: `39dc5f8`
- 关键证据:
  - `ccproxy/plugins/claude_api/models.py` 使用 schema 固定 `x-stainless-*`/`user-agent`/`anthropic-*` 字段别名。
  - `ccproxy/plugins/claude_api/detection_service.py` 有 cache 保存与 fallback 加载。
  - `ccproxy/data/claude_headers_fallback.json` 内置完整 fallback headers + system payload。
  - `CHANGELOG.md` 明确引入 package data fallback 机制。

### 样本 C: `snipeship/ccflare`

- 仓库: <https://github.com/snipeship/ccflare>
- 本地核查 commit: `6889212`
- 关键证据:
  - 代码中基本无 `x-stainless` / `<env>` / `prompt_env` 归一化实现。
  - `docs/api-http.md` 强调 session 路由防封号、自动 failover、token refresh。
  - 同文档声明“无可用账号时可无鉴权 fallback 转发”。

## 3. 竞品实现对比

| 维度 | cc-gateway | ccproxy-api | ccflare |
|---|---|---|---|
| Header 指纹 | 有（重写 UA，处理 billing header） | 有（schema + fallback headers） | 基本无专门层 |
| `<env>` 处理 | 有（prompt_env 驱动） | 主要依赖捕获/fallback payload，不是专门 normalizer 模块 | 无 |
| 配置形态 | 管理员维护 canonical config | 检测结果缓存 + package fallback | 以账号/路由管理为主 |
| 刷新机制 | OAuth 自动刷新清晰 | 有 token 管理与检测缓存流程 | Token refresh + 多账号切换 |
| 失败策略 | 更偏“继续服务”（重试） | 有 fallback 数据继续运行 | 强 availability（failover） |
| 封号风险控制 | 隐含（归一化减少漂移） | 中等（fallback 可能陈旧） | 主要靠 session 策略，非指纹层 |

## 4. 关键观察

1. `cc-gateway` 路线是“强 canonical profile”：
   - 管理面集中（`env` + `prompt_env`）
   - 执行面明确（header/prompt 重写）
   - 适合“身份一致性优先”的场景

2. `ccproxy-api` 路线是“检测驱动 + 缓存 + fallback 数据包”：
   - 好处是工程韧性高，CLI 版本变化时不至于立即瘫痪
   - 风险是 fallback 与真实环境可能发生长期漂移

3. `ccflare` 路线聚焦“账号池可用性”，不是“指纹一致性”：
   - 对我们当前 F017 Phase 1C 目标（owner 指纹一致）不够直接

## 5. 对 F017 的可落地结论

### 5.1 我们应采纳

1. 采用 `cc-gateway` 的“双配置分层”思想：
   - `env`（完整采集快照）
   - `prompt_env`（运行最小必需面）

2. 采用 `ccproxy-api` 的“检测缓存 + fallback 资产”思想，但收紧边界：
   - 允许 `last-known-good` fallback
   - 禁止硬编码通用默认值直接参与线上请求构造

3. 采用 `ccflare` 的“会话策略降低异常模式”思想：
   - 同一 session sticky 到同一 bridge
   - 避免跨 bridge 快速跳变导致行为特征异常

### 5.2 我们应避免

1. 避免长期 `warn`：会形成 silent drift。
2. 避免“无鉴权 fallback”这类 availability-first 策略迁移到我们的路径。
3. 避免把 fallback 数据当主数据源长期运行。

## 6. 推荐管理机制（归档版）

1. 采集与固化:
   - 首装采集 owner 本机指纹并写配置。
   - 保留 `last-known-good`。

2. 执行与校验:
   - preflight 校验必需字段。
   - 对 Claude caller 执行 `<env>` whole-block rewrite + canonical headers。
   - 非 Claude caller passthrough。

3. 失败与风控:
   - preflight 失败直接 `503 bridge_fingerprint_unavailable`，不发上游。
   - failure budget 触发 `SUSPENDED_FINGERPRINT`，暂停新外部会话路由。

4. 自动升级:
   - 启动 + 定时（默认 6h）+ 漂移触发重采集。
   - 候选配置需 probe 成功后再升为 active；失败回滚 `last-known-good`。

## 7. 对当前文档状态的同步结论

本调研结果与以下两份文档已对齐（策略一致）：

- `docs/features/F017-phase2-oauth-proxy.md`（Phase 1C 策略冻结）
- `docs/plans/F017-phase2-1C-risk-checklist.md`（失败熔断 + 自动刷新）

## 8. 附：本次源码核查命中点（摘要）

- `cc-gateway`
  - `config.example.yaml:59-67`
  - `src/rewriter.ts:147-190, 332-349`
  - `src/oauth.ts:62-79`
- `ccproxy-api`
  - `ccproxy/plugins/claude_api/models.py:38-110`
  - `ccproxy/plugins/claude_api/detection_service.py:410-423`
  - `ccproxy/data/claude_headers_fallback.json`（headers 字段）
  - `CHANGELOG.md:104-109`
- `ccflare`
  - `docs/api-http.md:852-860`
  - 代码检索未发现 `x-stainless`/`prompt_env`/`<env>` 归一化实现
