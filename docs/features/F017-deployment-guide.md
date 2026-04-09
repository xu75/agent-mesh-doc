---
feature_ids: [F017]
topics: [plan-pool, deployment, grey-release]
doc_kind: deployment-guide
created: 2026-04-08
parent: docs/features/F017-plan-pool.md
---

# F017: Plan Pool Deployment Guide

> Grey Release Plan for Phase 1.5 MVP
> 当前灰度入口仍是 `http://<pool>:3011` 直连；正式对外入口已裁定为标准 `443` 反向代理，内部 app 端口保持 `3011`。

## 1. Deployment Topology

```
              LAN / VPN
    ┌──────────────────────────────┐
    │                              │
    │   Hub Server (NAS / VPS)     │
    │   ┌────────────────────┐     │
    │   │ mesh-hub   :3010   │     │  ← existing, loopback OK
    │   │ pool-node  :3011   │     │  ← app port（灰度阶段外部直连）
    │   └────────────────────┘     │
    │   ┌────────────────────┐     │
    │   │ public API   :443  │     │  ← 正式入口（reverse proxy → 3011）
    │   └────────────────────┘     │
    │              │               │
    ├──────────────┼───────────────┤
    │              │               │
    │   ┌──────────┴──────────┐    │
    │   │   http://127.0.0.1    │    │  ← Hub↔Pool 走 loopback
    │   │       :3010         │    │
    │   └─────────────────────┘    │
    │                              │
    └────────┬──────────┬──────────┘
             │          │
      ┌──────┴───┐ ┌────┴─────┐
      │ Bridge   │ │ Consumer │
      │ (铲屎官) │ │ (小伙伴) │
      │ LAN/VPN  │ │ LAN/VPN  │
      └──────────┘ └──────────┘
```

## 2. Grey Release Phases

### Phase A: Pure Consumer Validation

| 角色 | 谁 | 做什么 | 需要装什么 |
|------|----|--------|-----------|
| Admin + Contributor | 铲屎官 | 跑 bridge + 管理 team | pool-node + bridge-cli |
| Pure Consumer | 小伙伴 A | 调 API 测试 | 无（curl / Claude Code） |
| Pure Consumer | 小伙伴 B | 调 API 测试（不同客户端） | 无（Cursor / 其他） |

**Phase A 目标**：验证 pool→bridge 主链路 + consumer quickstart 流程

### Phase B: Contributor Installation

| 角色 | 谁 | 做什么 |
|------|----|--------|
| Contributor | 小伙伴 A | 安装 bridge-cli，贡献额度 |
| Consumer | 铲屎官 | 现在可以消费别人的 bridge 了 |

**Phase B 目标**：验证 bridge 安装流程 + 双向互助 + self-route deny

## 3. Port Allocation

| Service | Port | Listen | Note |
|---------|------|--------|------|
| mesh-hub | 3010 | 127.0.0.1 | Existing, loopback |
| pool-node | 3011 | **0.0.0.0** | External (LAN/VPN) |
| bridge-cli | — | — | Outbound WS to Hub (no listen port) |
| bridge-api | — | — | Outbound WS to Hub (no listen port) |
| Cat Cafe frontend | 3003 | — | Unrelated |
| Cat Cafe API | 3004 | — | Unrelated |

## 3.1 Public Entry Decision (Post Grey)

- **当前灰度现状**：consumer / contributor 直接访问 `http://<pool>:3011`。
- **正式目标入口**：对外统一收敛到标准 `443`，由 reverse proxy 转发到内部 `3011`。
- **架构裁定**：保留 pool-node 独立进程；优先独立 API 域名/子域名，不把 provider-native `/v1/messages` 与 mesh-hub control plane 混在同一个 origin。
- **文档约定**：灰度 runbook 继续保留 `3011` 示例；面向正式对外接口的 spec/README 应优先写公共 base URL。

## 4. Environment Variables

### pool-node (Hub Server)

```bash
# === Required ===
POOL_LISTEN_HOST=0.0.0.0          # Must be 0.0.0.0 for external access
POOL_LISTEN_PORT=3011              # Default
MESH_HUB_URL=http://127.0.0.1:3010  # Hub on same machine

# === Optional (defaults shown) ===
POOL_NODE_ID=plan-pool
POOL_DB_PATH=./pool.sqlite
POOL_IDENTITY_DIR=.pool-identity
POOL_HEARTBEAT_MS=10000
POOL_LEASE_EXPIRE_INTERVAL_MS=60000
POOL_BINDING_CLEANUP_INTERVAL_MS=300000
```

### bridge-cli (Contributor / Provider Credential Machine)

> **Transport**: Bridge connects outbound to Hub via WebSocket (`/v1/ws`). No port
> forwarding or SSH tunnel needed — bridge works from any network with outbound
> HTTP/WS access to Hub. Hub dispatches INVOKE over the WS connection.

```bash
# === Required ===
MESH_HUB_URL=http://<hub-ip>:3010         # Hub address (reachable from credential machine via tunnel/relay in grey release)
POOL_NODE_URL=http://<hub-ip>:3011        # Pool-node address
POOL_API_KEY=pool_<your-member-key>       # Your member API key

# === Optional (defaults shown) ===
PLAN_BRIDGE_NODE_ID=plan-bridge-cli
PLAN_BRIDGE_CLI_BINARY=claude            # Must be in PATH
PLAN_BRIDGE_MAX_BUDGET_USD=0.5
PLAN_BRIDGE_MAX_CONCURRENCY=3
PLAN_BRIDGE_SESSION_TTL_MINUTES=30
POOL_PROVIDER=claude
POOL_ENDPOINT_ALLOWLIST=                 # Comma-separated, empty = official only
```

### bridge-api (API Key Contributor)

```bash
# === Required ===
MESH_HUB_URL=http://<hub-ip>:3010
PLAN_BRIDGE_API_KEY=sk-xxx               # Provider API key (NOT pool key)
POOL_NODE_URL=http://<hub-ip>:3011
POOL_API_KEY=pool_<your-member-key>

# === Optional (defaults shown) ===
PLAN_BRIDGE_API_NODE_ID=plan-bridge-api
PLAN_BRIDGE_API_PROVIDER=gpt
PLAN_BRIDGE_API_TARGET_URL=https://api.openai.com/v1/chat/completions
PLAN_BRIDGE_API_MAX_CONCURRENCY=5
```

## 5. Static Pages (pool-node)

### 5.1 Page Scope

| Path | Type | Auth | Priority | Content |
|------|------|------|----------|---------|
| `GET /` | Landing + Join | None | **P0** | Team intro + join form + consumer quickstart |
| `GET /admin` | Admin | Owner API key | P1 | Team management (invite/usage/suspend) |
| `GET /download` | Download | None | P2 | Bridge artifact list + install instructions |
| `GET /download/install.sh` | Script | None | P2 | One-click bridge installer |
| `GET /download/<artifact>` | Binary | None | P2 | Platform-specific tarball |

Page priority rationale: Landing/Join is the grey release main path (pure consumer). Admin is secondary. Download is Phase B only.

### 5.2 Landing Page (`/`) — **P0, Grey Release Critical Path**

Content:
- Team name + description
- **Interactive join form** (invite code + displayName → call `/v1/join` → show API key once)
- Consumer quickstart (copy-pastable Claude Code / Cursor / curl config)
- Link to `/download` for contributors
- **No secrets, no member list, no usage data**
- API key shown only to the person who just joined, in a one-time display

### 5.3 Admin Page (`/admin`)

Single HTML file, vanilla JS + fetch(). Auth via **owner API key** in sessionStorage (no separate admin token — see Section 11).

| Feature | API | UI |
|---------|-----|-----|
| Generate invite | POST /v1/admin/invites | Button → copyable code |
| Member table | GET /v1/admin/usage | Table: name / status / contributed / consumed / balance |
| Suspend member | PUT /v1/admin/members/:id/suspend | Toggle button |
| Adjust borrow limit | PUT /v1/admin/members/:id/borrow-limit | Inline edit (any member, including owner self) |
| Rotate key | POST /v1/admin/members/:id/rotate-key | Button + confirm |

**Not included** (Phase 2):
- Real-time monitoring / charts
- Bridge management page
- Member self-service portal

### 5.4 Download Page (`/download`)

Content:
- Bridge version
- Supported platforms (macOS arm64, macOS x64, Linux x64)
- Prerequisites (Node.js >= 20, Claude CLI / provider CLI)
- One-liner install command: `curl -fsSL http://<pool>:3011/download/install.sh | bash`
- Manual download links for tarballs
- **No API keys or secrets on this page**

## 6. Artifact Naming

```
plan-bridge-cli-<version>-<platform>-<arch>.tar.gz
plan-bridge-api-<version>-<platform>-<arch>.tar.gz

Examples:
  plan-bridge-cli-0.0.1-darwin-arm64.tar.gz
  plan-bridge-cli-0.0.1-darwin-x64.tar.gz
  plan-bridge-cli-0.0.1-linux-x64.tar.gz
  plan-bridge-api-0.0.1-darwin-arm64.tar.gz
  ...
```

Tarball contents:
```
plan-bridge-cli-0.0.1-darwin-arm64/
├── bin/
│   └── plan-bridge-cli           # esbuild bundled entry
├── node_modules/
│   └── better-sqlite3/           # Native addon (bridge-cli needs it? No — only pool needs sqlite)
├── .env.template                 # Template with all env vars
└── README.md                     # Quick setup guide
```

Note: bridge-cli and bridge-api do NOT use SQLite. They are pure JS bundles — no native addons needed. Only pool-node uses better-sqlite3.

## 7. install.sh Responsibilities

```bash
#!/bin/bash
# Plan Bridge Installer

# 1. Detect platform + arch
# 2. Check prerequisites
#    - node >= 20 (exit with install instructions if missing)
#    - claude CLI in PATH (for bridge-cli) or warn
# 3. Download tarball from pool-node
#    - curl -fsSL http://<pool>/download/plan-bridge-cli-<ver>-<platform>-<arch>.tar.gz
# 4. Extract to ~/.plan-bridge/ (or user-specified dir)
# 5. Generate .env from template
#    - Prompt for: MESH_HUB_URL, POOL_NODE_URL, POOL_API_KEY
#    - Pre-fill defaults where possible
# 6. Create service file
#    - macOS: ~/Library/LaunchAgents/com.agent-mesh.plan-bridge.plist
#    - Linux: ~/.config/systemd/user/plan-bridge.service
# 7. Print next steps
#    - "Run: launchctl load ..." or "systemctl --user start plan-bridge"
```

**Security**: install.sh does NOT contain any secrets. API key is entered by user during setup or pasted into .env manually.

## 8. Consumer Quickstart

For team members who only want to use the pool (no bridge):

```bash
# Step 1: Get invite code from admin (out of band — chat/email)

# Step 2: Join the team
curl -s -X POST http://<pool>:3011/v1/join \
  -H "Content-Type: application/json" \
  -d '{
    "code": "inv_xxxxxx",
    "displayName": "YourName",
    "publicKey": "optional-ed25519-pubkey"
  }'
# Response: {"memberId":"m-abc","apiKey":"pool_f4e5d6...","teamName":"Lab-Claude","borrowLimit":3}
# ⚠️  Save your apiKey! It's shown only once.

# Step 3: Test with curl
curl -s -X POST http://<pool>:3011/v1/messages \
  -H "Authorization: Bearer pool_f4e5d6..." \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Hello, Plan Pool!"}]}'

# Step 4: Configure your AI tool
# Claude Code:  ANTHROPIC_BASE_URL=http://<pool>:3011  ANTHROPIC_API_KEY=pool_f4e5d6...
# Cursor:       Settings → Models → Custom API Base URL = http://<pool>:3011
# Continue:     config.json → provider.apiBase = "http://<pool>:3011"
```

## 9. Admin Quickstart

```bash
POOL=http://<pool-ip>:3011

# Step 1: Create team (unprotected — returns owner API key)
curl -s -X POST $POOL/v1/admin/teams \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Lab-Claude",
    "providerScope": "claude",
    "ownerDisplayName": "铲屎官",
    "ownerPublicKey": "<your-ed25519-public-key>"
  }'
# → {"teamId":"t-xxx","memberId":"m-owner","apiKey":"pool_abc123..."}
#    ⚠️ Save this apiKey — it's your admin + consumer + bridge credential

OWNER_KEY=pool_abc123...  # The key from above

# Step 2: Generate invite code (single use, 48h TTL)
curl -s -X POST $POOL/v1/admin/invites \
  -H "Authorization: Bearer $OWNER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"maxUses":1,"ttlHours":48}'
# → {"code":"inv_xxxxxx"}  ← Send this to the invitee

# Step 3: Check usage
curl -s $POOL/v1/admin/usage \
  -H "Authorization: Bearer $OWNER_KEY" | jq .

# Step 4: Suspend a member
curl -s -X PUT $POOL/v1/admin/members/<member-id>/suspend \
  -H "Authorization: Bearer $OWNER_KEY"

# Step 5: Adjust borrow limit
curl -s -X PUT $POOL/v1/admin/members/<member-id>/borrow-limit \
  -H "Authorization: Bearer $OWNER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"limitUsd":10}'
```

## 10. Identity Model Summary

```
per-member API key (pool_xxx...)
    │
    ├── Join team:     POST /v1/join → returns this key
    │
    ├── Register bridge: POST /v1/bridges/register
    │                    Authorization: Bearer pool_xxx
    │                    → pool records ownerMemberId = you
    │
    └── Call API:      POST /v1/messages
                       Authorization: Bearer pool_xxx
                       → pool identifies caller = you
                       → self-route deny if you own the bridge
```

One key, three uses. No separate credentials needed.

## 11. Admin Model (Owner = Admin)

There is **no separate admin token**. The admin model works like this:

1. **Bootstrap**: `POST /v1/admin/teams` is unprotected (no auth) — anyone can create a team
2. **Team creation** returns the owner member's API key — this is the "admin credential"
3. **All other admin routes** require `role: "owner"` — checked via the owner's member API key
4. **Only the team owner** can create invites, suspend members, adjust limits, etc. `borrow-limit` applies to any member record, including the owner's own member.

```bash
# Step 1: Create team (unprotected bootstrap)
curl -X POST http://pool:3011/v1/admin/teams \
  -d '{"name":"Lab-Claude","providerScope":"claude","ownerDisplayName":"铲屎官","ownerPublicKey":"ed25519..."}'
# → {"teamId":"t-xxx","memberId":"m-owner","apiKey":"pool_owner_key..."}
#    ↑ This apiKey is your admin credential

# Step 2: All admin operations use this key
curl -X POST http://pool:3011/v1/admin/invites \
  -H "Authorization: Bearer pool_owner_key..."
```

> **Security**: `POST /v1/admin/teams` is currently unprotected. Before grey release on `0.0.0.0`, we MUST add bootstrap protection.

### Bootstrap Protection (must-fix before grey release)

**Two-layer approach** (both are minimal code):

1. **First-team-only gate** (default): After the first team is created, `POST /v1/admin/teams` returns 403. No env var needed.
   ```typescript
   // In api-gateway.ts, before team creation
   const existingTeams = teamManager.listTeams();
   if (existingTeams.length > 0) {
     return reply.status(403).send({ error: "bootstrap closed — team already exists" });
   }
   ```

2. **Optional POOL_BOOTSTRAP_TOKEN** env var: If set, `POST /v1/admin/teams` requires `Authorization: Bearer <token>`. Allows creating multiple teams.
   ```typescript
   const bootstrapToken = process.env.POOL_BOOTSTRAP_TOKEN;
   if (bootstrapToken && extractBearerToken(req) !== bootstrapToken) {
     return reply.status(401).send({ error: "bootstrap token required" });
   }
   ```

For grey release: first-team-only is sufficient. POOL_BOOTSTRAP_TOKEN is insurance for multi-team setups.

## 12. Deployment Checklist

### Pre-deployment
- [ ] Add `"start": "node dist/index.js"` to all three package.json
- [ ] Verify admin token configuration mechanism
- [ ] Build all three packages (`pnpm build`)
- [ ] Create systemd unit for pool-node

### Phase A Deployment
- [ ] Deploy pool-node on Hub server (0.0.0.0:3011)
- [ ] Deploy bridge-cli on 铲屎官 machine
- [ ] Create test team via admin API / admin page
- [ ] Generate invite codes for 2 testers
- [ ] Invite testers → verify consumer quickstart flow
- [ ] Smoke test: consumer → pool → bridge → CLI → response

### Phase B Deployment
- [ ] Tester installs bridge-cli via install.sh
- [ ] Verify bridge auto-registration
- [ ] Verify 铲屎官 can now consume (self-route deny allows others' bridges)
- [ ] Verify mutual credit accounting
