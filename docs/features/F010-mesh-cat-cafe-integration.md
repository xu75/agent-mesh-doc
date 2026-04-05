---
feature_ids: [F010]
related_features: [F001, F002, F004, F007, F009]
topics: [integration, cat-cafe, mesh-bridge, mcp-tools]
doc_kind: spec
created: 2026-04-02
---

# F010: Mesh-Cat Cafe Integration (信使模式 MVP)

> Status: complete | Owner: 宪宪

## Why

Agent Mesh 的 protocol/node/adapter/hub 四个 package 已经 production-verified，但还没有真正的 runtime 接入。Cat Cafe 是第一个接入方，需要验证完整的调用链：本地猫 -> Mesh Hub -> 远端节点 -> 响应返回。

## What

让 Cat Cafe 的猫猫通过 MCP tool 直接使用 mesh 能力：发现远端 agent、向远端发送任务并获取结果。

### 完整调用链

```
Cat (Claude Code session)
  → MCP tool (mesh_discover / mesh_invoke)
    → Sidecar HTTP service (localhost:3005)  [MVP test shim]
      → @agent-mesh/bridge (ClowderPluginAdapter + MeshClient)
        → Hub (mesh-hub.xyz, HTTPS + Ed25519 L0/L1/L2)
          → Target node callback (echo-node / future real agent)
            → Response 原路返回
```

### 交互模式

当前实现的是 **信使模式（Messenger / B0）**：
- 单次 invoke，read-only，本地猫中介
- 远端不能修改本地文件
- 本地猫把结果翻译给铲屎官

未来路线（按交付顺序）：
- B1: 伙伴模式 — 远端大脑 + 本地手脚，host-driven tool loop
- B2: STREAM — 流式返回
- C: 能力市场 — 专业服务（TTS、图片生成等）

## 已完成的工作

### 1. @agent-mesh/bridge package（从实验提升为正式 package）

**路径**: `packages/mesh-bridge/`

MeshBridge 类组合 ClowderPluginAdapter + MeshClient + MeshServer，提供统一的 init/invokeRemote/caps/startServer/close 接口。

- 依赖: `@agent-mesh/adapter` + `@agent-mesh/node` + `@agent-mesh/protocol`
- TLS 透传: `MeshBridgeConfig.tls` → `MeshClient`
- v0 Clowder-first: adapter 当前硬编码为 Clowder 语义，第二个 runtime 出现时再泛化
- 测试: 6 个集成测试（本地 Hub），全部通过（含 targetCatId e2e）
- 心跳: MeshBridge.startHeartbeat()/stopHeartbeat() 代理 MeshClient 内建心跳（`8a7b0ad`）

**Commits**:
- `a1dcfaa` feat(bridge): promote MeshBridge from experiment to @agent-mesh/bridge package
- `d0b6029` fix(bridge): add TLS passthrough, fix stale test comment
- `2c8b5b1` docs(bridge): add Clowder-first scope note to package.json

### 2. Cat Cafe MCP Tools（Clowder-AI 侧）

**路径**: `Clowder-AI/packages/mcp-server/src/tools/mesh-tools.ts`（在 `feat/mesh-mcp-tools` 分支）

两个工具：
- `mesh_discover` — 发现 mesh 上的远端 agent（调用 sidecar GET /discover）
- `mesh_invoke` — 向远端 agent 发送任务（调用 sidecar POST /invoke）

**关键发现**: MCP SDK (`@modelcontextprotocol/sdk`) 的 `server.tool()` 方法不接受 JSON Schema 对象作为参数 schema——会被 `isZodRawShapeCompat` 误判为 annotations。必须用 **Zod raw shape** 格式注册工具参数。

Clowder-AI 改动极小（3 个文件），核心运行时（API、前端、数据库）完全未动。新 Claude Code session 启动时 MCP server 自动加载新编译的 dist/，约等于热更新。

### 3. Hub 基础设施

**节点注册**（`/etc/mesh-hub/config.json`）：

| nodeId | catId | skills | 用途 |
|--------|-------|--------|------|
| cat-cafe-node | ragdoll-xianxian | architecture, coding, system-design | Cat Cafe 本地节点 |
| mesh-echo-node | mesh-echo | echo, ping | 回声测试节点 |
| mesh-reviewer-node | mesh-reviewer | code-review, static-analysis | 真实 code review 能力节点 |

**本地身份**: `.mesh-identity/identity.json`（Ed25519 keypair，**不入 git**）
- nodeId: cat-cafe-node
- publicKey: `VvUfevm33jM_c30f2b5w4u3-dORC5PnH3k1W_we9D40`

**Hub 服务**:
- `mesh-hub.service` — Mesh Hub（systemd, auto-restart）
- `mesh-echo.service` — Echo 测试节点（systemd, auto-restart, heartbeat 10s）
- `mesh-reviewer.service` — Code Review 节点（systemd, auto-restart, heartbeat 10s, port 3007）

**域名注意**: `mesh-hub.xyz` 会 301 重定向到 `www.mesh-hub.xyz`。Node.js fetch 默认不跟随重定向，必须用 `https://www.mesh-hub.xyz`。

### 4. Sidecar（测试用 shim）

**路径**: `experiments/e2-mesh-bridge/sidecar.ts`

轻量 HTTP 服务包装 MeshBridge，让 Clowder-AI 通过 fetch 调用 mesh 能力，避免跨 repo 依赖。

**明确定位**: TEST SHIM / MVP smoke-test only。长期方案是 Clowder-AI 直接依赖 `@agent-mesh/bridge` package。

启动方式：
```bash
MESH_HUB_URL=https://www.mesh-hub.xyz npx tsx experiments/e2-mesh-bridge/sidecar.ts
```

### 5. Reviewer Node（真实能力节点）

**路径**: `experiments/e2-mesh-bridge/reviewer-node.ts`

接受代码片段，执行静态分析，返回结构化 code review（P1/P2/P3 severity + verdict）。这是 F010 "remote cat does a real task" 的关键证据——不是 echo，是真实的代码审查能力。

- `reviewCode()` 检查: 安全（eval/innerHTML/empty-catch/destructive-ops）、类型安全（any/non-null-assertion）、代码质量（console.log/TODO）、可维护性（行数/行宽/exports）
- 返回: `ReviewResult { reviewer, verdict, findings[], summary, linesAnalyzed }`
- verdict: `APPROVE` | `APPROVE_WITH_COMMENTS` | `REQUEST_CHANGES`

### 6. E2E 验证证据

```
mesh_discover → 3 agents online (cat-cafe-node + mesh-echo-node + mesh-reviewer-node)
mesh_invoke   → echo 成功返回
  traceId:      815f0d4c-a2b9-449b-b60e-0c3e508cac5e
  latency:      0ms
mesh_invoke   → reviewer 真实 code review 返回
  traceId:      d68015ef-3bd2-4d1c-a5b8-cb1c7d83d4ed
  latency:      1ms
  verdict:      REQUEST_CHANGES (detected 3×P1, 2×P2, 1×P3)
  findings:     eval() injection, innerHTML XSS, empty catch, console.log, TODO, no exports
```

## 分层边界

```
agent-mesh repo (通用层):
  @agent-mesh/protocol  — 消息类型定义
  @agent-mesh/node      — MeshClient / MeshServer / Identity
  @agent-mesh/adapter   — Clowder <-> Mesh 语义翻译
  @agent-mesh/bridge    — 组合层（adapter + client + server）
  @agent-mesh/hub       — Hub 服务端

Clowder-AI repo (产品集成层):
  mesh-tools.ts         — MCP tool 定义（mesh_discover / mesh_invoke）
  server-toolsets.ts    — 注册入口

experiments/ (测试基础设施):
  sidecar.ts            — HTTP shim（仅测试用）
  echo-node.ts          — 回声测试节点
  reviewer-node.ts      — 真实 code review 能力节点
  run.ts                — 生产 E2E 脚本
  test-hello.ts         — HELLO 连通性测试
```

## 操作手册

### 测试 mesh（铲屎官视角）

1. 启动 sidecar: `MESH_HUB_URL=https://www.mesh-hub.xyz npx tsx experiments/e2-mesh-bridge/sidecar.ts`
2. 在 Cat Cafe 里对任意猫说："帮我看看 mesh 上有哪些远端 agent"
3. 或："用 mesh 给 mesh-echo-node 发一条消息：你好"

### 查看 Hub 日志

```bash
ssh mesh-hub "sudo journalctl -u mesh-hub -f --no-pager"    # Hub 日志
ssh mesh-hub "sudo journalctl -u mesh-echo -f --no-pager"   # Echo 节点日志
```

### 端口冲突

如果 sidecar 报 EADDRINUSE:
```bash
lsof -ti:3005 | xargs kill
```

## 已知限制与后续

- [ ] Sidecar 是临时方案，需替换为 Clowder-AI 直接依赖 @agent-mesh/bridge
- [ ] `startServer()` 的真实 inbound 集成测试缺失（当前只测了 adapter 映射）
- [x] ~~echo-node 只是回声~~ → mesh-reviewer-node 提供真实 code review 能力（`8a7b0ad`+）
- [x] ~~heartbeat 逻辑在 sidecar/echo-node 里手写~~ → MeshClient.startHeartbeat() 已内建（`8a7b0ad`）
- [ ] 伙伴模式（B1）的 host-driven tool loop 待设计
