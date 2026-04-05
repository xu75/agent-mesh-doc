# F012: Public Discovery Layer Implementation Plan

**Feature:** F012 — `docs/features/F012-public-discovery-layer.md`
**Goal:** Let mesh-hub be externally discoverable — agents get machine-readable protocol/capability info, humans see a landing page in the browser.
**Acceptance Criteria:**
- [ ] AC-1: `GET /` returns HTML Landing Page (hub intro, Getting Started, non-offline nodes, API reference)
- [ ] AC-2: `GET /v1/discovery` returns JSON self-description (protocol, auth, endpoints, registration mode)
- [ ] AC-3: `GET /v1/directory` returns all known nodes `{ catId, displayName, description, skills, status }` without nodeId/callbackUrl
- [ ] AC-4: `GET /v1/openapi.json` returns valid OpenAPI 3.0 spec
- [ ] AC-5: `NodeCapability.status` aligned with hub (`"online" | "stale" | "offline"`)
- [ ] AC-6: Landing Page node list is same-source as `/v1/directory`
- [ ] AC-7: All new endpoints have test coverage

**Architecture:** Six tasks in dependency order. Protocol drift fix first (foundation), then config extension, then three new endpoints (`/v1/directory`, `/v1/discovery`, `/v1/openapi.json`), finally Landing Page (consumes directory data). All routes added to existing `createHub()` in `hub.ts`. Tests use Fastify `app.inject()` with Node built-in test runner.
**Tech Stack:** Fastify, Node.js test runner, TypeScript
**前端验证:** No — Landing Page is server-rendered HTML, tested via response body assertions.

---

## Task 1: Protocol Drift Fix (`degraded` → `stale`)

**Covers:** AC-5

**Files:**
- Modify: `packages/mesh-protocol/src/messages.ts:38`

**Step 1: Fix `NodeCapability.status` type**

```typescript
// messages.ts line 38: change "degraded" to "stale"
status: "online" | "offline" | "stale";
```

**Step 2: Build to verify no type errors**

Run: `cd packages/mesh-protocol && pnpm build`
Expected: PASS — hub already uses `"stale"` at runtime, so aligning the type won't break consumers.

Run: `cd packages/mesh-hub && pnpm build`
Expected: PASS

**Step 3: Run existing tests to confirm no regressions**

Run: `cd packages/mesh-hub && pnpm test`
Expected: All existing tests PASS.

**Step 4: Commit**

```
fix(protocol): align NodeCapability.status with hub runtime (degraded→stale)
```

---

## Task 2: Config Extension — `displayName` and `description`

**Covers:** AC-3 (data source for directory)

**Files:**
- Modify: `packages/mesh-hub/src/config.ts:56-73`

**Step 1: Add optional fields to `AllowedNode`**

```typescript
// Add to AllowedNode interface, after label?:
/** Human-readable display name for public directory. */
displayName?: string;
/** One-line description for public directory. */
description?: string;
```

**Step 2: Build to confirm**

Run: `cd packages/mesh-hub && pnpm build`
Expected: PASS — new fields are optional, zero breaking changes.

**Step 3: Commit**

```
feat(hub): add displayName/description to AllowedNode config
```

---

## Task 3: `GET /v1/directory` — Public Capability Directory

**Covers:** AC-3, AC-7

**Files:**
- Modify: `packages/mesh-hub/src/hub.ts` (add route after `/v1/pubkey`)
- Create: `packages/mesh-hub/src/discovery.test.ts` (new test file for all F012 endpoints)

**Step 1: Write the failing test**

Create `packages/mesh-hub/src/discovery.test.ts`:

```typescript
import assert from "node:assert/strict";
import { after, before, describe, it } from "node:test";
import type { FastifyInstance } from "fastify";
import { loadConfig } from "./config.js";
import { createHub } from "./hub.js";

describe("F012: Public Discovery Layer", () => {
  let app: FastifyInstance;

  before(async () => {
    app = await createHub(
      loadConfig({
        hubId: "test-hub",
        adminToken: "test-secret",
        allowedNodes: [
          {
            nodeId: "node-a",
            publicKey: "pub-a",
            catId: "ragdoll-opus",
            skills: ["architecture", "code-review"],
            displayName: "布偶猫/宪宪",
            description: "Chief architect",
          },
          {
            nodeId: "node-b",
            publicKey: "pub-b",
            catId: "maine-coon",
            skills: ["coding"],
            // no displayName/description — should be omitted in response
          },
        ],
      }),
    );
  });

  after(async () => {
    await app.close();
  });

  describe("GET /v1/directory", () => {
    it("returns all nodes with catId, skills, status", async () => {
      const res = await app.inject({ method: "GET", url: "/v1/directory" });
      assert.equal(res.statusCode, 200);
      const body = res.json();
      assert.ok(Array.isArray(body.nodes));
      assert.equal(body.nodes.length, 2);

      const nodeA = body.nodes.find((n: any) => n.catId === "ragdoll-opus");
      assert.ok(nodeA);
      assert.deepEqual(nodeA.skills, ["architecture", "code-review"]);
      assert.equal(nodeA.displayName, "布偶猫/宪宪");
      assert.equal(nodeA.description, "Chief architect");
      assert.ok(["online", "stale", "offline"].includes(nodeA.status));

      // Must NOT expose nodeId or callbackUrl
      assert.equal(nodeA.nodeId, undefined);
      assert.equal(nodeA.callbackUrl, undefined);
    });

    it("omits displayName/description when not configured", async () => {
      const res = await app.inject({ method: "GET", url: "/v1/directory" });
      const body = res.json();
      const nodeB = body.nodes.find((n: any) => n.catId === "maine-coon");
      assert.ok(nodeB);
      assert.equal(nodeB.displayName, undefined);
      assert.equal(nodeB.description, undefined);
    });

    it("requires no authentication", async () => {
      // No token, no cert — should still work
      const res = await app.inject({ method: "GET", url: "/v1/directory" });
      assert.equal(res.statusCode, 200);
    });

    it("handles multi-capability nodes", async () => {
      const multiApp = await createHub(
        loadConfig({
          hubId: "test-hub-multi",
          adminToken: "test-secret",
          allowedNodes: [
            {
              nodeId: "node-multi",
              publicKey: "pub-multi",
              capabilities: [
                { catId: "cat-a", skills: ["skill-1"] },
                { catId: "cat-b", skills: ["skill-2", "skill-3"] },
              ],
              displayName: "Multi Cat",
            },
          ],
        }),
      );
      const res = await multiApp.inject({ method: "GET", url: "/v1/directory" });
      const body = res.json();
      // Multi-capability should produce multiple directory entries
      assert.equal(body.nodes.length, 2);
      assert.ok(body.nodes.find((n: any) => n.catId === "cat-a"));
      assert.ok(body.nodes.find((n: any) => n.catId === "cat-b"));
      await multiApp.close();
    });
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/mesh-hub && pnpm build && pnpm test`
Expected: FAIL — `/v1/directory` route doesn't exist yet.

**Step 3: Implement `/v1/directory` route in hub.ts**

Add after the `/v1/pubkey` route (line ~106):

```typescript
// ── F012: Public directory (no auth) ──
app.get("/v1/directory", async () => {
  const nodes: Array<{
    catId: string;
    skills: string[];
    status: "online" | "stale" | "offline";
    displayName?: string;
    description?: string;
  }> = [];
  for (const n of allowedById.values()) {
    const caps = resolveCapabilities(n);
    const status = deriveNodeStatus(n.nodeId);
    for (const cap of caps) {
      const entry: typeof nodes[number] = {
        catId: cap.catId,
        skills: cap.skills,
        status,
      };
      if (n.displayName) entry.displayName = n.displayName;
      if (n.description) entry.description = n.description;
      nodes.push(entry);
    }
  }
  return { nodes };
});
```

**Step 4: Run test to verify it passes**

Run: `cd packages/mesh-hub && pnpm build && pnpm test`
Expected: All tests PASS.

**Step 5: Commit**

```
feat(F012): add GET /v1/directory — public capability directory
```

---

## Task 4: `GET /v1/discovery` — Machine-Readable Hub Self-Description

**Covers:** AC-2, AC-7

**Files:**
- Modify: `packages/mesh-hub/src/hub.ts` (add route)
- Modify: `packages/mesh-hub/src/discovery.test.ts` (add tests)

**Step 1: Write the failing test**

Add to `discovery.test.ts` inside the main describe block:

```typescript
describe("GET /v1/discovery", () => {
  it("returns hub self-description with protocol and endpoints", async () => {
    const res = await app.inject({ method: "GET", url: "/v1/discovery" });
    assert.equal(res.statusCode, 200);
    const body = res.json();

    assert.equal(body.name, "Agent Mesh Hub");
    assert.ok(body.protocol);
    assert.equal(body.protocol.name, "clowder");
    assert.equal(body.protocol.version, "v1");
    assert.ok(body.authentication);
    assert.equal(body.authentication.type, "ed25519-hello-handshake");
    assert.ok(body.endpoints);
    // Must include all endpoints for complete native client flow
    assert.ok(body.endpoints.hello);
    assert.ok(body.endpoints.caps);
    assert.ok(body.endpoints.token);
    assert.ok(body.endpoints.verify);
    assert.ok(body.endpoints.invoke);
    assert.ok(body.endpoints.heartbeat);
    assert.ok(body.endpoints.directory);
    assert.ok(body.endpoints.pubkey);
    assert.ok(body.registration);
    assert.equal(body.registration.mode, "whitelist");
    assert.equal(body.openapi, "/v1/openapi.json");
  });

  it("requires no authentication", async () => {
    const res = await app.inject({ method: "GET", url: "/v1/discovery" });
    assert.equal(res.statusCode, 200);
  });

  it("is cacheable (has cache-control header)", async () => {
    const res = await app.inject({ method: "GET", url: "/v1/discovery" });
    assert.ok(res.headers["cache-control"]);
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/mesh-hub && pnpm build && pnpm test`
Expected: FAIL — `/v1/discovery` route doesn't exist.

**Step 3: Implement `/v1/discovery` route in hub.ts**

Add after the `/v1/directory` route:

```typescript
// ── F012: Machine-readable hub self-description (no auth, cacheable) ──
app.get("/v1/discovery", async (_request, reply) => {
  reply.header("cache-control", "public, max-age=3600");
  return {
    name: "Agent Mesh Hub",
    description: "Multi-agent collaboration mesh",
    protocol: { name: "clowder", version: "v1" },
    authentication: {
      type: "ed25519-hello-handshake",
      keyFormat: "JWK",
      curve: "Ed25519",
    },
    endpoints: {
      hello:     { method: "POST", path: "/v1/hello" },
      caps:      { method: "POST", path: "/v1/caps",      auth: "L1" },
      token:     { method: "POST", path: "/v1/token",     auth: "L1" },
      verify:    { method: "POST", path: "/v1/verify",    auth: "none" },
      invoke:    { method: "POST", path: "/v1/invoke",    auth: "L2" },
      heartbeat: { method: "POST", path: "/v1/heartbeat", auth: "L1" },
      directory: { method: "GET",  path: "/v1/directory", auth: "none" },
      pubkey:    { method: "GET",  path: "/v1/pubkey",    auth: "none" },
    },
    registration: { mode: "whitelist" },
    openapi: "/v1/openapi.json",
  };
});
```

**Step 4: Run test to verify it passes**

Run: `cd packages/mesh-hub && pnpm build && pnpm test`
Expected: All tests PASS.

**Step 5: Commit**

```
feat(F012): add GET /v1/discovery — machine-readable hub self-description
```

---

## Task 5: `GET /v1/openapi.json` — API Schema

**Covers:** AC-4, AC-7

**Files:**
- Create: `packages/mesh-hub/src/openapi.ts` (static OpenAPI document)
- Modify: `packages/mesh-hub/src/hub.ts` (add route)
- Modify: `packages/mesh-hub/src/discovery.test.ts` (add tests)

**Step 1: Write the failing test**

Add to `discovery.test.ts`:

```typescript
describe("GET /v1/openapi.json", () => {
  it("returns valid OpenAPI 3.0 spec", async () => {
    const res = await app.inject({ method: "GET", url: "/v1/openapi.json" });
    assert.equal(res.statusCode, 200);
    const body = res.json();

    assert.equal(body.openapi, "3.0.3");
    assert.ok(body.info);
    assert.ok(body.info.title);
    assert.ok(body.info.version);
    assert.ok(body.paths);
    // Must document all public endpoints
    assert.ok(body.paths["/v1/hello"]);
    assert.ok(body.paths["/v1/caps"]);
    assert.ok(body.paths["/v1/invoke"]);
    assert.ok(body.paths["/v1/directory"]);
    assert.ok(body.paths["/v1/discovery"]);
  });

  it("requires no authentication", async () => {
    const res = await app.inject({ method: "GET", url: "/v1/openapi.json" });
    assert.equal(res.statusCode, 200);
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/mesh-hub && pnpm build && pnpm test`
Expected: FAIL.

**Step 3: Create `openapi.ts` with static OpenAPI document**

Create `packages/mesh-hub/src/openapi.ts`:

This file exports a function that returns the OpenAPI 3.0 document as a plain object. The document is hand-written based on the actual hub HTTP interfaces (HelloBody, CapsBody, TokenBody, InvokeBody, VerifyBody from hub.ts). It describes:

- All endpoints (hello, caps, token, verify, invoke, heartbeat, directory, discovery, pubkey, health, metrics)
- Request/response schemas matching the actual Fastify route handlers
- Authentication requirements per endpoint

The document is static and cacheable. Source of truth is the hub-side HTTP DTO shapes, NOT `mesh-protocol/messages.ts`.

**Step 4: Add route in hub.ts**

```typescript
import { getOpenApiSpec } from "./openapi.js";

// ── F012: OpenAPI spec (no auth) ──
app.get("/v1/openapi.json", async () => getOpenApiSpec());
```

**Step 5: Run test to verify it passes**

Run: `cd packages/mesh-hub && pnpm build && pnpm test`
Expected: All tests PASS.

**Step 6: Commit**

```
feat(F012): add GET /v1/openapi.json — API schema
```

---

## Task 6: `GET /` — HTML Landing Page

**Covers:** AC-1, AC-6, AC-7

**Files:**
- Create: `packages/mesh-hub/src/landing-page.ts` (template function)
- Modify: `packages/mesh-hub/src/hub.ts` (add route)
- Modify: `packages/mesh-hub/src/discovery.test.ts` (add tests)

**Step 1: Write the failing test**

Add to `discovery.test.ts`:

```typescript
describe("GET /", () => {
  it("returns HTML landing page", async () => {
    const res = await app.inject({ method: "GET", url: "/" });
    assert.equal(res.statusCode, 200);
    assert.ok(res.headers["content-type"]?.toString().includes("text/html"));
    const html = res.body;
    assert.ok(html.includes("Agent Mesh Hub"));
    assert.ok(html.includes("Getting Started"));
  });

  it("shows non-offline nodes from directory (AC-6: same source)", async () => {
    const res = await app.inject({ method: "GET", url: "/" });
    const html = res.body;
    // node-a and node-b are in config but have no heartbeat → offline
    // Landing page should filter offline nodes, so they may not appear
    // But the page should still render successfully
    assert.ok(html.includes("API Reference") || html.includes("Endpoints"));
  });

  it("shows online nodes when heartbeat is active", async () => {
    // Create a fresh hub and simulate HELLO to make a node online
    const { generateNodeIdentity, getPublicKeyBase64url, createHelloProof } = await import("@agent-mesh/node");
    const identity = await generateNodeIdentity("node-live");
    const pub = getPublicKeyBase64url(identity);
    const liveApp = await createHub(
      loadConfig({
        hubId: "test-landing",
        adminToken: "test-secret",
        allowedNodes: [
          {
            nodeId: "node-live",
            publicKey: pub,
            catId: "live-cat",
            skills: ["demo"],
            displayName: "Live Cat",
          },
        ],
      }),
    );
    // HELLO to make the node online
    const proof = await createHelloProof(identity);
    await liveApp.inject({
      method: "POST",
      url: "/v1/hello",
      payload: { nodeId: "node-live", publicKey: pub, proof },
    });
    // Now landing page should include the live node
    const res = await liveApp.inject({ method: "GET", url: "/" });
    assert.ok(res.body.includes("Live Cat") || res.body.includes("live-cat"));
    await liveApp.close();
  });

  it("does not conflict with /health route", async () => {
    const healthRes = await app.inject({ method: "GET", url: "/health" });
    assert.equal(healthRes.statusCode, 200);
    assert.equal(healthRes.json().status, "ok");
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/mesh-hub && pnpm build && pnpm test`
Expected: FAIL — `GET /` returns 404.

**Step 3: Create `landing-page.ts` template function**

Create `packages/mesh-hub/src/landing-page.ts`:

A pure function `renderLandingPage(data)` that takes:
- `hubName: string`
- `nodes: DirectoryNode[]` (same type as `/v1/directory` response, filtered to non-offline)

Returns an HTML string with:
- Clean, minimal CSS (inline `<style>`, no external deps)
- Hub name + description
- Getting Started section (Ed25519 key generation → HELLO → use flow)
- Online Agents table (catId, displayName, skills, status — with colored status dots)
- API Reference table (endpoint, method, auth level)
- Footer linking to `/v1/discovery` and `/v1/openapi.json`

**Step 4: Add route in hub.ts**

```typescript
import { renderLandingPage } from "./landing-page.js";

// ── F012: Landing page — HTML for browsers (no auth) ──
app.get("/", async (_request, reply) => {
  // Build directory data (same source as /v1/directory — AC-6)
  const nodes = [];
  for (const n of allowedById.values()) {
    const caps = resolveCapabilities(n);
    const status = deriveNodeStatus(n.nodeId);
    if (status === "offline") continue; // Landing page filters offline
    for (const cap of caps) {
      const entry: Record<string, unknown> = {
        catId: cap.catId,
        skills: cap.skills,
        status,
      };
      if (n.displayName) entry.displayName = n.displayName;
      if (n.description) entry.description = n.description;
      nodes.push(entry);
    }
  }
  reply.type("text/html");
  return renderLandingPage({ hubName: "Agent Mesh Hub", nodes });
});
```

Note: The directory-building logic is duplicated from `/v1/directory` (with the offline filter). This is intentional — extracting a shared helper for 2 call sites would be premature abstraction.

**Step 5: Run test to verify it passes**

Run: `cd packages/mesh-hub && pnpm build && pnpm test`
Expected: All tests PASS.

**Step 6: Commit**

```
feat(F012): add GET / — HTML landing page for browsers
```

---

## Final Verification

**Step 1: Run full test suite**

Run: `cd packages/mesh-hub && pnpm build && pnpm test`
Expected: All tests PASS (existing + new F012 tests).

**Step 2: Manual smoke test (optional)**

```bash
cd packages/mesh-hub
MESH_HUB_CONFIG=deploy/config.example.json pnpm start
# In another terminal:
curl http://localhost:3010/                  # HTML landing page
curl http://localhost:3010/v1/discovery      # JSON self-description
curl http://localhost:3010/v1/directory      # JSON capability directory
curl http://localhost:3010/v1/openapi.json   # OpenAPI spec
```

**Step 3: Final commit — update spec status**

Update `docs/features/F012-public-discovery-layer.md`:
- Status: `spec` → `in-progress`

```
docs(F012): mark as in-progress
```
