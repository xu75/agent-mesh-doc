# F017 Phase 2 — Phase 1B: Routing + Hard-fail Implementation Plan

**Feature:** F017 — `docs/features/F017-phase2-oauth-proxy.md` §6.2
**Goal:** Pool 层集成 OAuth proxy bridge，多 bridge 能力路由正确，session 硬绑定生效（binding 失效 → 503，不 fallback）。
**Acceptance Criteria:**
- ~~AC-P2-07: oauth-proxy bridge 注册时上报 capabilities~~ ✅ done (PR #46)
- ~~AC-P2-08: gateway 将 tool_use 请求路由到 oauth/api bridge，不路由到 cli bridge~~ ✅ done (70a771d)
- ~~AC-P2-09: session 硬绑定: 同一 session 的请求始终走同一 bridge~~ ✅ done (82740e7, d5f2bb8)
- ~~AC-P2-10: bridge 离线 → 503 session_expired（不 fallback 到其他 bridge）~~ ✅ done (d5f2bb8)
- ~~AC-P2-11: 自路由条件放行~~ ✅ updated — `oauth-proxy` bridge 可声明 `allowOwnerRoute=true`；`cli-chat` 始终禁止；自路由独立并发 + 零金额记账（2026-04-15 CVO 决策）
- ~~AC-P2-19: gateway 支持 `x-api-key` header 认证（Claude Code 默认方式）~~ ✅ done (PR #45)
- ~~AC-P2-20: bridge auth unhealthy（token 刷新失败 / 持续 401/403）→ 触发 session 解绑~~ ✅ done (ebcffd5)
- ~~AC-P2-21: 含 structured input（`system[]`、thinking、非文本 content block）的请求不路由到 cli bridge~~ ✅ done (70a771d)
**Architecture:** plan-pool 层改造（pool.ts 路由引擎 + api-gateway.ts HTTP 层），不动 bridge 侧代码（plan-bridge-oauth 已发送 capabilities）。改动集中在 Layer 1 Governance（plan-pool）。
**Tech Stack:** TypeScript, node:test, Fastify, better-sqlite3
**前端验证:** No — 纯后端路由逻辑

---

## Terminal Schema（最终类型形态）

所有 Task 围绕这些类型构建，每步 extend-only，不重写。

```typescript
// pool.ts — 新增类型
export interface BridgeCapabilities {
  supportsTools: boolean;
  supportsStreaming: boolean;
  supportsStructuredInput: boolean;
  bridgeType: "cli-chat" | "api-proxy" | "oauth-proxy";
}

export const DEFAULT_CAPABILITIES: BridgeCapabilities = {
  supportsTools: false,
  supportsStreaming: false,
  supportsStructuredInput: false,
  bridgeType: "cli-chat",
};

// BridgeEntry — 新增字段
export interface BridgeEntry {
  // ... 现有字段不变
  capabilities: BridgeCapabilities;   // Task 2
  authHealthy: boolean;               // Task 6
}

// SelectBridgeRequest — 新增字段
export interface SelectBridgeRequest {
  // ... 现有字段不变
  requiresTools?: boolean;            // Task 3
  requiresStructuredInput?: boolean;  // Task 3
}

// selectBridge 返回类型变更 (Task 5)
// Old: BridgeEntry | null
// New: BridgeEntry | { sessionExpired: true } | null

// NoRouteReason — 新增
| "CAPABILITY_MISMATCH"              // Task 3
| "AUTH_UNHEALTHY"                   // Task 6
```

---

## Task 1: x-api-key 认证支持 (AC-P2-19)

**Files:**
- Modify: `packages/plan-pool/src/api-gateway.ts:73-90`
- Test: `packages/plan-pool/src/api-gateway.test.ts`

**Step 1: Write failing test**

```typescript
// api-gateway.test.ts — 在 describe("ApiGateway") 内新增
it("authenticates via x-api-key header (AC-P2-19)", async () => {
  const { apiKey } = tm.createTeam({
    name: "Lab-xapi",
    providerScope: "claude",
    ownerDisplayName: "Xapi",
    ownerPublicKey: "pk-xapi",
  });
  const resp = await app.inject({
    method: "GET",
    url: "/v1/models",
    headers: { "x-api-key": apiKey },
  });
  assert.equal(resp.statusCode, 200);
  const body = JSON.parse(resp.body);
  assert.ok(Array.isArray(body.data));
});

it("x-api-key takes precedence over Authorization header (AC-P2-19)", async () => {
  const { apiKey } = tm.createTeam({
    name: "Lab-xapi2",
    providerScope: "claude",
    ownerDisplayName: "Xapi2",
    ownerPublicKey: "pk-xapi2",
  });
  const resp = await app.inject({
    method: "GET",
    url: "/v1/models",
    headers: {
      "x-api-key": apiKey,
      authorization: "Bearer invalid-token",
    },
  });
  assert.equal(resp.statusCode, 200, "x-api-key should win over invalid Bearer");
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/plan-pool && npx tsx --test src/api-gateway.test.ts 2>&1 | tail -20`
Expected: FAIL — `x-api-key` not recognized, returns 401

**Step 3: Implement — rename + extend extractBearerToken**

```typescript
// api-gateway.ts:73-77 — replace extractBearerToken
function extractApiToken(req: FastifyRequest): string | null {
  // Priority 1: x-api-key (Claude Code default)
  const xApiKey = req.headers["x-api-key"];
  if (typeof xApiKey === "string" && xApiKey) return xApiKey;
  // Priority 2: Authorization: Bearer (existing clients)
  const auth = req.headers.authorization;
  if (auth?.startsWith("Bearer ")) return auth.slice(7);
  return null;
}
```

Update `authenticate()` at line 86-90:
```typescript
function authenticate(req: FastifyRequest): AuthResult | null {
  const token = extractApiToken(req);  // was extractBearerToken
  if (!token) return null;
  return teamManager.authenticateApiKey(token);
}
```

**Step 4: Run test to verify it passes**

Run: `cd packages/plan-pool && npx tsx --test src/api-gateway.test.ts 2>&1 | tail -20`
Expected: PASS

**Step 5: Commit**

```bash
git add packages/plan-pool/src/api-gateway.ts packages/plan-pool/src/api-gateway.test.ts
git commit -m "feat(F017-P2): AC-P2-19 — x-api-key header auth support"
```

---

## Task 2: BridgeCapabilities 类型 + 注册 (AC-P2-07)

**Files:**
- Modify: `packages/plan-pool/src/pool.ts:1-20` (types)
- Modify: `packages/plan-pool/src/api-gateway.ts:628-696` (registration endpoint)
- Test: `packages/plan-pool/src/pool.test.ts`
- Test: `packages/plan-pool/src/api-gateway.test.ts`

**Step 1: Write failing test — pool.test.ts**

```typescript
// pool.test.ts — 更新 makeBridge helper，新增 capabilities 测试
// 先更新 makeBridge 添加 capabilities 默认值（编译会失败因为 BridgeEntry 还没有该字段）

it("bridge with capabilities is stored and retrievable", () => {
  const router = new PoolRouter();
  const bridge = makeBridge("b-oauth", "alice", {
    capabilities: {
      supportsTools: true,
      supportsStreaming: true,
      supportsStructuredInput: true,
      bridgeType: "oauth-proxy",
    },
  });
  router.addBridge(bridge);
  const stored = router.getBridge("b-oauth");
  assert.ok(stored);
  assert.equal(stored.capabilities.supportsTools, true);
  assert.equal(stored.capabilities.bridgeType, "oauth-proxy");
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/plan-pool && npx tsx --test src/pool.test.ts 2>&1 | tail -20`
Expected: FAIL — `capabilities` not in BridgeEntry type

**Step 3: Implement — add types to pool.ts**

```typescript
// pool.ts — 在 ShareMode 之后新增
export interface BridgeCapabilities {
  supportsTools: boolean;
  supportsStreaming: boolean;
  supportsStructuredInput: boolean;
  bridgeType: "cli-chat" | "api-proxy" | "oauth-proxy";
}

export const DEFAULT_CAPABILITIES: BridgeCapabilities = {
  supportsTools: false,
  supportsStreaming: false,
  supportsStructuredInput: false,
  bridgeType: "cli-chat",
};
```

Add to BridgeEntry:
```typescript
export interface BridgeEntry {
  // ... existing fields
  capabilities: BridgeCapabilities;
}
```

Update `makeBridge` in pool.test.ts:
```typescript
function makeBridge(id: string, ownerId: string, overrides?: Partial<BridgeEntry>): BridgeEntry {
  return {
    // ... existing defaults
    capabilities: DEFAULT_CAPABILITIES,
    ...overrides,
  };
}
```

**Step 4: Run pool tests**

Run: `cd packages/plan-pool && npx tsx --test src/pool.test.ts 2>&1 | tail -20`
Expected: PASS

**Step 5: Write failing test — registration endpoint parses capabilities**

```typescript
// api-gateway.test.ts
it("POST /v1/bridges/register stores capabilities (AC-P2-07)", async () => {
  const ledger = new Ledger(poolDb.db);
  const router = new PoolRouter();
  const appWithRouter = await createApiGateway({
    db: poolDb.db, teamManager: tm, ledger, router,
  });
  const { apiKey } = tm.createTeam({
    name: "Lab-Cap",
    providerScope: "claude",
    ownerDisplayName: "Cap",
    ownerPublicKey: "pk-cap",
  });
  const resp = await appWithRouter.inject({
    method: "POST",
    url: "/v1/bridges/register",
    headers: { "x-api-key": apiKey },
    payload: {
      clientName: "bridge-oauth-1",
      provider: "claude",
      shareMode: "idle-only",
      externalMaxConcurrency: 2,
      attestation: { binaryHash: "oauth-proxy", version: "0.1.0" },
      capabilities: {
        supportsTools: true,
        supportsStreaming: true,
        supportsStructuredInput: true,
        bridgeType: "oauth-proxy",
      },
    },
  });
  assert.equal(resp.statusCode, 200);
  const body = JSON.parse(resp.body);
  const bridge = router.getBridge(body.bridgeId);
  assert.ok(bridge, "bridge should be in router");
  assert.equal(bridge.capabilities.supportsTools, true);
  assert.equal(bridge.capabilities.bridgeType, "oauth-proxy");
  await appWithRouter.close();
});

it("POST /v1/bridges/register defaults to cli-chat capabilities when omitted (AC-P2-07)", async () => {
  const ledger = new Ledger(poolDb.db);
  const router = new PoolRouter();
  const appWithRouter = await createApiGateway({
    db: poolDb.db, teamManager: tm, ledger, router,
  });
  const { apiKey } = tm.createTeam({
    name: "Lab-NoCap",
    providerScope: "claude",
    ownerDisplayName: "NoCap",
    ownerPublicKey: "pk-nocap",
  });
  const resp = await appWithRouter.inject({
    method: "POST",
    url: "/v1/bridges/register",
    headers: { "x-api-key": apiKey },
    payload: {
      clientName: "bridge-cli-1",
      provider: "claude",
      attestation: { binaryHash: "sha256-abc", version: "1.0.0" },
    },
  });
  assert.equal(resp.statusCode, 200);
  const body = JSON.parse(resp.body);
  const bridge = router.getBridge(body.bridgeId);
  assert.ok(bridge);
  assert.equal(bridge.capabilities.supportsTools, false);
  assert.equal(bridge.capabilities.bridgeType, "cli-chat");
  await appWithRouter.close();
});
```

**Step 6: Run test to verify it fails**

Run: `cd packages/plan-pool && npx tsx --test src/api-gateway.test.ts 2>&1 | tail -20`
Expected: FAIL — capabilities not parsed/stored

**Step 7: Implement — update registration endpoint**

```typescript
// api-gateway.ts:639-645 — extend body type
const body = req.body as {
  clientName?: string;
  provider?: string;
  shareMode?: string;
  externalMaxConcurrency?: number;
  attestation?: { binaryHash: string; version: string };
  capabilities?: Partial<BridgeCapabilities>;  // NEW
};
```

```typescript
// api-gateway.ts:672-688 — add capabilities to router.addBridge()
import { DEFAULT_CAPABILITIES, type BridgeCapabilities } from "./pool.js";

// Inside the registration handler, before router.addBridge:
const capabilities: BridgeCapabilities = {
  ...DEFAULT_CAPABILITIES,
  ...body.capabilities,
};

router.addBridge({
  // ... existing fields
  capabilities,  // NEW
});
```

**Step 8: Run all tests**

Run: `cd packages/plan-pool && npx tsx --test src/api-gateway.test.ts src/pool.test.ts 2>&1 | tail -20`
Expected: PASS

**Step 9: Commit**

```bash
git add packages/plan-pool/src/pool.ts packages/plan-pool/src/pool.test.ts packages/plan-pool/src/api-gateway.ts packages/plan-pool/src/api-gateway.test.ts
git commit -m "feat(F017-P2): AC-P2-07 — BridgeCapabilities type + registration"
```

---

## Task 3: Capability-based 路由 (AC-P2-08, AC-P2-21)

**Files:**
- Modify: `packages/plan-pool/src/pool.ts:108-127` (selectBridge), `pool.ts:204-260` (getRejectionReason)
- Modify: `packages/plan-pool/src/api-gateway.ts:430-456` (/v1/messages handler)
- Test: `packages/plan-pool/src/pool.test.ts`
- Test: `packages/plan-pool/src/api-gateway.test.ts`

**Step 1: Write failing test — pool 层 capability 过滤**

```typescript
// pool.test.ts
it("capability routing: rejects cli bridge for tool_use request (AC-P2-08)", () => {
  const router = new PoolRouter();
  router.addBridge(makeBridge("b-cli", "alice", {
    shareMode: "capped",
    capabilities: { ...DEFAULT_CAPABILITIES, bridgeType: "cli-chat" },
  }));
  const result = router.selectBridge({
    callerId: "bob",
    teamId: "team-1",
    provider: "claude",
    namespace: "ns-1",
    requiresTools: true,
  });
  assert.equal(result, null);
  assert.equal(router.diagnoseNoRoute({
    callerId: "bob", teamId: "team-1", provider: "claude",
    namespace: "ns-1", requiresTools: true,
  }), "CAPABILITY_MISMATCH");
});

it("capability routing: selects oauth bridge for tool_use request (AC-P2-08)", () => {
  const router = new PoolRouter();
  router.addBridge(makeBridge("b-cli", "alice", {
    shareMode: "capped",
    capabilities: { ...DEFAULT_CAPABILITIES, bridgeType: "cli-chat" },
  }));
  router.addBridge(makeBridge("b-oauth", "carol", {
    shareMode: "capped",
    capabilities: {
      supportsTools: true, supportsStreaming: true,
      supportsStructuredInput: true, bridgeType: "oauth-proxy",
    },
  }));
  const result = router.selectBridge({
    callerId: "bob", teamId: "team-1", provider: "claude",
    namespace: "ns-1", requiresTools: true,
  });
  assert.ok(result);
  assert.equal((result as BridgeEntry).id, "b-oauth");
});

it("capability routing: rejects cli bridge for structured input (AC-P2-21)", () => {
  const router = new PoolRouter();
  router.addBridge(makeBridge("b-cli", "alice", {
    shareMode: "capped",
    capabilities: { ...DEFAULT_CAPABILITIES, bridgeType: "cli-chat" },
  }));
  const result = router.selectBridge({
    callerId: "bob", teamId: "team-1", provider: "claude",
    namespace: "ns-1", requiresStructuredInput: true,
  });
  assert.equal(result, null);
});

it("capability routing: no-capability request routes to any bridge", () => {
  const router = new PoolRouter();
  router.addBridge(makeBridge("b-cli", "alice", {
    shareMode: "capped",
    capabilities: { ...DEFAULT_CAPABILITIES, bridgeType: "cli-chat" },
  }));
  const result = router.selectBridge({
    callerId: "bob", teamId: "team-1", provider: "claude", namespace: "ns-1",
  });
  assert.ok(result, "plain text request should route to cli bridge");
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/plan-pool && npx tsx --test src/pool.test.ts 2>&1 | tail -20`
Expected: FAIL — `requiresTools` not in SelectBridgeRequest / no capability check

**Step 3: Implement — extend SelectBridgeRequest + getRejectionReason**

```typescript
// pool.ts — SelectBridgeRequest 新增可选字段
export interface SelectBridgeRequest {
  callerId: string;
  teamId: string;
  provider: string;
  namespace: string;
  requiresTools?: boolean;
  requiresStructuredInput?: boolean;
}

// pool.ts — NoRouteReason 新增
export type NoRouteReason =
  // ... existing
  | "CAPABILITY_MISMATCH";

// pool.ts — getRejectionReason 新增 capability 检查（在 heartbeat 检查之后）
private getRejectionReason(
  bridge: BridgeEntry,
  callerId: string,
  req?: SelectBridgeRequest,
): NoRouteReason | null {
  // ... existing checks (heartbeat, self-route, capacity, shareMode)

  // Capability check (after shareMode, before return null)
  if (req?.requiresTools && !bridge.capabilities.supportsTools) return "CAPABILITY_MISMATCH";
  if (req?.requiresStructuredInput && !bridge.capabilities.supportsStructuredInput) return "CAPABILITY_MISMATCH";

  return null;
}
```

注意：`getRejectionReason` 签名需要传入 `req` 以获取 capability 需求。`canAcceptExternal` 和 `getAvailableBridges` 也需要传递 `req`。`selectBridge` 和 `diagnoseNoRoute` 已有 `req` 参数，向下传递即可。

**Step 4: Run pool tests**

Run: `cd packages/plan-pool && npx tsx --test src/pool.test.ts 2>&1 | tail -20`
Expected: PASS

**Step 5: Write failing test — gateway 层 capability 检测**

```typescript
// api-gateway.test.ts — 新增 describe block
describe("ApiGateway — capability routing (AC-P2-08, AC-P2-21)", () => {
  // Setup: router with cli + oauth bridges, ledger, proxyInvokeBridge mock
  // Test: POST /v1/messages with tools[] → routed to oauth bridge (not cli)
  // Test: POST /v1/messages with system[] → routed to oauth bridge
  // Test: POST /v1/messages plain text → routed to any bridge
});
```

具体测试需要 mock `proxyInvokeBridge` 来验证哪个 bridge 被选中。通过检查 `router.getBridge()` 的 `externalActiveSessions` 变化来间接验证路由目标。

**Step 6: Implement — gateway 层 request capability 检测**

```typescript
// api-gateway.ts — 新增 helper（在 buildProxyHeaders 附近）
function detectRequestCapabilities(body: unknown): {
  requiresTools: boolean;
  requiresStructuredInput: boolean;
} {
  const b = body as Record<string, unknown>;
  const requiresTools = Array.isArray(b.tools) && b.tools.length > 0;
  const requiresStructuredInput =
    Array.isArray(b.system) || // system[] 而非 string
    (Array.isArray(b.messages) && (b.messages as unknown[]).some((m: unknown) => {
      const msg = m as Record<string, unknown>;
      return Array.isArray(msg.content) && (msg.content as unknown[]).some((c: unknown) => {
        const block = c as Record<string, unknown>;
        return block.type !== "text";
      });
    }));
  return { requiresTools, requiresStructuredInput };
}
```

在 `/v1/messages` handler 中，`acquireRoutedLease` 调用前检测 capabilities 并传入：

```typescript
// api-gateway.ts — POST /v1/messages handler 内
const caps = detectRequestCapabilities(body);
const routed = acquireRoutedLease(auth, namespace, caps);
```

`acquireRoutedLease` 签名扩展：
```typescript
function acquireRoutedLease(
  auth: AuthResult,
  namespace: string,
  caps?: { requiresTools: boolean; requiresStructuredInput: boolean },
): ...
```

传递给 `router.selectBridge({ ...req, ...caps })`。

**Step 7: Run all tests**

Run: `cd packages/plan-pool && npx tsx --test src/pool.test.ts src/api-gateway.test.ts 2>&1 | tail -20`
Expected: PASS

**Step 8: Commit**

```bash
git add packages/plan-pool/src/pool.ts packages/plan-pool/src/pool.test.ts packages/plan-pool/src/api-gateway.ts packages/plan-pool/src/api-gateway.test.ts
git commit -m "feat(F017-P2): AC-P2-08/21 — capability-based routing"
```

---

## Task 4: Session affinity key 优先级 (AC-P2-09 partial)

**Files:**
- Modify: `packages/plan-pool/src/api-gateway.ts:448` (namespace resolution)
- Test: `packages/plan-pool/src/api-gateway.test.ts`

**Step 1: Write failing test**

```typescript
// api-gateway.test.ts
it("session affinity prefers X-Claude-Code-Session-Id over x-namespace (AC-P2-09)", async () => {
  // Setup: router + ledger + proxyInvokeBridge mock that records bridge selection
  const ledger = new Ledger(poolDb.db);
  const router = new PoolRouter();
  let capturedBridge: BridgeEntry | null = null;
  const proxyInvokeBridge = async (bridge: BridgeEntry, _req: ProxyBridgeRequest) => {
    capturedBridge = bridge;
    return { status: 200, headers: {}, body: '{"type":"message","content":[]}' } as ProxyBridgeResult;
  };
  const appR = await createApiGateway({
    db: poolDb.db, teamManager: tm, ledger, router, proxyInvokeBridge,
  });
  // Register two bridges
  const { apiKey, teamId, memberId } = tm.createTeam({
    name: "Lab-Session", providerScope: "claude",
    ownerDisplayName: "Sess", ownerPublicKey: "pk-sess",
  });
  // ... add bridges to router, make requests with X-Claude-Code-Session-Id
  // Verify same session ID → same bridge across requests
  await appR.close();
});
```

**Step 2: Run test to verify it fails**

Expected: FAIL — `X-Claude-Code-Session-Id` not used as namespace

**Step 3: Implement — namespace resolution priority**

```typescript
// api-gateway.ts:448 — 替换 namespace 解析（/v1/messages handler 内）
// Old:
const namespace = (req.headers["x-namespace"] as string) ?? crypto.randomUUID();
// New:
const namespace =
  (req.headers["x-claude-code-session-id"] as string) ??
  (req.headers["x-namespace"] as string) ??
  crypto.randomUUID();
```

同样更新 `/v1/messages/count_tokens`（line 526）和 `/v1/chat/completions`（line 568）中的 namespace 解析。

**Step 4: Run tests**

Run: `cd packages/plan-pool && npx tsx --test src/api-gateway.test.ts 2>&1 | tail -20`
Expected: PASS

**Step 5: Commit**

```bash
git add packages/plan-pool/src/api-gateway.ts packages/plan-pool/src/api-gateway.test.ts
git commit -m "feat(F017-P2): AC-P2-09 — session affinity key priority"
```

---

## Task 5: Hard-fail selectBridge + 503 session_expired (AC-P2-09, AC-P2-10)

**核心状态机反转**：binding 失效 → 503（不 fallback round-robin）。

**Files:**
- Modify: `packages/plan-pool/src/pool.ts:108-127` (selectBridge return type + logic)
- Modify: `packages/plan-pool/src/api-gateway.ts:225-268` (acquireRoutedLease 处理 sessionExpired)
- Test: `packages/plan-pool/src/pool.test.ts`
- Test: `packages/plan-pool/src/api-gateway.test.ts`

**Step 1: Write failing test — pool 层 hard-fail**

```typescript
// pool.test.ts
it("hard-fail: binding exists but bridge offline → sessionExpired (AC-P2-10)", () => {
  const router = new PoolRouter();
  const bridge = makeBridge("b-alice", "alice", { shareMode: "capped" });
  router.addBridge(bridge);
  const req: SelectBridgeRequest = {
    callerId: "bob", teamId: "team-1", provider: "claude", namespace: "conv-1",
  };
  // First request: bind to bridge
  const first = router.selectBridge(req);
  assert.ok(first);
  router.updateBinding(req, first as BridgeEntry);

  // Bridge goes offline (remove from pool)
  router.removeBridge("b-alice");

  // Second request: binding exists but bridge gone → hard-fail
  const second = router.selectBridge(req);
  assert.ok(second !== null && "sessionExpired" in (second as object),
    "should return sessionExpired, not null or fallback");
});

it("hard-fail: binding exists but bridge at capacity → sessionExpired (AC-P2-10)", () => {
  const router = new PoolRouter();
  router.addBridge(makeBridge("b-alice", "alice", {
    shareMode: "capped", externalActiveSessions: 1, externalMaxConcurrency: 1,
  }));
  router.addBridge(makeBridge("b-carol", "carol", { shareMode: "capped" }));
  const req: SelectBridgeRequest = {
    callerId: "bob", teamId: "team-1", provider: "claude", namespace: "conv-1",
  };
  // Bind to alice
  router.updateBinding(req, router.getBridge("b-alice")!);

  // Alice at capacity → hard-fail (don't fallback to carol)
  const result = router.selectBridge(req);
  assert.ok(result !== null && "sessionExpired" in (result as object));
});

it("no binding + no bridges → null (not sessionExpired)", () => {
  const router = new PoolRouter();
  const result = router.selectBridge({
    callerId: "bob", teamId: "team-1", provider: "claude", namespace: "ns-1",
  });
  assert.equal(result, null, "no binding → null for 429, not sessionExpired");
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/plan-pool && npx tsx --test src/pool.test.ts 2>&1 | tail -20`
Expected: FAIL — selectBridge returns null (fallback), not sessionExpired

**Step 3: Implement — selectBridge return type change**

```typescript
// pool.ts — selectBridge 返回类型
selectBridge(req: SelectBridgeRequest): BridgeEntry | { sessionExpired: true } | null {
  const stickyKey = `${req.callerId}:${req.provider}:${req.namespace}`;

  // 1. Check session-sticky binding
  const binding = this.bindings.get(stickyKey);
  if (binding) {
    const bridge = this.bridges.get(binding.bridgeId);
    if (bridge && bridge.teamId === req.teamId
        && bridge.provider === req.provider
        && this.canAcceptExternal(bridge, req.callerId, req)) {
      binding.lastActive = Date.now();
      return bridge;
    }
    // CHANGED: binding invalid → hard-fail (不删除 binding，不 fallback)
    return { sessionExpired: true };
  }

  // 2. No binding → round-robin (first request for this session)
  const available = this.getAvailableBridges(req.callerId, req.teamId, req.provider, req);
  if (available.length === 0) return null;
  return available[this.roundRobinIndex++ % available.length];
}
```

**Step 4: Update acquireRoutedLease to handle sessionExpired**

```typescript
// api-gateway.ts — acquireRoutedLease 内
const result = router.selectBridge({ callerId: auth.memberId, teamId: auth.teamId, provider: auth.providerScope, namespace, ...caps });

// NEW: hard-fail check
if (result && "sessionExpired" in result) {
  return { error: "session_expired", statusCode: 503 };
}
const bridge = result;
if (!bridge) {
  return { error: "RATE_LIMITED", statusCode: 429, reason: router.diagnoseNoRoute(...) };
}
```

**Step 5: 503 response format in /v1/messages handler**

```typescript
// api-gateway.ts — /v1/messages handler 内，处理 503 session_expired
if ("error" in proxyResult && proxyResult.error === "session_expired") {
  reply.status(503);
  reply.header("X-Session-Expired", "true");
  return {
    type: "error",
    error: {
      type: "session_expired",
      message: "Service temporarily unavailable. Please retry with a new session.",
    },
  };
}
```

**Step 6: Run all tests**

Run: `cd packages/plan-pool && npx tsx --test src/pool.test.ts src/api-gateway.test.ts 2>&1 | tail -20`
Expected: PASS

注意：现有 pool.test.ts 中 "session-sticky" 测试的 `selectBridge` 返回值类型可能需要调整（从 `BridgeEntry` 到 `BridgeEntry | { sessionExpired: true } | null`）。需要在断言中做类型窄化。

**Step 7: Commit**

```bash
git add packages/plan-pool/src/pool.ts packages/plan-pool/src/pool.test.ts packages/plan-pool/src/api-gateway.ts packages/plan-pool/src/api-gateway.test.ts
git commit -m "feat(F017-P2): AC-P2-09/10 — hard-fail selectBridge + 503 session_expired"
```

---

## Task 6: Auth unhealthy → session 解绑 (AC-P2-20)

**Files:**
- Modify: `packages/plan-pool/src/pool.ts` (BridgeEntry.authHealthy + unbindBridge + getRejectionReason)
- Modify: `packages/plan-pool/src/api-gateway.ts:628-696` (registration endpoint parses authHealthy)
- Test: `packages/plan-pool/src/pool.test.ts`
- Test: `packages/plan-pool/src/api-gateway.test.ts`

**Step 1: Write failing test — pool 层**

```typescript
// pool.test.ts
it("auth unhealthy bridge is rejected (AC-P2-20)", () => {
  const router = new PoolRouter();
  router.addBridge(makeBridge("b-alice", "alice", {
    shareMode: "capped", authHealthy: false,
  }));
  const result = router.selectBridge({
    callerId: "bob", teamId: "team-1", provider: "claude", namespace: "ns-1",
  });
  assert.equal(result, null);
  assert.equal(router.diagnoseNoRoute({
    callerId: "bob", teamId: "team-1", provider: "claude", namespace: "ns-1",
  }), "AUTH_UNHEALTHY");
});

it("auth unhealthy triggers session unbind (AC-P2-20)", () => {
  const router = new PoolRouter();
  const bridge = makeBridge("b-alice", "alice", { shareMode: "capped" });
  router.addBridge(bridge);
  const req: SelectBridgeRequest = {
    callerId: "bob", teamId: "team-1", provider: "claude", namespace: "conv-1",
  };
  const first = router.selectBridge(req);
  assert.ok(first);
  router.updateBinding(req, first as BridgeEntry);
  assert.equal(router.bindingCount, 1);

  // Bridge reports auth unhealthy → unbind
  router.markAuthUnhealthy("b-alice");
  assert.equal(router.bindingCount, 0, "bindings to unhealthy bridge should be removed");
});
```

**Step 2: Run test to verify it fails**

Expected: FAIL — `authHealthy` not in BridgeEntry, `markAuthUnhealthy` not defined

**Step 3: Implement**

```typescript
// pool.ts — BridgeEntry 新增
export interface BridgeEntry {
  // ... existing
  capabilities: BridgeCapabilities;
  authHealthy: boolean;
}

// pool.ts — PoolRouter 新增方法
markAuthUnhealthy(bridgeId: string): void {
  const bridge = this.bridges.get(bridgeId);
  if (bridge) {
    bridge.authHealthy = false;
    // Unbind all sessions pointing to this bridge
    for (const [key, binding] of this.bindings) {
      if (binding.bridgeId === bridgeId) {
        this.bindings.delete(key);
      }
    }
  }
}

// pool.ts — getRejectionReason 新增（在 heartbeat 检查之后）
if (!bridge.authHealthy) return "AUTH_UNHEALTHY";

// pool.ts — NoRouteReason 新增
| "AUTH_UNHEALTHY"
```

Update `makeBridge` in pool.test.ts: add `authHealthy: true` default.

**Step 4: Write failing test — registration endpoint parses authHealthy**

```typescript
// api-gateway.test.ts
it("POST /v1/bridges/register with authHealthy=false triggers unbind (AC-P2-20)", async () => {
  // Setup router with bridge + binding, then re-register with authHealthy: false
  // Verify binding removed
});
```

**Step 5: Implement — registration endpoint**

```typescript
// api-gateway.ts — registration body type 新增
authHealthy?: boolean;

// router.addBridge 调用中新增
authHealthy: body.authHealthy !== false, // default true

// 当 authHealthy=false 时调用 router.markAuthUnhealthy
if (body.authHealthy === false && router) {
  router.markAuthUnhealthy(assignedId);
}
```

**Step 6: Run all tests**

Run: `cd packages/plan-pool && npx tsx --test src/pool.test.ts src/api-gateway.test.ts 2>&1 | tail -20`
Expected: PASS

**Step 7: Commit**

```bash
git add packages/plan-pool/src/pool.ts packages/plan-pool/src/pool.test.ts packages/plan-pool/src/api-gateway.ts packages/plan-pool/src/api-gateway.test.ts
git commit -m "feat(F017-P2): AC-P2-20 — auth unhealthy triggers session unbind"
```

---

## Task 7: Self-route 回归验证 (AC-P2-11)

**Files:**
- Test: `packages/plan-pool/src/pool.test.ts`

**Step 1: Write regression test**

```typescript
// pool.test.ts — 在所有新功能之后验证 self-route deny 仍然生效
it("self-route deny still works with capabilities + hard-fail (AC-P2-11)", () => {
  const router = new PoolRouter();
  router.addBridge(makeBridge("b-alice", "alice", {
    shareMode: "capped",
    capabilities: {
      supportsTools: true, supportsStreaming: true,
      supportsStructuredInput: true, bridgeType: "oauth-proxy",
    },
  }));
  // Alice requests with tools → should NOT route to own bridge
  const result = router.selectBridge({
    callerId: "alice", teamId: "team-1", provider: "claude",
    namespace: "ns-1", requiresTools: true,
  });
  assert.equal(result, null);
  assert.equal(router.diagnoseNoRoute({
    callerId: "alice", teamId: "team-1", provider: "claude",
    namespace: "ns-1", requiresTools: true,
  }), "SELF_ROUTE_DENIED");
});

it("self-route deny: binding to own bridge → sessionExpired (AC-P2-11)", () => {
  const router = new PoolRouter();
  const bridge = makeBridge("b-alice", "alice", { shareMode: "capped" });
  router.addBridge(bridge);
  // Manually create a binding (simulating a race or admin override)
  router.updateBinding(
    { callerId: "alice", teamId: "team-1", provider: "claude", namespace: "ns-1" },
    bridge,
  );
  // Self-route with existing binding → hard-fail (not fallback)
  const result = router.selectBridge({
    callerId: "alice", teamId: "team-1", provider: "claude", namespace: "ns-1",
  });
  assert.ok(result !== null && "sessionExpired" in (result as object),
    "self-route with binding should hard-fail, not silently succeed");
});
```

**Step 2: Run test**

Run: `cd packages/plan-pool && npx tsx --test src/pool.test.ts 2>&1 | tail -20`
Expected: PASS（回归验证，不需要新代码）

**Step 3: Commit**

```bash
git add packages/plan-pool/src/pool.test.ts
git commit -m "test(F017-P2): AC-P2-11 — self-route deny regression with capabilities + hard-fail"
```

---

## 依赖关系 & 执行顺序

```
Task 1 (x-api-key)  ──┐
Task 2 (capabilities) ─┼─→ Task 3 (capability routing) ─┐
Task 4 (session key)  ─┘                                 ├─→ Task 7 (regression)
Task 5 (hard-fail)    ─→ Task 6 (auth unhealthy) ────────┘
```

Tasks 1, 2, 4, 5 无互相依赖，可并行开发（但建议按顺序 commit 保持 git 历史清晰）。
Task 3 依赖 Task 2（需要 BridgeCapabilities 类型）。
Task 6 依赖 Task 2（authHealthy 字段）和 Task 5（hard-fail 语义）。
Task 7 是回归验证，最后执行。

## 不动的文件

- `packages/plan-bridge-oauth/` — 已在 Phase 1A 完成 capabilities 声明，不需要改动
- `team.ts`, `credit.ts`, `ledger.ts`, `circuit-breaker.ts` — 不涉及
- `format-anthropic.ts`, `format-openai.ts` — 不涉及
- `plan-bridge-cli/` — 不涉及（cli bridge 不发送 capabilities，使用 DEFAULT_CAPABILITIES）
