---
feature_ids: [F018]
related_features: [F017]
topics: [plan-pool, admin-panel, member-management, self-service, quota, time-based-access]
doc_kind: spec
created: 2026-04-11
layer: application
owner_module: plan-pool
status: p0-deployed-test
phase: 0
depends_on:
  - { id: F017, type: blocking, note: "基于 Plan Pool Phase 1.5 的管理页面" }
roundtable: docs/discussions/F018-admin-roundtable.md
---

# F018: Plan Pool Admin Enhancement — 管理面板 + 成员自服务

> Status: **P0 deployed to test** | Owner: @gpt52 (impl) + @opus (review/merge)
> Roundtable 1: 2026-04-11（opus / gemini25 / gpt52）— 时间管理 + 自服务面板 + 架构评审
> Roundtable 2: 2026-04-11（opus / gemini25 / gpt52）— 5H/1W 速率限制头脑风暴
> 铲屎官裁定：UTC 整点桶、默认值 5H=$5 1W=$40、P0 含速率限制
> **@gpt52 formal review passed** 2026-04-11 — 4 findings 全部修复后放行
> **P0 merged** 2026-04-11 — PR #35 (impl) + PR #38 (fix: admin layout + member $ visibility)
> **P0 deployed to test** 2026-04-12 — test.mesh-hub.xyz:3011, CVO 验收中

## Why

Plan Pool Phase 1.5 (F017) 的管理面板是 MVP 级别：owner 能看成员列表和调用记录，能 suspend 和改额度。但存在两个核心缺失：

1. **只能用额度管理，不能用时间管理** — 实际场景中 owner 经常需要"给这个人用 7 天"或"月底到期自动停"，目前只能手动 suspend。
2. **成员完全没有自服务能力** — 拿到 API key 后就是盲飞，看不到额度、过期时间、使用记录。

铲屎官原话："时间和额度应该是 AND 的关系，必须既有时间又有额度才可用。"

## What

### P0 — 铲屎官直接要求

#### P0-1: 时间 + 额度双维度管理

**模型（@gpt52 架构修订）：**

```
可调用(invoke/register) = status == 'active' 
                         AND expires_at > now 
                         AND available_usd > 0
                         AND 5h_window_used < rate_limit_5h_usd
                         AND weekly_window_used < rate_limit_weekly_usd
可自服务(read /me)      = status == 'active'  (即使 expired/限速，仍可看到原因)
```

> **关键设计决策：过期 ≠ 自动 suspend，两者分离。**
> - `status` 只表示 **人工硬状态**（owner 手动 suspend/reactivate）
> - `expires_at` 只表示 **时间门槛**（到期后 invoke 被拒，但 key 不 revoke、status 不变）
> - 前端/API 返回 `effectiveStatus`（canonical enum，按优先级）: suspended / expired / quota_exhausted / rate_limited_weekly / rate_limited_5h / active
>
> 理由（@gpt52）：如果到期就 auto-suspend + revoke key，成员连 `/v1/me` 都进不去，看不到"为什么不能用"。
> 而且 `status='suspended'` 同时表达"人工封禁"和"时间到期"会污染状态机、审计无法区分。

**实现：**
- DB: `members` 表加 `expires_at INTEGER` (unix ms, nullable = 永不过期)，加索引 `(status, expires_at)`
- API: `PUT /v1/admin/members/:id/expiry` — 设置/延期/清除过期时间
- 鉴权拆分: `authenticateApiKey()` → `AuthContext`，按路由做准入判断:
  - `canInvoke`: active + not expired + has quota + within rate limits (5H & 7D)
  - `canRegisterBridge`: active + not expired
  - `canAdmin`: active + role=owner
  - `canReadSelf`: active（即使 expired 也允许，保留自服务只读入口）
- 信用检查: `checkQuota()` 增加 `expires_at` 校验，区分 `EXPIRED`(403) 和 `QUOTA_EXHAUSTED`(402)
- Bridge 联动: suspend/expire 时立即调用 `PoolRouter.evictMemberBridges(memberId)` 从路由表摘除，不等心跳超时
- 邀请码生成时可设默认成员有效期

#### P0-3: 滚动窗口速率限制（5H + 1W）

> 铲屎官场景：1 个 Max 订阅分给 5 个成员，每人 ≈ Pro 级别速率，防止一个人用光所有额度。
> 参考 Claude（消息/5小时）和 OpenAI（TPM/RPM），按金额控制。成员端 quota 和 per-call 显示 $，5H/7D 窗口显示百分比。

**数据模型：**
```sql
-- members 表新增
rate_limit_5h_usd    REAL NULL,  -- 每 5 小时最大消费金额，NULL = 不限
rate_limit_weekly_usd REAL NULL,  -- 每周最大消费金额，NULL = 不限

-- teams 表新增（团队默认值）
default_rate_limit_5h_usd    REAL NULL DEFAULT 5.0,   -- $5/5h
default_rate_limit_weekly_usd REAL NULL DEFAULT 40.0   -- $40/7d
```

**检查逻辑（`checkRateLimit()`）— UTC 整点桶：**
```ts
// 5H 桶：含当前 UTC 小时共 5 个整点小时
// 例：now = 10:32 UTC → bucketStart = 06:00 UTC (floor(10) - 4 = 6)
const nowHour = Math.floor(Date.now() / 3600_000) * 3600_000; // 当前 UTC 整点
const bucketStart5h = nowHour - 4 * 3600_000; // 含当前小时共 5 个整点小时

const used5h = db.prepare(
  `SELECT COALESCE(SUM(normalized_cost), 0) as total FROM invocations
   WHERE source_member_id = ? AND created_at >= ?
   AND lease_status IN ('pending','settled','expired')`
).get(memberId, bucketStart5h);

// 7D 桶：当前 UTC 日向前推 7 个自然天
const nowDay = Math.floor(Date.now() / 86400_000) * 86400_000; // 当前 UTC 日 00:00
const bucketStart7d = nowDay - 6 * 86400_000; // 含当天共 7 个自然天
// 同理...
```

**错误码：**
- `RATE_LIMITED_5H`(429) + `Retry-After: 距下一个 UTC 整点的秒数`
- `RATE_LIMITED_WEEKLY`(429) + `Retry-After: 距下一个 UTC 日 00:00 的秒数`

> 注：Retry-After 指向下一个桶边界（整点/日切），此时最早的一小时/一天消费会滑出窗口。

**Team 默认值管理（@gpt52 review 补充）：**
- API: `PUT /v1/admin/team/settings` — 设置 team 默认值（default_borrow_limit_usd, default_rate_limit_5h_usd, default_rate_limit_weekly_usd）
- Admin UI: Team Settings 卡片（独立于成员编辑，放在邀请码区块下方）

**Admin 成员编辑（@gemini25 渐进式展开）：**
- 成员编辑 dialog 分三组（@gpt52 建议）：Access（status, expires_at）/ Budget（borrow limit）/ Rate Windows（5H/7D）
- Rate Limits 默认显示 checkbox `[x] Use Team Defaults ($5/5H, $40/7D)`
- 取消勾选才展开两个数字输入框 — 90% 场景一个 checkbox 搞定

**Member 端（@gemini25 木桶效应仪表盘）：**
- 主视觉只展示**当前最接近耗尽的限制**作为大进度条
- 其他健康指标折叠为小 Badge
- Quota 和 per-call 显示 $，5H/7D 窗口显示百分比

**effectiveStatus 优先级（按恢复难度降序，@gemini25）：**
`suspended > expired > quota_exhausted > rate_limited_weekly > rate_limited_5h > active`

**窗口设计（铲屎官裁定 2026-04-11，@gpt52 review 收敛）：**
- **UTC 整点桶**：5H = 含当前 UTC 小时共 5 个整点小时（`floor(now_hour) - 4h`）；7D = 含当天共 7 个 UTC 自然天（`floor(now_day) - 6d`）
- 所有时间基于 **UTC**，不涉及本地时区
- 计入: `pending + settled + expired`（防 TOCTOU 并发打穿 + 保守结算一致）
- 不计入: `rolled_back`（系统失败不应让用户承担）
- `Retry-After`: 距下一个桶边界（UTC 整点 / UTC 日 00:00）的秒数
- 管理员下调 limit 后，不回收 in-flight，从下一次 invoke 开始 429（@gpt52）

**默认值（铲屎官裁定）：**
- `5H = $5.00`（按 Pro 订阅估算）
- `1W = $40.00`（按 Pro 订阅估算）
- **前置依赖**：`credit.ts` 价格表必须更新（目前不认新模型名，回落 $0.02 会打歪美元限速，@gpt52）

#### P0-2: 成员自服务面板

**新增 API（member API key 认证）：**
- `GET /v1/me` — 成员信息，响应体：
  ```json
  {
    "memberId": "...",
    "displayName": "...",
    "effectiveStatus": "active|suspended|expired|quota_exhausted|rate_limited_weekly|rate_limited_5h",
    "blockingReason": "5H rate limit exceeded" | null,
    "nextAvailableAt": 1735700400000 | null,
    "retryAfterSeconds": 1823 | null,
    "quotaPercent": 78,
    "rateLimitPercent5h": 32,
    "rateLimitPercent7d": 12,
    "expiresAt": 1735700400000 | null,
    "credentialStatus": "active",
    "credentialLastRotatedAt": 1735600000000,
    "teamName": "..."
  }
  ```
- `GET /v1/me/usage` — 个人用量统计（按模型/天聚合）
- `GET /v1/me/invocations?limit=N` — 个人调用记录

**新增页面 `member.html`（`/member`）：**

信息层次设计（@gemini25 建议）：
- **L1 Hero Card（木桶效应 @gemini25 + 恢复时间 @gpt52）**:
  - 主状态：可用/不可用 + 主阻塞原因 + **预计恢复时间**（"约 2 小时后恢复"）
  - Quota 和 per-call 显示 $，5H/7D 窗口显示百分比 — 必须补"何时恢复"才有行动意义
- **L2 三小卡**: 总额度% / 5H窗口% / 7D窗口%（含"进行中请求预留"说明）
- **L3 凭证**: Credential 状态 + 最后轮换时间 + 接入文档快速入口（不回显 API key，DB 只存 hash，@gpt52）
- **L4 用量趋势**: 最近 7 天消耗趋势（CSS 柱状图或轻量 Chart.js）
- **L5 调用明细**: 最近 50 条调用记录（时间、模型、消耗）

### P1 — 体验显著提升

#### P1-1: Unsuspend / Reactivate（@gpt52 架构建议）
- API: `PUT /v1/admin/members/:id/reactivate`
- 逻辑: `status='suspended'` → `status='active'`
- **reactivate 和 rotate-key 分离**：reactivate 不自动发新 key
  - 如果是人工 suspend（key 已 revoke）→ reactivate 后 owner 需手动 rotate-key
  - 如果是时间到期（key 未 revoke、status 未变）→ owner 只需延长 expires_at，无需 reactivate
- Admin UI: suspended 成员显示 Reactivate 按钮

#### P1-2: Admin 内联编辑（@gemini25 UX 方案）
- **方案 A（推荐）**: Native `<dialog>` 模态框 — 点击 Edit 弹出居中 Modal，时间/额度分区块，Save/Cancel
- **方案 B（进阶）**: 右侧抽屉 Side Drawer — 点击成员行右滑出编辑面板，左侧表格保持可见
- UX 要点: `<input type="datetime-local">` 选时间；`[x] Never expires` checkbox 禁用日期选择器
- Admin container max-width 从 800px 放宽到 1200px 以适应多列数据

#### P1-3: 邀请码管理
- API: `GET /v1/admin/invites` — 列出有效邀请码
- API: `DELETE /v1/admin/invites/:codeHash` — 撤销
- Admin UI: 邀请码列表卡片（状态、用量、过期时间）

### P2 — 锦上添花

- P2-1: 操作审计日志（admin actions log）
- P2-2: 用量可视化图表
- P2-3: 调用列表分页 + 筛选
- P2-4: 用量告警（接近额度/过期提醒）
- P2-5: Bridge 管理面板

## Acceptance Criteria

### P0 AC
- [x] 成员有 `expires_at` 字段，到期后 invoke/register 被拒（403 EXPIRED）
- [x] 过期 ≠ suspend：过期成员 status 不变、key 不 revoke，仍可访问 `/v1/me`
- [x] Owner 可在 admin 面板设置/修改成员过期时间和额度
- [x] 时间、额度、速率限制是 AND 关系：全部满足才可调用
- [x] 5H/1W 滚动窗口速率限制：按金额控制，超限返回 429 + Retry-After
- [x] Admin 端可设每个成员的 5H/1W 限额（金额），也可设团队默认值
- [x] suspend/expire 时立即从路由表摘除成员 bridge（不等心跳超时）
- [x] 成员用 API key 可登录 `/member` 查看 effectiveStatus
- [x] Member 端显示 quota $（available/total）+ per-call $，5H/7D 窗口显示百分比
- [x] 成员可看到自己的调用历史和用量统计

### P1 AC
- [ ] Owner 可 reactivate 被 suspend 的成员
- [ ] Admin 编辑操作使用内联表单（非 prompt 弹窗）
- [ ] Owner 可查看和撤销已创建的邀请码

## Industry Reference

- **OpenAI Platform**: Project-level budget + monthly reset, member roles, usage dashboard
- **Anthropic Console**: Workspace spend limits, Admin API, 6 roles, cost/usage reporting
- **Stripe Dashboard**: Self-service developer portal, real-time usage, billing breakdown
- **Apigee**: Time-based + quota-based access control, developer portal self-service

## Roundtable Decisions (2026-04-11)

### UX Decisions (@gemini25)

| 决策 | 结论 |
|------|------|
| Admin container width | 800px → 1200px |
| 成员面板信息层次 | L1 Hero Card(木桶效应) → L2 凭证 → L3 趋势 → L4 明细 |
| Admin 编辑交互 | `<dialog>` 模态框（推荐）或 Side Drawer（进阶） |
| 时间选择器 | `<input type="datetime-local">` + Never expires checkbox |
| 设计框架 | 继续原生 CSS 变量，可选 Tailwind CDN 加速 |
| Rate Limit 编辑 | 渐进式展开：默认 checkbox "Use Team Defaults"，取消才展开输入框 |
| Member Hero Card | 木桶效应：主视觉展示最紧迫限制，其余折叠为小 Badge |
| effectiveStatus 优先级 | suspended > expired > quota_exhausted > rate_limited_weekly > rate_limited_5h > active |
| 滑动窗口 | UTC 整点桶（铲屎官裁定整点对齐，@gpt52 review 收敛为 UTC） |
| 默认值 | 5H=$5, 1W=$40（铲屎官按 Pro 订阅估算裁定） |

### Architecture Decisions (@gpt52)

| 决策 | 结论 | 理由 |
|------|------|------|
| 过期 vs suspend | **分离**：expires_at 是时间门槛，status 是人工硬状态 | 合并会污染状态机、阻断自服务面板 |
| 鉴权模型 | 拆为 `canInvoke/canRegisterBridge/canAdmin/canReadSelf` | 过期成员需要 `/v1/me` 只读入口 |
| 错误码映射 | `suspended/expired` → 403, `quota_exhausted` → 402, `rate_limited` → 429+Retry-After | HTTP 状态码与 effectiveStatus 解耦（@gpt52 review） |
| Bridge 联动 | suspend/expire 时立即从 router 摘除 | 否则有 ~2 分钟路由泄漏窗口 |
| Reactivate + Key | 分离操作，reactivate 不自动发新 key | 时间到期不 revoke key，延长 expires_at 即可恢复 |
| In-flight 请求 | 已放行的请求允许跑完，不中途 kill | 最小化对用户的干扰 |

### Architecture Risks (@gpt52 发现)

| # | 风险 | 严重度 | 对策 |
|---|------|--------|------|
| R1 | suspend 后 bridge 仍在路由表（~2min 窗口） | **P0** | `evictMemberBridges()` 立即摘除 |
| R2 | auto-suspend+revoke 与自服务面板冲突 | **P0** | 过期不 suspend、不 revoke（已采纳） |
| R3 | status 同时表达人工封禁和时间到期 | P1 | 分离 status 和 expires_at 语义 |
| R4 | 过期成员仍能续 bridge heartbeat | P1 | `canRegisterBridge` 检查 expires_at |
| R5 | 额度检查有并发竞态（read-then-write） | P2 | 非 F018 问题，但成员面板会放大感知 |
| R6 | `credit.ts` 价格表只认旧模型名，不匹配回落 $0.02 | **P0** | 直接打歪美元窗口限速，必须前置更新 |
| R7 | pending 预留只有 $0.02，对高成本模型不足以防 burst | P1 | 可基于 requestedModel 做更强预估 |

## Open Questions

1. ~~过期成员自动 suspend 后，是否需要通知机制？~~ → 已改为不 auto-suspend
2. 成员面板是否需要"申请续期"功能（发请求给 owner）？
3. ~~是否需要 grace period（过期后缓冲期）？~~ → 架构上已有：过期仅拒 invoke，不 revoke
4. P2 的用量图表用什么方案？纯 CSS/SVG 还是引入 Chart.js？
5. Bridge 贡献者是否应获得速率限制加成系数？（@gemini25 建议 x1.5，待铲屎官决策）→ Phase 2
6. 长上下文单次重击（单请求 $4）瞬间打爆 5H 窗口：MVP 靠调大个人限额解决，后续是否需要 Token Bucket burst？
7. 凌晨无人时段窗口让渡 → Phase 2（本质是 burst-pool / market-based fairness，@gpt52）
8. ~~默认值用 conservative beta(5/25) 还是 power(20/100)？~~ → 铲屎官裁定 5H=$5, 1W=$40
