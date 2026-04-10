---
feature_ids: [F018]
related_features: [F017]
topics: [plan-pool, admin-panel, member-management, self-service, quota, time-based-access]
doc_kind: spec
created: 2026-04-11
layer: application
owner_module: plan-pool
status: discussion
phase: 0
depends_on:
  - { id: F017, type: blocking, note: "基于 Plan Pool Phase 1.5 的管理页面" }
roundtable: docs/discussions/F018-admin-roundtable.md
---

# F018: Plan Pool Admin Enhancement — 管理面板 + 成员自服务

> Status: **discussion** | Owner: TBD
> Roundtable: 2026-04-11（opus / gemini25 / gpt52）
> 铲屎官两个核心发现 + 深度调研 8 项额外问题

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
可调用(invoke/register) = status == 'active' AND expires_at > now AND available_usd > 0
可自服务(read /me)      = status == 'active'  (即使 expired，仍可看到"为什么不能用")
```

> **关键设计决策：过期 ≠ 自动 suspend，两者分离。**
> - `status` 只表示 **人工硬状态**（owner 手动 suspend/reactivate）
> - `expires_at` 只表示 **时间门槛**（到期后 invoke 被拒，但 key 不 revoke、status 不变）
> - 前端/API 返回 `effectiveStatus`: active / expired / suspended / quota_exhausted
>
> 理由（@gpt52）：如果到期就 auto-suspend + revoke key，成员连 `/v1/me` 都进不去，看不到"为什么不能用"。
> 而且 `status='suspended'` 同时表达"人工封禁"和"时间到期"会污染状态机、审计无法区分。

**实现：**
- DB: `members` 表加 `expires_at INTEGER` (unix ms, nullable = 永不过期)，加索引 `(status, expires_at)`
- API: `PUT /v1/admin/members/:id/expiry` — 设置/延期/清除过期时间
- 鉴权拆分: `authenticateApiKey()` → `AuthContext`，按路由做准入判断:
  - `canInvoke`: active + not expired + has quota
  - `canRegisterBridge`: active + not expired
  - `canAdmin`: active + role=owner
  - `canReadSelf`: active（即使 expired 也允许，保留自服务只读入口）
- 信用检查: `checkQuota()` 增加 `expires_at` 校验，区分 `EXPIRED`(403) 和 `QUOTA_EXHAUSTED`(402)
- Bridge 联动: suspend/expire 时立即调用 `PoolRouter.evictMemberBridges(memberId)` 从路由表摘除，不等心跳超时
- 邀请码生成时可设默认成员有效期

#### P0-2: 成员自服务面板

**新增 API（member API key 认证）：**
- `GET /v1/me` — 成员信息（quota, expiry, status, team info）
- `GET /v1/me/usage` — 个人用量统计（按模型/天聚合）
- `GET /v1/me/invocations?limit=N` — 个人调用记录

**新增页面 `member.html`（`/member`）：**

信息层次设计（@gemini25 建议）：
- **L1 Hero Card**: 双仪表盘 — 剩余额度($) + 剩余时间(天/小时) + 总体状态 Badge
- **L2 凭证**: 模糊化 API Key（点击复制）+ 接入文档快速入口
- **L3 用量趋势**: 最近 7 天消耗趋势（CSS 柱状图或轻量 Chart.js）
- **L4 调用明细**: 最近 50 条调用记录（时间、模型、消耗）

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
- [ ] 成员有 `expires_at` 字段，到期后 invoke/register 被拒（403 EXPIRED）
- [ ] 过期 ≠ suspend：过期成员 status 不变、key 不 revoke，仍可访问 `/v1/me`
- [ ] Owner 可在 admin 面板设置/修改成员过期时间和额度
- [ ] 时间和额度是 AND 关系：两者都满足才可调用
- [ ] suspend/expire 时立即从路由表摘除成员 bridge（不等心跳超时）
- [ ] 成员用 API key 可登录 `/member` 查看自己的额度、过期时间、effectiveStatus
- [ ] 成员可看到自己的调用历史和用量统计

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
| 成员面板信息层次 | L1 Hero Card → L2 凭证 → L3 趋势 → L4 明细 |
| Admin 编辑交互 | `<dialog>` 模态框（推荐）或 Side Drawer（进阶） |
| 时间选择器 | `<input type="datetime-local">` + Never expires checkbox |
| 设计框架 | 继续原生 CSS 变量，可选 Tailwind CDN 加速 |

### Architecture Decisions (@gpt52)

| 决策 | 结论 | 理由 |
|------|------|------|
| 过期 vs suspend | **分离**：expires_at 是时间门槛，status 是人工硬状态 | 合并会污染状态机、阻断自服务面板 |
| 鉴权模型 | 拆为 `canInvoke/canRegisterBridge/canAdmin/canReadSelf` | 过期成员需要 `/v1/me` 只读入口 |
| 过期错误码 | `EXPIRED`(403) vs `QUOTA_EXHAUSTED`(402) 区分 | 前端需要告知用户具体原因 |
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

## Open Questions

1. ~~过期成员自动 suspend 后，是否需要通知机制？~~ → 已改为不 auto-suspend
2. 成员面板是否需要"申请续期"功能（发请求给 owner）？
3. 是否需要 grace period（过期后缓冲期）？→ 架构上已有：过期仅拒 invoke，不 revoke
4. P2 的用量图表用什么方案？纯 CSS/SVG 还是引入 Chart.js？
