# F017-WS Hub Dual-Transport Implementation Plan

**Feature:** F017-WS — `docs/features/F017-ws-dispatch.md`
**Goal:** Hub 传输层升级为 outbound WebSocket + HTTP callback 双传输，消除 SSH 隧道依赖，所有 mesh 节点受益
**Acceptance Criteria:** AC-1 ~ AC-15（见 spec），plan 逐条覆盖
**Architecture:** Hub `/v1/invoke` 治理逻辑不变，仅把"最后一跳"从 `fetch(callbackUrl)` 扩展为 WS 优先 + callback fallback。节点通过 outbound WS 连 Hub，Hub 管理连接池。pool-node 零改动。
**Tech Stack:** Node.js native `ws` (via Fastify websocket plugin), `jose` (existing L2 verification)
**前端验证:** No — 纯后端 infra 改动

---

## Terminal Schema (Final Form)

```typescript
// ── packages/mesh-hub/src/ws-registry.ts ──

interface PendingInvocation {
  traceId: string;
  resolve: (result: WsResultMessage) => void;
  reject: (error: Error) => void;
  timer: ReturnType<typeof setTimeout>;
}

interface WsSession {
  nodeId: string;
  ws: WebSocket;
  meshCertificate: string;
  connectedAt: number;
  pending: Map<string, PendingInvocation>;
}

// Hub → Node WS messages
type HubWsMessage =
  | { type: "WS_REGISTERED"; ok: true }
  | { type: "WS_REGISTERED"; ok: false; error: string }
  | { type: "INVOKE"; traceId: string; invocationToken: string;
      sourceNodeId: string; payload: { task: string; context?: string; scope: string[] } }
  | { type: "CANCEL"; traceId: string };

// Node → Hub WS messages
type NodeWsMessage =
  | { type: "WS_REGISTER"; nodeId: string; meshCertificate: string }
  | { type: "RESULT"; traceId: string; status: "success" | "error";
      payload: string; durationMs: number };

class WsRegistry {
  has(nodeId: string): boolean;
  register(nodeId: string, ws: WebSocket, meshCertificate: string): void;
  remove(nodeId: string): void;
  dispatch(nodeId: string, payload: object, timeoutMs: number): Promise<WsResultMessage>;
  handleResult(nodeId: string, msg: WsResultMessage): void;
  closeByNodeId(nodeId: string, code: number, reason: string): void;
  destroy(): void;
}

// ── packages/mesh-node/src/ws-channel.ts ──

interface WsChannelConfig {
  hubUrl: string;
  nodeId: string;
  handler: InvocationHandler;
  hubPublicKeyJwk?: JsonWebKey;
  reconnectMaxMs?: number;
}

class WsChannel {
  constructor(config: WsChannelConfig);
  connect(meshCertificate: string): Promise<void>;
  close(): void;
  isConnected(): boolean;
}
```

---

## Task 1: WsRegistry — Hub 侧 WS 连接注册表

**Files:**
- Create: `packages/mesh-hub/src/ws-registry.ts`
- Test: `packages/mesh-hub/src/ws-registry.test.ts`

**Covers:** AC-1 (partial), AC-7, AC-9, AC-10

### Step 1: Write failing test — register + has + remove

```typescript
// ws-registry.test.ts
it("registers a WS session and reports has()", () => {
  const registry = new WsRegistry();
  const mockWs = createMockWs();
  registry.register("node-a", mockWs, "cert-a");
  assert.equal(registry.has("node-a"), true);
  assert.equal(registry.has("node-b"), false);
  registry.remove("node-a");
  assert.equal(registry.has("node-a"), false);
});
```

### Step 2: Run test → FAIL (`WsRegistry` not defined)

```bash
cd packages/mesh-hub && npx tsx --test src/ws-registry.test.ts
```

### Step 3: Implement WsRegistry skeleton

Implement `WsRegistry` class with `register`, `has`, `remove` using `Map<string, WsSession>`.

### Step 4: Run test → PASS

### Step 5: Write failing test — dispatch + RESULT

```typescript
it("dispatches INVOKE via WS and resolves on RESULT", async () => {
  const registry = new WsRegistry();
  const mockWs = createMockWs();
  registry.register("node-a", mockWs, "cert-a");
  const dispatchPromise = registry.dispatch("node-a", { type: "INVOKE", traceId: "t1", ... }, 5000);
  // Simulate RESULT coming back
  registry.handleResult("node-a", { type: "RESULT", traceId: "t1", status: "success", payload: "{}", durationMs: 100 });
  const result = await dispatchPromise;
  assert.equal(result.status, "success");
  // Verify WS received the INVOKE
  assert.equal(mockWs.sent.length, 1);
});
```

### Step 6: Run test → FAIL

### Step 7: Implement dispatch + handleResult

`dispatch` creates a `PendingInvocation` with Promise, sends INVOKE via `ws.send()`, returns Promise. `handleResult` looks up pending by traceId, calls `resolve`.

### Step 8: Run test → PASS

### Step 9: Write failing test — dispatch timeout

```typescript
it("rejects dispatch after timeout and sends CANCEL", async () => {
  const registry = new WsRegistry();
  const mockWs = createMockWs();
  registry.register("node-a", mockWs, "cert-a");
  await assert.rejects(
    registry.dispatch("node-a", { ... traceId: "t2" ... }, 50), // 50ms timeout
    /timeout/,
  );
  // CANCEL should have been sent
  const cancelMsg = mockWs.sent.find(m => JSON.parse(m).type === "CANCEL");
  assert.ok(cancelMsg);
});
```

### Step 10: Run test → FAIL

### Step 11: Implement timeout + CANCEL

In `dispatch`, `setTimeout` → reject + send CANCEL + clean up pending.

### Step 12: Run test → PASS

### Step 13: Write failing test — disconnect clears pending

```typescript
it("rejects all pending invocations on disconnect", async () => {
  const registry = new WsRegistry();
  const mockWs = createMockWs();
  registry.register("node-a", mockWs, "cert-a");
  const p1 = registry.dispatch("node-a", { ...traceId: "t3"... }, 5000);
  registry.remove("node-a"); // simulate disconnect
  await assert.rejects(p1, /disconnected/);
});
```

### Step 14: Run test → FAIL

### Step 15: Implement disconnect cleanup

`remove` iterates all pending invocations for the node, calls `reject(new Error("disconnected"))`.

### Step 16: Run test → PASS

### Step 17: Write failing test — closeByNodeId

```typescript
it("closeByNodeId sends close frame with custom code", () => {
  const registry = new WsRegistry();
  const mockWs = createMockWs();
  registry.register("node-a", mockWs, "cert-a");
  registry.closeByNodeId("node-a", 4001, "credential_revoked");
  assert.equal(mockWs.closedWith?.code, 4001);
  assert.equal(registry.has("node-a"), false);
});
```

### Step 18: Run test → FAIL → Implement → PASS

### Step 19: Commit

```bash
git add packages/mesh-hub/src/ws-registry.ts packages/mesh-hub/src/ws-registry.test.ts
git commit -m "feat(mesh-hub): add WsRegistry for dual-transport dispatch [宪宪/Opus-46🐾]"
```

---

## Task 2: Hub WS Endpoint + Dual-Transport Invoke

**Files:**
- Modify: `packages/mesh-hub/src/hub.ts`
- Modify: `packages/mesh-hub/src/hub.test.ts`
- Dependency: `ws-registry.ts` from Task 1

**Covers:** AC-1, AC-2, AC-3, AC-6, AC-14

### Step 20: Write failing test — WS upgrade + WS_REGISTER

```typescript
// hub.test.ts — new describe block
describe("WebSocket transport", () => {
  it("accepts WS connection at /v1/ws and processes WS_REGISTER", async () => {
    // HELLO first to get meshCertificate
    const helloRes = await doHello(app, nodeBIdentity);
    const { meshCertificate } = helloRes;
    // Connect WS
    const ws = new WebSocket(`ws://127.0.0.1:${port}/v1/ws`);
    ws.send(JSON.stringify({ type: "WS_REGISTER", nodeId: "node-b", meshCertificate }));
    const reply = await waitForMessage(ws);
    assert.deepEqual(reply, { type: "WS_REGISTERED", ok: true });
    ws.close();
  });
});
```

### Step 21: Run test → FAIL (no /v1/ws route)

### Step 22: Install @fastify/websocket + add WS endpoint to hub.ts

```bash
cd packages/mesh-hub && pnpm add @fastify/websocket
```

Add to `hub.ts`:
1. `import websocket from "@fastify/websocket"`
2. `await app.register(websocket)`
3. `/v1/ws` route with WS upgrade, WS_REGISTER validation (verify meshCertificate against identity service)
4. Instantiate `WsRegistry` in `createHub`

### Step 23: Run test → PASS

### Step 24: Write failing test — WS_REGISTER with invalid cert

```typescript
it("rejects WS_REGISTER with invalid meshCertificate", async () => {
  const ws = new WebSocket(`ws://127.0.0.1:${port}/v1/ws`);
  ws.send(JSON.stringify({ type: "WS_REGISTER", nodeId: "node-b", meshCertificate: "bad" }));
  const reply = await waitForMessage(ws);
  assert.equal(reply.ok, false);
});
```

### Step 25: Run test → FAIL → Implement cert validation → PASS

### Step 26: Write failing test — invoke via WS dispatch

```typescript
it("POST /v1/invoke dispatches via WS when node has WS session", async () => {
  // 1. HELLO for node-b → get cert
  // 2. WS connect + WS_REGISTER for node-b
  // 3. node-a gets L2 token targeting node-b
  // 4. POST /v1/invoke → Hub should dispatch via WS (not callback)
  // 5. Simulate node-b receiving INVOKE on WS, sending RESULT
  // 6. Verify caller gets the result
});
```

### Step 27: Run test → FAIL

### Step 28: Modify hub.ts `/v1/invoke` handler

Change the dispatch section (~L497-589):
```typescript
// After all L2 validation, audit recording...
if (wsRegistry.has(targetNodeId)) {
  // WS path
  try {
    const result = await wsRegistry.dispatch(targetNodeId, forwardPayload, config.invokeTimeoutMs);
    metrics.recordInvocation("success", Date.now() - invokeStartMs);
    auditStore.record({ event: "INVOKE_RESULT", traceId, timestamp: Date.now(), invocationId, status: "success" });
    return { type: "RESULT", traceId, invocationId, ...result };
  } catch (err) { ... }
} else if (targetNode?.callbackUrl) {
  // Existing callback path (unchanged)
  ...
} else {
  return reply.code(502).send({ error: "target node has no registered transport" });
}
```

### Step 29: Run test → PASS

### Step 30: Write failing test — invoke fallback to callback when no WS

```typescript
it("POST /v1/invoke falls back to callback when no WS session", async () => {
  // Same as existing invoke test — node-b has callbackUrl, no WS
  // Should work exactly as before
});
```

### Step 31: Run test → PASS (existing callback path unchanged)

### Step 32: Write failing test — WS liveness integration

```typescript
it("WS connection updates node liveness (heartbeat alignment)", async () => {
  // 1. WS_REGISTER
  // 2. Verify node status is "online"
  // 3. WS disconnect
  // 4. Verify node status degrades
});
```

### Step 33: Run test → FAIL → Implement WS ping/liveness hook → PASS

### Step 34: Run full existing test suite

```bash
cd packages/mesh-hub && pnpm test
```
All existing tests must still pass (AC-14: callback path unbroken).

### Step 35: Commit

```bash
git add packages/mesh-hub/
git commit -m "feat(mesh-hub): add /v1/ws endpoint and dual-transport invoke dispatch [宪宪/Opus-46🐾]"
```

---

## Task 3: WsChannel — Node 侧 outbound WS 连接

**Files:**
- Create: `packages/mesh-node/src/ws-channel.ts`
- Create: `packages/mesh-node/src/ws-channel.test.ts`
- Modify: `packages/mesh-node/src/index.ts`

**Covers:** AC-2, AC-4, AC-5

### Step 36: Write failing test — WS_REGISTER + receive INVOKE

```typescript
it("connects to Hub WS, sends WS_REGISTER, and receives INVOKE", async () => {
  // Start mock WS server
  const mockHub = createMockWsServer();
  const handler = async (inv: InboundInvocation) => ({ status: "success", payload: "{}", durationMs: 50 });
  const channel = new WsChannel({ hubUrl: mockHub.url, nodeId: "test-node", handler });
  await channel.connect("valid-cert");
  // Mock hub should have received WS_REGISTER
  const regMsg = mockHub.lastReceived();
  assert.equal(regMsg.type, "WS_REGISTER");
  // Mock hub sends INVOKE
  mockHub.send({ type: "INVOKE", traceId: "t1", invocationToken: "tok", sourceNodeId: "caller", payload: { task: "hello", scope: ["read"] } });
  // Channel should process and send RESULT
  const result = await mockHub.waitForMessage();
  assert.equal(result.type, "RESULT");
  assert.equal(result.traceId, "t1");
  channel.close();
});
```

### Step 37: Run test → FAIL

### Step 38: Implement WsChannel core

- Connect via `new WebSocket(hubUrl)`
- Send WS_REGISTER on open
- Listen for messages: INVOKE → call handler → send RESULT
- L2 verification: verify `invocationToken` using hub public key (reuse MeshServer's verification logic)

### Step 39: Run test → PASS

### Step 40: Write failing test — L2 token verification in WS mode

```typescript
it("rejects INVOKE with invalid invocationToken", async () => {
  // Send INVOKE with bad token → RESULT(error) should be sent back
});
```

### Step 41: Run test → FAIL → Implement L2 verification → PASS

### Step 42: Write failing test — CANCEL handling

```typescript
it("responds to CANCEL by not sending RESULT for cancelled traceId", async () => {
  // Start a long-running handler
  // Send CANCEL mid-flight
  // Verify handler is signaled (via AbortSignal or similar)
});
```

### Step 43: Run test → FAIL → Implement CANCEL support → PASS

### Step 44: Write failing test — reconnect with exponential backoff

```typescript
it("reconnects with exponential backoff after disconnect", async () => {
  // Connect → disconnect → verify reconnect attempts
  // Verify backoff: 1s, 2s, 4s...
});
```

### Step 45: Run test → FAIL → Implement reconnect logic → PASS

### Step 46: Write failing test — no reconnect on close code 4001

```typescript
it("does not reconnect on close code 4001 (credential_revoked)", async () => {
  // Connect → close with code 4001 → verify no reconnect
});
```

### Step 47: Run test → FAIL → Implement close code check → PASS

### Step 48: Export WsChannel from index.ts

```typescript
// packages/mesh-node/src/index.ts
export { WsChannel } from "./ws-channel.js";
export type { WsChannelConfig } from "./ws-channel.js";
```

### Step 49: Commit

```bash
git add packages/mesh-node/src/ws-channel.ts packages/mesh-node/src/ws-channel.test.ts packages/mesh-node/src/index.ts
git commit -m "feat(mesh-node): add WsChannel for outbound WS transport [宪宪/Opus-46🐾]"
```

---

## Task 4: plan-bridge-cli — 迁移到 WsChannel

**Files:**
- Modify: `packages/plan-bridge-cli/src/bridge.ts`
- Modify: `packages/plan-bridge-cli/src/bridge.test.ts` (if exists)

**Covers:** AC-8, AC-11

### Step 50: Write failing test — bridge uses WsChannel instead of MeshServer

```typescript
it("bridge connects via WsChannel and handles INVOKE", async () => {
  // Start mock Hub with WS support
  // Start bridge pointing at mock hub
  // Verify: HELLO → WS_REGISTER → bridge is ready
  // Send INVOKE via WS → verify CLI is spawned → RESULT returned
});
```

### Step 51: Run test → FAIL

### Step 52: Refactor bridge.ts

Remove:
- `MeshServer` instantiation and `server.start()`
- `server.setVerificationKey()` call
- Remove `PLAN_BRIDGE_PORT` usage

Add:
- After `client.hello()`, fetch Hub pubkey, create `WsChannel` with handler
- `channel.connect(meshCertificate)`
- CANCEL support: handler receives AbortSignal, calls `child.kill('SIGTERM')`

Keep:
- `MeshClient.hello()` for L1 cert (still needed for WS_REGISTER)
- Pool registration (`POST /v1/bridges/register` to pool-node)
- Identity, attestation, endpoint validation, session manager, CLI wrapper

### Step 53: Run test → PASS

### Step 54: Run existing bridge tests

```bash
cd packages/plan-bridge-cli && pnpm test
```

### Step 55: Commit

```bash
git add packages/plan-bridge-cli/
git commit -m "feat(plan-bridge-cli): migrate from MeshServer to WsChannel [宪宪/Opus-46🐾]"
```

---

## Task 5: plan-bridge-api — 迁移到 WsChannel

**Files:**
- Modify: `packages/plan-bridge-api/src/bridge.ts`
- Modify: `packages/plan-bridge-api/src/proxy.ts` (add AbortController)

**Covers:** AC-8, AC-12

### Step 56: Write failing test — bridge-api uses WsChannel

Same pattern as Task 4 but with API proxy handler.

### Step 57: Run test → FAIL

### Step 58: Refactor bridge-api bridge.ts + proxy.ts

- `bridge.ts`: same refactor as cli — remove MeshServer, add WsChannel
- `proxy.ts`: add `AbortController` + `signal` to fetch calls for CANCEL support

### Step 59: Run test → PASS

### Step 60: Commit

```bash
git add packages/plan-bridge-api/
git commit -m "feat(plan-bridge-api): migrate from MeshServer to WsChannel [宪宪/Opus-46🐾]"
```

---

## Task 6: 端到端集成测试

**Files:**
- Modify: `packages/mesh-hub/src/hub.test.ts`

**Covers:** AC-13

### Step 61: Write e2e test

```typescript
describe("E2E: WS dual-transport invoke", () => {
  it("caller → Hub → [WS] → handler → RESULT", async () => {
    // 1. Hub with two nodes (caller: node-a, target: node-b)
    // 2. node-b connects via WS
    // 3. node-a requests token, posts /v1/invoke
    // 4. Hub dispatches via WS to node-b
    // 5. node-b handler processes, returns RESULT via WS
    // 6. Caller receives response
    // Verify: audit trail has INVOKE_SENT + INVOKE_RESULT
  });

  it("invoke falls back to callback when WS disconnects mid-flight", async () => {
    // node-b has both WS + callbackUrl
    // node-b WS disconnects → Hub falls back to callback
  });
});
```

### Step 62: Run test → FAIL → wire up → PASS

### Step 63: Commit

```bash
git commit -m "test(mesh-hub): add e2e dual-transport invoke tests [宪宪/Opus-46🐾]"
```

---

## Task 7: install.sh + download.html 更新

**Files:**
- Modify: `packages/plan-pool/public/download/install.sh`
- Modify: `packages/plan-pool/public/download.html`

**Covers:** AC-15

### Step 64: Update install.sh

- Remove `PLAN_BRIDGE_PORT` from .env template
- Remove SSH tunnel instructions from comments/output
- Keep `MESH_HUB_URL` (bridge still HELLOs to Hub)
- Keep `POOL_NODE_URL` + `POOL_API_KEY`

### Step 65: Update download.html

- Remove SSH tunnel section
- Update prerequisites (remove "SSH access to Hub server")
- Update architecture diagram if present

### Step 66: Commit

```bash
git add packages/plan-pool/public/
git commit -m "docs(install): remove SSH tunnel instructions, update for WS transport [宪宪/Opus-46🐾]"
```

---

## Task 8: 文档同步

**Files:**
- Already done: `docs/decisions/2026-04-01-mesh-hub-mvp-communication-architecture.md` (ADR amendment)
- Modify: `docs/features/F017-plan-pool.md` (topology diagram)
- Modify: `docs/features/F017-deployment-guide.md` (remove SSH, update topology)
- Modify: `docs/ARCHITECTURE.md` + `docs/ARCHITECTURE.zh-CN.md`

**Covers:** Spec 文档同步清单 (必须级)

### Step 67: Update F017-plan-pool.md topology

Update the deployment topology diagram: replace `via Hub Relay` with `WS direct` for bridge path.

### Step 68: Update F017-deployment-guide.md

Remove SSH/tunnel sections. Update bridge deployment to show WS connection flow.

### Step 69: Update ARCHITECTURE docs

Add transport layer note to Phase 1 description in both zh-CN and English versions.

### Step 70: Commit

```bash
git add docs/
git commit -m "docs: sync architecture docs for Hub dual-transport [宪宪/Opus-46🐾]"
```

---

## AC Coverage Matrix

| AC | Task | Verified By |
|----|------|-------------|
| AC-1: Hub `/v1/ws` endpoint | Task 2 | Step 20-23 |
| AC-2: WS_REGISTER with meshCertificate | Task 2 + 3 | Step 20, 36 |
| AC-3: WS优先/callback fallback | Task 2 | Step 26-31 |
| AC-4: L2 token local verification in WS | Task 3 | Step 40-41 |
| AC-5: Reconnect with backoff, 4001 exception | Task 3 | Step 44-47 |
| AC-6: WS + liveness alignment | Task 2 | Step 32-33 |
| AC-7: Timeout → 504 + CANCEL | Task 1 | Step 9-12 |
| AC-8: CANCEL → SIGTERM / abort | Task 4 + 5 | Step 52, 58 |
| AC-9: WS disconnect → in-flight fail | Task 1 | Step 13-16 |
| AC-10: Cert revocation → close 4001 | Task 1 | Step 17-18 |
| AC-11: bridge-cli uses WsChannel | Task 4 | Step 50-53 |
| AC-12: bridge-api uses WsChannel | Task 5 | Step 56-59 |
| AC-13: E2E test | Task 6 | Step 61-62 |
| AC-14: Callback path still works | Task 2 | Step 30-31, 34 |
| AC-15: install.sh/download.html updated | Task 7 | Step 64-65 |

## Dependency Graph

```
Task 1 (WsRegistry)
  └─→ Task 2 (Hub WS endpoint + dual invoke)
       └─→ Task 3 (WsChannel, node side)
            ├─→ Task 4 (bridge-cli migration)
            ├─→ Task 5 (bridge-api migration)
            └─→ Task 6 (e2e integration test)
Task 7 (install.sh) — independent, can parallel with Task 4-6
Task 8 (docs) — after all code done
```
