# F011: Local TTS Bridge Example Implementation Plan

**Feature:** F011 — `docs/features/F011-local-tts-bridge-example.md`
**Goal:** Deliver a reproducible TTS example chain: Caller Agent -> Mesh Hub -> TTS Bridge -> Local TTS Runtime -> Audio Result, validating how local models are safely reused in the mesh.
**Acceptance Criteria:**
- [ ] AC-1: `bridge-tts` completes HELLO + heartbeat registration; Hub CAPS shows `tts.synthesize` online.
- [ ] AC-2: Remote agent via mesh invoke gets playable audio result (base64).
- [ ] AC-3: Scope insufficient returns auditable denial (non-500, semantic error code).
- [ ] AC-4: End-to-end traceId trackable across caller/hub/bridge/runtime.
- [ ] AC-5: Docs provide "one command start + one command invoke" minimal repro.

**Architecture:** Three layers: `packages/bridge-tts/` (TypeScript mesh node — identity, HELLO, heartbeat, INVOKE handler), `bridges/qwen3-tts/` (Python FastAPI HTTP wrapper — BYO Runtime), and example caller script. Bridge-tts follows reviewer-node.ts pattern exactly. Bridge calls runtime via local HTTP. Hub scope enforcement handles AC-3 natively; bridge adds defense-in-depth scope guard. TraceId flows through all layers via invocation payload + HTTP headers.
**Tech Stack:** TypeScript (mesh node), Python + FastAPI (runtime wrapper), pnpm workspace
**前端验证:** No

**Design Decisions (confirmed with CVO):**
1. Base64 audio output (contract reserves `audioUrl` for future)
2. Standalone Python HTTP process (FastAPI), bridge calls via `TTS_RUNTIME_URL`
3. Bridge does 1 quick retry on local HTTP failure; mesh layer does not retry
4. Default voice/speed; contract reserves optional `voice`/`speed` params

---

## Task 1: TTS Contract Types

**Covers:** Foundation for AC-2

**Files:**
- Create: `packages/bridge-tts/package.json`
- Create: `packages/bridge-tts/tsconfig.json`
- Create: `packages/bridge-tts/src/types.ts`
- Create: `packages/bridge-tts/src/index.ts`

**Step 1: Create package scaffold**

`packages/bridge-tts/package.json`:
```json
{
  "name": "@agent-mesh/bridge-tts",
  "version": "0.0.1",
  "description": "TTS bridge capability for agent-mesh — synthesize text to speech via local runtime",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "lint": "tsc --noEmit",
    "test": "node --test dist/**/*.test.js"
  },
  "dependencies": {
    "@agent-mesh/node": "workspace:*"
  },
  "devDependencies": {
    "@types/node": "^22.13.10"
  }
}
```

`packages/bridge-tts/tsconfig.json`:
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

**Step 2: Define terminal schema types**

`packages/bridge-tts/src/types.ts`:
```typescript
/** TTS synthesize request — sent as INVOKE payload.task (JSON string). */
export interface TtsSynthesizeRequest {
  text: string;
  voice?: string;
  speed?: number;
}

/** TTS synthesize result — returned as INVOKE response payload (JSON string). */
export interface TtsSynthesizeResult {
  audio: string;         // base64-encoded audio data
  audioUrl?: string;     // reserved for future: URL to audio file
  format: "wav" | "mp3";
  durationMs: number;
  sampleRate: number;
}

/** Configuration for the TTS bridge node. */
export interface TtsBridgeConfig {
  hubUrl: string;
  nodeId: string;
  identityDir: string;
  port: number;
  /** URL of the local TTS runtime (e.g. http://127.0.0.1:5001) */
  runtimeUrl: string;
  /** Heartbeat interval in milliseconds. Default: 10000 */
  heartbeatIntervalMs?: number;
}
```

`packages/bridge-tts/src/index.ts`:
```typescript
export type {
  TtsSynthesizeRequest,
  TtsSynthesizeResult,
  TtsBridgeConfig,
} from "./types.js";
```

**Step 3: Build to verify**

Run: `cd packages/bridge-tts && pnpm install && pnpm build`
Expected: PASS — types compile cleanly.

**Step 4: Commit**

```bash
git add packages/bridge-tts/
git commit -m "feat(F011): scaffold bridge-tts package with TTS contract types"
```

---

## Task 2: Synthesizer Client (bridge → runtime HTTP)

**Covers:** AC-2 (audio result), AC-4 (traceId propagation to runtime)

**Files:**
- Create: `packages/bridge-tts/src/synthesizer.ts`
- Create: `packages/bridge-tts/src/synthesizer.test.ts`
- Modify: `packages/bridge-tts/src/index.ts`

**Step 1: Write the failing test**

`packages/bridge-tts/src/synthesizer.test.ts`:
```typescript
import { describe, it, after, before } from "node:test";
import assert from "node:assert/strict";
import { createServer, type Server } from "node:http";
import { synthesize } from "./synthesizer.js";

describe("synthesize()", () => {
  let mockServer: Server;
  let port: number;

  before(async () => {
    mockServer = createServer((req, res) => {
      // Collect body
      const chunks: Buffer[] = [];
      req.on("data", (c) => chunks.push(c));
      req.on("end", () => {
        const body = JSON.parse(Buffer.concat(chunks).toString());
        // Echo back a fake audio result
        res.writeHead(200, { "content-type": "application/json" });
        res.end(JSON.stringify({
          audio_base64: "AAAA",   // fake base64
          format: "wav",
          duration_ms: 1200,
          sample_rate: 24000,
        }));
      });
    });
    await new Promise<void>((resolve) => {
      mockServer.listen(0, "127.0.0.1", () => resolve());
    });
    const addr = mockServer.address();
    port = typeof addr === "object" && addr ? addr.port : 0;
  });

  after(async () => {
    await new Promise<void>((resolve) => mockServer.close(() => resolve()));
  });

  it("calls runtime and returns TtsSynthesizeResult", async () => {
    const result = await synthesize(
      `http://127.0.0.1:${port}`,
      { text: "hello world" },
      "trace-123",
    );
    assert.equal(result.audio, "AAAA");
    assert.equal(result.format, "wav");
    assert.equal(result.durationMs, 1200);
    assert.equal(result.sampleRate, 24000);
  });

  it("retries once on transient failure then succeeds", async () => {
    let callCount = 0;
    const flaky = createServer((req, res) => {
      const chunks: Buffer[] = [];
      req.on("data", (c) => chunks.push(c));
      req.on("end", () => {
        callCount++;
        if (callCount === 1) {
          res.writeHead(503);
          res.end("busy");
          return;
        }
        res.writeHead(200, { "content-type": "application/json" });
        res.end(JSON.stringify({
          audio_base64: "BBBB",
          format: "wav",
          duration_ms: 500,
          sample_rate: 24000,
        }));
      });
    });
    await new Promise<void>((resolve) => {
      flaky.listen(0, "127.0.0.1", () => resolve());
    });
    const flakyAddr = flaky.address();
    const flakyPort = typeof flakyAddr === "object" && flakyAddr ? flakyAddr.port : 0;

    const result = await synthesize(
      `http://127.0.0.1:${flakyPort}`,
      { text: "retry me" },
      "trace-retry",
    );
    assert.equal(result.audio, "BBBB");
    assert.equal(callCount, 2);

    await new Promise<void>((resolve) => flaky.close(() => resolve()));
  });

  it("throws after retry exhaustion", async () => {
    const dead = createServer((_req, res) => {
      res.writeHead(500);
      res.end("dead");
    });
    await new Promise<void>((resolve) => {
      dead.listen(0, "127.0.0.1", () => resolve());
    });
    const deadAddr = dead.address();
    const deadPort = typeof deadAddr === "object" && deadAddr ? deadAddr.port : 0;

    await assert.rejects(
      () => synthesize(`http://127.0.0.1:${deadPort}`, { text: "fail" }, "trace-fail"),
      { message: /TTS runtime error/ },
    );

    await new Promise<void>((resolve) => dead.close(() => resolve()));
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/bridge-tts && pnpm build && pnpm test`
Expected: FAIL — `synthesize` not exported.

**Step 3: Write minimal implementation**

`packages/bridge-tts/src/synthesizer.ts`:
```typescript
import type { TtsSynthesizeRequest, TtsSynthesizeResult } from "./types.js";

interface RuntimeResponse {
  audio_base64: string;
  format: "wav" | "mp3";
  duration_ms: number;
  sample_rate: number;
}

/**
 * Call the local TTS runtime HTTP endpoint.
 * Retries once on transient failure (5xx / network error).
 */
export async function synthesize(
  runtimeUrl: string,
  request: TtsSynthesizeRequest,
  traceId: string,
): Promise<TtsSynthesizeResult> {
  const url = `${runtimeUrl}/synthesize`;
  const body = JSON.stringify({
    text: request.text,
    voice: request.voice,
    speed: request.speed,
  });
  const headers: Record<string, string> = {
    "content-type": "application/json",
    "x-trace-id": traceId,
  };

  let lastError: Error | null = null;
  for (let attempt = 0; attempt < 2; attempt++) {
    try {
      const res = await fetch(url, { method: "POST", headers, body });
      if (!res.ok) {
        lastError = new Error(`TTS runtime error: ${res.status} ${res.statusText}`);
        continue; // retry once
      }
      const data = (await res.json()) as RuntimeResponse;
      return {
        audio: data.audio_base64,
        format: data.format,
        durationMs: data.duration_ms,
        sampleRate: data.sample_rate,
      };
    } catch (err) {
      lastError = err instanceof Error ? err : new Error(String(err));
      // retry once on network error
    }
  }
  throw lastError ?? new Error("TTS runtime error: unknown");
}
```

Update `packages/bridge-tts/src/index.ts`:
```typescript
export type {
  TtsSynthesizeRequest,
  TtsSynthesizeResult,
  TtsBridgeConfig,
} from "./types.js";
export { synthesize } from "./synthesizer.js";
```

**Step 4: Run test to verify it passes**

Run: `cd packages/bridge-tts && pnpm build && pnpm test`
Expected: 3/3 PASS.

**Step 5: Commit**

```bash
git add packages/bridge-tts/src/synthesizer.ts packages/bridge-tts/src/synthesizer.test.ts packages/bridge-tts/src/index.ts
git commit -m "feat(F011): synthesizer client with retry + traceId propagation"
```

---

## Task 3: TTS INVOKE Handler

**Covers:** AC-2 (audio result), AC-3 (scope guard), AC-4 (traceId logging)

**Files:**
- Create: `packages/bridge-tts/src/handler.ts`
- Create: `packages/bridge-tts/src/handler.test.ts`
- Modify: `packages/bridge-tts/src/index.ts`

**Step 1: Write the failing test**

`packages/bridge-tts/src/handler.test.ts`:
```typescript
import { describe, it } from "node:test";
import assert from "node:assert/strict";
import { createTtsHandler } from "./handler.js";
import type { InboundInvocation } from "@agent-mesh/node";

function makeInvocation(overrides: Partial<InboundInvocation> = {}): InboundInvocation {
  return {
    invocationId: "inv-1",
    traceId: "trace-1",
    sourceNodeId: "caller",
    targetNodeId: "tts-bridge",
    invocationToken: "tok",
    payload: {
      task: JSON.stringify({ text: "hello" }),
      scope: ["tts"],
    },
    ...overrides,
  };
}

describe("createTtsHandler()", () => {
  // Stub synthesizer: returns fixed result
  const stubSynthesize = async () => ({
    audio: "AAAA",
    format: "wav" as const,
    durationMs: 1000,
    sampleRate: 24000,
  });

  it("returns success with audio result", async () => {
    const handler = createTtsHandler("http://stub", stubSynthesize);
    const result = await handler(makeInvocation());
    assert.equal(result.status, "success");
    const payload = JSON.parse(result.payload);
    assert.equal(payload.audio, "AAAA");
    assert.equal(payload.format, "wav");
  });

  it("rejects when scope lacks 'tts'", async () => {
    const handler = createTtsHandler("http://stub", stubSynthesize);
    const result = await handler(makeInvocation({
      payload: { task: JSON.stringify({ text: "hi" }), scope: ["read"] },
    }));
    assert.equal(result.status, "error");
    const payload = JSON.parse(result.payload);
    assert.equal(payload.code, "SCOPE_INSUFFICIENT");
  });

  it("rejects when task JSON is invalid", async () => {
    const handler = createTtsHandler("http://stub", stubSynthesize);
    const result = await handler(makeInvocation({
      payload: { task: "not json", scope: ["tts"] },
    }));
    assert.equal(result.status, "error");
    const payload = JSON.parse(result.payload);
    assert.equal(payload.code, "INVALID_PAYLOAD");
  });

  it("rejects when text is empty", async () => {
    const handler = createTtsHandler("http://stub", stubSynthesize);
    const result = await handler(makeInvocation({
      payload: { task: JSON.stringify({ text: "" }), scope: ["tts"] },
    }));
    assert.equal(result.status, "error");
    const payload = JSON.parse(result.payload);
    assert.equal(payload.code, "INVALID_PAYLOAD");
  });

  it("returns error when synthesizer throws", async () => {
    const failSynthesize = async () => { throw new Error("GPU OOM"); };
    const handler = createTtsHandler("http://stub", failSynthesize);
    const result = await handler(makeInvocation());
    assert.equal(result.status, "error");
    assert.ok(result.payload.includes("GPU OOM"));
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/bridge-tts && pnpm build && pnpm test`
Expected: FAIL — `createTtsHandler` not exported.

**Step 3: Write minimal implementation**

`packages/bridge-tts/src/handler.ts`:
```typescript
import type { InboundInvocation, InvocationResponse } from "@agent-mesh/node";
import type { TtsSynthesizeRequest, TtsSynthesizeResult } from "./types.js";

type SynthesizeFn = (
  runtimeUrl: string,
  request: TtsSynthesizeRequest,
  traceId: string,
) => Promise<TtsSynthesizeResult>;

/**
 * Create an INVOKE handler for tts.synthesize.
 * Validates scope + payload, calls synthesizer, returns structured result.
 */
export function createTtsHandler(
  runtimeUrl: string,
  synthesizeFn: SynthesizeFn,
) {
  return async (inv: InboundInvocation): Promise<InvocationResponse> => {
    const start = Date.now();

    // Defense-in-depth: scope guard (Hub already enforces, but belt-and-suspenders)
    if (!inv.payload.scope.includes("tts")) {
      return {
        status: "error",
        payload: JSON.stringify({
          code: "SCOPE_INSUFFICIENT",
          message: "tts.synthesize requires scope 'tts'",
          traceId: inv.traceId,
        }),
        durationMs: Date.now() - start,
      };
    }

    // Parse request
    let request: TtsSynthesizeRequest;
    try {
      request = JSON.parse(inv.payload.task) as TtsSynthesizeRequest;
      if (!request.text || typeof request.text !== "string") {
        throw new Error("text is required");
      }
    } catch {
      return {
        status: "error",
        payload: JSON.stringify({
          code: "INVALID_PAYLOAD",
          message: "payload.task must be JSON with non-empty 'text' field",
          traceId: inv.traceId,
        }),
        durationMs: Date.now() - start,
      };
    }

    // Synthesize
    try {
      console.log(
        `[tts-bridge] synthesize traceId=${inv.traceId} from=${inv.sourceNodeId} text=${request.text.length}chars`,
      );
      const result = await synthesizeFn(runtimeUrl, request, inv.traceId);
      console.log(
        `[tts-bridge] done traceId=${inv.traceId} format=${result.format} duration=${result.durationMs}ms`,
      );
      return {
        status: "success",
        payload: JSON.stringify(result),
        durationMs: Date.now() - start,
      };
    } catch (err) {
      const message = err instanceof Error ? err.message : "synthesize failed";
      console.error(`[tts-bridge] error traceId=${inv.traceId}: ${message}`);
      return {
        status: "error",
        payload: JSON.stringify({
          code: "SYNTHESIZE_FAILED",
          message,
          traceId: inv.traceId,
        }),
        durationMs: Date.now() - start,
      };
    }
  };
}
```

Update `packages/bridge-tts/src/index.ts`:
```typescript
export type {
  TtsSynthesizeRequest,
  TtsSynthesizeResult,
  TtsBridgeConfig,
} from "./types.js";
export { synthesize } from "./synthesizer.js";
export { createTtsHandler } from "./handler.js";
```

**Step 4: Run test to verify it passes**

Run: `cd packages/bridge-tts && pnpm build && pnpm test`
Expected: All tests PASS.

**Step 5: Commit**

```bash
git add packages/bridge-tts/src/handler.ts packages/bridge-tts/src/handler.test.ts packages/bridge-tts/src/index.ts
git commit -m "feat(F011): TTS handler with scope guard + payload validation"
```

---

## Task 4: Bridge Node Main Entry (lifecycle)

**Covers:** AC-1 (HELLO + heartbeat + CAPS visible), AC-4 (traceId end-to-end)

**Files:**
- Create: `packages/bridge-tts/src/bridge.ts`
- Modify: `packages/bridge-tts/src/index.ts`

**Step 1: Write bridge main — following reviewer-node.ts pattern**

`packages/bridge-tts/src/bridge.ts`:
```typescript
/**
 * TTS Bridge Node — mesh node that wraps a local TTS runtime.
 *
 * Usage:
 *   TTS_RUNTIME_URL=http://127.0.0.1:5001 npx tsx packages/bridge-tts/src/bridge.ts
 *
 * Env vars:
 *   MESH_HUB_URL        — Hub URL (default: http://127.0.0.1:3010)
 *   TTS_NODE_ID          — Node ID (default: tts-bridge)
 *   TTS_IDENTITY_DIR     — Identity storage dir (default: .tts-identity)
 *   TTS_PORT             — Callback server port (default: 3008)
 *   TTS_RUNTIME_URL      — Local TTS runtime URL (required)
 *   TTS_HEARTBEAT_MS     — Heartbeat interval ms (default: 10000)
 */

import {
  MeshClient,
  MeshServer,
  generateNodeIdentity,
  getPublicKeyBase64url,
  loadIdentity,
  saveIdentity,
} from "@agent-mesh/node";
import { createTtsHandler } from "./handler.js";
import { synthesize } from "./synthesizer.js";

const HUB_URL = process.env.MESH_HUB_URL ?? "http://127.0.0.1:3010";
const NODE_ID = process.env.TTS_NODE_ID ?? "tts-bridge";
const IDENTITY_DIR = process.env.TTS_IDENTITY_DIR ?? ".tts-identity";
const PORT = Number(process.env.TTS_PORT ?? "3008");
const RUNTIME_URL = process.env.TTS_RUNTIME_URL;
const HEARTBEAT_MS = Number(process.env.TTS_HEARTBEAT_MS ?? "10000");

async function main() {
  if (!RUNTIME_URL) {
    console.error("[tts-bridge] FATAL: TTS_RUNTIME_URL is required");
    process.exit(1);
  }

  // Identity
  let identity;
  try {
    identity = await loadIdentity(IDENTITY_DIR);
    if (identity.nodeId !== NODE_ID) {
      identity = await generateNodeIdentity(NODE_ID);
      await saveIdentity(identity, IDENTITY_DIR);
    }
  } catch {
    identity = await generateNodeIdentity(NODE_ID);
    await saveIdentity(identity, IDENTITY_DIR);
  }

  const pubKey = getPublicKeyBase64url(identity);
  console.log(`[tts-bridge] nodeId:      ${NODE_ID}`);
  console.log(`[tts-bridge] publicKey:   ${pubKey}`);
  console.log(`[tts-bridge] hubUrl:      ${HUB_URL}`);
  console.log(`[tts-bridge] runtimeUrl:  ${RUNTIME_URL}`);

  // Callback server with TTS handler
  const handler = createTtsHandler(RUNTIME_URL, synthesize);
  const server = new MeshServer(handler, { port: PORT, host: "127.0.0.1" });
  const callbackUrl = await server.start();
  console.log(`[tts-bridge] callback:    ${callbackUrl}`);

  // HELLO handshake
  const client = new MeshClient({ hubUrl: HUB_URL, identity });
  console.log("[tts-bridge] Sending HELLO...");
  await client.hello();
  console.log("[tts-bridge] HELLO OK");

  // L2 verification key
  const pubkey = await fetch(`${HUB_URL}/v1/pubkey`).then(
    (r) => r.json() as Promise<JsonWebKey>,
  );
  await server.setVerificationKey(pubkey, NODE_ID);

  // Heartbeat
  client.startHeartbeat(HEARTBEAT_MS);
  console.log(`[tts-bridge] heartbeat started (every ${HEARTBEAT_MS}ms)`);

  console.log(`\n[tts-bridge] Ready — accepting tts.synthesize requests...`);

  // Graceful shutdown
  for (const sig of ["SIGINT", "SIGTERM"] as const) {
    process.on(sig, async () => {
      console.log(`\n[tts-bridge] ${sig} — shutting down...`);
      await client.close();
      await server.stop();
      process.exit(0);
    });
  }
}

main().catch((e) => {
  console.error("[tts-bridge] FATAL:", e);
  process.exit(1);
});
```

**Step 2: Build to verify**

Run: `cd packages/bridge-tts && pnpm build`
Expected: PASS — compiles without error.

**Step 3: Commit**

```bash
git add packages/bridge-tts/src/bridge.ts packages/bridge-tts/src/index.ts
git commit -m "feat(F011): TTS bridge node main entry — HELLO + heartbeat + handler lifecycle"
```

---

## Task 5: Python Runtime Wrapper

**Covers:** AC-2 (actual audio generation), AC-4 (traceId in runtime logs)

**Files:**
- Create: `bridges/qwen3-tts/server.py`
- Create: `bridges/qwen3-tts/requirements.txt`

**Step 1: Create FastAPI server**

`bridges/qwen3-tts/server.py`:
```python
"""
Qwen3-TTS local runtime wrapper.

Exposes POST /synthesize for bridge-tts to call.
BYO Runtime: assumes Qwen3-TTS model is installed and importable.

Usage:
  pip install -r requirements.txt
  python server.py                     # default port 5001
  TTS_PORT=5002 python server.py       # custom port
"""

import base64
import io
import os
import time
import logging

from fastapi import FastAPI, Request
from pydantic import BaseModel

logging.basicConfig(level=logging.INFO, format="[qwen3-tts] %(message)s")
logger = logging.getLogger(__name__)

app = FastAPI(title="Qwen3-TTS Runtime", version="0.1.0")

# Lazy-load model to avoid import-time GPU allocation
_model = None
_processor = None


def _load_model():
    """Load Qwen3-TTS model. Called once on first request."""
    global _model, _processor
    if _model is not None:
        return
    logger.info("Loading Qwen3-TTS model...")
    try:
        from transformers import AutoModelForCausalLM, AutoProcessor
        import torch

        model_name = os.environ.get("TTS_MODEL_NAME", "Qwen/Qwen3-TTS-Base")
        device = os.environ.get("TTS_DEVICE", "cpu")

        _processor = AutoProcessor.from_pretrained(model_name)
        _model = AutoModelForCausalLM.from_pretrained(model_name).to(device)
        logger.info(f"Model loaded: {model_name} on {device}")
    except ImportError:
        logger.warning("transformers/torch not installed — using stub mode (silence)")
    except Exception as e:
        logger.error(f"Model load failed: {e} — falling back to stub mode")


class SynthesizeRequest(BaseModel):
    text: str
    voice: str | None = None
    speed: float | None = None


class SynthesizeResponse(BaseModel):
    audio_base64: str
    format: str = "wav"
    duration_ms: int
    sample_rate: int = 24000


@app.post("/synthesize")
async def synthesize(req: SynthesizeRequest, request: Request) -> SynthesizeResponse:
    trace_id = request.headers.get("x-trace-id", "unknown")
    logger.info(f"synthesize traceId={trace_id} text={len(req.text)}chars")
    start = time.time()

    _load_model()

    if _model is not None and _processor is not None:
        # Real model inference
        import torch
        import soundfile as sf

        inputs = _processor(text=req.text, return_tensors="pt")
        device = next(_model.parameters()).device
        inputs = {k: v.to(device) for k, v in inputs.items()}

        with torch.no_grad():
            output = _model.generate(**inputs, max_new_tokens=2048)

        audio_array = output[0].cpu().numpy()
        buf = io.BytesIO()
        sf.write(buf, audio_array, 24000, format="WAV")
        audio_bytes = buf.getvalue()
        duration_ms = int(len(audio_array) / 24000 * 1000)
    else:
        # Stub mode: generate 1s of silence (for testing without model)
        import struct

        sample_rate = 24000
        num_samples = sample_rate  # 1 second
        buf = io.BytesIO()
        # WAV header
        buf.write(b"RIFF")
        data_size = num_samples * 2  # 16-bit mono
        buf.write(struct.pack("<I", 36 + data_size))
        buf.write(b"WAVE")
        buf.write(b"fmt ")
        buf.write(struct.pack("<IHHIIHH", 16, 1, 1, sample_rate, sample_rate * 2, 2, 16))
        buf.write(b"data")
        buf.write(struct.pack("<I", data_size))
        buf.write(b"\x00" * data_size)
        audio_bytes = buf.getvalue()
        duration_ms = 1000

    elapsed = time.time() - start
    audio_b64 = base64.b64encode(audio_bytes).decode("ascii")
    logger.info(f"done traceId={trace_id} duration={duration_ms}ms elapsed={elapsed:.2f}s")

    return SynthesizeResponse(
        audio_base64=audio_b64,
        format="wav",
        duration_ms=duration_ms,
        sample_rate=24000,
    )


@app.get("/health")
async def health():
    return {"status": "ok", "model_loaded": _model is not None}


if __name__ == "__main__":
    import uvicorn

    port = int(os.environ.get("TTS_PORT", "5001"))
    logger.info(f"Starting on port {port}")
    uvicorn.run(app, host="127.0.0.1", port=port)
```

`bridges/qwen3-tts/requirements.txt`:
```
fastapi>=0.100.0
uvicorn>=0.23.0
pydantic>=2.0.0
# BYO Runtime: uncomment if model is installed locally
# torch
# transformers
# soundfile
```

**Step 2: Verify stub mode starts**

Run: `cd bridges/qwen3-tts && pip install fastapi uvicorn pydantic && python server.py &`
Run: `curl -s -X POST http://127.0.0.1:5001/synthesize -H 'content-type: application/json' -d '{"text":"hello"}' | python -c "import sys,json; d=json.load(sys.stdin); print(f'format={d[\"format\"]} duration={d[\"duration_ms\"]}ms b64_len={len(d[\"audio_base64\"])}')""`
Expected: `format=wav duration=1000ms b64_len=...` (stub silence output)

**Step 3: Commit**

```bash
git add bridges/qwen3-tts/
git commit -m "feat(F011): qwen3-tts Python runtime wrapper with stub fallback"
```

---

## Task 6: Hub Config Entry + Example Caller

**Covers:** AC-1 (CAPS visible), AC-2 (end-to-end invoke), AC-5 (minimal repro)

**Files:**
- Create: `examples/tts-bridge/config-tts-bridge.json` — Hub config snippet
- Create: `examples/tts-bridge/invoke-tts.ts` — Minimal caller
- Create: `examples/tts-bridge/start.sh` — One-command startup
- Create: `examples/tts-bridge/README.md` — AC-5 docs

**Step 1: Hub config snippet**

`examples/tts-bridge/config-tts-bridge.json`:
```json
{
  "$comment": "Add this entry to your Hub's allowedNodes array. Replace publicKey after first identity generation.",
  "nodeId": "tts-bridge",
  "publicKey": "REPLACE_WITH_GENERATED_PUBLIC_KEY",
  "maxScopes": ["read", "tts"],
  "capabilities": [
    {
      "catId": "tts-bridge",
      "skills": ["tts.synthesize"]
    }
  ],
  "callbackUrl": "http://127.0.0.1:3008"
}
```

**Step 2: Minimal caller script**

`examples/tts-bridge/invoke-tts.ts`:
```typescript
/**
 * Minimal TTS invoke demo — calls tts-bridge through mesh.
 *
 * Usage:
 *   MESH_HUB_URL=http://127.0.0.1:3010 npx tsx examples/tts-bridge/invoke-tts.ts "Hello world"
 *
 * Prerequisites:
 *   1. Hub running with tts-bridge in allowedNodes
 *   2. TTS bridge running (packages/bridge-tts/src/bridge.ts)
 *   3. TTS runtime running (bridges/qwen3-tts/server.py)
 *   4. This caller node registered in Hub config
 */

import {
  MeshClient,
  generateNodeIdentity,
  loadIdentity,
  saveIdentity,
} from "@agent-mesh/node";

const HUB_URL = process.env.MESH_HUB_URL ?? "http://127.0.0.1:3010";
const CALLER_ID = process.env.CALLER_NODE_ID ?? "tts-demo-caller";
const IDENTITY_DIR = ".tts-demo-identity";

const text = process.argv[2] ?? "Hello, this is a TTS test from agent-mesh.";

async function main() {
  // Identity
  let identity;
  try {
    identity = await loadIdentity(IDENTITY_DIR);
  } catch {
    identity = await generateNodeIdentity(CALLER_ID);
    await saveIdentity(identity, IDENTITY_DIR);
  }

  const client = new MeshClient({ hubUrl: HUB_URL, identity });

  // HELLO
  console.log(`[caller] HELLO to ${HUB_URL}...`);
  await client.hello();
  console.log("[caller] HELLO OK");

  // CAPS — verify tts.synthesize is visible
  const caps = await client.caps();
  const ttsCap = caps.capabilities.find((c) =>
    c.skills.includes("tts.synthesize"),
  );
  if (!ttsCap) {
    console.error("[caller] tts.synthesize not found in CAPS! Available:", caps.capabilities);
    process.exit(1);
  }
  console.log(`[caller] Found tts.synthesize on node=${ttsCap.nodeId} status=${ttsCap.status}`);

  // INVOKE
  console.log(`[caller] Invoking tts.synthesize: "${text.slice(0, 50)}..."`);
  const result = await client.invoke("tts-bridge", JSON.stringify({ text }), {
    scope: ["tts"],
  });
  console.log(`[caller] Result: status=${result.status} traceId=${result.traceId}`);

  if (result.status === "success") {
    const payload = JSON.parse(result.payload);
    console.log(`[caller] Audio: format=${payload.format} duration=${payload.durationMs}ms sampleRate=${payload.sampleRate}`);
    console.log(`[caller] Base64 length: ${payload.audio.length} chars`);

    // Optionally save to file
    const fs = await import("node:fs");
    const outPath = `tts-output-${Date.now()}.wav`;
    fs.writeFileSync(outPath, Buffer.from(payload.audio, "base64"));
    console.log(`[caller] Saved to ${outPath}`);
  } else {
    console.error("[caller] Error:", result.payload);
  }

  await client.close();
}

main().catch((e) => {
  console.error("[caller] FATAL:", e);
  process.exit(1);
});
```

**Step 3: Startup script**

`examples/tts-bridge/start.sh`:
```bash
#!/usr/bin/env bash
# One-command startup for the TTS bridge demo.
# Prerequisites: Hub running at MESH_HUB_URL with tts-bridge in allowedNodes.
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
ROOT_DIR="$(cd "$SCRIPT_DIR/../.." && pwd)"

echo "=== Starting Qwen3-TTS Runtime (stub mode) ==="
cd "$ROOT_DIR/bridges/qwen3-tts"
python server.py &
RUNTIME_PID=$!
sleep 1

echo "=== Starting TTS Bridge Node ==="
cd "$ROOT_DIR"
TTS_RUNTIME_URL=http://127.0.0.1:5001 npx tsx packages/bridge-tts/src/bridge.ts &
BRIDGE_PID=$!
sleep 2

echo ""
echo "=== TTS Bridge Demo Ready ==="
echo "Runtime PID: $RUNTIME_PID"
echo "Bridge  PID: $BRIDGE_PID"
echo ""
echo "Test invoke:"
echo "  npx tsx examples/tts-bridge/invoke-tts.ts \"Hello world\""
echo ""
echo "Stop: kill $RUNTIME_PID $BRIDGE_PID"

# Wait for either to exit
wait -n $RUNTIME_PID $BRIDGE_PID 2>/dev/null || true
kill $RUNTIME_PID $BRIDGE_PID 2>/dev/null || true
```

**Step 4: README (AC-5)**

`examples/tts-bridge/README.md`:
```markdown
# TTS Bridge Demo

End-to-end TTS via agent-mesh: Caller -> Hub -> TTS Bridge -> Qwen3-TTS Runtime.

## Prerequisites

1. Hub running at `http://127.0.0.1:3010` (or set `MESH_HUB_URL`)
2. Hub config includes `tts-bridge` and `tts-demo-caller` in `allowedNodes` (see `config-tts-bridge.json`)
3. Python 3.10+ with `pip install fastapi uvicorn pydantic`

## Quick Start

```bash
# 1. Start runtime + bridge (one command)
bash examples/tts-bridge/start.sh

# 2. Invoke (one command)
npx tsx examples/tts-bridge/invoke-tts.ts "Hello from agent-mesh"
```

## Manual Start

```bash
# Terminal 1: TTS Runtime
cd bridges/qwen3-tts
python server.py

# Terminal 2: TTS Bridge
TTS_RUNTIME_URL=http://127.0.0.1:5001 npx tsx packages/bridge-tts/src/bridge.ts

# Terminal 3: Invoke
npx tsx examples/tts-bridge/invoke-tts.ts "Hello world"
```

## Verify AC

| AC | How to verify |
|----|---------------|
| AC-1 | Bridge logs show `HELLO OK` + `heartbeat started`; caller's CAPS shows `tts.synthesize` online |
| AC-2 | Caller logs show `Audio: format=wav duration=...` and `tts-output-*.wav` is playable |
| AC-3 | Invoke with `scope: ["read"]` instead of `["tts"]` → Hub returns `scope denied` |
| AC-4 | Same `traceId` appears in caller, bridge, and runtime logs |
| AC-5 | This README — two commands: `start.sh` + `invoke-tts.ts` |
```

**Step 5: Commit**

```bash
git add examples/tts-bridge/
git commit -m "feat(F011): example caller + config + startup script + AC-5 docs"
```

---

## Task 7: Integration Verification

**Covers:** AC-1 through AC-5 (full chain)

**Step 1: Build all packages**

Run: `pnpm install && pnpm build`
Expected: PASS — all packages compile.

**Step 2: Run unit tests**

Run: `cd packages/bridge-tts && pnpm test`
Expected: All synthesizer + handler tests PASS.

**Step 3: End-to-end smoke test (local Hub)**

Prerequisites: Hub running with tts-bridge + tts-demo-caller in config.

1. Start runtime: `cd bridges/qwen3-tts && python server.py`
2. Start bridge: `TTS_RUNTIME_URL=http://127.0.0.1:5001 npx tsx packages/bridge-tts/src/bridge.ts`
3. Verify AC-1: Bridge logs `HELLO OK` + `heartbeat started`
4. Invoke: `npx tsx examples/tts-bridge/invoke-tts.ts "Test"`
5. Verify AC-2: Caller gets wav file
6. Verify AC-3: Change scope to `["read"]` → Hub returns `scope denied`
7. Verify AC-4: Same traceId in all three process logs

**Step 4: Final commit**

```bash
git commit -m "chore(F011): integration verification complete — all 5 ACs met"
```

---

## File Inventory

| Path | Purpose | New/Modify |
|------|---------|------------|
| `packages/bridge-tts/package.json` | Package manifest | New |
| `packages/bridge-tts/tsconfig.json` | TypeScript config | New |
| `packages/bridge-tts/src/types.ts` | TTS contract types | New |
| `packages/bridge-tts/src/index.ts` | Barrel exports | New |
| `packages/bridge-tts/src/synthesizer.ts` | Runtime HTTP client | New |
| `packages/bridge-tts/src/synthesizer.test.ts` | Synthesizer tests | New |
| `packages/bridge-tts/src/handler.ts` | INVOKE handler | New |
| `packages/bridge-tts/src/handler.test.ts` | Handler tests | New |
| `packages/bridge-tts/src/bridge.ts` | Node main entry | New |
| `bridges/qwen3-tts/server.py` | Python runtime wrapper | New |
| `bridges/qwen3-tts/requirements.txt` | Python dependencies | New |
| `examples/tts-bridge/config-tts-bridge.json` | Hub config snippet | New |
| `examples/tts-bridge/invoke-tts.ts` | Caller demo | New |
| `examples/tts-bridge/start.sh` | One-command startup | New |
| `examples/tts-bridge/README.md` | AC-5 docs | New |
