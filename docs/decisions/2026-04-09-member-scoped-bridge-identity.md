---
feature_ids: [F017]
topics: [bridge-identity, registration, routing, security, plan-pool]
doc_kind: decision
created: 2026-04-09
---

# Member-Scoped Bridge Identity (2026-04-09)

## Status

Accepted

## Context

plan-bridge-cli 的 `PLAN_BRIDGE_NODE_ID` 默认值硬编码为 `"plan-bridge-cli"`。此值直接作为 `bridgeId` 发送到 plan-pool 注册接口，而 `bridges` 表以 `id` (即 bridgeId) 作为全局 PRIMARY KEY。

当多个 contributor 使用默认安装流程时：

1. **同 team 冲突**：第二个 member 的 `registerBridge()` UPDATE 覆盖第一个 member 的 `owner_member_id`，导致第一人的 bridge 被劫持
2. **跨 team 冲突**：第二个 team 的 member UPDATE 覆盖第一个 team 的 `team_id`，造成跨 team 数据污染
3. **UPDATE 无归属校验**：`registerBridge()` 的 UPDATE 语句仅按 `WHERE id = ?` 匹配，不校验 `team_id` 和 `owner_member_id`，任何知道 bridgeId 的人都能覆写

此外，`install.sh` 生成的 `.env` 中完全没有 `PLAN_BRIDGE_NODE_ID` 配置项，所有安装实例都使用默认值。

## Decision

### 1. Server-Assigned Bridge ID + Member-Scoped Uniqueness

- `bridges.id` 改为 server 生成的 UUID，不再由 client 控制
- 新增 `client_name` 列存储 client 提供的名称（原 `bridgeId` 字段值）
- 添加 `UNIQUE(team_id, owner_member_id, client_name)` 约束
- `registerBridge()` 按 `(team_id, owner_member_id, client_name)` 查找，而非按 `id` 查找

### 2. Registration UPDATE 加归属校验

- UPDATE 语句必须包含 `WHERE team_id = ? AND owner_member_id = ?`
- 不同 member 发送相同 `client_name` 时创建独立的 bridge 行
- 同 member 发送相同 `client_name` 时视为重注册（heartbeat / attestation check）

### 3. Client 默认值改为唯一

- `PLAN_BRIDGE_NODE_ID` 默认值从 `"plan-bridge-cli"` 改为 `${hostname}-${random}`
- `install.sh` 自动生成唯一的 `PLAN_BRIDGE_NODE_ID` 写入 `.env`
- `download.html` 手动安装说明同步更新

### 4. API 兼容性

- 注册接口仍接受 `bridgeId` 字段（语义变为 client-provided name）
- 返回值新增 `assignedBridgeId` 字段（server-assigned UUID）
- PoolRouter、Ledger、invocations 全部使用 server-assigned ID 做路由和记账

## Consequences

### Positive

- 多 contributor 默认安装不再冲突
- Server 兜底保证 bridge 归属隔离，即使 client 重名
- Attestation / suspend / rotate-key 语义不受影响（均基于 server-assigned ID）

### Negative

- Schema 变更需要 migration（新建表 + 迁移数据）
- 已注册的 bridge 数据需要迁移：原 `id` 变为 `client_name`，生成新 UUID 作为 `id`

### Neutral

- `bridgeId` 和 `nodeId` 保持耦合（client 侧同一个值），仅在 pool 侧解耦为 server-assigned ID
- 向后兼容：API 字段名不变，仅语义调整

## Affected Components

| Component | Change |
|-----------|--------|
| `packages/plan-pool/src/db.ts` | Schema: `bridges` 表结构变更 + migration |
| `packages/plan-pool/src/team.ts` | `registerBridge()` 查找逻辑 + 返回值 |
| `packages/plan-pool/src/api-gateway.ts` | 注册接口返回 `assignedBridgeId`; router sync 用 server ID |
| `packages/plan-bridge-cli/src/bridge.ts` | 默认 `NODE_ID` 改为 hostname-random |
| `packages/plan-pool/public/download/install.sh` | `.env` 生成唯一 NODE_ID |
| `packages/plan-pool/public/download.html` | 文档同步 |
| `docs/features/F017-plan-pool.md` | Schema 说明 amendment |

## References

- Root cause: `bridge.ts:23` (`NODE_ID` 硬编码) + `team.ts:229-230` (全局 PK 查找) + `db.ts:53-54` (无复合约束)
- Architecture check: Section 8 Q3 "Does it change registration, routing, or identity semantics?" → Yes → ADR required
