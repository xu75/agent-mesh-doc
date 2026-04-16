---
feature_ids: [F019]
related_features: [F017, F018]
topics: [multi-team, governance, super-admin, team-lifecycle, cross-team-identity]
doc_kind: spec
created: 2026-04-11
layer: application
owner_module: plan-pool
status: spec-approved
phase: 0
depends_on:
  - { id: F017, type: blocking, note: "基于 Plan Pool 的 team 模型" }
  - { id: F018, type: blocking, note: "单 team 管理能力是多 team 的前提" }
---

# F019: Multi-Team Governance — 多 Team 治理

> Status: **spec-approved** | Owner: 宪宪
> 发起人：宪宪 | CVO 批准拉起讨论：2026-04-11
> Roundtable 1: 2026-04-11（opus / gemini25 / gpt52）— 初版讨论 + 5 点 review 修订

## Why

F017 设计之初就预留了多 team 架构（schema、API scoping、隔离逻辑均已到位），spec 原话：

> "同一个 pool-node 可托管多个 team（如 'Lab-Claude' 和 'Lab-GPT'）。team 之间账本隔离。"

但**默认部署是 first-team-only 模式**——创建第一个 team 后 `POST /v1/admin/teams` 返回 403。即使设了 `POOL_BOOTSTRAP_TOKEN` 可以解锁多 team 创建，从"能建"到"能用"之间还有显著的产品缺口：

1. **无超级管理视角** — 没有跨 team 的全局管理面（team 列表、汇总用量、生命周期管理）
2. **Team 创建只有裸 API** — 没有面向用户的创建/管理流程
3. **无跨 team 身份** — 一个自然人无法同时属于多个 team（每次 join 产生独立 member，互不关联）
4. **Admin 页面是单 team 视角** — admin.html 假设只有一个 team，没有 team 切换
5. **Team 只能创建，不能删除/归档** — 没有生命周期终结逻辑
6. **成员无法发现 team** — 没有 team 列表/发现机制，只能通过邀请码加入

CVO 原话（2026-04-11）："当前的环境是不是还是只能建一个 team，是时候立项，支持多 team 了。"

## What

### 现有基础（不重复建设）

| 已有能力 | 来源 |
|---------|------|
| teams 表支持多行、team_id 隔离查询 | F017 schema |
| API 层按 auth.teamId 自动 scope | F017 api-gateway |
| 每 team 独立 provider_scope + 账本 | F017 设计 |
| Bootstrap token 解锁多 team 创建 | F017 部署保护 |
| Team 内成员管理（suspend/expiry/rate-limit） | F018 |

### P0 — Super-Admin + Team 生命周期

#### P0-1: Super-Admin 角色与 API

**问题**：当前 admin 权限是 team-scoped 的（owner 只能管自己的 team，认证从 API key 推导 `auth.teamId`）。多 team 场景需要一个跨 team 的管理角色，且该角色必须能进入任意 team 的管理上下文。

> **Roundtable 裁定：选方案 B — 独立 `super_admin` 主体**
> - 否决方案 A（bootstrap token 是静态密码，不适合长期运营）
> - 否决方案 C（第一个 owner = super-admin 会导致权限膨胀和概念混淆）
> - 依据：@gemini25 UX 视角 + @gpt52 架构分析

**数据模型**：

```sql
-- 独立于 team 的 super-admin 凭证
CREATE TABLE super_admins (
  id TEXT PRIMARY KEY,
  display_name TEXT NOT NULL,
  key_hash TEXT NOT NULL UNIQUE,
  created_at INTEGER NOT NULL
);
```

> 设计原则：super-admin 不自动获得任何 team 的 owner 权限。team 内操作仍需 team owner 身份。super-admin 的能力是**跨 team 视角 + team 生命周期管理**。

**Super-Admin API**：

Super-admin 进入 team 上下文走**显式 teamId 路径参数**（`/v1/super/teams/:id/...`），不从 auth 推导。

```
# 全局视角
GET  /v1/super/teams                    — 列出所有 team（名称、provider_scope、成员数、总用量、status）
POST /v1/super/teams                    — 创建新 team（替代当前 bootstrap 入口）
GET  /v1/super/usage                    — 跨 team 用量汇总（仅报表聚合，不做账本合并）

# 进入 team 上下文（act-as，只读）
GET  /v1/super/teams/:id/members        — 查看 team 成员
GET  /v1/super/teams/:id/usage          — 查看 team 用量
GET  /v1/super/teams/:id/bridges        — 查看 team bridges

# Team 生命周期
PUT  /v1/super/teams/:id/archive        — 归档 team
PUT  /v1/super/teams/:id/unarchive      — 恢复 team
```

> **@gpt52 Finding 修复**：明确 super-admin 通过 `/v1/super/teams/:id/...` 路径进入 team 上下文，而非复用现有 team-scoped admin 页面。现有 `admin.html` 的 team owner 管理功能不变。

#### P0-2: Team 生命周期

> **Roundtable 裁定：P0 只做 archive，不做 hard delete**

**数据模型变更**（@gpt52 Finding 修复：team 有自己的状态字段，不借 member expiry 代表达）：

```sql
ALTER TABLE teams ADD COLUMN status TEXT NOT NULL DEFAULT 'active';  -- active | archived
ALTER TABLE teams ADD COLUMN archived_at INTEGER;
ALTER TABLE teams ADD COLUMN archived_by TEXT;  -- super_admin id
```

```
状态机：active → archived → active（可恢复）
```

**归档联动（停写不停读）**：
- **停写**：拒绝新调用请求（API gateway 检查 `teams.status`）、拒绝 bridge 注册/心跳、拒绝新成员 join
- 调用 `PoolRouter.evictTeamBridges(teamId)` 从路由表摘除所有 bridges
- **不修改任何 member 的 status 或 expires_at**（F018 原则：不同语义不混用同一状态字段）
- **保持可读**：
  - `GET /v1/me` — 已归档 team 的成员仍可查看自己的状态和历史（effectiveStatus 显示 `team_archived`）
  - Super-admin 只读查询（`/v1/super/teams/:id/*`）正常可用
  - Team owner 的 `admin.html` 可查看历史数据（只读模式）

**恢复**：super-admin 可 unarchive，team 回到 active，bridges 需重新注册。

#### P0-3: Admin 面板 — 多 Team 导航

当前 `admin.html` 直接按 owner API key 加载单个 team。多 team 需要：

- Super-admin 登录（独立凭证）后看到 team 列表
- 每个 team 卡片显示：名称、provider_scope、成员数、状态（active/archived）
- 点击 team 进入**只读**的 team 详情（通过 `/v1/super/teams/:id/...`）
- "创建新 Team" 入口
- 全局 dashboard（跨 team 用量汇总）
- Team owner 的管理页面（`admin.html`）独立存在，不受影响

### P1 — 跨 Team 身份 + Bridge 注册策略

#### P1-1: 跨 Team 用户身份

> **Roundtable 裁定：方向钉死 `users` 表 + 复用 `members` 表（加 `user_id` FK），P1 实现**
> - 否决方案 B（public_key 聚合）— @gpt52 论证：public_key 既不是唯一约束也不是认证主体，聚合方案不自洽
> - 否决方案 C（不做）— 体验不可接受
> - P0 不建空 schema（违反 P1 原则"终态基座不是脚手架"），但 P0 实现不做任何阻断 `users` 表引入的决策

**P1 数据模型**：

```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  display_name TEXT NOT NULL,
  auth_key_hash TEXT NOT NULL UNIQUE,  -- 用户级认证 key
  created_at INTEGER NOT NULL
);

-- members 表加外键
ALTER TABLE members ADD COLUMN user_id TEXT REFERENCES users(id);
```

> 注意：不新建 `memberships` 表。现有 `members` 表已经是 team-scoped 的 membership record，加 `user_id` FK 即可把多个 member 关联到同一 user。

**迁移策略**：P1 实现时，为每个已存在的 member 自动创建对应 user，按 public_key 去重合并。

**P1 API 变化**：
- `GET /v1/me` 返回用户级信息 + 所属 team 列表
- 调用 API 时通过 header 指定目标 team（如 `X-Team-Id`），默认使用"最近活跃 team"
- join 新 team 时自动关联到已有 user

#### P1-2: Team 可见性

> **Roundtable 裁定：默认 private / invite-only**

- Super-admin 可见所有 team
- 成员只看到"我加入的 teams"（通过 `/v1/me`）
- 不做公开 team 发现广场（当前规模不需要）
- Team 加入仍为邀请制（invite code）

#### P1-3: Per-Team Bridge 注册策略

> **@gpt52 Finding 修复**：原 BACKLOG 的 `registrationMode` 是 Hub 层节点注册概念，不等于 pool-side 的 bridge 注册策略。此处独立定义为 `bridgeEnrollmentPolicy`。

- 每个 team 可设置 `bridgeEnrollmentPolicy`: `open`（任何成员可注册 bridge）| `owner-only`（仅 owner 可注册）
- 这是 pool-node 内部的策略，与 Hub 的 `registrationMode` 无关
- Team admin 通过 `PUT /v1/admin/team/settings` 管理

## Roundtable 裁定记录

| Q# | 问题 | 裁定 | 依据 |
|----|------|------|------|
| Q1 | Super-admin 认证方案 | 方案 B：独立 `super_admin` 主体 | gemini25 UX + gpt52 架构 |
| Q2 | Team 删除策略 | P0 只做 archive，不做 hard delete | 共识 |
| Q3 | 跨 team 身份 P0 做不做 | P0 钉方向（`users + memberships`），P1 实现 | gpt52：空 schema 是脚手架 |
| Q4 | Team 发现 | 默认 private，super-admin 全可见 | gpt52 安全优先 |
| Q5 | 跨 team 计费汇总 | 只做报表聚合，不做账本/额度合并 | 共识 |
| Q6 | Bridge 跨 team 共享 | **非目标** — 会击穿 provider_scope + 额度归因 + 自路由禁止 | gpt52：F017 已绑定 team+owner |

## 非目标（明确不做）

- 跨 team 额度映射/转移 — F017 已明确 "team 之间账本隔离"
- Bridge 跨 team 共享 — 会击穿 provider_scope、额度归因和 self-route invariants（F017 约束）
- 多 pool-node 联邦 — 超出当前架构范围
- RBAC 细粒度权限 — owner/member 二元模型目前够用
- 公开 Team 发现广场 — 当前规模不需要，默认 invite-only

## References

- [F017 Plan Pool spec](F017-plan-pool.md) — team 模型定义、部署保护
- [F018 Admin Enhancement spec](F018-plan-pool-admin.md) — 单 team 管理能力
- [F017 Deployment Guide](F017-deployment-guide.md) — bootstrap token 机制
- BACKLOG.md line 55 — 原"Team-level registration policy"条目
