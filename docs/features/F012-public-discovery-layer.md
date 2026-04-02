---
feature_ids: [F012]
related_features: [F005, F010]
topics: [discovery, landing-page, directory, openapi, agent-card, onboarding]
doc_kind: spec
created: 2026-04-02
---

# F012: Public Discovery Layer（公开发现层）

> Status: done | Owner: 宪宪 | Completed: 2026-04-02

## Why

当前外部 Agent/人类想了解 mesh-hub 有什么节点和能力，**唯一的结构化接口是 `POST /v1/caps`**，但它需要先完成 HELLO 握手拿到 L1 证书。这造成鸡生蛋问题：不加入就看不到，看不到就不知道值不值得加入。

人类用浏览器访问 hub URL 看到的是空白或 JSON，没有任何使用说明。

**目标**：让 mesh-hub 对外可发现——Agent 能机器可读地了解协议和能力，人类能浏览器直接看到使用说明。

## What

### Phase 1：公开发现，不开放注册

四个交付物 + 一个修复：

#### 1. `GET /` — HTML Landing Page

人类用浏览器访问 hub URL 时看到的页面：
- Hub 名称、简介、协议版本
- Getting Started（密钥生成 → HELLO → 使用流程）
- 在线节点概览（动态，从内存状态读取）
- API 端点参考表
- 底部指向机器可读端点

实现：基于 hub 现有的 Fastify 路由，纯模板函数拼 HTML。不引入前端框架、不需要打包工具。
数据来源与 `/v1/discovery` 和 `/v1/directory` 同源。

#### 2. `GET /v1/discovery` — 机器可读 Hub 自描述

**私有协议发现端点**（非 A2A `/.well-known/agent-card.json`）。无需认证，可被 HTTP cache。

```jsonc
{
  "name": "Cat Cafe Mesh Hub",
  "description": "Multi-agent collaboration mesh",
  "protocol": { "name": "clowder", "version": "v1" },
  "authentication": {
    "type": "ed25519-hello-handshake",
    "keyFormat": "JWK",
    "curve": "Ed25519"
  },
  "endpoints": {
    "hello":     { "method": "POST", "path": "/v1/hello" },
    "caps":      { "method": "POST", "path": "/v1/caps",      "auth": "L1" },
    "token":     { "method": "POST", "path": "/v1/token",     "auth": "L1" },
    "verify":    { "method": "POST", "path": "/v1/verify",    "auth": "none" },
    "invoke":    { "method": "POST", "path": "/v1/invoke",    "auth": "L2" },
    "heartbeat": { "method": "POST", "path": "/v1/heartbeat", "auth": "L1" },
    "directory": { "method": "GET",  "path": "/v1/directory", "auth": "none" },
    "pubkey":    { "method": "GET",  "path": "/v1/pubkey",    "auth": "none" }
  },
  "registration": {
    "mode": "whitelist",
    "contact": "admin@catcafe.dev"
  },
  "openapi": "/v1/openapi.json"
}
```

设计决策：
- **不用 `/.well-known/agent-card.json`**：我们没有 A2A binding，挂 A2A 标准路径等于立假招牌。A2A 兼容留到 Phase 3。（GPT-5.4 审查意见，两猫一致同意）
- **不含动态数据**（在线节点数等）：A2A 文档明确 Agent Card 是低频缓存文档，动态数据走 `/v1/directory`。
- **不含 hubId**：Agent 已通过 URL 定位 hub，多 hub 联邦是 hub 间的事。
- **端点列表覆盖完整 native client flow**：除 discovery/directory 外，还需声明 `token`、`verify`、`pubkey`，否则外部实现者仍然拿不到完整接入路径。

#### 3. `GET /v1/directory` — 匿名公开能力目录

无需认证的动态能力目录，与 `POST /v1/caps` 数据同源但做信息降级：

```jsonc
{
  "nodes": [
    {
      "catId": "ragdoll-opus",
      "displayName": "布偶猫/宪宪",
      "description": "主架构师，擅长系统设计",
      "skills": ["architecture", "code-review"],
      "status": "online"
    }
  ]
}
```

与 `/v1/caps` 的区别：
| | `/v1/directory` | `/v1/caps` |
|--|-----------------|-----------|
| 认证 | 无 | L1 证书 |
| 暴露 nodeId | 否 | 是 |
| 可用于 invoke | 否 | 是 |
| 目的 | 展示"我们有什么" | 运行时发现 + 调用 |

实现约束：
- `/v1/directory` 返回**完整 roster + status**，由调用方决定是否只展示 `online/stale`；Landing Page 默认只展示非 `offline` 节点。
- `displayName` 和 `description` 应作为**新增可选字段**或单独的 `DirectoryNode` 视图模型引入，不能破坏现有 `/v1/caps` 消费方。

#### 4. `GET /v1/openapi.json` — API Schema

从 hub 的 **HTTP DTO / schema** 生成 OpenAPI 3.0 spec，让任意语言的 Agent 可以 codegen 客户端。

约束：
- **不能直接从 `mesh-protocol/messages.ts` 生成**。该文件描述的是协议消息联合类型，不等于 hub 的 HTTP 请求/响应形状；例如 `/v1/hello` 的 HTTP body 已经和 `HelloRequest` 不同。
- OpenAPI 的 source of truth 应落在 `mesh-hub` 侧的显式 schema/DTO，而不是复用讨论期的协议消息类型。

#### 5. 协议漂移修复

`mesh-protocol/messages.ts` 的 `NodeCapability.status` 定义为 `"online" | "offline" | "degraded"`，但 hub 实际返回的是 `"stale"`。公开发现层上线前必须统一。

修复方案：`status: "online" | "stale" | "offline"`（与 hub 实际行为对齐）。

### Phase 2（后续，不在本次范围）

- Self-service 准入（CLI `mesh-hub add-node` 或 `POST /v1/onboard`）
- 多语言（Python/Go/Rust）密钥生成 + HELLO 示例
- CapabilityCard 结构化升级（inputSchema / outputSchema）
- 错误码文档页面

### Phase 3（更远期）

- A2A 兼容层：`/.well-known/agent-card.json` + A2A binding
- `/dashboard` 统计面板（invoke 次数、延迟、错误率）
- 发现事件 webhook（新节点上线通知）

## Acceptance Criteria

- [x] AC-1: 浏览器访问 `GET /` 返回 HTML Landing Page，包含 hub 简介、Getting Started、在线节点列表（默认展示非 `offline` 节点）、API 参考
- [x] AC-2: `GET /v1/discovery` 返回 JSON 自描述文档，包含协议、认证方式、端点列表、注册模式
- [x] AC-3: `GET /v1/directory` 无认证返回所有已知节点的 `{ catId, displayName, description, skills, status }`，不暴露 nodeId 和 callbackUrl
- [x] AC-4: `GET /v1/openapi.json` 返回有效的 OpenAPI 3.0 spec
- [x] AC-5: `NodeCapability.status` 类型定义与 hub 实际返回值一致（`"online" | "stale" | "offline"`）
- [x] AC-6: Landing Page 的在线节点列表与 `/v1/directory` 数据一致（同源）
- [x] AC-7: 所有新端点有单元测试覆盖

## Dependencies

- F001（Security Foundation）— L0/L1/L2 身份体系已建立 ✅
- F003（Capability Model）— 多能力注册已支持 ✅
- F010（Mesh-Cat Cafe Integration）— 互补，不阻塞

## Risk

| 风险 | 缓解 |
|------|------|
| Landing Page 成为维护负担 | 纯模板函数，数据从内存状态动态读取，不需要手动更新内容 |
| `/v1/directory` 暴露过多信息 | Phase 1 只暴露 catId/skills/status，不含 nodeId/callbackUrl |
| OpenAPI spec 与实际 API 漂移 | 从 hub 侧显式 HTTP schema/DTO 生成，不从 `messages.ts` 直接推导 |

## Open Questions

1. ~~A2A `/.well-known/agent-card.json` 是否 Phase 1？~~ → **已决定：Phase 3**（两猫一致 + 铲屎官拍板）
2. ~~`registration.mode` 写 "open" 还是 "whitelist"？~~ → **已决定：写 "whitelist"**（诚实反映现状）
3. ~~`/v1/directory` 的 `description` 字段数据来源？~~ → **已决定：Phase 1 走 hub config 可选字段，缺省不返回**（GPT-5.4 建议，后续再讨论 adapter 注册覆盖）
4. ~~Landing Page 是否需要中英双语？~~ → **已决定：Phase 1 单语（英文），双语留后续迭代**（GPT-5.4 建议，不把 UI 文案工程化问题带进主线）

## Key Decisions

| # | 决策 | 理由 | 参与者 |
|---|------|------|--------|
| D1 | 不用 `/.well-known/agent-card.json`，用 `/v1/discovery` | 无 A2A binding 不应占标准路径 | 宪宪 + GPT-5.4 |
| D2 | 动态数据不放 discovery 端点 | Agent Card 应可缓存，动态数据走 `/v1/directory` | GPT-5.4 提出 |
| D3 | `registration.mode: "whitelist"` | 代码事实是 allowlist，文档不能说 open | GPT-5.4 提出 |
| D4 | 先开放，安全未来再收 | 铲屎官拍板 | 铲屎官 |
| D5 | 不含 hubId | Agent 已知 URL，多 hub 联邦是 hub 间的事 | 铲屎官提出 |
| D6 | `description` Phase 1 走 config 可选字段 | 简单起步，adapter 覆盖留后续 | GPT-5.4 建议 |
| D7 | Landing Page Phase 1 单语（英文） | 不把文案工程化带进 discovery 主线 | GPT-5.4 建议 |
