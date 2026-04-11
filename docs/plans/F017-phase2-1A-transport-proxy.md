# F017 Phase 2 — Phase 1A: Transport + Proxy Implementation Plan

**Feature:** F017 — `docs/features/F017-phase2-oauth-proxy.md`
**Goal:** 单 bridge 能代理 Claude Code 请求到 Anthropic 并返回（含 streaming），通过 WS 隧道中继
**Acceptance Criteria:**
- AC-P2-01: bridge 启动后自动读取本地 OAuth token
- AC-P2-02: 非流式请求: Claude Code → bridge → Anthropic → 正确响应
- AC-P2-03: 流式请求: SSE 全程 pipe，client 实时收到 chunk
- AC-P2-04: tool_use blocks 正确透传（bridge 不解析/不执行）
- AC-P2-05: usage 数据正确提取并上报
- AC-P2-06: owner CLI 在线时 bridge 只读 token 不刷新
- AC-P2-16: `/v1/messages/count_tokens` 透传
- AC-P2-17: client 断开 → gateway 发 `proxy-abort` → bridge AbortController 取消
- AC-P2-18: `proxy-request` 发出后 N 秒无首帧 → 504
- AC-P2-22: CLI 离线 + token 过期 + grace 2min → bridge 兜底刷新并写回
- AC-P2-23: 401 → 重读 Keychain + 重试一次 → 仍失败标 unhealthy
**Architecture:** Hub 新增 `/v1/proxy-invoke` HTTP→WS 桥接端点。Gateway 调用此端点，Hub 通过 WS 转发 proxy-request 到 Bridge。Bridge 直连 Anthropic，SSE 通过 proxy 帧回传。非流式/流式统一走 proxy 帧协议。
**Tech Stack:** TypeScript, ws, node:child_process (Keychain), jose (L2), undici (streaming fetch)
**前端验证:** No

---

## 不做 (Out of Scope)

- Phase 1B: 多 bridge 路由、session 硬绑定、hard-fail 语义、capability-based routing
- Phase 1C: `<env>` block 指纹归一化、E2E Claude Code 工作流
- backpressure（spec 明确标为独立高风险子任务）

---

## Terminal Schema (Final Form)

所有 Task 的实现目标——不是中间态，不会被后续步骤重写。

```typescript
// ── @agent-mesh/protocol/src/messages.ts（扩展） ──

/** Gateway → Hub → Bridge: forward HTTP request to upstream provider */
export interface ProxyRequestFrame {
  type: "proxy-request";
  streamId: string;        // UUID, 每个请求唯一
  method: string;          // "POST"
  path: string;            // "/v1/messages" | "/v1/messages/count_tokens"
  headers: Record<string, string>;  // caller 透传 headers（已去除 auth）
  body: string;            // raw JSON string
}

/** Bridge → Hub → Gateway: raw SSE event from upstream */
export interface ProxySseChunkFrame {
  type: "proxy-sse-chunk";
  streamId: string;
  data: string;            // 一个完整 SSE event（"event: ...\ndata: ...\n\n"）
}

/** Bridge → Hub → Gateway: request completed */
export interface ProxyEndFrame {
  type: "proxy-end";
  streamId: string;
  status: number;          // upstream HTTP status
  body?: string;           // 非流式: 完整 response body
  usage?: { model: string; input_tokens: number; output_tokens: number };
}

/** Bridge → Hub → Gateway: request failed */
export interface ProxyErrorFrame {
  type: "proxy-error";
  streamId: string;
  error: string;
  status?: number;
}

/** Gateway → Hub → Bridge: client disconnected */
export interface ProxyAbortFrame {
  type: "proxy-abort";
  streamId: string;
}

export type ProxyFrame =
  | ProxyRequestFrame | ProxySseChunkFrame | ProxyEndFrame
  | ProxyErrorFrame | ProxyAbortFrame;

// ── packages/plan-bridge-oauth/src/token-provider.ts ──

export interface OAuthCredentials {
  accessToken: string;
  refreshToken: string;
  expiresAt: number;       // Unix ms
  scopes: string[];
}

export interface TokenProvider {
  getAccessToken(): Promise<string>;
  isHealthy(): boolean;
  refresh(): Promise<void>;
  onStatusChange(cb: (healthy: boolean) => void): void;
  close(): void;
}

// ── packages/mesh-hub/src/proxy-streams.ts（新增） ──

export interface ProxyStream {
  streamId: string;
  sourceNodeId: string;    // Hub HTTP 调用者不用——通过 reply 对象直写
  targetNodeId: string;
  reply: FastifyReply;     // Hub HTTP 响应（SSE 或 JSON）
  isStreaming: boolean;
  firstChunkReceived: boolean;
  timer: ReturnType<typeof setTimeout>;
  aborted: boolean;
}

// ── packages/mesh-node/src/ws-channel.ts（扩展） ──

export type ProxyRequestHandler = (
  frame: ProxyRequestFrame,
  sender: ProxySender,
  signal: AbortSignal,
) => Promise<void>;

export interface ProxySender {
  chunk(data: string): void;
  end(status: number, body?: string, usage?: object): void;
  error(error: string, status?: number): void;
}
```

---

## Task Dependency Graph

```
Stream A (Token)              Stream B (WS Infrastructure)
─────────────               ─────────────────────────
T1  scaffold                  T2  proxy frame types (protocol)
  ↓                             ↓           ↓
T3  credential reader         T5  WsRegistry   T7  WsChannel
  ↓                           streaming      proxy frames
T4  B+ guardrails               ↓              ↓
  │                           T6  Hub         T8  MeshClient
  │                           /v1/proxy-      proxyInvoke()
  │                           invoke            ↓
  │                             ↓           T14  Gateway
  │                             └──────┐   proxy path
  ↓                                    ↓
T9  non-streaming proxy ◀──── T7 converge
  ↓
T10 SSE streaming proxy
  ↓
T11 usage extraction
  ↓           ↓
T12 abort   T13 count_tokens
  ↓
T15 bridge main + registration
```

**并行窗口**：T1+T2 并行 → T3-T4 || T5-T8 两条线并行 → T9 起汇合。

---

## Task 1: plan-bridge-oauth 包骨架

**Files:**
- Create: `packages/plan-bridge-oauth/package.json`
- Create: `packages/plan-bridge-oauth/tsconfig.json`
- Create: `packages/plan-bridge-oauth/tsconfig.build.json`
- Create: `packages/plan-bridge-oauth/src/index.ts`

**Step 1: Create package.json**

```json
{
  "name": "@agent-mesh/plan-bridge-oauth",
  "version": "0.0.1",
  "description": "OAuth proxy bridge — Claude Max subscription proxy with streaming",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "lint": "tsc --noEmit",
    "start": "node dist/bridge.js",
    "test": "tsc && node --test dist/**/*.test.js"
  },
  "dependencies": {
    "@agent-mesh/node": "workspace:*",
    "@agent-mesh/protocol": "workspace:*"
  },
  "devDependencies": {
    "@types/node": "^22.13.10"
  }
}
```

**Step 2: Create tsconfig.json** (same pattern as plan-bridge-api)

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

**Step 3: Create tsconfig.build.json**

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["src/**/*.test.ts"]
}
```

**Step 4: Create src/index.ts**

```typescript
export { TokenProvider, createTokenProvider } from "./token-provider.js";
```

**Step 5: pnpm install + build**

```bash
cd packages/plan-bridge-oauth && pnpm install && pnpm build
```

**Step 6: Commit**

```bash
git add packages/plan-bridge-oauth/
git commit -m "feat(F017-P2): scaffold plan-bridge-oauth package"
```

---

## Task 2: Proxy Frame Types (protocol)

**Files:**
- Modify: `packages/mesh-protocol/src/messages.ts`

**Step 1: Write test**

- Test file: `packages/mesh-protocol/src/proxy-frames.test.ts`

```typescript
import { describe, it } from "node:test";
import assert from "node:assert/strict";
import type {
  ProxyRequestFrame, ProxySseChunkFrame, ProxyEndFrame,
  ProxyErrorFrame, ProxyAbortFrame,
} from "./messages.js";

describe("proxy frame types", () => {
  it("ProxyRequestFrame has required fields", () => {
    const f: ProxyRequestFrame = {
      type: "proxy-request",
      streamId: "abc",
      method: "POST",
      path: "/v1/messages",
      headers: { "anthropic-version": "2023-06-01" },
      body: '{"model":"claude-sonnet-4-20250514"}',
    };
    assert.equal(f.type, "proxy-request");
    assert.equal(typeof f.body, "string");
  });

  it("ProxySseChunkFrame carries raw SSE event text", () => {
    const f: ProxySseChunkFrame = {
      type: "proxy-sse-chunk",
      streamId: "abc",
      data: 'event: message_start\ndata: {"type":"message_start"}\n\n',
    };
    assert.ok(f.data.includes("event:"));
  });

  it("ProxyEndFrame includes status and optional usage", () => {
    const f: ProxyEndFrame = {
      type: "proxy-end",
      streamId: "abc",
      status: 200,
      usage: { model: "claude-sonnet-4-20250514", input_tokens: 10, output_tokens: 5 },
    };
    assert.equal(f.status, 200);
    assert.equal(f.usage?.input_tokens, 10);
  });

  it("ProxyErrorFrame includes error message", () => {
    const f: ProxyErrorFrame = {
      type: "proxy-error",
      streamId: "abc",
      error: "upstream timeout",
      status: 504,
    };
    assert.equal(f.status, 504);
  });

  it("ProxyAbortFrame is minimal", () => {
    const f: ProxyAbortFrame = { type: "proxy-abort", streamId: "abc" };
    assert.equal(f.type, "proxy-abort");
  });
});
```

**Step 2: Add types to messages.ts**

在 `StreamChunk` 定义之后、`ResultMessage` 之前添加：

```typescript
// ── Proxy Frames (F017 Phase 2: OAuth proxy streaming) ──

export interface ProxyRequestFrame {
  type: "proxy-request";
  streamId: string;
  method: string;
  path: string;
  headers: Record<string, string>;
  body: string;
}

export interface ProxySseChunkFrame {
  type: "proxy-sse-chunk";
  streamId: string;
  data: string;
}

export interface ProxyEndFrame {
  type: "proxy-end";
  streamId: string;
  status: number;
  body?: string;
  usage?: { model: string; input_tokens: number; output_tokens: number };
}

export interface ProxyErrorFrame {
  type: "proxy-error";
  streamId: string;
  error: string;
  status?: number;
}

export interface ProxyAbortFrame {
  type: "proxy-abort";
  streamId: string;
}

export type ProxyFrame =
  | ProxyRequestFrame
  | ProxySseChunkFrame
  | ProxyEndFrame
  | ProxyErrorFrame
  | ProxyAbortFrame;
```

更新 `MeshMessage` union：

```typescript
export type MeshMessage =
  | HelloRequest | HelloResponse | CapsRequest | CapsResponse
  | InvokeRequest | StreamChunk | ResultMessage | ErrorMessage
  | ProxyFrame;  // ← 新增
```

**Step 3: Verify**

```bash
cd packages/mesh-protocol && pnpm build && pnpm test
```

**Step 4: Commit**

```bash
git commit -m "feat(protocol): add proxy frame types for F017 OAuth streaming"
```

---

## Task 3: TokenProvider — Credential Reader (AC-P2-01)

**Files:**
- Create: `packages/plan-bridge-oauth/src/token-provider.ts`
- Create: `packages/plan-bridge-oauth/src/token-provider.test.ts`

**Step 1: Write test**

```typescript
import { describe, it, beforeEach, afterEach } from "node:test";
import assert from "node:assert/strict";
import { readCredentials, type OAuthCredentials } from "./token-provider.js";
import { writeFileSync, mkdirSync, rmSync } from "node:fs";
import { join } from "node:path";

describe("readCredentials", () => {
  const tmpDir = join(import.meta.dirname, ".tmp-test-creds");
  const credPath = join(tmpDir, ".credentials.json");

  beforeEach(() => { mkdirSync(tmpDir, { recursive: true }); });
  afterEach(() => { rmSync(tmpDir, { recursive: true, force: true }); });

  it("reads credentials from JSON file", async () => {
    const cred = {
      claudeAiOauth: {
        accessToken: "sk-ant-oat01-test",
        refreshToken: "sk-ant-ort01-test",
        expiresAt: Date.now() + 3600_000,
        scopes: ["user:inference"],
      },
    };
    writeFileSync(credPath, JSON.stringify(cred));
    const result = await readCredentials({ credentialPath: credPath });
    assert.equal(result.accessToken, "sk-ant-oat01-test");
    assert.equal(result.refreshToken, "sk-ant-ort01-test");
    assert.deepEqual(result.scopes, ["user:inference"]);
  });

  it("prefers CLAUDE_CODE_OAUTH_TOKEN env var", async () => {
    process.env.CLAUDE_CODE_OAUTH_TOKEN = "sk-ant-oat01-envtest";
    try {
      const result = await readCredentials({ credentialPath: credPath });
      assert.equal(result.accessToken, "sk-ant-oat01-envtest");
      assert.equal(result.refreshToken, "");
      assert.equal(result.expiresAt, Infinity);
    } finally {
      delete process.env.CLAUDE_CODE_OAUTH_TOKEN;
    }
  });

  it("throws on missing file", async () => {
    await assert.rejects(
      () => readCredentials({ credentialPath: "/nonexistent/path" }),
      /credential/i,
    );
  });
});
```

**Step 2: Implement**

```typescript
import { readFile, writeFile } from "node:fs/promises";
import { execSync } from "node:child_process";
import { platform } from "node:os";

export interface OAuthCredentials {
  accessToken: string;
  refreshToken: string;
  expiresAt: number;
  scopes: string[];
}

interface CredentialSource {
  credentialPath?: string;
}

/** Read OAuth credentials from env var, Keychain, or file. */
export async function readCredentials(
  opts: CredentialSource = {},
): Promise<OAuthCredentials> {
  // Layer 0: env var override (highest priority)
  const envToken = process.env.CLAUDE_CODE_OAUTH_TOKEN;
  if (envToken) {
    return {
      accessToken: envToken,
      refreshToken: "",
      expiresAt: Infinity,
      scopes: ["user:inference"],
    };
  }

  // Layer 1: platform-specific storage
  if (platform() === "darwin" && !opts.credentialPath) {
    return readFromKeychain();
  }

  // Layer 2: credential file
  const path = opts.credentialPath ?? defaultCredentialPath();
  return readFromFile(path);
}

function defaultCredentialPath(): string {
  const home = process.env.HOME ?? process.env.USERPROFILE ?? "~";
  return `${home}/.claude/.credentials.json`;
}

async function readFromFile(path: string): Promise<OAuthCredentials> {
  let raw: string;
  try {
    raw = await readFile(path, "utf-8");
  } catch {
    throw new Error(`Cannot read credential file: ${path}`);
  }
  return parseCredentialJson(raw);
}

function readFromKeychain(): OAuthCredentials {
  try {
    const raw = execSync(
      "security find-generic-password -s 'Claude Code-credentials' -w",
      { encoding: "utf-8", timeout: 5000 },
    ).trim();
    return parseCredentialJson(raw);
  } catch {
    throw new Error("Cannot read from macOS Keychain (Claude Code-credentials)");
  }
}

function parseCredentialJson(raw: string): OAuthCredentials {
  const data = JSON.parse(raw);
  const oauth = data.claudeAiOauth;
  if (!oauth?.accessToken) {
    throw new Error("Invalid credential format: missing claudeAiOauth.accessToken");
  }
  return {
    accessToken: oauth.accessToken,
    refreshToken: oauth.refreshToken ?? "",
    expiresAt: oauth.expiresAt ?? Infinity,
    scopes: oauth.scopes ?? [],
  };
}

/** Write credentials back to platform storage (for refresh write-back). */
export async function writeCredentials(
  creds: OAuthCredentials,
  opts: CredentialSource = {},
): Promise<void> {
  const data = {
    claudeAiOauth: {
      accessToken: creds.accessToken,
      refreshToken: creds.refreshToken,
      expiresAt: creds.expiresAt,
      scopes: creds.scopes,
    },
  };
  if (platform() === "darwin" && !opts.credentialPath) {
    writeToKeychain(JSON.stringify(data));
    return;
  }
  const path = opts.credentialPath ?? defaultCredentialPath();
  await writeFile(path, JSON.stringify(data, null, 2), "utf-8");
}

function writeToKeychain(json: string): void {
  // Delete then re-add (Keychain has no "update password" command)
  try {
    execSync(
      "security delete-generic-password -s 'Claude Code-credentials'",
      { encoding: "utf-8", timeout: 5000 },
    );
  } catch { /* may not exist */ }
  execSync(
    `security add-generic-password -s 'Claude Code-credentials' -a 'Claude Code' -w '${json.replace(/'/g, "'\\''")}'`,
    { encoding: "utf-8", timeout: 5000 },
  );
}
```

**Step 3: Verify**

```bash
cd packages/plan-bridge-oauth && pnpm build && pnpm test
```

**Step 4: Commit**

```bash
git commit -m "feat(bridge-oauth): credential reader — Keychain/file/env (AC-P2-01)"
```

---

## Task 4: TokenProvider — B+ Refresh Guardrails (AC-P2-06, AC-P2-22, AC-P2-23)

**Files:**
- Modify: `packages/plan-bridge-oauth/src/token-provider.ts`
- Create: `packages/plan-bridge-oauth/src/token-provider.test.ts`（扩展 Task 3 的测试）

**Step 1: Write tests**

```typescript
import { describe, it, mock, beforeEach, afterEach } from "node:test";
import assert from "node:assert/strict";
import { createTokenProvider, type TokenProvider } from "./token-provider.js";

describe("TokenProvider B+ guardrails", () => {
  it("returns cached token when valid (READ-FIRST)", async () => {
    const provider = createTokenProvider({
      readCredentials: async () => ({
        accessToken: "valid-token",
        refreshToken: "rt",
        expiresAt: Date.now() + 3600_000,
        scopes: ["user:inference"],
      }),
      pollIntervalMs: 100_000, // won't fire in test
    });
    const token = await provider.getAccessToken();
    assert.equal(token, "valid-token");
    assert.ok(provider.isHealthy());
    provider.close();
  });

  it("re-reads credential source on each poll tick", async () => {
    let callCount = 0;
    const provider = createTokenProvider({
      readCredentials: async () => ({
        accessToken: `token-${++callCount}`,
        refreshToken: "rt",
        expiresAt: Date.now() + 3600_000,
        scopes: ["user:inference"],
      }),
      pollIntervalMs: 50,
    });
    await provider.getAccessToken();
    assert.equal(callCount, 1);
    // Wait for poll
    await new Promise((r) => setTimeout(r, 120));
    await provider.getAccessToken();
    assert.ok(callCount >= 2, `expected >=2 reads, got ${callCount}`);
    provider.close();
  });

  it("enters GRACE PERIOD when token expired, waits before refresh", async () => {
    let readCount = 0;
    let refreshCalled = false;
    const provider = createTokenProvider({
      readCredentials: async () => {
        readCount++;
        return {
          accessToken: "expired-token",
          refreshToken: "rt",
          expiresAt: Date.now() - 1000, // already expired
          scopes: ["user:inference"],
        };
      },
      refreshToken: async () => { refreshCalled = true; return null as any; },
      gracePeriodMs: 100, // short for test
      pollIntervalMs: 30,
    });
    // First call: enters grace period, does NOT refresh yet
    await assert.rejects(() => provider.getAccessToken(), /expired/i);
    assert.ok(!refreshCalled, "should not refresh during grace period");
    provider.close();
  });

  it("performs LAST-RESORT REFRESH after grace period expires", async () => {
    let refreshCalled = false;
    const refreshedCred = {
      accessToken: "refreshed-token",
      refreshToken: "new-rt",
      expiresAt: Date.now() + 3600_000,
      scopes: ["user:inference"],
    };
    const provider = createTokenProvider({
      readCredentials: async () => ({
        accessToken: "expired-token",
        refreshToken: "rt",
        expiresAt: Date.now() - 1000,
        scopes: ["user:inference"],
      }),
      refreshToken: async () => { refreshCalled = true; return refreshedCred; },
      writeCredentials: async () => {},
      gracePeriodMs: 50,
      pollIntervalMs: 20,
    });
    // Wait past grace period
    await new Promise((r) => setTimeout(r, 100));
    const token = await provider.getAccessToken();
    assert.ok(refreshCalled, "should have called refresh");
    assert.equal(token, "refreshed-token");
    provider.close();
  });

  it("marks unhealthy after 3 consecutive refresh failures", async () => {
    let statusChanges: boolean[] = [];
    const provider = createTokenProvider({
      readCredentials: async () => ({
        accessToken: "expired",
        refreshToken: "rt",
        expiresAt: Date.now() - 1000,
        scopes: ["user:inference"],
      }),
      refreshToken: async () => { throw new Error("refresh failed"); },
      gracePeriodMs: 10,
      pollIntervalMs: 10,
      maxRefreshFailures: 3,
    });
    provider.onStatusChange((healthy) => statusChanges.push(healthy));
    // Wait for grace + multiple refresh attempts
    await new Promise((r) => setTimeout(r, 200));
    assert.ok(!provider.isHealthy());
    assert.ok(statusChanges.includes(false));
    provider.close();
  });

  it("401 RECOVERY: re-reads then retries before marking unhealthy", async () => {
    let reads = 0;
    const provider = createTokenProvider({
      readCredentials: async () => ({
        accessToken: reads++ === 0 ? "stale-token" : "fresh-token",
        refreshToken: "rt",
        expiresAt: Date.now() + 3600_000,
        scopes: ["user:inference"],
      }),
      pollIntervalMs: 100_000,
    });
    // Simulate 401: caller reports auth failure
    const token1 = await provider.getAccessToken();
    assert.equal(token1, "stale-token");
    // Force re-read (401 recovery path)
    provider.invalidate();
    const token2 = await provider.getAccessToken();
    assert.equal(token2, "fresh-token");
    provider.close();
  });
});
```

**Step 2: Implement `createTokenProvider()`**

在 `token-provider.ts` 末尾添加：

```typescript
const TOKEN_URL = "https://platform.claude.com/v1/oauth/token";
const CLIENT_ID = "9d1c250a-e61b-44d9-88ed-5944d1962f5e";

export interface TokenProviderOptions {
  /** Override credential reader (for testing). */
  readCredentials?: () => Promise<OAuthCredentials>;
  /** Override refresh logic (for testing). */
  refreshToken?: (rt: string) => Promise<OAuthCredentials>;
  /** Override write-back (for testing). */
  writeCredentials?: (creds: OAuthCredentials) => Promise<void>;
  /** Keychain/file poll interval (default: 30000). */
  pollIntervalMs?: number;
  /** Grace period after expiry before attempting refresh (default: 120000). */
  gracePeriodMs?: number;
  /** Consecutive refresh failures before marking unhealthy (default: 3). */
  maxRefreshFailures?: number;
  /** Credential file path override. */
  credentialPath?: string;
  /** Token URL override. */
  tokenUrl?: string;
  /** Client ID override. */
  clientId?: string;
}

export interface TokenProvider {
  getAccessToken(): Promise<string>;
  isHealthy(): boolean;
  refresh(): Promise<void>;
  invalidate(): void;
  onStatusChange(cb: (healthy: boolean) => void): void;
  close(): void;
}

export function createTokenProvider(opts: TokenProviderOptions = {}): TokenProvider {
  const pollMs = opts.pollIntervalMs ?? 30_000;
  const graceMs = opts.gracePeriodMs ?? 120_000;
  const maxFailures = opts.maxRefreshFailures ?? 3;
  const tokenUrl = opts.tokenUrl ?? TOKEN_URL;
  const clientId = opts.clientId ?? CLIENT_ID;

  let cached: OAuthCredentials | null = null;
  let healthy = true;
  let refreshFailures = 0;
  let graceEnteredAt: number | null = null;
  let pollTimer: ReturnType<typeof setInterval> | null = null;
  let closed = false;
  let pollInFlight = false;       // guard: prevent overlapping pollOnce()
  let refreshInFlight = false;    // guard: prevent overlapping refresh
  const listeners: Array<(healthy: boolean) => void> = [];

  const doRead = opts.readCredentials ?? (() => readCredentials({
    credentialPath: opts.credentialPath,
  }));
  const doRefresh = opts.refreshToken ?? (async (rt: string) => {
    const res = await fetch(tokenUrl, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({
        grant_type: "refresh_token",
        refresh_token: rt,
        client_id: clientId,
      }),
    });
    if (!res.ok) throw new Error(`refresh failed: ${res.status}`);
    const data = await res.json() as Record<string, unknown>;
    return {
      accessToken: data.access_token as string,
      refreshToken: data.refresh_token as string,
      expiresAt: Date.now() + (data.expires_in as number) * 1000,
      scopes: (data.scope as string ?? "").split(" "),
    };
  });
  const doWrite = opts.writeCredentials ?? ((creds: OAuthCredentials) =>
    writeCredentials(creds, { credentialPath: opts.credentialPath }));

  function setHealthy(h: boolean) {
    if (healthy !== h) {
      healthy = h;
      for (const cb of listeners) cb(h);
    }
  }

  function isExpired(creds: OAuthCredentials): boolean {
    return Date.now() >= creds.expiresAt;
  }

  async function pollOnce(): Promise<void> {
    if (closed || pollInFlight) return;
    pollInFlight = true;
    try {
      cached = await doRead();
      if (!isExpired(cached)) {
        graceEnteredAt = null;
        refreshFailures = 0;
        setHealthy(true);
        return;
      }
      // Token expired — grace period logic
      if (graceEnteredAt === null) {
        graceEnteredAt = Date.now();
      }
      if (Date.now() - graceEnteredAt < graceMs) {
        return; // still in grace period, wait for CLI to refresh
      }
      // Past grace period — last-resort refresh
      await doLastResortRefresh();
    } catch {
      // read failed — keep using cached if available
    } finally {
      pollInFlight = false;
    }
  }

  async function doLastResortRefresh(): Promise<void> {
    if (refreshInFlight) return;
    refreshInFlight = true;
    if (!cached?.refreshToken) {
      setHealthy(false);
      refreshInFlight = false;
      return;
    }
    // Double-check: re-read once more before refreshing
    try {
      const fresh = await doRead();
      if (!isExpired(fresh)) {
        cached = fresh;
        graceEnteredAt = null;
        refreshFailures = 0;
        setHealthy(true);
        refreshInFlight = false;
        return;
      }
    } catch { /* proceed with refresh */ }

    try {
      const newCreds = await doRefresh(cached.refreshToken);
      // Validate scope
      if (!newCreds.scopes.includes("user:inference")) {
        throw new Error("refreshed token missing user:inference scope");
      }
      cached = newCreds;
      graceEnteredAt = null;
      refreshFailures = 0;
      setHealthy(true);
      // Write back so CLI picks up new token
      await doWrite(newCreds).catch(() => {});
    } catch {
      refreshFailures++;
      if (refreshFailures >= maxFailures) {
        setHealthy(false);
      }
    } finally {
      refreshInFlight = false;
    }
  }

  // Start polling
  pollTimer = setInterval(() => { pollOnce().catch(() => {}); }, pollMs);
  // Initial read
  const initPromise = pollOnce();

  return {
    async getAccessToken(): Promise<string> {
      await initPromise;
      if (!cached) throw new Error("No credentials available");
      if (isExpired(cached) && graceEnteredAt !== null) {
        throw new Error("Token expired (grace period or refresh in progress)");
      }
      return cached.accessToken;
    },
    isHealthy: () => healthy,
    async refresh() { await doLastResortRefresh(); },
    invalidate() {
      cached = null;
      pollOnce().catch(() => {});
    },
    onStatusChange(cb) { listeners.push(cb); },
    close() {
      closed = true;
      if (pollTimer) { clearInterval(pollTimer); pollTimer = null; }
    },
  };
}
```

**Step 3: Verify**

```bash
cd packages/plan-bridge-oauth && pnpm build && pnpm test
```

**Step 4: Commit**

```bash
git commit -m "feat(bridge-oauth): B+ TokenProvider with 4-layer guardrails (AC-P2-06/22/23)"
```

---

## Task 5: WsRegistry Streaming Session Support

**Files:**
- Create: `packages/mesh-hub/src/proxy-streams.ts`
- Test: `packages/mesh-hub/src/proxy-streams.test.ts`

**设计决策**：独立 ProxyStreamManager 类，不修改 WsRegistry。WsRegistry 管 WS 连接 + unary dispatch，ProxyStreamManager 管 streaming proxy session。Hub 在 WS message handler 中根据 frame type 路由到对应 manager。

**Step 1: Write test**

```typescript
import { describe, it } from "node:test";
import assert from "node:assert/strict";
import { ProxyStreamManager } from "./proxy-streams.js";

describe("ProxyStreamManager", () => {
  it("registers and retrieves a stream", () => {
    const mgr = new ProxyStreamManager();
    const chunks: string[] = [];
    mgr.register("s1", {
      onChunk: (data) => chunks.push(data),
      onEnd: () => {},
      onError: () => {},
      timeoutMs: 5000,
      onTimeout: () => {},
    });
    assert.ok(mgr.has("s1"));
    mgr.handleChunk("s1", "chunk1");
    assert.deepEqual(chunks, ["chunk1"]);
    mgr.destroy();
  });

  it("calls onEnd and removes stream on proxy-end", () => {
    const mgr = new ProxyStreamManager();
    let ended = false;
    let endUsage: unknown = null;
    mgr.register("s1", {
      onChunk: () => {},
      onEnd: (_status, _body, usage) => { ended = true; endUsage = usage; },
      onError: () => {},
      timeoutMs: 5000,
      onTimeout: () => {},
    });
    mgr.handleEnd("s1", 200, undefined, { model: "m", input_tokens: 1, output_tokens: 2 });
    assert.ok(ended);
    assert.ok(!mgr.has("s1"));
    mgr.destroy();
  });

  it("calls onError and removes stream on proxy-error", () => {
    const mgr = new ProxyStreamManager();
    let errorMsg = "";
    mgr.register("s1", {
      onChunk: () => {},
      onEnd: () => {},
      onError: (err) => { errorMsg = err; },
      timeoutMs: 5000,
      onTimeout: () => {},
    });
    mgr.handleError("s1", "upstream failed", 502);
    assert.equal(errorMsg, "upstream failed");
    assert.ok(!mgr.has("s1"));
    mgr.destroy();
  });

  it("calls onTimeout when first chunk not received within timeout", async () => {
    const mgr = new ProxyStreamManager();
    let timedOut = false;
    mgr.register("s1", {
      onChunk: () => {},
      onEnd: () => {},
      onError: () => {},
      timeoutMs: 50,
      onTimeout: () => { timedOut = true; },
    });
    await new Promise((r) => setTimeout(r, 100));
    assert.ok(timedOut);
    assert.ok(!mgr.has("s1"));
    mgr.destroy();
  });

  it("cancels timeout after first chunk", async () => {
    const mgr = new ProxyStreamManager();
    let timedOut = false;
    mgr.register("s1", {
      onChunk: () => {},
      onEnd: () => {},
      onError: () => {},
      timeoutMs: 80,
      onTimeout: () => { timedOut = true; },
    });
    mgr.handleChunk("s1", "first");
    await new Promise((r) => setTimeout(r, 120));
    assert.ok(!timedOut, "timeout should be cancelled after first chunk");
    mgr.handleEnd("s1", 200);
    mgr.destroy();
  });
});
```

**Step 2: Implement**

```typescript
/** ProxyStreamManager — manages streaming proxy sessions for Hub relay. */

interface StreamCallbacks {
  onChunk: (data: string) => void;
  onEnd: (status: number, body?: string, usage?: object) => void;
  onError: (error: string) => void;
  timeoutMs: number;
  onTimeout: () => void;
}

interface ActiveStream {
  callbacks: StreamCallbacks;
  timer: ReturnType<typeof setTimeout>;
  firstChunkReceived: boolean;
}

export class ProxyStreamManager {
  private streams = new Map<string, ActiveStream>();

  register(streamId: string, callbacks: StreamCallbacks): void {
    const timer = setTimeout(() => {
      const stream = this.streams.get(streamId);
      if (stream && !stream.firstChunkReceived) {
        callbacks.onTimeout();
        this.streams.delete(streamId);
      }
    }, callbacks.timeoutMs);

    this.streams.set(streamId, { callbacks, timer, firstChunkReceived: false });
  }

  has(streamId: string): boolean {
    return this.streams.has(streamId);
  }

  handleChunk(streamId: string, data: string): void {
    const stream = this.streams.get(streamId);
    if (!stream) return;
    if (!stream.firstChunkReceived) {
      stream.firstChunkReceived = true;
      clearTimeout(stream.timer);
    }
    stream.callbacks.onChunk(data);
  }

  handleEnd(streamId: string, status: number, body?: string, usage?: object): void {
    const stream = this.streams.get(streamId);
    if (!stream) return;
    clearTimeout(stream.timer);
    stream.callbacks.onEnd(status, body, usage);
    this.streams.delete(streamId);
  }

  handleError(streamId: string, error: string, status?: number): void {
    const stream = this.streams.get(streamId);
    if (!stream) return;
    clearTimeout(stream.timer);
    stream.callbacks.onError(error);
    this.streams.delete(streamId);
  }

  remove(streamId: string): void {
    const stream = this.streams.get(streamId);
    if (!stream) return;
    clearTimeout(stream.timer);
    this.streams.delete(streamId);
  }

  destroy(): void {
    for (const [id, stream] of this.streams) {
      clearTimeout(stream.timer);
    }
    this.streams.clear();
  }
}
```

**Step 3: Verify**

```bash
cd packages/mesh-hub && pnpm build && pnpm test
```

**Step 4: Commit**

```bash
git commit -m "feat(hub): ProxyStreamManager for streaming proxy sessions"
```

---

## Task 6: Hub `/v1/proxy-invoke` Endpoint (AC-P2-18)

**Files:**
- Modify: `packages/mesh-hub/src/hub.ts`

**设计要点**：
- 复用 `/v1/invoke` 的 L2 auth + scope + jti + liveness 逻辑（提取为共享函数）
- 新端点接受 `{ invocationToken, targetNodeId, stream, proxyRequest: { method, path, headers, body } }`
- Hub 生成 streamId，注册到 ProxyStreamManager
- 通过 WsRegistry 发送 proxy-request 帧到 target bridge
- 流式：SSE 响应；非流式：等 proxy-end 后返回 JSON

**Step 1: Write test**

```typescript
// packages/mesh-hub/src/proxy-invoke.test.ts
import { describe, it } from "node:test";
import assert from "node:assert/strict";
// Integration test with Hub + mock WS bridge — see verify step below
```

**Step 2: Extract auth validation helper from /v1/invoke**

在 hub.ts 中提取 L2 验证逻辑为内部函数：

```typescript
async function validateInvocationAuth(
  invocationToken: string,
  targetNodeId: string,
  payload: { scope: string[] },
): Promise<
  | { ok: true; claims: L2Claims; traceId: string; invocationId: string }
  | { ok: false; status: number; error: string; code?: number }
> {
  // Steps 1-6 from existing /v1/invoke (verify L2, audience, cat, cert, scope, jti, liveness)
  // Returns claims on success or error details on failure
}
```

**Step 3: Add /v1/proxy-invoke endpoint**

```typescript
interface ProxyInvokeBody {
  invocationToken: string;
  targetNodeId: string;
  stream: boolean;
  proxyRequest: {
    method: string;
    path: string;
    headers: Record<string, string>;
    body: string;
  };
}

app.post<{ Body: ProxyInvokeBody }>("/v1/proxy-invoke", async (request, reply) => {
  const { invocationToken, targetNodeId, stream, proxyRequest } = request.body ?? {} as ProxyInvokeBody;
  if (!invocationToken || !targetNodeId || !proxyRequest) {
    return reply.code(400).send({ error: "invocationToken, targetNodeId, proxyRequest required" });
  }

  // Reuse L2 auth validation (scope: ["invoke"])
  const auth = await validateInvocationAuth(invocationToken, targetNodeId, { scope: ["invoke"] });
  if (!auth.ok) return reply.code(auth.status).send({ error: auth.error, code: auth.code });

  // Check WS session
  if (!wsRegistry.has(targetNodeId)) {
    return reply.code(503).send({ error: "target node has no WS session" });
  }

  const streamId = crypto.randomUUID();

  // Send proxy-request frame to bridge
  const session = wsRegistry.getWs(targetNodeId);
  if (!session) return reply.code(503).send({ error: "target WS session lost" });

  const proxyRequestFrame = {
    type: "proxy-request" as const,
    streamId,
    method: proxyRequest.method,
    path: proxyRequest.path,
    headers: proxyRequest.headers,
    body: proxyRequest.body,
  };

  if (stream) {
    // SSE response
    reply.raw.writeHead(200, {
      "content-type": "text/event-stream",
      "cache-control": "no-cache",
      connection: "keep-alive",
      "x-stream-id": streamId,
    });

    const cleanup = () => {
      proxyStreamManager.remove(streamId);
      // Send abort to bridge
      try { session.send(JSON.stringify({ type: "proxy-abort", streamId })); } catch {}
    };

    request.raw.on("close", cleanup);

    let sideChannelUsage: object | undefined;
    proxyStreamManager.register(streamId, {
      onChunk: (data) => { reply.raw.write(data); }, // 原样写出 Anthropic SSE，不包装
      onEnd: (_status, _body, usage) => {
        sideChannelUsage = usage; // usage 走旁路，不注入客户端流
        reply.raw.end();
      },
      onError: (error) => {
        reply.raw.write(`event: error\ndata: ${JSON.stringify({ error })}\n\n`);
        reply.raw.end();
      },
      timeoutMs: config.invokeTimeoutMs,
      onTimeout: () => {
        reply.raw.write(`event: error\ndata: ${JSON.stringify({ error: "first chunk timeout" })}\n\n`);
        reply.raw.end();
      },
    });

    session.send(JSON.stringify(proxyRequestFrame));
    // Response is managed by proxyStreamManager callbacks — don't return here
    // Fastify: use reply.hijack() to take over the response
    await reply.hijack();
  } else {
    // Non-streaming: wait for proxy-end (carries status + body + usage)
    const result = await new Promise<{ status: number; body?: string; usage?: object; error?: string }>((resolve) => {
      proxyStreamManager.register(streamId, {
        onChunk: () => {}, // non-streaming should not receive chunks
        onEnd: (status, body, usage) => resolve({ status, body, usage }),
        onError: (error) => resolve({ error, status: 502 }),
        timeoutMs: config.invokeTimeoutMs,
        onTimeout: () => resolve({ error: "proxy timeout", status: 504 }),
      });
      session.send(JSON.stringify(proxyRequestFrame));
    });

    if (result.error) {
      return reply.code(result.status).send({ error: result.error });
    }
    // Non-streaming: proxy-end carries upstream status + raw response body
    const replyStatus = result.status;
    if (result.body) {
      return reply.code(replyStatus).header("content-type", "application/json").send(result.body);
    }
    return reply.code(replyStatus).send({ error: "empty response from bridge" });
  }
});
```

**Step 4: Extend WS message handler to route proxy frames**

在 hub.ts 的 WS `socket.on("message")` handler 中添加：

```typescript
// After existing RESULT handling:
} else if (msg.type === "proxy-sse-chunk" && registeredNodeId) {
  proxyStreamManager.handleChunk(msg.streamId as string, msg.data as string);
} else if (msg.type === "proxy-end" && registeredNodeId) {
  proxyStreamManager.handleEnd(
    msg.streamId as string,
    msg.status as number,
    msg.body as string | undefined,
    msg.usage as object | undefined,
  );
} else if (msg.type === "proxy-error" && registeredNodeId) {
  proxyStreamManager.handleError(
    msg.streamId as string,
    msg.error as string,
    msg.status as number | undefined,
  );
}
```

**Step 5: Verify**

```bash
cd packages/mesh-hub && pnpm build && pnpm test
```

**Step 6: Commit**

```bash
git commit -m "feat(hub): /v1/proxy-invoke endpoint — HTTP→WS streaming relay (AC-P2-18)"
```

---

## Task 7: WsChannel Proxy Frame Handling

**Files:**
- Modify: `packages/mesh-node/src/ws-channel.ts`

**Step 1: Write test**

```typescript
// packages/mesh-node/src/ws-channel-proxy.test.ts
import { describe, it } from "node:test";
import assert from "node:assert/strict";
// Tests use WsChannel with a mock WS server — pattern from existing ws-channel tests

describe("WsChannel proxy frames", () => {
  it("dispatches proxy-request to proxyHandler", async () => {
    let receivedFrame: unknown = null;
    // Create WsChannel with proxyHandler
    // Send a proxy-request message on mock WS
    // Assert proxyHandler was called with the frame
    // (Full integration test setup with mock Hub)
  });

  it("sends proxy-sse-chunk frames via ProxySender.chunk()", async () => {
    // proxyHandler calls sender.chunk("sse data")
    // Assert WS sent { type: "proxy-sse-chunk", streamId, data: "sse data" }
  });

  it("sends proxy-end frame via ProxySender.end()", async () => {
    // proxyHandler calls sender.end(200, body, usage)
    // Assert WS sent { type: "proxy-end", streamId, status: 200, body, usage }
  });

  it("aborts via AbortSignal on proxy-abort frame", async () => {
    // Send proxy-abort for active streamId
    // Assert AbortSignal.aborted === true
  });
});
```

**Step 2: Extend WsChannelConfig**

```typescript
export type ProxyRequestHandler = (
  frame: { streamId: string; method: string; path: string; headers: Record<string, string>; body: string },
  sender: ProxySender,
  signal: AbortSignal,
) => Promise<void>;

export interface ProxySender {
  chunk(data: string): void;
  end(status: number, body?: string, usage?: object): void;
  error(error: string, status?: number): void;
}

export interface WsChannelConfig {
  // ... existing fields ...
  /** Handler for proxy-request frames (F017 Phase 2). */
  proxyHandler?: ProxyRequestHandler;
}
```

**Step 3: Add proxy frame handling in WsChannel**

在 `_connect()` 的 message handler 中，INVOKE/CANCEL 之后添加：

```typescript
if (msg.type === "proxy-request" && this.config.proxyHandler) {
  await this.handleProxyRequest(msg);
  return;
}

if (msg.type === "proxy-abort") {
  const streamId = msg.streamId as string;
  const controller = this.activeProxyStreams.get(streamId);
  if (controller) {
    controller.abort();
    this.activeProxyStreams.delete(streamId);
  }
  return;
}
```

新增 `activeProxyStreams` map 和 handler：

```typescript
private activeProxyStreams = new Map<string, AbortController>();

private async handleProxyRequest(msg: Record<string, unknown>): Promise<void> {
  const streamId = msg.streamId as string;
  const controller = new AbortController();
  this.activeProxyStreams.set(streamId, controller);

  const sender: ProxySender = {
    chunk: (data) => this.sendFrame({ type: "proxy-sse-chunk", streamId, data }),
    end: (status, body?, usage?) => this.sendFrame({ type: "proxy-end", streamId, status, body, usage }),
    error: (error, status?) => this.sendFrame({ type: "proxy-error", streamId, error, status }),
  };

  try {
    await this.config.proxyHandler!(
      {
        streamId,
        method: msg.method as string,
        path: msg.path as string,
        headers: msg.headers as Record<string, string>,
        body: msg.body as string,
      },
      sender,
      controller.signal,
    );
  } catch (err) {
    if (!controller.signal.aborted) {
      sender.error(err instanceof Error ? err.message : "proxy handler error");
    }
  } finally {
    this.activeProxyStreams.delete(streamId);
  }
}

private sendFrame(frame: Record<string, unknown>): void {
  if (this.ws && this.connected) {
    this.ws.send(JSON.stringify(frame));
  }
}
```

在 `close()` 中清理：

```typescript
for (const [, controller] of this.activeProxyStreams) {
  controller.abort();
}
this.activeProxyStreams.clear();
```

**Step 4: Verify**

```bash
cd packages/mesh-node && pnpm build && pnpm test
```

**Step 5: Commit**

```bash
git commit -m "feat(ws-channel): proxy-request/abort frame handling with ProxySender"
```

---

## Task 8: MeshClient `proxyInvoke()` Method

**Files:**
- Modify: `packages/mesh-node/src/client.ts`

**Step 1: Write test**

```typescript
// packages/mesh-node/src/client-proxy.test.ts
describe("MeshClient.proxyInvoke()", () => {
  it("sends proxy-invoke request to Hub and streams SSE chunks", async () => {
    // Setup: mock Hub /v1/proxy-invoke that returns SSE
    // Call client.proxyInvoke(targetNodeId, { method, path, headers, body, stream: true })
    // Assert: receives async iterable of SSE events
  });

  it("returns full response for non-streaming requests", async () => {
    // Setup: mock Hub /v1/proxy-invoke that returns JSON
    // Call client.proxyInvoke(targetNodeId, { method, path, headers, body, stream: false })
    // Assert: returns { status, body, usage }
  });
});
```

**Step 2: Implement**

```typescript
export interface ProxyInvokeOptions {
  method: string;
  path: string;
  headers: Record<string, string>;
  body: string;
  stream: boolean;
  scope?: string[];
  targetCatId?: string;
}

export interface ProxyInvokeResult {
  status: number;
  body: string;
  usage?: { model: string; input_tokens: number; output_tokens: number };
}

/** Invoke a proxy bridge — returns full response (non-streaming) or async SSE iterable (streaming). */
async proxyInvoke(
  targetNodeId: string,
  opts: ProxyInvokeOptions & { stream: false },
): Promise<ProxyInvokeResult>;
async proxyInvoke(
  targetNodeId: string,
  opts: ProxyInvokeOptions & { stream: true },
): Promise<AsyncIterable<string>>;
async proxyInvoke(
  targetNodeId: string,
  opts: ProxyInvokeOptions,
): Promise<ProxyInvokeResult | AsyncIterable<string>> {
  this.requireCert();
  const scope = opts.scope ?? ["invoke"];
  const traceId = crypto.randomUUID();
  const invocationToken = await this.requestToken(targetNodeId, scope, {
    targetCatId: opts.targetCatId,
    traceId,
  });

  const res = await fetch(`${this.hubUrl}/v1/proxy-invoke`, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({
      invocationToken,
      targetNodeId,
      stream: opts.stream,
      proxyRequest: {
        method: opts.method,
        path: opts.path,
        headers: opts.headers,
        body: opts.body,
      },
    }),
  });

  if (!res.ok) {
    const errBody = await res.text();
    throw new Error(`proxy-invoke failed: ${res.status} ${errBody}`);
  }

  if (!opts.stream) {
    const body = await res.text();
    return { status: res.status, body };
  }

  // Streaming: return async iterable over SSE events
  return this._readSseStream(res);
}

private async *_readSseStream(res: Response): AsyncIterable<string> {
  const reader = res.body!.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      buffer += decoder.decode(value, { stream: true });

      // Parse SSE events from buffer
      const parts = buffer.split("\n\n");
      buffer = parts.pop()!; // last part is incomplete
      for (const part of parts) {
        if (!part.trim()) continue;
        yield part + "\n\n";
      }
    }
    if (buffer.trim()) yield buffer;
  } finally {
    reader.releaseLock();
  }
}
```

**Step 3: Export from index.ts**

确保 `ProxyInvokeOptions` 和 `ProxyInvokeResult` 从 `@agent-mesh/node` 导出。

**Step 4: Verify**

```bash
cd packages/mesh-node && pnpm build && pnpm test
```

**Step 5: Commit**

```bash
git commit -m "feat(mesh-client): proxyInvoke() with SSE streaming support"
```

---

## Task 9: Bridge Non-Streaming Proxy (AC-P2-02, AC-P2-04)

**Files:**
- Create: `packages/plan-bridge-oauth/src/proxy.ts`
- Create: `packages/plan-bridge-oauth/src/proxy.test.ts`

**Step 1: Write test**

```typescript
import { describe, it } from "node:test";
import assert from "node:assert/strict";
import { buildAnthropicRequest, handleProxyRequest } from "./proxy.js";

describe("buildAnthropicRequest", () => {
  it("injects OAuth token via x-api-key header", () => {
    const req = buildAnthropicRequest({
      accessToken: "sk-ant-oat01-test",
      path: "/v1/messages",
      callerHeaders: { "anthropic-version": "2023-06-01", "content-type": "application/json" },
      body: '{"model":"claude-sonnet-4-20250514","messages":[]}',
    });
    assert.equal(req.headers["x-api-key"], "sk-ant-oat01-test");
    assert.equal(req.headers["anthropic-version"], "2023-06-01");
    assert.ok(!req.headers["authorization"]); // no Bearer
  });

  it("strips caller auth headers (x-api-key, authorization)", () => {
    const req = buildAnthropicRequest({
      accessToken: "sk-ant-oat01-test",
      path: "/v1/messages",
      callerHeaders: {
        "x-api-key": "team-key-should-be-stripped",
        "authorization": "Bearer team-key",
        "anthropic-version": "2023-06-01",
      },
      body: "{}",
    });
    assert.equal(req.headers["x-api-key"], "sk-ant-oat01-test"); // replaced with OAuth token
  });

  it("strips x-anthropic-billing-header", () => {
    const req = buildAnthropicRequest({
      accessToken: "tok",
      path: "/v1/messages",
      callerHeaders: { "x-anthropic-billing-header": "fingerprint-hash" },
      body: "{}",
    });
    assert.ok(!req.headers["x-anthropic-billing-header"]);
  });

  it("passes body through unmodified (tool_use transparency, AC-P2-04)", () => {
    const body = JSON.stringify({
      model: "claude-sonnet-4-20250514",
      messages: [{ role: "user", content: "hi" }],
      tools: [{ name: "bash", description: "run commands" }],
    });
    const req = buildAnthropicRequest({
      accessToken: "tok", path: "/v1/messages", callerHeaders: {}, body,
    });
    assert.equal(req.body, body); // exact same string, not re-serialized
  });
});
```

**Step 2: Implement**

```typescript
const ANTHROPIC_BASE = "https://api.anthropic.com";
const PASSTHROUGH_HEADERS = [
  "anthropic-version", "anthropic-beta", "content-type", "x-claude-code-session-id",
];
const DISCARD_HEADERS = [
  "x-api-key", "authorization", "x-anthropic-billing-header",
  "host", "connection", "transfer-encoding",
];

export interface BuildRequestInput {
  accessToken: string;
  path: string;
  callerHeaders: Record<string, string>;
  body: string;
}

export interface AnthropicRequest {
  url: string;
  headers: Record<string, string>;
  body: string;
}

export function buildAnthropicRequest(input: BuildRequestInput): AnthropicRequest {
  const headers: Record<string, string> = {
    "x-api-key": input.accessToken,
  };

  // Passthrough caller headers (lowercase match)
  const discardSet = new Set(DISCARD_HEADERS);
  const passthroughSet = new Set(PASSTHROUGH_HEADERS);
  for (const [key, val] of Object.entries(input.callerHeaders)) {
    const lk = key.toLowerCase();
    if (discardSet.has(lk)) continue;
    if (passthroughSet.has(lk)) {
      headers[lk] = val;
    }
  }

  // Ensure content-type
  if (!headers["content-type"]) {
    headers["content-type"] = "application/json";
  }

  return {
    url: `${ANTHROPIC_BASE}${input.path}`,
    headers,
    body: input.body, // passthrough — not re-serialized
  };
}

export type ProxySender = import("@agent-mesh/node").ProxySender;

/**
 * Handle a proxy-request frame: construct Anthropic request, fetch, return result.
 * Non-streaming: sends proxy-end with full response body.
 */
export async function handleNonStreamingProxy(
  accessToken: string,
  frame: { path: string; headers: Record<string, string>; body: string },
  sender: ProxySender,
  signal: AbortSignal,
): Promise<void> {
  const req = buildAnthropicRequest({
    accessToken,
    path: frame.path,
    callerHeaders: frame.headers,
    body: frame.body,
  });

  const res = await fetch(req.url, {
    method: "POST",
    headers: req.headers,
    body: req.body,
    signal,
  });

  const body = await res.text();

  // Extract usage from response
  let usage: { model: string; input_tokens: number; output_tokens: number } | undefined;
  try {
    const parsed = JSON.parse(body);
    if (parsed.usage) {
      usage = {
        model: parsed.model ?? "unknown",
        input_tokens: parsed.usage.input_tokens ?? 0,
        output_tokens: parsed.usage.output_tokens ?? 0,
      };
    }
  } catch {}

  sender.end(res.status, body, usage);
}
```

**Step 3: Verify**

```bash
cd packages/plan-bridge-oauth && pnpm build && pnpm test
```

**Step 4: Commit**

```bash
git commit -m "feat(bridge-oauth): non-streaming proxy with header normalization (AC-P2-02/04)"
```

---

## Task 10: Bridge SSE Streaming Proxy (AC-P2-03)

**Files:**
- Modify: `packages/plan-bridge-oauth/src/proxy.ts`
- Extend: `packages/plan-bridge-oauth/src/proxy.test.ts`

**Step 1: Write test**

```typescript
describe("handleStreamingProxy", () => {
  it("sends SSE chunks via sender.chunk()", async () => {
    // Mock fetch that returns SSE body
    const chunks: string[] = [];
    const mockSender = {
      chunk: (data: string) => chunks.push(data),
      end: () => {},
      error: () => {},
    };
    // ... setup mock fetch, call handleStreamingProxy
    // Assert chunks received in order
  });

  it("sends proxy-end with usage from message_delta event", async () => {
    // Mock SSE stream with message_delta containing usage
    let endUsage: unknown;
    const mockSender = {
      chunk: () => {},
      end: (_s: number, _b?: string, u?: object) => { endUsage = u; },
      error: () => {},
    };
    // ... call handleStreamingProxy
    // Assert endUsage has correct input_tokens / output_tokens
  });

  it("aborts upstream fetch when signal fires", async () => {
    // Create AbortController, abort after first chunk
    // Assert fetch was aborted, sender.error() NOT called (abort is intentional)
  });
});
```

**Step 2: Implement**

```typescript
/**
 * Handle a streaming proxy request: fetch with SSE, pipe chunks via sender.
 * Usage is extracted from the message_delta event (Anthropic sends usage at stream end).
 */
export async function handleStreamingProxy(
  accessToken: string,
  frame: { path: string; headers: Record<string, string>; body: string },
  sender: ProxySender,
  signal: AbortSignal,
): Promise<void> {
  const req = buildAnthropicRequest({
    accessToken,
    path: frame.path,
    callerHeaders: frame.headers,
    body: frame.body,
  });

  const res = await fetch(req.url, {
    method: "POST",
    headers: req.headers,
    body: req.body,
    signal,
  });

  if (!res.ok || !res.body) {
    const errBody = await res.text();
    sender.error(errBody, res.status);
    return;
  }

  // Parse SSE stream, extract usage from message_delta
  const reader = res.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";
  let usage: { model: string; input_tokens: number; output_tokens: number } | undefined;
  let model = "unknown";

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      if (signal.aborted) break;

      buffer += decoder.decode(value, { stream: true });
      const events = buffer.split("\n\n");
      buffer = events.pop()!;

      for (const event of events) {
        if (!event.trim()) continue;

        // Send raw SSE event to gateway via WS
        sender.chunk(event + "\n\n");

        // Tee: extract usage (don't consume — just read)
        const dataMatch = event.match(/^data:\s*(.+)$/m);
        if (dataMatch) {
          try {
            const data = JSON.parse(dataMatch[1]);
            if (data.type === "message_start" && data.message?.model) {
              model = data.message.model;
            }
            if (data.type === "message_delta" && data.usage) {
              usage = {
                model,
                input_tokens: data.usage.input_tokens ?? 0,
                output_tokens: data.usage.output_tokens ?? 0,
              };
            }
          } catch {}
        }
      }
    }

    sender.end(res.status, undefined, usage);
  } catch (err) {
    if (signal.aborted) return; // intentional abort — no error
    sender.error(err instanceof Error ? err.message : "streaming error");
  } finally {
    reader.releaseLock();
  }
}
```

**Step 3: Verify**

```bash
cd packages/plan-bridge-oauth && pnpm build && pnpm test
```

**Step 4: Commit**

```bash
git commit -m "feat(bridge-oauth): SSE streaming proxy with usage tee (AC-P2-03/05)"
```

---

## Task 11: Usage Extraction + Reporting (AC-P2-05)

**Files:**
- Create: `packages/plan-bridge-oauth/src/usage.ts`
- Test: `packages/plan-bridge-oauth/src/usage.test.ts`

**说明**：非流式 usage 已在 Task 9 的 `handleNonStreamingProxy` 中提取，流式 usage 已在 Task 10 的 `handleStreamingProxy` 中从 `message_delta` 提取。此 Task 封装上报逻辑。

**Step 1: Write test**

```typescript
import { describe, it } from "node:test";
import assert from "node:assert/strict";
import { formatUsageReport } from "./usage.js";

describe("formatUsageReport", () => {
  it("formats usage for pool-node ledger reporting", () => {
    const report = formatUsageReport({
      model: "claude-sonnet-4-20250514",
      input_tokens: 100,
      output_tokens: 50,
    });
    assert.equal(report.model, "claude-sonnet-4-20250514");
    assert.equal(report.inputTokens, 100);
    assert.equal(report.outputTokens, 50);
  });

  it("handles missing usage gracefully", () => {
    const report = formatUsageReport(undefined);
    assert.equal(report.model, "unknown");
    assert.equal(report.inputTokens, 0);
    assert.equal(report.outputTokens, 0);
  });
});
```

**Step 2: Implement**

```typescript
export interface UsageReport {
  model: string;
  inputTokens: number;
  outputTokens: number;
}

export function formatUsageReport(
  usage?: { model: string; input_tokens: number; output_tokens: number },
): UsageReport {
  if (!usage) return { model: "unknown", inputTokens: 0, outputTokens: 0 };
  return {
    model: usage.model,
    inputTokens: usage.input_tokens,
    outputTokens: usage.output_tokens,
  };
}
```

**Step 3: Commit**

```bash
git commit -m "feat(bridge-oauth): usage extraction and reporting (AC-P2-05)"
```

---

## Task 12: Abort + First-Chunk Timeout (AC-P2-17, AC-P2-18)

**已覆盖于前序 Tasks**：

- **AC-P2-17 (proxy-abort)**: Task 6 Hub WS handler 转发 `proxy-abort`，Task 7 WsChannel `handleProxyRequest` 中 `activeProxyStreams` + `AbortController` 已实现
- **AC-P2-18 (首帧超时)**: Task 5 `ProxyStreamManager.register()` 中 `timeoutMs` + `onTimeout` 已实现，Task 6 Hub `/v1/proxy-invoke` 中 `onTimeout` 返回 504

此 Task 为集成验证——写端到端测试确认两条路径：

**Step 1: Write integration test**

```typescript
// packages/mesh-hub/src/proxy-abort-timeout.test.ts
import { describe, it } from "node:test";
import assert from "node:assert/strict";

describe("proxy abort + timeout integration", () => {
  it("AC-P2-17: client disconnect triggers proxy-abort to bridge", async () => {
    // 1. Start Hub + connect mock bridge via WS
    // 2. Send proxy-invoke to Hub, immediately abort the HTTP request
    // 3. Assert bridge receives proxy-abort frame
    // 4. Assert bridge AbortSignal fires
  });

  it("AC-P2-18: no first chunk within timeout → 504", async () => {
    // 1. Start Hub + connect mock bridge that never responds
    // 2. Send proxy-invoke with short timeout (100ms)
    // 3. Assert HTTP response is 504 with "first chunk timeout"
  });
});
```

**Step 2: Verify existing implementation passes these tests**

```bash
cd packages/mesh-hub && pnpm build && pnpm test
```

**Step 3: Commit**

```bash
git commit -m "test(hub): proxy abort + first-chunk timeout integration tests (AC-P2-17/18)"
```

---

## Task 13: count_tokens Passthrough (AC-P2-16)

**Files:**
- 无需新文件——复用 Task 9 的 `handleNonStreamingProxy`

**说明**：`/v1/messages/count_tokens` 与 `/v1/messages` 走完全相同的 proxy 路径，区别仅在 `path` 字段。Bridge 不区分——passthrough。Gateway/MeshClient 在 `proxyRequest.path` 中传入 `/v1/messages/count_tokens` 即可。

**Step 1: Write test**

```typescript
describe("count_tokens passthrough (AC-P2-16)", () => {
  it("proxies /v1/messages/count_tokens to Anthropic", async () => {
    const req = buildAnthropicRequest({
      accessToken: "tok",
      path: "/v1/messages/count_tokens",
      callerHeaders: { "anthropic-version": "2023-06-01" },
      body: '{"model":"claude-sonnet-4-20250514","messages":[{"role":"user","content":"hi"}]}',
    });
    assert.equal(req.url, "https://api.anthropic.com/v1/messages/count_tokens");
  });
});
```

**Step 2: Verify**

既有 `buildAnthropicRequest` 已支持任意 path，无需修改。测试通过即确认。

**Step 3: Commit**

```bash
git commit -m "test(bridge-oauth): count_tokens passthrough verification (AC-P2-16)"
```

---

## Task 14: Gateway Proxy Path (Phase 1A 最小集)

**Files:**
- Modify: `packages/plan-pool/src/api-gateway.ts`
- Modify: `packages/plan-pool/src/index.ts`

**说明**：Phase 1A 仅需最小网关支持——让 `/v1/messages` 端点能将 oauth-proxy bridge 的请求走 `proxyInvoke` 路径。完整路由（bridge 类型检测、capability routing）是 Phase 1B。

**Step 1: Extend `createApiGateway` options**

```typescript
// In api-gateway.ts — add to ApiGatewayOptions:
interface ApiGatewayOptions {
  // ... existing
  /** Streaming proxy invocation — for oauth/api bridges. */
  proxyInvokeBridge?: (
    bridge: BridgeEntry,
    req: { method: string; path: string; headers: Record<string, string>; body: string; stream: boolean },
  ) => Promise<ProxyInvokeResult | AsyncIterable<string>>;
}
```

**Step 2: Add `proxyPassthrough()` in api-gateway.ts**

```typescript
async function proxyPassthrough(
  req: FastifyRequest,
  reply: FastifyReply,
  bridge: BridgeEntry,
  proxyInvoke: NonNullable<ApiGatewayOptions["proxyInvokeBridge"]>,
): Promise<void> {
  const rawBody = JSON.stringify(req.body);
  const isStreaming = typeof req.body === "object" && (req.body as Record<string, unknown>).stream === true;

  // Extract passthrough headers from caller
  const callerHeaders: Record<string, string> = {};
  for (const key of ["anthropic-version", "anthropic-beta", "content-type", "x-claude-code-session-id"]) {
    const val = req.headers[key];
    if (typeof val === "string") callerHeaders[key] = val;
  }

  const result = await proxyInvoke(bridge, {
    method: "POST",
    path: req.url,  // "/v1/messages" or "/v1/messages/count_tokens"
    headers: callerHeaders,
    body: rawBody,
    stream: isStreaming,
  });

  if (!isStreaming) {
    // Non-streaming: result is ProxyInvokeResult
    const r = result as { status: number; body: string };
    reply.code(r.status).header("content-type", "application/json").send(r.body);
    return;
  }

  // Streaming: result is AsyncIterable<string>
  reply.raw.writeHead(200, {
    "content-type": "text/event-stream",
    "cache-control": "no-cache",
    connection: "keep-alive",
  });
  for await (const chunk of result as AsyncIterable<string>) {
    reply.raw.write(chunk);
  }
  reply.raw.end();
}
```

**Step 3: Wire in index.ts**

```typescript
// In index.ts, add proxyInvokeBridge closure alongside invokeBridge:
const proxyInvokeBridge = async (
  bridge: BridgeEntry,
  req: { method: string; path: string; headers: Record<string, string>; body: string; stream: boolean },
) => {
  return client.proxyInvoke(bridge.clientName, {
    method: req.method,
    path: req.path,
    headers: req.headers,
    body: req.body,
    stream: req.stream,
  });
};

// Pass to createApiGateway:
const app = await createApiGateway({
  // ... existing
  proxyInvokeBridge,
});
```

**Step 4: Verify**

```bash
cd packages/plan-pool && pnpm build && pnpm test
```

**Step 5: Commit**

```bash
git commit -m "feat(plan-pool): minimal gateway proxy path for oauth-proxy bridges"
```

---

## Task 15: Bridge Main + Registration

**Files:**
- Create: `packages/plan-bridge-oauth/src/bridge.ts`

**Step 1: Implement bridge main** (follows plan-bridge-api pattern)

```typescript
import {
  MeshClient, WsChannel, getPublicKeyBase64url, loadOrCreateIdentity,
} from "@agent-mesh/node";
import { createTokenProvider, type TokenProvider } from "./token-provider.js";
import { handleNonStreamingProxy, handleStreamingProxy } from "./proxy.js";

const HUB_URL = process.env.MESH_HUB_URL ?? "http://127.0.0.1:3010";
const NODE_ID = process.env.PLAN_BRIDGE_OAUTH_NODE_ID ?? "plan-bridge-oauth";
const IDENTITY_DIR = process.env.PLAN_BRIDGE_OAUTH_IDENTITY_DIR ?? ".plan-bridge-oauth-identity";
const MAX_CONCURRENCY = Number(process.env.PLAN_BRIDGE_OAUTH_MAX_CONCURRENCY ?? "2");
const HEARTBEAT_MS = Number(process.env.PLAN_BRIDGE_OAUTH_HEARTBEAT_MS ?? "10000");
const POOL_NODE_URL = process.env.POOL_NODE_URL ?? "";
const POOL_API_KEY = process.env.POOL_API_KEY ?? "";
const BRIDGE_VERSION = process.env.PLAN_BRIDGE_OAUTH_VERSION ?? "0.1.0";

async function main(): Promise<void> {
  // 1. Create TokenProvider
  const tokenProvider = createTokenProvider({
    credentialPath: process.env.CLAUDE_CREDENTIAL_PATH,
  });
  tokenProvider.onStatusChange((healthy) => {
    console.log(`[bridge-oauth] token health: ${healthy ? "OK" : "UNHEALTHY"}`);
  });

  // Verify initial token read
  try {
    const token = await tokenProvider.getAccessToken();
    console.log(`[bridge-oauth] OAuth token loaded (${token.slice(0, 20)}...)`);
  } catch (err) {
    console.error(`[bridge-oauth] FATAL: cannot read OAuth token: ${err}`);
    process.exit(1);
  }

  // 2. A2A identity + client
  const identity = await loadOrCreateIdentity(IDENTITY_DIR, NODE_ID);
  console.log(`[bridge-oauth] nodeId: ${NODE_ID}`);
  console.log(`[bridge-oauth] publicKey: ${getPublicKeyBase64url(identity)}`);

  const client = new MeshClient({ hubUrl: HUB_URL, identity });
  console.log("[bridge-oauth] Sending HELLO...");
  const helloResult = await client.hello();
  console.log("[bridge-oauth] HELLO OK");

  // 3. Hub public key for L2 verification
  const hubPubkeyRes = await fetch(`${HUB_URL}/v1/pubkey`);
  if (!hubPubkeyRes.ok) throw new Error(`Failed to fetch hub pubkey: ${hubPubkeyRes.status}`);
  const hubPubkey = await hubPubkeyRes.json() as JsonWebKey;

  // 4. WsChannel with proxy handler
  let activeSessions = 0;

  const hubWsUrl = HUB_URL.replace(/^http/, "ws") + "/v1/ws";
  const wsChannel = new WsChannel({
    hubWsUrl,
    nodeId: NODE_ID,
    handler: async () => ({ status: "error" as const, payload: "use proxy-request", durationMs: 0 }),
    hubPublicKeyJwk: hubPubkey,
    proxyHandler: async (frame, sender, signal) => {
      if (activeSessions >= MAX_CONCURRENCY) {
        sender.error("max concurrency reached", 429);
        return;
      }
      activeSessions++;
      try {
        const accessToken = await tokenProvider.getAccessToken();
        const body = frame.body;
        const isStreaming = (() => { try { return JSON.parse(body).stream === true; } catch { return false; } })();

        if (isStreaming) {
          await handleStreamingProxy(accessToken, frame, sender, signal);
        } else {
          await handleNonStreamingProxy(accessToken, frame, sender, signal);
        }
      } catch (err) {
        if (signal.aborted) return;
        // 401 recovery: re-read token and retry once
        if (err instanceof Error && err.message.includes("401")) {
          tokenProvider.invalidate();
          try {
            const freshToken = await tokenProvider.getAccessToken();
            const isStreaming = (() => { try { return JSON.parse(frame.body).stream === true; } catch { return false; } })();
            if (isStreaming) {
              await handleStreamingProxy(freshToken, frame, sender, signal);
            } else {
              await handleNonStreamingProxy(freshToken, frame, sender, signal);
            }
            return;
          } catch {
            sender.error("auth recovery failed", 401);
            return;
          }
        }
        sender.error(err instanceof Error ? err.message : "proxy error");
      } finally {
        activeSessions--;
      }
    },
    log: {
      info: (...args: unknown[]) => console.log("[ws]", ...args),
      warn: (...args: unknown[]) => console.warn("[ws]", ...args),
    },
  });
  await wsChannel.connect(helloResult.meshCertificate);
  console.log("[bridge-oauth] WS connected to Hub");

  // 5. Pool registration
  if (POOL_NODE_URL && POOL_API_KEY) {
    try {
      const regRes = await fetch(`${POOL_NODE_URL}/v1/bridges/register`, {
        method: "POST",
        headers: { "content-type": "application/json", authorization: `Bearer ${POOL_API_KEY}` },
        body: JSON.stringify({
          bridgeId: NODE_ID,
          provider: "claude",
          shareMode: "idle-only",
          externalMaxConcurrency: MAX_CONCURRENCY,
          attestation: { binaryHash: "oauth-proxy", version: BRIDGE_VERSION },
          capabilities: {
            supportsTools: true,
            supportsStreaming: true,
            supportsStructuredInput: true,
            bridgeType: "oauth-proxy",
          },
        }),
      });
      if (regRes.ok) {
        console.log("[bridge-oauth] pool registration OK");
      } else {
        console.error(`[bridge-oauth] pool registration failed: ${regRes.status}`);
      }
    } catch (err) {
      console.error(`[bridge-oauth] pool registration error: ${err}`);
    }
  }

  // 6. Heartbeat
  client.startHeartbeat(HEARTBEAT_MS);
  console.log("[bridge-oauth] Ready — proxying requests...");

  // 7. Graceful shutdown
  for (const sig of ["SIGINT", "SIGTERM"] as const) {
    process.on(sig, async () => {
      console.log(`[bridge-oauth] ${sig} — shutting down...`);
      tokenProvider.close();
      wsChannel.close();
      await client.close();
      process.exit(0);
    });
  }
}

main().catch((error) => {
  console.error("[bridge-oauth] FATAL:", error);
  process.exit(1);
});
```

**Step 2: Verify**

```bash
cd packages/plan-bridge-oauth && pnpm build && pnpm start
# Expected: reads OAuth token, connects to Hub, registers with pool
```

**Step 3: Commit**

```bash
git commit -m "feat(bridge-oauth): bridge main with A2A registration + capability (AC-P2-01..06)"
```

---

## AC Coverage Matrix

| AC | Task(s) | 验证方式 |
|---|---|---|
| AC-P2-01 | T3, T15 | Unit test: credential reader + bridge startup |
| AC-P2-02 | T9, T15 | Unit test: non-streaming proxy + E2E via bridge |
| AC-P2-03 | T10, T15 | Unit test: SSE streaming + E2E via bridge |
| AC-P2-04 | T9 | Unit test: body passthrough (tool_use JSON 不被修改) |
| AC-P2-05 | T10, T11 | Unit test: usage extraction from response + SSE stream |
| AC-P2-06 | T4 | Unit test: polling reads, no refresh when token valid |
| AC-P2-16 | T13 | Unit test: path passthrough for count_tokens |
| AC-P2-17 | T7, T12 | Integration test: abort propagation through WS |
| AC-P2-18 | T5, T6, T12 | Integration test: first-chunk timeout → 504 |
| AC-P2-22 | T4 | Unit test: grace period + last-resort refresh + write-back |
| AC-P2-23 | T4, T15 | Unit test: invalidate → re-read; 401 recovery in bridge |

---

## Risk Notes

1. **Hub `reply.hijack()`**：Fastify 的 `reply.hijack()` 用于 SSE 手动写入。需验证 Fastify 5 的 hijack 行为——若不支持则改用 `reply.raw` 直接写入 + 跳过 Fastify 生命周期。
2. **WsChannel 双 handler**：现有 `handler` (InvocationHandler) + 新增 `proxyHandler`。message handler 按 type 分发，无冲突。但需确保 proxy-request 不经过 L2 本地校验（L2 已在 Hub `/v1/proxy-invoke` 校验过）。
3. **MeshClient `proxyInvoke()` 的 `fetch()` streaming**：Node.js 18+ 的 `fetch()` 支持 `response.body` 作为 ReadableStream。若部署环境 Node < 18，需用 `undici` 的 `fetch`（mesh-node 已依赖 undici）。
4. **Token 并发安全**：`pollOnce()` 和 `invalidate()` 都可能触发异步 credential 读取/刷新。由于 `await` 点会交出控制权，纯 JS 单线程不足以防止重入。已通过 `pollInFlight` / `refreshInFlight` 显式布尔锁防止双读/双刷/双写回（gpt52 review fix）。
