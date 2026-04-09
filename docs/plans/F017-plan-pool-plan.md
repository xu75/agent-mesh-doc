# F017: Plan Pool Implementation Plan

**Feature:** F017 — `docs/features/F017-plan-pool.md`
**Goal:** 团队互助号池——多人共享 AI 订阅，按需调度，透支额度模型控制消耗，Provider-native HTTP API 暴露。
**Acceptance Criteria:**
- [ ] AC-1: CLI 封装型 plan-bridge 可通过 `claude -p --session-id --output-format stream-json --tools ""` 封装 non-API Plan 为纯对话 A2A 节点。
- [ ] AC-2: 每个调用方映射独立的 named session（UUID），上下文隔离。
- [ ] AC-3: 并发上限由 `maxConcurrency` 配置（默认 3），超限返回 `RATE_LIMITED`。
- [ ] AC-4: API 代理型 plan-bridge 通过 HTTP 转发实现，key 不暴露给调用方。
- [ ] AC-5: plan-bridge 注册时声明 `provider` 和 `providerModel`。
- [ ] AC-6: pool-node 主入口为 Provider-native HTTP API + 辅入口 A2A card。
- [ ] AC-7: 请求分发策略：session-sticky + round-robin fallback。
- [ ] AC-8: 所有 bridge 不可用时返回 `RATE_LIMITED`（HTTP 429）。
- [ ] AC-9: SQLite ledger 采用 lease-based 记账。
- [ ] AC-10: provider 验真——从响应提取实际 model，不一致标记 `verified: false`。
- [ ] AC-11: Hub `registrationMode: "open"` 后自动注册（TOFU）。
- [ ] AC-12: `whitelist` 模式行为不变。
- [ ] AC-13: 动态注册节点 maxScopes 不超上限。
- [ ] AC-14: `--tools ""` 生效——无法执行本地工具。
- [ ] AC-15: 跨调用方 session 隔离。
- [ ] AC-16: 超预算/超并发返回 429 + `RATE_LIMITED`。
- [ ] AC-17: CLI 进程 `spawn(shell: false)` + 最小化 env + 隔离 cwd。
- [ ] AC-18: provider 二进制 hash 校验启动时执行。
- [ ] AC-19: Session TTL 可配置（默认 30 分钟），过期自动回收。
- [ ] AC-20: L1.5 环境探针——BASE_URL 校验。
- [ ] AC-21: HELLO 携带 attestation 字段，MVP warn-only。
- [ ] AC-22: Circuit Breaker 三态熔断器。
- [ ] AC-23: 根据 provider_scope 自动暴露对应 HTTP 端点。
- [ ] AC-24: per-member API key 认证。
- [ ] AC-25: 流式输出——Anthropic SSE + OpenAI SSE 格式。
- [ ] AC-26: `GET /v1/models` 返回可用 model 列表。
- [ ] AC-27: Team 一等实体，provider_scope 绑定。
- [ ] AC-28: 一次性邀请码加入 + TOFU + per-member API key。
- [ ] AC-29: 透支额度模型（Mutual Credit）。
- [ ] AC-30: 成本计算优先级（provider > token定价 > fallback）。
- [ ] AC-31: 自路由禁止。
- [ ] AC-32: Owner-first（shareMode: off/idle-only/capped）。
- [ ] AC-33: 管理员操作 MVP 最小集。

**Architecture:** 两个新 package：`packages/plan-pool/`（pool-node + api-gateway + team/credit/ledger）和 `packages/plan-bridge-cli/`（CLI 封装型 bridge）。F013-lite 改动约 60 行在 `packages/mesh-hub/src/hub.ts`。SQLite 做持久化。Fastify 做 HTTP server。复用 `@agent-mesh/node` 的 MeshClient/MeshServer 协议层。
**Tech Stack:** TypeScript, SQLite (better-sqlite3), Fastify, node:test, pnpm workspace
**前端验证:** No

---

## Task 1: plan-pool Package Scaffold + SQLite Schema

**Covers:** Foundation for AC-9, AC-27, AC-28, AC-29

**Files:**
- Create: `packages/plan-pool/package.json`
- Create: `packages/plan-pool/tsconfig.json`
- Create: `packages/plan-pool/src/db.ts`
- Create: `packages/plan-pool/src/db.test.ts`

**Step 1: Create package scaffold**

`packages/plan-pool/package.json`:
```json
{
  "name": "@agent-mesh/plan-pool",
  "version": "0.0.1",
  "description": "Plan Pool — subscription sharing with team management and credit system",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "lint": "tsc --noEmit",
    "test": "node --test dist/**/*.test.js"
  },
  "dependencies": {
    "@agent-mesh/node": "workspace:*",
    "@agent-mesh/protocol": "workspace:*",
    "better-sqlite3": "^11.0.0",
    "fastify": "^5.2.1"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^7.6.0",
    "@types/node": "^22.13.10"
  }
}
```

`packages/plan-pool/tsconfig.json`:
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

**Step 2: Write the failing test for DB schema initialization**

`packages/plan-pool/src/db.test.ts`:
```typescript
import { describe, it, beforeEach, afterEach } from 'node:test'
import assert from 'node:assert/strict'
import { createDatabase, type PoolDatabase } from './db.js'

describe('PoolDatabase', () => {
  let db: PoolDatabase

  beforeEach(() => {
    db = createDatabase(':memory:')
  })

  afterEach(() => {
    db.close()
  })

  it('creates all tables', () => {
    const tables = db.db
      .prepare("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")
      .all() as { name: string }[]
    const names = tables.map(t => t.name)
    assert.ok(names.includes('teams'))
    assert.ok(names.includes('members'))
    assert.ok(names.includes('api_keys'))
    assert.ok(names.includes('invites'))
    assert.ok(names.includes('bridges'))
    assert.ok(names.includes('invocations'))
  })

  it('creates member_balances view', () => {
    const views = db.db
      .prepare("SELECT name FROM sqlite_master WHERE type='view'")
      .all() as { name: string }[]
    assert.ok(views.some(v => v.name === 'member_balances'))
  })
})
```

**Step 3: Run test to verify it fails**

```bash
cd packages/plan-pool && pnpm install && pnpm build && pnpm test
```
Expected: FAIL — `createDatabase` not defined.

**Step 4: Write minimal implementation**

`packages/plan-pool/src/db.ts`:
```typescript
import Database from 'better-sqlite3'

export interface PoolDatabase {
  db: Database.Database
  close(): void
}

export function createDatabase(path: string): PoolDatabase {
  const db = new Database(path)
  db.pragma('journal_mode = WAL')
  db.pragma('foreign_keys = ON')

  db.exec(`
    CREATE TABLE IF NOT EXISTS teams (
      id                       TEXT PRIMARY KEY,
      name                     TEXT NOT NULL,
      provider_scope           TEXT NOT NULL,
      owner_member_id          TEXT NOT NULL,
      default_borrow_limit_usd REAL NOT NULL DEFAULT 30.0,
      created_at               INTEGER NOT NULL
    );

    CREATE TABLE IF NOT EXISTS members (
      id                        TEXT PRIMARY KEY,
      team_id                   TEXT NOT NULL REFERENCES teams(id),
      display_name              TEXT NOT NULL,
      public_key                TEXT NOT NULL,
      role                      TEXT NOT NULL DEFAULT 'member',
      status                    TEXT NOT NULL DEFAULT 'active',
      borrow_limit_override_usd REAL,
      joined_at                 INTEGER NOT NULL,
      last_active_at            INTEGER NOT NULL
    );

    CREATE TABLE IF NOT EXISTS api_keys (
      key_hash    TEXT PRIMARY KEY,
      team_id     TEXT NOT NULL REFERENCES teams(id),
      member_id   TEXT NOT NULL REFERENCES members(id),
      status      TEXT NOT NULL DEFAULT 'active',
      created_at  INTEGER NOT NULL
    );

    CREATE TABLE IF NOT EXISTS invites (
      code_hash   TEXT PRIMARY KEY,
      team_id     TEXT NOT NULL REFERENCES teams(id),
      expires_at  INTEGER NOT NULL,
      max_uses    INTEGER NOT NULL DEFAULT 1,
      used_count  INTEGER NOT NULL DEFAULT 0,
      created_by  TEXT NOT NULL REFERENCES members(id),
      created_at  INTEGER NOT NULL
    );

    CREATE TABLE IF NOT EXISTS bridges (
      id                       TEXT PRIMARY KEY,
      team_id                  TEXT NOT NULL REFERENCES teams(id),
      owner_member_id          TEXT NOT NULL REFERENCES members(id),
      share_mode               TEXT NOT NULL DEFAULT 'idle-only',
      external_max_concurrency INTEGER NOT NULL DEFAULT 1,
      last_owner_active_at     INTEGER,
      attestation_hash         TEXT,
      attestation_version      TEXT,
      created_at               INTEGER NOT NULL
    );

    CREATE TABLE IF NOT EXISTS invocations (
      id              INTEGER PRIMARY KEY AUTOINCREMENT,
      idempotency_key TEXT NOT NULL UNIQUE,
      team_id         TEXT NOT NULL,
      source_member_id TEXT NOT NULL,
      bridge_id       TEXT NOT NULL,
      bridge_owner_id TEXT NOT NULL,
      provider        TEXT NOT NULL,
      observed_model  TEXT,
      verified        BOOLEAN DEFAULT 1,
      input_tokens    INTEGER DEFAULT 0,
      output_tokens   INTEGER DEFAULT 0,
      normalized_cost REAL DEFAULT 0,
      cost_source     TEXT DEFAULT 'fallback',
      lease_status    TEXT NOT NULL DEFAULT 'pending',
      lease_expires   INTEGER,
      status          TEXT NOT NULL,
      created_at      INTEGER NOT NULL
    );

    CREATE INDEX IF NOT EXISTS idx_team ON invocations(team_id, created_at);
    CREATE INDEX IF NOT EXISTS idx_source ON invocations(source_member_id, created_at);
    CREATE INDEX IF NOT EXISTS idx_bridge_owner ON invocations(bridge_owner_id, created_at);
    CREATE INDEX IF NOT EXISTS idx_lease ON invocations(lease_status) WHERE lease_status = 'pending';

    -- 余额视图：settled 正常计入 + expired 保守占用（防超时刷爆）
    CREATE VIEW IF NOT EXISTS member_balances AS
    SELECT
      m.id AS member_id,
      m.team_id,
      m.display_name,
      COALESCE(c.contributed, 0) AS contributed_usd,
      COALESCE(d.consumed, 0) AS consumed_usd,
      COALESCE(c.contributed, 0) - COALESCE(d.consumed, 0) AS balance_usd,
      COALESCE(m.borrow_limit_override_usd, t.default_borrow_limit_usd) AS borrow_limit_usd,
      COALESCE(c.contributed, 0) - COALESCE(d.consumed, 0)
        + COALESCE(m.borrow_limit_override_usd, t.default_borrow_limit_usd) AS available_usd
    FROM members m
    JOIN teams t ON m.team_id = t.id
    LEFT JOIN (
      SELECT bridge_owner_id, SUM(normalized_cost) AS contributed
      FROM invocations WHERE status = 'success' AND lease_status = 'settled'
      GROUP BY bridge_owner_id
    ) c ON c.bridge_owner_id = m.id
    LEFT JOIN (
      -- consumed = settled 实际消耗 + expired 保守占用
      SELECT source_member_id, SUM(normalized_cost) AS consumed
      FROM invocations
      WHERE (status = 'success' AND lease_status = 'settled')
         OR (lease_status = 'expired')
      GROUP BY source_member_id
    ) d ON d.source_member_id = m.id;
  `)

  return {
    db,
    close() { db.close() },
  }
}
```

**Step 5: Run test to verify it passes**

```bash
pnpm build && pnpm test
```
Expected: PASS

**Step 6: Commit**

```bash
git add packages/plan-pool/
git commit -m "feat(F017): plan-pool package scaffold + SQLite schema"
```

---

## Task 2: Circuit Breaker

**Covers:** AC-22

**Files:**
- Create: `packages/plan-pool/src/circuit-breaker.ts`
- Create: `packages/plan-pool/src/circuit-breaker.test.ts`

**Step 1: Write the failing test**

```typescript
import { describe, it } from 'node:test'
import assert from 'node:assert/strict'
import { CircuitBreaker } from './circuit-breaker.js'

describe('CircuitBreaker', () => {
  it('starts closed', () => {
    const cb = new CircuitBreaker()
    assert.equal(cb.state, 'closed')
  })

  it('classifies 429 as quota', () => {
    const cb = new CircuitBreaker()
    assert.equal(cb.classify(429), 'quota')
  })

  it('classifies 500 as transient', () => {
    const cb = new CircuitBreaker()
    assert.equal(cb.classify(500), 'transient')
  })

  it('classifies 401 as misconfigured', () => {
    const cb = new CircuitBreaker()
    assert.equal(cb.classify(401), 'misconfigured')
  })

  it('opens after transient threshold (4 failures)', () => {
    const cb = new CircuitBreaker()
    for (let i = 0; i < 4; i++) cb.recordFailure('transient')
    assert.equal(cb.state, 'open')
  })

  it('opens immediately on misconfigured', () => {
    const cb = new CircuitBreaker()
    cb.recordFailure('misconfigured')
    assert.equal(cb.state, 'open')
  })

  it('opens on quota', () => {
    const cb = new CircuitBreaker()
    cb.recordFailure('quota')
    assert.equal(cb.state, 'open')
  })

  it('transitions to half-open after probe interval', () => {
    const cb = new CircuitBreaker({ probeIntervalMs: 10 })
    cb.recordFailure('transient')
    cb.recordFailure('transient')
    cb.recordFailure('transient')
    cb.recordFailure('transient')
    assert.equal(cb.state, 'open')
    // Simulate time passing
    cb._setLastFailureForTest(Date.now() - 20)
    assert.equal(cb.state, 'half-open')
  })

  it('resets to closed on success', () => {
    const cb = new CircuitBreaker({ probeIntervalMs: 10 })
    cb.recordFailure('transient')
    cb.recordFailure('transient')
    cb.recordFailure('transient')
    cb.recordFailure('transient')
    cb._setLastFailureForTest(Date.now() - 20)
    assert.equal(cb.state, 'half-open')
    cb.reset()
    assert.equal(cb.state, 'closed')
    assert.equal(cb.failureCount, 0)
  })
})
```

**Step 2: Run test — expect FAIL**

**Step 3: Implement**

```typescript
export type BreakerState = 'closed' | 'open' | 'half-open'
export type ErrorCategory = 'quota' | 'transient' | 'misconfigured'

export interface CircuitBreakerOptions {
  transientThreshold?: number  // default 4
  probeIntervalMs?: number     // default 60_000
}

export class CircuitBreaker {
  private _state: BreakerState = 'closed'
  private _failureCount = 0
  private _lastFailure = 0
  private _lastCategory: ErrorCategory | null = null
  private readonly transientThreshold: number
  private readonly probeIntervalMs: number

  constructor(opts?: CircuitBreakerOptions) {
    this.transientThreshold = opts?.transientThreshold ?? 4
    this.probeIntervalMs = opts?.probeIntervalMs ?? 60_000
  }

  get state(): BreakerState {
    if (this._state === 'open' && Date.now() - this._lastFailure >= this.probeIntervalMs) {
      return 'half-open'
    }
    return this._state
  }

  get failureCount(): number { return this._failureCount }

  classify(httpStatus: number): ErrorCategory {
    if (httpStatus === 429) return 'quota'
    if (httpStatus === 401 || httpStatus === 403) return 'misconfigured'
    return 'transient'
  }

  recordFailure(category: ErrorCategory): void {
    this._lastFailure = Date.now()
    this._lastCategory = category

    if (category === 'misconfigured' || category === 'quota') {
      this._state = 'open'
      this._failureCount++
      return
    }
    // transient: count-based
    this._failureCount++
    if (this._failureCount >= this.transientThreshold) {
      this._state = 'open'
    }
  }

  reset(): void {
    this._state = 'closed'
    this._failureCount = 0
    this._lastCategory = null
  }

  /** Test helper only */
  _setLastFailureForTest(ts: number): void {
    this._lastFailure = ts
  }
}
```

**Step 4: Run test — expect PASS**

**Step 5: Commit**

```bash
git add packages/plan-pool/src/circuit-breaker*
git commit -m "feat(F017): circuit breaker with three-state model (AC-22)"
```

---

## Task 3: Normalized Cost Calculation

**Covers:** AC-30

**Files:**
- Create: `packages/plan-pool/src/credit.ts`
- Create: `packages/plan-pool/src/credit.test.ts`

**Step 1: Write the failing test**

```typescript
import { describe, it } from 'node:test'
import assert from 'node:assert/strict'
import { calculateNormalizedCost } from './credit.js'

describe('calculateNormalizedCost', () => {
  it('uses provider-reported costUsd first', () => {
    const cost = calculateNormalizedCost({
      costUsd: 0.05,
      model: 'claude-sonnet-4',
      inputTokens: 1000,
      outputTokens: 500,
    })
    assert.equal(cost.amount, 0.05)
    assert.equal(cost.source, 'provider')
  })

  it('falls back to token × official pricing', () => {
    const cost = calculateNormalizedCost({
      costUsd: null,
      model: 'claude-sonnet-4',
      inputTokens: 1_000_000,
      outputTokens: 100_000,
    })
    // 1M * 3/1M + 100K * 15/1M = 3 + 1.5 = 4.5
    assert.equal(cost.source, 'token_pricing')
    assert.ok(Math.abs(cost.amount - 4.5) < 0.001)
  })

  it('falls back to fixed weight when model unknown', () => {
    const cost = calculateNormalizedCost({
      costUsd: null,
      model: 'unknown-model',
      inputTokens: null,
      outputTokens: null,
    })
    assert.equal(cost.amount, 0.02)
    assert.equal(cost.source, 'fallback')
  })

  it('falls back to fixed weight when tokens missing', () => {
    const cost = calculateNormalizedCost({
      costUsd: null,
      model: 'claude-sonnet-4',
      inputTokens: null,
      outputTokens: null,
    })
    assert.equal(cost.amount, 0.02)
    assert.equal(cost.source, 'fallback')
  })
})
```

**Step 2: Run test — expect FAIL**

**Step 3: Implement**

```typescript
export const OFFICIAL_PRICING: Record<string, { input: number; output: number }> = {
  'claude-sonnet-4':   { input: 3.0 / 1e6, output: 15.0 / 1e6 },
  'claude-sonnet-4-6': { input: 3.0 / 1e6, output: 15.0 / 1e6 },
  'claude-opus-4':     { input: 15.0 / 1e6, output: 75.0 / 1e6 },
  'claude-opus-4-6':   { input: 15.0 / 1e6, output: 75.0 / 1e6 },
  'gpt-4o':            { input: 2.5 / 1e6, output: 10.0 / 1e6 },
  'gpt-4o-mini':       { input: 0.15 / 1e6, output: 0.6 / 1e6 },
}

export interface UsageInput {
  costUsd: number | null
  model: string | null
  inputTokens: number | null
  outputTokens: number | null
}

export interface NormalizedCost {
  amount: number
  source: 'provider' | 'token_pricing' | 'fallback'
}

export function calculateNormalizedCost(usage: UsageInput): NormalizedCost {
  if (usage.costUsd != null) {
    return { amount: usage.costUsd, source: 'provider' }
  }
  if (usage.model) {
    const pricing = OFFICIAL_PRICING[usage.model]
    if (pricing && usage.inputTokens != null && usage.outputTokens != null) {
      const amount = usage.inputTokens * pricing.input + usage.outputTokens * pricing.output
      return { amount, source: 'token_pricing' }
    }
  }
  return { amount: 0.02, source: 'fallback' }
}
```

**Step 4: Run test — expect PASS**

**Step 5: Commit**

```bash
git add packages/plan-pool/src/credit*
git commit -m "feat(F017): normalized cost calculation with three-tier priority (AC-30)"
```

---

## Task 4: Ledger — Lease-based Accounting

**Covers:** AC-9

**Files:**
- Create: `packages/plan-pool/src/ledger.ts`
- Create: `packages/plan-pool/src/ledger.test.ts`

**Step 1: Write the failing test**

```typescript
import { describe, it, beforeEach, afterEach } from 'node:test'
import assert from 'node:assert/strict'
import { createDatabase, type PoolDatabase } from './db.js'
import { Ledger } from './ledger.js'

describe('Ledger', () => {
  let poolDb: PoolDatabase
  let ledger: Ledger

  beforeEach(() => {
    poolDb = createDatabase(':memory:')
    ledger = new Ledger(poolDb.db)
  })

  afterEach(() => { poolDb.close() })

  it('acquireLease creates pending record', () => {
    const leaseId = ledger.acquireLease({
      idempotencyKey: 'req-1',
      teamId: 'team-1',
      sourceMemberId: 'alice',
      bridgeId: 'bridge-1',
      bridgeOwnerId: 'bob',
      provider: 'claude',
      leaseTTLMs: 30_000,
    })
    assert.ok(leaseId > 0)
    const row = poolDb.db.prepare('SELECT * FROM invocations WHERE id = ?').get(leaseId) as any
    assert.equal(row.lease_status, 'pending')
    assert.equal(row.source_member_id, 'alice')
  })

  it('acquireLease is idempotent', () => {
    const id1 = ledger.acquireLease({
      idempotencyKey: 'req-1', teamId: 'team-1', sourceMemberId: 'alice',
      bridgeId: 'bridge-1', bridgeOwnerId: 'bob', provider: 'claude', leaseTTLMs: 30_000,
    })
    const id2 = ledger.acquireLease({
      idempotencyKey: 'req-1', teamId: 'team-1', sourceMemberId: 'alice',
      bridgeId: 'bridge-1', bridgeOwnerId: 'bob', provider: 'claude', leaseTTLMs: 30_000,
    })
    assert.equal(id1, id2)
  })

  it('settleLease updates to settled with cost', () => {
    const id = ledger.acquireLease({
      idempotencyKey: 'req-2', teamId: 'team-1', sourceMemberId: 'alice',
      bridgeId: 'bridge-1', bridgeOwnerId: 'bob', provider: 'claude', leaseTTLMs: 30_000,
    })
    ledger.settleLease(id, {
      observedModel: 'claude-sonnet-4',
      inputTokens: 1000, outputTokens: 500,
      normalizedCost: 0.05, costSource: 'provider',
      status: 'success',
    })
    const row = poolDb.db.prepare('SELECT * FROM invocations WHERE id = ?').get(id) as any
    assert.equal(row.lease_status, 'settled')
    assert.equal(row.normalized_cost, 0.05)
    assert.equal(row.status, 'success')
  })

  it('rollbackLease updates to rolled_back', () => {
    const id = ledger.acquireLease({
      idempotencyKey: 'req-3', teamId: 'team-1', sourceMemberId: 'alice',
      bridgeId: 'bridge-1', bridgeOwnerId: 'bob', provider: 'claude', leaseTTLMs: 30_000,
    })
    ledger.rollbackLease(id, { status: 'error' })
    const row = poolDb.db.prepare('SELECT * FROM invocations WHERE id = ?').get(id) as any
    assert.equal(row.lease_status, 'rolled_back')
    assert.equal(row.status, 'error')
  })

  it('expireStaleLeases applies conservative settlement with fallback cost', () => {
    const id = ledger.acquireLease({
      idempotencyKey: 'req-4', teamId: 'team-1', sourceMemberId: 'alice',
      bridgeId: 'bridge-1', bridgeOwnerId: 'bob', provider: 'claude', leaseTTLMs: 1,
    })
    // Manually set lease_expires to past
    poolDb.db.prepare('UPDATE invocations SET lease_expires = ? WHERE id = ?')
      .run(Date.now() - 1000, id)
    const expired = ledger.expireStaleLeases()
    assert.equal(expired, 1)
    // Conservative: keeps fallback cost ($0.02) so quota is consumed
    const row = poolDb.db.prepare('SELECT * FROM invocations WHERE id = ?').get(id) as any
    assert.equal(row.lease_status, 'expired')
    assert.equal(row.status, 'conservative_settle')
    assert.equal(row.normalized_cost, 0.02) // fallback cost preserved
  })
})
```

**Step 2: Run test — expect FAIL**

**Step 3: Implement**

```typescript
import type Database from 'better-sqlite3'

export interface AcquireLeaseInput {
  idempotencyKey: string
  teamId: string
  sourceMemberId: string
  bridgeId: string
  bridgeOwnerId: string
  provider: string
  leaseTTLMs: number
}

export interface SettleLeaseInput {
  observedModel?: string | null
  inputTokens?: number
  outputTokens?: number
  normalizedCost?: number
  costSource?: string
  status: string
}

export class Ledger {
  private insertStmt: Database.Statement
  private findByKeyStmt: Database.Statement
  private settleStmt: Database.Statement
  private rollbackStmt: Database.Statement
  private expireStmt: Database.Statement

  constructor(private db: Database.Database) {
    this.insertStmt = db.prepare(`
      INSERT INTO invocations (idempotency_key, team_id, source_member_id, bridge_id,
        bridge_owner_id, provider, lease_status, lease_expires, status, created_at)
      VALUES (?, ?, ?, ?, ?, ?, 'pending', ?, 'pending', ?)
    `)
    this.findByKeyStmt = db.prepare(
      'SELECT id FROM invocations WHERE idempotency_key = ?'
    )
    this.settleStmt = db.prepare(`
      UPDATE invocations SET lease_status = 'settled',
        observed_model = ?, input_tokens = ?, output_tokens = ?,
        normalized_cost = ?, cost_source = ?, status = ?, verified = 1
      WHERE id = ? AND lease_status = 'pending'
    `)
    this.rollbackStmt = db.prepare(`
      UPDATE invocations SET lease_status = 'rolled_back', status = ?
      WHERE id = ? AND lease_status = 'pending'
    `)
    // Conservative settlement: expired leases keep fallback cost ($0.02)
    // so repeated timeouts still consume quota (prevents bridge abuse)
    this.expireStmt = db.prepare(`
      UPDATE invocations SET lease_status = 'expired',
        status = 'conservative_settle', normalized_cost = 0.02,
        cost_source = 'fallback'
      WHERE lease_status = 'pending' AND lease_expires < ?
    `)
  }

  acquireLease(input: AcquireLeaseInput): number {
    const existing = this.findByKeyStmt.get(input.idempotencyKey) as { id: number } | undefined
    if (existing) return existing.id

    const now = Date.now()
    const result = this.insertStmt.run(
      input.idempotencyKey, input.teamId, input.sourceMemberId,
      input.bridgeId, input.bridgeOwnerId, input.provider,
      now + input.leaseTTLMs, now,
    )
    return Number(result.lastInsertRowid)
  }

  settleLease(leaseId: number, input: SettleLeaseInput): void {
    this.settleStmt.run(
      input.observedModel ?? null, input.inputTokens ?? 0,
      input.outputTokens ?? 0, input.normalizedCost ?? 0,
      input.costSource ?? 'fallback', input.status, leaseId,
    )
  }

  rollbackLease(leaseId: number, input: { status: string }): void {
    this.rollbackStmt.run(input.status, leaseId)
  }

  expireStaleLeases(): number {
    const result = this.expireStmt.run(Date.now())
    return result.changes
  }
}
```

**Step 4: Run test — expect PASS**

**Step 5: Commit**

```bash
git add packages/plan-pool/src/ledger*
git commit -m "feat(F017): lease-based ledger with idempotent acquire/settle/rollback (AC-9)"
```

---

## Task 5: Team Management — Create + Invite + Join

**Covers:** AC-27, AC-28, AC-24

**Files:**
- Create: `packages/plan-pool/src/team.ts`
- Create: `packages/plan-pool/src/team.test.ts`

**Step 1: Write the failing test**

```typescript
import { describe, it, beforeEach, afterEach } from 'node:test'
import assert from 'node:assert/strict'
import { createDatabase, type PoolDatabase } from './db.js'
import { TeamManager } from './team.js'

describe('TeamManager', () => {
  let poolDb: PoolDatabase
  let tm: TeamManager

  beforeEach(() => {
    poolDb = createDatabase(':memory:')
    tm = new TeamManager(poolDb.db)
  })

  afterEach(() => { poolDb.close() })

  it('createTeam inserts team + owner member + api key', () => {
    const result = tm.createTeam({
      name: 'Lab-Claude',
      providerScope: 'claude',
      ownerDisplayName: 'Alice',
      ownerPublicKey: 'pk-alice',
    })
    assert.ok(result.teamId)
    assert.ok(result.memberId)
    assert.ok(result.apiKey)

    const team = poolDb.db.prepare('SELECT * FROM teams WHERE id = ?').get(result.teamId) as any
    assert.equal(team.name, 'Lab-Claude')
    assert.equal(team.provider_scope, 'claude')

    const member = poolDb.db.prepare('SELECT * FROM members WHERE id = ?').get(result.memberId) as any
    assert.equal(member.role, 'owner')
    assert.equal(member.display_name, 'Alice')
  })

  it('createInvite returns code + stores hash', () => {
    const { teamId, memberId } = tm.createTeam({
      name: 'Lab', providerScope: 'claude',
      ownerDisplayName: 'Alice', ownerPublicKey: 'pk-a',
    })
    const invite = tm.createInvite({ teamId, createdBy: memberId, maxUses: 3, ttlHours: 24 })
    assert.ok(invite.code)
    assert.ok(invite.code.length >= 16)
  })

  it('joinTeam with valid invite creates member + api key', () => {
    const { teamId, memberId } = tm.createTeam({
      name: 'Lab', providerScope: 'claude',
      ownerDisplayName: 'Alice', ownerPublicKey: 'pk-a',
    })
    const { code } = tm.createInvite({ teamId, createdBy: memberId, maxUses: 1, ttlHours: 24 })
    const result = tm.joinTeam({ code, displayName: 'Bob', publicKey: 'pk-bob' })
    assert.ok(result.memberId)
    assert.ok(result.apiKey)
    assert.equal(result.teamName, 'Lab')
  })

  it('joinTeam rejects expired invite', () => {
    const { teamId, memberId } = tm.createTeam({
      name: 'Lab', providerScope: 'claude',
      ownerDisplayName: 'Alice', ownerPublicKey: 'pk-a',
    })
    const { code } = tm.createInvite({ teamId, createdBy: memberId, maxUses: 1, ttlHours: 0 })
    // Manually expire it
    poolDb.db.prepare('UPDATE invites SET expires_at = ?').run(Date.now() - 1000)
    assert.throws(() => tm.joinTeam({ code, displayName: 'Bob', publicKey: 'pk-bob' }),
      { message: /expired|invalid/ })
  })

  it('joinTeam rejects used-up invite', () => {
    const { teamId, memberId } = tm.createTeam({
      name: 'Lab', providerScope: 'claude',
      ownerDisplayName: 'Alice', ownerPublicKey: 'pk-a',
    })
    const { code } = tm.createInvite({ teamId, createdBy: memberId, maxUses: 1, ttlHours: 24 })
    tm.joinTeam({ code, displayName: 'Bob', publicKey: 'pk-bob' })
    assert.throws(() => tm.joinTeam({ code, displayName: 'Carol', publicKey: 'pk-carol' }),
      { message: /expired|invalid|used/ })
  })

  it('authenticateApiKey returns team + member info', () => {
    const { teamId, memberId, apiKey } = tm.createTeam({
      name: 'Lab', providerScope: 'claude',
      ownerDisplayName: 'Alice', ownerPublicKey: 'pk-a',
    })
    const auth = tm.authenticateApiKey(apiKey)
    assert.ok(auth)
    assert.equal(auth!.teamId, teamId)
    assert.equal(auth!.memberId, memberId)
  })

  it('authenticateApiKey returns null for unknown key', () => {
    assert.equal(tm.authenticateApiKey('unknown'), null)
  })
})
```

**Step 2: Run test — expect FAIL**

**Step 3: Implement `team.ts`**

Key operations: `createTeam`, `createInvite`, `joinTeam`, `authenticateApiKey`, `registerBridge`, `suspendMember`, `setBorrowLimit`, `rotateApiKey`. API key = `pool_<random-hex>`, stored as SHA-256 hash.

`registerBridge({ bridgeId, teamId, ownerMemberId, provider, shareMode, externalMaxConcurrency, attestation })`:
- Validates `provider` matches `team.provider_scope` → reject on mismatch
- First call: INSERT into `bridges` + pin `attestation_hash` + `attestation_version` (TOFU)
- Subsequent: if attestation changed → set `share_mode='off'` (suspend sharing) + return `{ pinChanged: true }` → admin must re-approve
- If attestation unchanged → update `last_owner_active_at` only

**Step 4: Run test — expect PASS**

**Step 5: Commit**

```bash
git add packages/plan-pool/src/team*
git commit -m "feat(F017): team management — create, invite, join, API key auth (AC-27,28,24)"
```

---

## Task 6: Credit System — Balance Check + Quota Exhaustion

**Covers:** AC-29, AC-31

**Files:**
- Modify: `packages/plan-pool/src/credit.ts` (add `checkQuota`)
- Modify: `packages/plan-pool/src/credit.test.ts` (add quota tests)

**Step 1: Write the failing test**

```typescript
describe('checkQuota', () => {
  let poolDb: PoolDatabase
  let tm: TeamManager
  let ledger: Ledger

  beforeEach(() => {
    poolDb = createDatabase(':memory:')
    tm = new TeamManager(poolDb.db)
    ledger = new Ledger(poolDb.db)
  })

  afterEach(() => { poolDb.close() })

  it('new member can use up to borrow limit', () => {
    const { teamId, memberId } = tm.createTeam({
      name: 'Lab', providerScope: 'claude',
      ownerDisplayName: 'Alice', ownerPublicKey: 'pk-a',
    })
    const result = checkQuota(poolDb.db, memberId)
    assert.ok(result.allowed)
    assert.equal(result.availableUsd, 30.0) // default borrow limit
  })

  it('rejects when quota exhausted', () => {
    const { teamId, memberId } = tm.createTeam({
      name: 'Lab', providerScope: 'claude',
      ownerDisplayName: 'Alice', ownerPublicKey: 'pk-a',
    })
    // Simulate consumption past borrow limit
    for (let i = 0; i < 4; i++) {
      const id = ledger.acquireLease({
        idempotencyKey: `req-${i}`, teamId, sourceMemberId: memberId,
        bridgeId: 'b1', bridgeOwnerId: 'other', provider: 'claude', leaseTTLMs: 30_000,
      })
      ledger.settleLease(id, { normalizedCost: 1.0, costSource: 'fallback', status: 'success' })
    }
    // consumed = $4, balance = -$4, available = -$4 + $3 = -$1
    const result = checkQuota(poolDb.db, memberId)
    assert.ok(!result.allowed)
  })
})
```

**Step 2: Run test — expect FAIL**

**Step 3: Implement `checkQuota`**

```typescript
export interface QuotaResult {
  allowed: boolean
  availableUsd: number
  balanceUsd: number
  borrowLimitUsd: number
}

export function checkQuota(db: Database.Database, memberId: string): QuotaResult {
  const row = db.prepare(
    'SELECT available_usd, balance_usd, borrow_limit_usd FROM member_balances WHERE member_id = ?'
  ).get(memberId) as { available_usd: number; balance_usd: number; borrow_limit_usd: number } | undefined

  if (!row) return { allowed: false, availableUsd: 0, balanceUsd: 0, borrowLimitUsd: 0 }

  return {
    allowed: row.available_usd > 0,
    availableUsd: row.available_usd,
    balanceUsd: row.balance_usd,
    borrowLimitUsd: row.borrow_limit_usd,
  }
}
```

**Step 4: Run test — expect PASS**

**Step 5: Commit**

```bash
git add packages/plan-pool/src/credit*
git commit -m "feat(F017): mutual credit quota check with borrow limit (AC-29)"
```

---

## Task 7: Pool-node Router — Session-sticky + Self-route Deny + Owner-first

**Covers:** AC-7, AC-8, AC-31, AC-32

**Files:**
- Create: `packages/plan-pool/src/pool.ts`
- Create: `packages/plan-pool/src/pool.test.ts`

**Step 1: Write the failing test**

```typescript
import { describe, it, beforeEach } from 'node:test'
import assert from 'node:assert/strict'
import { PoolRouter, type BridgeEntry } from './pool.js'

function makeBridge(id: string, ownerId: string, overrides?: Partial<BridgeEntry>): BridgeEntry {
  return {
    id, teamId: 'team-1', ownerMemberId: ownerId, provider: 'claude',
    maxConcurrency: 3, activeSessions: 0, shareMode: 'idle-only',
    externalMaxConcurrency: 1, externalActiveSessions: 0,
    lastOwnerActiveAt: 0, ownerIdleTimeout: 30 * 60 * 1000,
    ...overrides,
  }
}

describe('PoolRouter', () => {
  // --- selectBridge request includes namespace (spec: callerId + provider + namespace) ---

  it('self-route deny: never routes caller to own bridge', () => {
    const router = new PoolRouter()
    router.addBridge(makeBridge('b-alice', 'alice'))
    const result = router.selectBridge({ callerId: 'alice', teamId: 'team-1', provider: 'claude', namespace: 'ns-1' })
    assert.equal(result, null) // only bridge is alice's own → no match
  })

  it('selects available bridge for different caller', () => {
    const router = new PoolRouter()
    router.addBridge(makeBridge('b-alice', 'alice'))
    const result = router.selectBridge({ callerId: 'bob', teamId: 'team-1', provider: 'claude', namespace: 'ns-1' })
    assert.ok(result)
    assert.equal(result!.id, 'b-alice')
  })

  it('session-sticky: same caller+provider+namespace → same bridge', () => {
    const router = new PoolRouter()
    router.addBridge(makeBridge('b-alice', 'alice'))
    router.addBridge(makeBridge('b-carol', 'carol'))
    const req = { callerId: 'bob', teamId: 'team-1', provider: 'claude', namespace: 'conv-1' }
    const first = router.selectBridge(req)
    assert.ok(first)
    router.updateBinding(req, first!)
    const second = router.selectBridge(req)
    assert.equal(second!.id, first!.id)
  })

  it('session-sticky: same caller, different namespace → independent binding', () => {
    const router = new PoolRouter()
    router.addBridge(makeBridge('b-alice', 'alice', { shareMode: 'capped' }))
    router.addBridge(makeBridge('b-carol', 'carol', { shareMode: 'capped' }))
    const req1 = { callerId: 'bob', teamId: 'team-1', provider: 'claude', namespace: 'conv-1' }
    const req2 = { callerId: 'bob', teamId: 'team-1', provider: 'claude', namespace: 'conv-2' }
    const first = router.selectBridge(req1)
    router.updateBinding(req1, first!)
    const second = router.selectBridge(req2)
    // Different namespace → may get different bridge (round-robin)
    assert.ok(first && second)
  })

  it('owner-first idle-only: rejects external when owner active', () => {
    const router = new PoolRouter()
    router.addBridge(makeBridge('b-alice', 'alice', {
      shareMode: 'idle-only', lastOwnerActiveAt: Date.now(),
    }))
    const result = router.selectBridge({ callerId: 'bob', teamId: 'team-1', provider: 'claude', namespace: 'ns-1' })
    assert.equal(result, null) // owner is active → not available for external
  })

  it('owner-first idle-only: allows external when owner idle', () => {
    const router = new PoolRouter()
    router.addBridge(makeBridge('b-alice', 'alice', {
      shareMode: 'idle-only', lastOwnerActiveAt: Date.now() - 31 * 60 * 1000,
    }))
    const result = router.selectBridge({ callerId: 'bob', teamId: 'team-1', provider: 'claude', namespace: 'ns-1' })
    assert.ok(result)
  })

  it('shareMode off: never available for external', () => {
    const router = new PoolRouter()
    router.addBridge(makeBridge('b-alice', 'alice', { shareMode: 'off' }))
    const result = router.selectBridge({ callerId: 'bob', teamId: 'team-1', provider: 'claude', namespace: 'ns-1' })
    assert.equal(result, null)
  })

  it('returns null when all bridges busy', () => {
    const router = new PoolRouter()
    router.addBridge(makeBridge('b-alice', 'alice', {
      shareMode: 'capped', externalActiveSessions: 1, externalMaxConcurrency: 1,
    }))
    const result = router.selectBridge({ callerId: 'bob', teamId: 'team-1', provider: 'claude', namespace: 'ns-1' })
    assert.equal(result, null)
  })

  it('round-robin across multiple bridges', () => {
    const router = new PoolRouter()
    router.addBridge(makeBridge('b-alice', 'alice', { shareMode: 'capped' }))
    router.addBridge(makeBridge('b-carol', 'carol', { shareMode: 'capped' }))
    const r1 = router.selectBridge({ callerId: 'bob', teamId: 'team-1', provider: 'claude', namespace: 'ns-1' })
    const r2 = router.selectBridge({ callerId: 'dave', teamId: 'team-1', provider: 'claude', namespace: 'ns-1' })
    // Should pick different bridges (round-robin)
    assert.ok(r1 && r2)
    assert.notEqual(r1.id, r2.id)
  })
})
```

**Step 2: Run test — expect FAIL**

**Step 3: Implement `pool.ts`**

Core logic: `selectBridge` checks self-route deny → session-sticky → `canAcceptExternal` (owner-first) → round-robin fallback. `updateBinding` / `cleanupStaleBindings` for session affinity.

**Sticky key = `callerId:provider:namespace`** (matches spec AC-7). `SelectBridgeRequest` includes `namespace: string`. HTTP callers without stable namespace should auto-generate UUID per request → no `--resume`, stateless mode. Only callers providing stable namespace get session continuity.

**Step 4: Run test — expect PASS**

**Step 5: Commit**

```bash
git add packages/plan-pool/src/pool*
git commit -m "feat(F017): pool router with session-sticky, self-route deny, owner-first (AC-7,8,31,32)"
```

---

## Task 8: CLI Wrapper — Spawn + stream-json Parse + Usage Extraction

**Covers:** AC-1, AC-14, AC-17

**Files:**
- Create: `packages/plan-bridge-cli/package.json`
- Create: `packages/plan-bridge-cli/tsconfig.json`
- Create: `packages/plan-bridge-cli/src/cli-wrapper.ts`
- Create: `packages/plan-bridge-cli/src/cli-wrapper.test.ts`

**Step 1: Create package scaffold**

```json
{
  "name": "@agent-mesh/plan-bridge-cli",
  "version": "0.0.1",
  "description": "CLI-based plan-bridge — wraps Claude CLI for chat-only mode",
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

**Step 2: Write the failing test**

Test `buildCliArgs` (pure function, no external dependency) and `parseStreamJsonLine` (parse one NDJSON line from Claude CLI output).

```typescript
import { describe, it } from 'node:test'
import assert from 'node:assert/strict'
import { buildCliArgs, parseStreamJsonLine, extractUsageFromLines } from './cli-wrapper.js'

describe('buildCliArgs', () => {
  it('includes mandatory safety flags', () => {
    const args = buildCliArgs({
      sessionId: '550e8400-e29b-41d4-a716-446655440000',
      prompt: 'hello',
      maxBudgetUsd: 0.5,
    })
    assert.ok(args.includes('--tools'))
    assert.ok(args.includes(''))  // empty string = disable all tools
    assert.ok(args.includes('--output-format'))
    assert.ok(args.includes('stream-json'))
    assert.ok(args.includes('--session-id'))
    assert.ok(args.includes('-p'))
    assert.ok(args.includes('--setting-sources'))
    assert.ok(args.includes('--strict-mcp-config'))
  })

  it('includes max-budget-usd', () => {
    const args = buildCliArgs({
      sessionId: 'abc', prompt: 'hello', maxBudgetUsd: 0.5,
    })
    assert.ok(args.includes('--max-budget-usd'))
    assert.ok(args.includes('0.5'))
  })
})

describe('parseStreamJsonLine', () => {
  it('parses assistant text message', () => {
    const line = JSON.stringify({ type: 'assistant', message: { content: [{ type: 'text', text: 'hello' }] } })
    const parsed = parseStreamJsonLine(line)
    assert.equal(parsed?.type, 'assistant')
  })

  it('parses result with usage', () => {
    const line = JSON.stringify({
      type: 'result',
      result: 'hello',
      cost_usd: 0.03,
      input_tokens: 100,
      output_tokens: 50,
      model: 'claude-sonnet-4',
    })
    const parsed = parseStreamJsonLine(line)
    assert.equal(parsed?.type, 'result')
    assert.equal(parsed?.costUsd, 0.03)
    assert.equal(parsed?.inputTokens, 100)
  })

  it('returns null for unparseable line', () => {
    assert.equal(parseStreamJsonLine('not json'), null)
  })
})
```

**Step 3: Run test — expect FAIL**

**Step 4: Implement**

`buildCliArgs` returns the safety-hardened arg array. `parseStreamJsonLine` parses one NDJSON line. `invokeCli` spawns the actual process (tested in integration).

**Step 5: Run test — expect PASS**

**Step 6: Commit**

```bash
git add packages/plan-bridge-cli/
git commit -m "feat(F017): CLI wrapper with safety flags + stream-json parser (AC-1,14,17)"
```

---

## Task 9: Session Manager — Per-caller + TTL + Resume

**Covers:** AC-2, AC-15, AC-19

**Files:**
- Create: `packages/plan-bridge-cli/src/session-manager.ts`
- Create: `packages/plan-bridge-cli/src/session-manager.test.ts`

**Step 1: Write the failing test**

```typescript
import { describe, it } from 'node:test'
import assert from 'node:assert/strict'
import { SessionManager } from './session-manager.js'

// SessionManager key = callerId:namespace (composite key)
// Same caller with different namespace → different CLI sessions → no context leak
describe('SessionManager', () => {
  it('creates new session for unknown caller+namespace', () => {
    const sm = new SessionManager({ ttlMinutes: 30 })
    const s = sm.getOrCreate('caller-1', 'conv-1')
    assert.ok(s.sessionId)
    assert.equal(s.isResume, false)
  })

  it('returns existing session for same caller+namespace (resume)', () => {
    const sm = new SessionManager({ ttlMinutes: 30 })
    const s1 = sm.getOrCreate('caller-1', 'conv-1')
    const s2 = sm.getOrCreate('caller-1', 'conv-1')
    assert.equal(s1.sessionId, s2.sessionId)
    assert.equal(s2.isResume, true)
  })

  it('same caller, different namespace → different sessions (no context leak)', () => {
    const sm = new SessionManager({ ttlMinutes: 30 })
    const s1 = sm.getOrCreate('caller-1', 'conv-1')
    const s2 = sm.getOrCreate('caller-1', 'conv-2')
    assert.notEqual(s1.sessionId, s2.sessionId)
  })

  it('different callers get different sessions', () => {
    const sm = new SessionManager({ ttlMinutes: 30 })
    const s1 = sm.getOrCreate('caller-1', 'conv-1')
    const s2 = sm.getOrCreate('caller-2', 'conv-1')
    assert.notEqual(s1.sessionId, s2.sessionId)
  })

  it('expires session after TTL', () => {
    const sm = new SessionManager({ ttlMinutes: 0 }) // 0 min TTL for test
    const s1 = sm.getOrCreate('caller-1', 'conv-1')
    // Manually set lastActive to past
    sm._setLastActiveForTest('caller-1:conv-1', Date.now() - 60_000)
    sm.cleanup()
    const s2 = sm.getOrCreate('caller-1', 'conv-1')
    assert.notEqual(s1.sessionId, s2.sessionId)
    assert.equal(s2.isResume, false)
  })

  it('tracks active session count', () => {
    const sm = new SessionManager({ ttlMinutes: 30 })
    assert.equal(sm.activeCount, 0)
    sm.getOrCreate('caller-1', 'conv-1')
    assert.equal(sm.activeCount, 1)
    sm.getOrCreate('caller-2', 'conv-1')
    assert.equal(sm.activeCount, 2)
  })
})
```

**Step 2: Run test — expect FAIL**

**Step 3: Implement**

Map of `callerId:namespace → { sessionId: UUID, lastActive: number }`. Composite key ensures same caller with different namespaces gets independent CLI sessions (no `--resume` context leak). `cleanup()` sweeps expired entries. `getOrCreate(callerId, namespace)` returns existing or creates new with `crypto.randomUUID()`. HTTP callers without stable namespace → api-gateway auto-generates UUID per request → `isResume` always false → no `--resume`.

**Step 4: Run test — expect PASS**

**Step 5: Commit**

```bash
git add packages/plan-bridge-cli/src/session-manager*
git commit -m "feat(F017): session manager with per-caller isolation + TTL (AC-2,15,19)"
```

---

## Task 10: Provider Attestation — L1 Binary + L1.5 Env Probe + L2 Response

**Covers:** AC-18, AC-20, AC-21, AC-10

**Files:**
- Create: `packages/plan-bridge-cli/src/attestation.ts`
- Create: `packages/plan-bridge-cli/src/attestation.test.ts`

**Step 1: Write the failing test**

Test the pure logic: `isOfficialEndpoint`, `buildAttestation` (from pre-computed inputs), `verifyResponse` (claimed vs observed).

```typescript
import { describe, it } from 'node:test'
import assert from 'node:assert/strict'
import { isOfficialEndpoint, verifyResponseModel, OFFICIAL_ENDPOINTS } from './attestation.js'

describe('isOfficialEndpoint', () => {
  it('recognizes official Claude endpoint', () => {
    assert.ok(isOfficialEndpoint('claude', 'https://api.anthropic.com'))
  })

  it('rejects unknown endpoint', () => {
    assert.ok(!isOfficialEndpoint('claude', 'https://evil.example.com'))
  })

  it('returns true for null (default endpoint)', () => {
    assert.ok(isOfficialEndpoint('claude', null))
  })
})

describe('verifyResponseModel', () => {
  it('matches when claimed provider matches observed model', () => {
    const result = verifyResponseModel('claude', 'claude-sonnet-4')
    assert.ok(result.verified)
  })

  it('flags mismatch', () => {
    const result = verifyResponseModel('claude', 'gpt-4o')
    assert.ok(!result.verified)
  })

  it('verified=true when observed is null (cannot verify)', () => {
    const result = verifyResponseModel('claude', null)
    assert.ok(result.verified) // cannot disprove → trust
  })
})
```

**Step 2: Run test — expect FAIL**

**Step 3: Implement**

`OFFICIAL_ENDPOINTS` map, `isOfficialEndpoint` checks URL hostname, `verifyResponseModel` checks if observed model starts with provider prefix, `buildAttestation` assembles the full attestation object (binary hash + env probe done at startup via separate async functions).

**Step 4: Run test — expect PASS**

**Step 5: Commit**

```bash
git add packages/plan-bridge-cli/src/attestation*
git commit -m "feat(F017): provider attestation — endpoint whitelist + response verify (AC-10,18,20,21)"
```

---

## Task 11: Plan-bridge Node — A2A Protocol Layer

**Covers:** AC-1, AC-3, AC-5

**Files:**
- Create: `packages/plan-bridge-cli/src/bridge.ts`
- Create: `packages/plan-bridge-cli/src/index.ts`

**Step 1: Write bridge.ts**

Following the `bridge-tts/src/bridge.ts` pattern exactly:
1. `loadOrCreateIdentity` (reuse from bridge-tts)
2. `main()`: create handler → `MeshServer` → `MeshClient.hello()` → `startHeartbeat()`
3. Handler: extract prompt + namespace from invoke payload → `sessionManager.getOrCreate(callerId, namespace)` → `invokeCli(sessionId, prompt, isResume)` → parse result → return

The handler enforces `maxConcurrency` via semaphore counter. Excess returns `RATE_LIMITED`.

This is glue code — the core logic is in cli-wrapper.ts / session-manager.ts / attestation.ts (already tested). Bridge integration test is in Task 17.

**Step 2: Verify builds**

```bash
cd packages/plan-bridge-cli && pnpm build
```
Expected: compiles without error.

**Step 3: Commit**

```bash
git add packages/plan-bridge-cli/src/bridge.ts packages/plan-bridge-cli/src/index.ts
git commit -m "feat(F017): plan-bridge-cli A2A protocol layer (AC-1,3,5)"
```

---

## Task 12: API Gateway — Anthropic Format (`/v1/messages`)

**Covers:** AC-23, AC-25 (Anthropic part)

**Files:**
- Create: `packages/plan-pool/src/format-anthropic.ts`
- Create: `packages/plan-pool/src/format-anthropic.test.ts`

**Step 1: Write the failing test**

```typescript
import { describe, it } from 'node:test'
import assert from 'node:assert/strict'
import { anthropicToPrompt, toAnthropicResponse, toAnthropicSSEChunk } from './format-anthropic.js'

describe('anthropicToPrompt', () => {
  it('converts Anthropic messages array to plain prompt', () => {
    const messages = [
      { role: 'user', content: 'Hello, how are you?' },
    ]
    const prompt = anthropicToPrompt(messages)
    assert.equal(prompt, 'Hello, how are you?')
  })

  it('joins multi-turn messages', () => {
    const messages = [
      { role: 'user', content: 'Hi' },
      { role: 'assistant', content: 'Hello!' },
      { role: 'user', content: 'How are you?' },
    ]
    const prompt = anthropicToPrompt(messages)
    assert.ok(prompt.includes('Hi'))
    assert.ok(prompt.includes('How are you?'))
  })
})

describe('toAnthropicResponse', () => {
  it('wraps text into Anthropic Messages response format', () => {
    const resp = toAnthropicResponse({
      text: 'Hello!', model: 'claude-sonnet-4',
      inputTokens: 10, outputTokens: 5,
    })
    assert.equal(resp.type, 'message')
    assert.equal(resp.role, 'assistant')
    assert.equal(resp.content[0].type, 'text')
    assert.equal(resp.content[0].text, 'Hello!')
    assert.equal(resp.model, 'claude-sonnet-4')
  })
})

describe('toAnthropicSSEChunk', () => {
  it('formats as content_block_delta SSE', () => {
    const chunk = toAnthropicSSEChunk('Hello')
    assert.ok(chunk.includes('event: content_block_delta'))
    assert.ok(chunk.includes('"Hello"'))
  })
})
```

**Step 2: Run test — expect FAIL**

**Step 3: Implement format conversion functions**

**Step 4: Run test — expect PASS**

**Step 5: Commit**

```bash
git add packages/plan-pool/src/format-anthropic*
git commit -m "feat(F017): Anthropic format conversion — messages + SSE (AC-23,25)"
```

---

## Task 13: API Gateway — OpenAI Format (`/v1/chat/completions`)

**Covers:** AC-23, AC-25 (OpenAI part)

**Files:**
- Create: `packages/plan-pool/src/format-openai.ts`
- Create: `packages/plan-pool/src/format-openai.test.ts`

**Step 1: Write the failing test**

Same pattern as Task 12 but for OpenAI ChatCompletions format: `openaiToPrompt`, `toOpenAIResponse`, `toOpenAISSEChunk`.

**Step 2: Run test — expect FAIL**

**Step 3: Implement**

**Step 4: Run test — expect PASS**

**Step 5: Commit**

```bash
git add packages/plan-pool/src/format-openai*
git commit -m "feat(F017): OpenAI format conversion — chat completions + SSE (AC-23,25)"
```

---

## Task 14: API Gateway + Admin API — HTTP Server

**Covers:** AC-6, AC-24, AC-26, AC-33

**Files:**
- Create: `packages/plan-pool/src/api-gateway.ts`
- Create: `packages/plan-pool/src/admin-api.ts`
- Create: `packages/plan-pool/src/api-gateway.test.ts`

**Step 1: Write the failing test**

Test the Fastify app via `inject()` (no real HTTP needed):

```typescript
import { describe, it, beforeEach, afterEach } from 'node:test'
import assert from 'node:assert/strict'
import { createApiGateway } from './api-gateway.js'
import { createDatabase, type PoolDatabase } from './db.js'
import { TeamManager } from './team.js'

describe('ApiGateway', () => {
  let poolDb: PoolDatabase
  let tm: TeamManager
  let app: any  // FastifyInstance

  beforeEach(async () => {
    poolDb = createDatabase(':memory:')
    tm = new TeamManager(poolDb.db)
    const { teamId, apiKey } = tm.createTeam({
      name: 'Lab', providerScope: 'claude',
      ownerDisplayName: 'Alice', ownerPublicKey: 'pk-a',
    })
    app = await createApiGateway({ db: poolDb.db, teamManager: tm })
  })

  afterEach(async () => {
    await app.close()
    poolDb.close()
  })

  it('rejects unauthenticated request', async () => {
    const resp = await app.inject({ method: 'POST', url: '/v1/messages', payload: {} })
    assert.equal(resp.statusCode, 401)
  })

  it('GET /v1/models rejects without auth', async () => {
    const resp = await app.inject({ method: 'GET', url: '/v1/models' })
    assert.equal(resp.statusCode, 401)
  })

  it('GET /v1/models returns team-scoped model list with valid key', async () => {
    const { apiKey } = tm.createTeam({
      name: 'Lab-GPT', providerScope: 'gpt',
      ownerDisplayName: 'Dave', ownerPublicKey: 'pk-d',
    })
    const resp = await app.inject({
      method: 'GET', url: '/v1/models',
      headers: { authorization: `Bearer ${apiKey}` },
    })
    assert.equal(resp.statusCode, 200)
    const body = JSON.parse(resp.body)
    assert.ok(Array.isArray(body.data))
    // Should only see models for this team's provider_scope, not other teams
  })

  it('POST /v1/bridges/register validates provider_scope + pins attestation', async () => {
    const { teamId, memberId, apiKey } = tm.createTeam({
      name: 'Lab-Claude2', providerScope: 'claude',
      ownerDisplayName: 'Eve', ownerPublicKey: 'pk-e',
    })
    const resp = await app.inject({
      method: 'POST', url: '/v1/bridges/register',
      headers: { authorization: `Bearer ${apiKey}` },
      payload: {
        bridgeId: 'bridge-eve', provider: 'claude',
        shareMode: 'idle-only', maxConcurrency: 3,
        attestation: { binaryHash: 'sha256-abc', version: '1.0.0' },
      },
    })
    assert.equal(resp.statusCode, 200)
    const body = JSON.parse(resp.body)
    assert.equal(body.pinned, true)
  })

  it('POST /v1/bridges/register rejects provider_scope mismatch', async () => {
    const { apiKey } = tm.createTeam({
      name: 'Lab-Claude3', providerScope: 'claude',
      ownerDisplayName: 'Frank', ownerPublicKey: 'pk-f',
    })
    const resp = await app.inject({
      method: 'POST', url: '/v1/bridges/register',
      headers: { authorization: `Bearer ${apiKey}` },
      payload: { bridgeId: 'bridge-f', provider: 'gpt' }, // mismatch!
    })
    assert.equal(resp.statusCode, 400) // provider doesn't match team scope
  })

  it('POST /v1/join with valid invite returns apiKey', async () => {
    const { teamId, memberId } = tm.createTeam({
      name: 'Lab2', providerScope: 'claude',
      ownerDisplayName: 'Bob', ownerPublicKey: 'pk-b',
    })
    const { code } = tm.createInvite({ teamId, createdBy: memberId, maxUses: 1, ttlHours: 24 })
    const resp = await app.inject({
      method: 'POST', url: '/v1/join',
      payload: { code, displayName: 'Carol', publicKey: 'pk-carol' },
    })
    assert.equal(resp.statusCode, 200)
    const body = JSON.parse(resp.body)
    assert.ok(body.apiKey)
  })
})
```

**Step 2: Run test — expect FAIL**

**Step 3: Implement**

`createApiGateway` creates a Fastify instance, registers routes:
- `POST /v1/messages` — Anthropic format handler (auth → quota check → pool.invoke → format response)
- `POST /v1/chat/completions` — OpenAI format handler
- `GET /v1/models` — **requires API key auth**, returns models scoped to caller's team only (prevents cross-team topology leak)
- `POST /v1/join` — invite join endpoint
- `POST /v1/bridges/register` — **bridge registration** (auth via member API key):
  - Validate member identity + `team.provider_scope` matches bridge's declared provider
  - First registration: pin attestation hash/version (TOFU)
  - Subsequent: if attestation changed → set `share_mode='off'` + alert admin for confirmation
  - Insert/update `bridges` table + add to `PoolRouter`
- `POST /v1/admin/teams` — create team (pool owner only)
- `POST /v1/admin/invites` — create invite (team owner/admin)
- `PUT /v1/admin/members/:id/suspend` — suspend member
- `PUT /v1/admin/members/:id/borrow-limit` — set borrow limit
- `POST /v1/admin/members/:id/rotate-key` — rotate API key
- `GET /v1/admin/usage` — usage stats JSON

**Step 4: Run test — expect PASS**

**Step 5: Commit**

```bash
git add packages/plan-pool/src/api-gateway* packages/plan-pool/src/admin-api*
git commit -m "feat(F017): API gateway + admin endpoints with Fastify (AC-6,24,26,33)"
```

---

## Task 15: Pool-node Main — Orchestration + Hub Registration

**Covers:** AC-6 (A2A 辅), AC-10, AC-22 (integration)

**Files:**
- Create: `packages/plan-pool/src/config.ts`
- Create: `packages/plan-pool/src/index.ts`

**Step 1: Implement config + main**

`config.ts`: Load pool configuration from JSON/env (port, hubUrl, dbPath, teams).

`index.ts`: Main entry point:
1. `createDatabase(dbPath)` → init schema
2. `TeamManager(db)` + `Ledger(db)` + `PoolRouter()`
3. `createApiGateway(...)` → start HTTP server on `listenPort`
4. `MeshClient({ hubUrl, identity })` → `hello()` → `startHeartbeat()`
5. Register as A2A node (agentId from config, e.g. `claude-pool`)
6. Handle A2A invoke requests → route to pool logic
7. Periodic: `ledger.expireStaleLeases()` + `router.cleanupStaleBindings()`

**Step 2: Verify builds**

```bash
cd packages/plan-pool && pnpm build
```

**Step 3: Commit**

```bash
git add packages/plan-pool/src/config.ts packages/plan-pool/src/index.ts
git commit -m "feat(F017): pool-node main entry — config + orchestration + A2A registration (AC-6)"
```

---

## Task 16: F013-lite — Hub Open Registration Mode

**Covers:** AC-11, AC-12, AC-13

**Files:**
- Modify: `packages/mesh-hub/src/config.ts` (add `registrationMode` + `defaultMaxScopes`)
- Modify: `packages/mesh-hub/src/hub.ts` (extend HELLO handler)
- Modify: `packages/mesh-hub/src/hub.test.ts` (add open registration tests)

**Step 1: Write the failing test**

```typescript
// Add to existing hub.test.ts
describe('F013-lite: open registration', () => {
  it('unknown node auto-registers in open mode', async () => {
    // create Hub with registrationMode: 'open'
    // send HELLO from unknown nodeId with valid proof
    // expect HELLO_ACK (not 403)
  })

  it('whitelist mode rejects unknown node (backward compat)', async () => {
    // create Hub with registrationMode: 'whitelist' (default)
    // send HELLO from unknown nodeId
    // expect 403
  })

  it('open mode applies defaultMaxScopes cap', async () => {
    // create Hub with registrationMode: 'open', defaultMaxScopes: ['read']
    // register unknown node
    // request token with scope ['read', 'invoke']
    // expect scope denied for 'invoke'
  })

  it('TOFU: rejects second HELLO with different publicKey for same nodeId', async () => {
    // register node in open mode
    // second HELLO with same nodeId but different publicKey
    // expect 403
  })
})
```

**Step 2: Run test — expect FAIL**

**Step 3: Modify hub.ts HELLO handler**

In the HELLO handler, after the whitelist check fails:
```typescript
// Existing: return 403 "node not in allowlist"
// New: if registrationMode === 'open', auto-register via TOFU
if (config.registrationMode === 'open') {
  const existing = dynamicNodes.get(nodeId)
  if (existing && existing.publicKey !== publicKey) {
    return reply.code(403).send({ error: 'publicKey mismatch for registered node' })
  }
  if (!existing) {
    dynamicNodes.set(nodeId, {
      nodeId, publicKey,
      maxScopes: config.defaultMaxScopes ?? ['read'],
    })
  }
  // Continue with normal proof verification + certificate issuance
}
```

Add `registrationMode` and `defaultMaxScopes` to `HubConfig`. ~60 lines total change.

**Step 4: Run test — expect PASS**

**Step 5: Commit**

```bash
git add packages/mesh-hub/src/config.ts packages/mesh-hub/src/hub.ts packages/mesh-hub/src/hub.test.ts
git commit -m "feat(F013-lite): Hub open registration mode with TOFU (AC-11,12,13)"
```

---

## Task 17: Integration Test — End-to-end Flow

**Covers:** Validates AC-1 through AC-33 in combination

**Files:**
- Create: `packages/plan-pool/src/integration.test.ts`

**Step 1: Write integration test**

Tests the full flow without external dependencies (uses `:memory:` SQLite, mock bridge):

```typescript
import { describe, it, beforeEach, afterEach } from 'node:test'
import assert from 'node:assert/strict'
import { createDatabase } from './db.js'
import { TeamManager } from './team.js'
import { Ledger } from './ledger.js'
import { PoolRouter, type BridgeEntry } from './pool.js'
import { checkQuota } from './credit.js'
import { calculateNormalizedCost } from './credit.js'

describe('Integration: team → invite → join → route → ledger → credit', () => {
  it('full lifecycle', () => {
    const poolDb = createDatabase(':memory:')
    const tm = new TeamManager(poolDb.db)
    const ledger = new Ledger(poolDb.db)
    const router = new PoolRouter()

    // 1. Create team
    const { teamId, memberId: aliceId, apiKey: aliceKey } = tm.createTeam({
      name: 'Lab-Claude', providerScope: 'claude',
      ownerDisplayName: 'Alice', ownerPublicKey: 'pk-alice',
    })

    // 2. Alice creates invite
    const { code } = tm.createInvite({ teamId, createdBy: aliceId, maxUses: 2, ttlHours: 24 })

    // 3. Bob joins
    const { memberId: bobId, apiKey: bobKey } = tm.joinTeam({
      code, displayName: 'Bob', publicKey: 'pk-bob',
    })

    // 4. Alice registers a bridge (via TeamManager, not raw INSERT)
    const bridgeId = 'bridge-alice'
    tm.registerBridge({
      bridgeId, teamId, ownerMemberId: aliceId, provider: 'claude',
      shareMode: 'idle-only', externalMaxConcurrency: 1,
      attestation: { binaryHash: 'sha256-abc', version: '1.0.0' },
    })

    router.addBridge({
      id: bridgeId, teamId, ownerMemberId: aliceId, provider: 'claude',
      maxConcurrency: 3, activeSessions: 0, shareMode: 'idle-only',
      externalMaxConcurrency: 1, externalActiveSessions: 0,
      lastOwnerActiveAt: 0, ownerIdleTimeout: 30 * 60 * 1000,
    })

    // 5. Bob's quota check — should have $30 available (borrow limit)
    const q1 = checkQuota(poolDb.db, bobId)
    assert.ok(q1.allowed)
    assert.equal(q1.availableUsd, 30.0)

    // 6. Self-route deny: Alice cannot route to her own bridge
    const selfRoute = router.selectBridge({ callerId: aliceId, teamId, provider: 'claude', namespace: 'ns-1' })
    assert.equal(selfRoute, null)

    // 7. Bob can route to Alice's bridge
    const bobRoute = router.selectBridge({ callerId: bobId, teamId, provider: 'claude', namespace: 'ns-1' })
    assert.ok(bobRoute)
    assert.equal(bobRoute!.id, bridgeId)

    // 8. Lease-based accounting
    const leaseId = ledger.acquireLease({
      idempotencyKey: 'req-1', teamId, sourceMemberId: bobId,
      bridgeId, bridgeOwnerId: aliceId, provider: 'claude', leaseTTLMs: 30_000,
    })
    const cost = calculateNormalizedCost({
      costUsd: 0.05, model: 'claude-sonnet-4', inputTokens: 1000, outputTokens: 500,
    })
    ledger.settleLease(leaseId, {
      observedModel: 'claude-sonnet-4', inputTokens: 1000, outputTokens: 500,
      normalizedCost: cost.amount, costSource: cost.source, status: 'success',
    })

    // 9. Check balances
    // Bob consumed $0.05, Alice contributed $0.05
    const qBob = checkQuota(poolDb.db, bobId)
    assert.ok(Math.abs(qBob.balanceUsd - (-0.05)) < 0.001)
    assert.ok(Math.abs(qBob.availableUsd - 2.95) < 0.001)

    const qAlice = checkQuota(poolDb.db, aliceId)
    assert.ok(Math.abs(qAlice.balanceUsd - 0.05) < 0.001)

    // 10. Auth check
    const authBob = tm.authenticateApiKey(bobKey)
    assert.ok(authBob)
    assert.equal(authBob!.memberId, bobId)

    poolDb.close()
  })
})
```

**Step 2: Run test — expect PASS** (all components already tested individually)

**Step 3: Commit**

```bash
git add packages/plan-pool/src/integration.test.ts
git commit -m "test(F017): integration test — full lifecycle (team→invite→route→ledger→credit)"
```

---

## Task 18: Workspace Wiring + Final Build

**Covers:** Build infrastructure

**Files:**
- Modify: `pnpm-workspace.yaml` (add new packages)
- Verify: all packages build and test

**Step 1: Update pnpm-workspace**

Add `packages/plan-pool`, `packages/plan-bridge-cli`, and `packages/plan-bridge-api` to workspace config.

**Step 2: Full build + test**

```bash
pnpm install && pnpm -r build && pnpm -r test
```
Expected: all packages build and all tests pass.

**Step 3: Commit**

```bash
git add pnpm-workspace.yaml pnpm-lock.yaml
git commit -m "chore(F017): wire plan-pool + plan-bridge-cli + plan-bridge-api into pnpm workspace"
```

---

## Task 19: API Proxy Bridge — HTTP Forwarding Type

**Covers:** AC-4

**Files:**
- Create: `packages/plan-bridge-api/package.json`
- Create: `packages/plan-bridge-api/tsconfig.json`
- Create: `packages/plan-bridge-api/src/proxy.ts`
- Create: `packages/plan-bridge-api/src/proxy.test.ts`
- Create: `packages/plan-bridge-api/src/bridge.ts`

**Step 1: Create package scaffold**

Same pattern as plan-bridge-cli. Dependencies: `@agent-mesh/node`.

**Step 2: Write the failing test**

```typescript
import { describe, it } from 'node:test'
import assert from 'node:assert/strict'
import { buildProxyRequest, extractUsageFromHeaders } from './proxy.js'

describe('buildProxyRequest', () => {
  it('forwards to target URL without exposing API key in response', () => {
    const req = buildProxyRequest({
      targetUrl: 'https://api.openai.com/v1/chat/completions',
      apiKey: 'sk-secret-key',
      body: { messages: [{ role: 'user', content: 'hi' }], model: 'gpt-4o' },
    })
    assert.equal(req.url, 'https://api.openai.com/v1/chat/completions')
    assert.ok(req.headers['authorization'].includes('sk-secret-key'))
    // Key stays in request headers, never in response body
  })
})

describe('extractUsageFromHeaders', () => {
  it('extracts model from x-model header', () => {
    const usage = extractUsageFromHeaders({
      'x-model': 'gpt-4o',
      'x-request-id': 'req-123',
    })
    assert.equal(usage.model, 'gpt-4o')
  })
})
```

**Step 3: Run test — expect FAIL**

**Step 4: Implement**

`proxy.ts`: HTTP proxy — `fetch()` to target with API key in header, stream response back, extract usage from response headers/body. Key never leaves bridge process.

`bridge.ts`: Same A2A lifecycle as plan-bridge-cli (MeshClient hello + heartbeat + MeshServer), but handler does HTTP proxy instead of CLI spawn.

**Step 5: Run test — expect PASS**

**Step 6: Commit**

```bash
git add packages/plan-bridge-api/
git commit -m "feat(F017): API proxy bridge — HTTP forwarding with key isolation (AC-4)"
```

---

## Review Findings Resolution Log

| # | Finding | Severity | Resolution |
|---|---------|----------|------------|
| 1 | AC-4 api-proxy-bridge 无实现任务 | P1 | 新增 Task 19（plan-bridge-api 包） |
| 2 | Session 语义退回单映射，丢失 namespace | P1 | Task 7 sticky key 改为 `callerId:provider:namespace`；Task 9 SessionManager key 改为 `callerId:namespace`；无 namespace 时 auto-UUID → 无状态 |
| 3 | Bridge 挂载流程缺失（注册/pin/provider_scope 校验） | P1 | Task 5 新增 `registerBridge` 方法；Task 14 新增 `POST /v1/bridges/register`（provider_scope 校验 + TOFU attestation pin + 变更暂停）；集成测试改用 `registerBridge` |
| 4 | Lease 超时未做保守策略结算 | P1 | Task 1 `member_balances` view 扩展为包含 `lease_status='expired'` 的保守占用；Task 4 `expireStaleLeases` 改为 `conservative_settle` + 保留 $0.02 fallback cost |
| 5 | `/v1/models` 匿名暴露跨 team 拓扑 | P2 | Task 14 改为 `GET /v1/models` 需要 API key 认证，按 `team_id` 过滤返回 |

---

## Summary

| Task | Component | ACs | Est. Lines |
|------|-----------|-----|-----------|
| 1 | Package scaffold + SQLite schema | 9,27 | ~120 |
| 2 | Circuit Breaker | 22 | ~80 |
| 3 | Normalized Cost | 30 | ~50 |
| 4 | Ledger (lease-based) | 9 | ~100 |
| 5 | Team Management + Bridge Registration | 27,28,24 | ~250 |
| 6 | Credit System (quota) | 29,31 | ~40 |
| 7 | Pool Router | 7,8,31,32 | ~150 |
| 8 | CLI Wrapper | 1,14,17 | ~120 |
| 9 | Session Manager | 2,15,19 | ~60 |
| 10 | Attestation | 10,18,20,21 | ~80 |
| 11 | Bridge Node (A2A — CLI) | 1,3,5 | ~80 |
| 12 | Format: Anthropic | 23,25 | ~80 |
| 13 | Format: OpenAI | 23,25 | ~80 |
| 14 | API Gateway + Admin | 6,24,26,33 | ~250 |
| 15 | Pool-node Main | 6 | ~80 |
| 16 | F013-lite (Hub) | 11,12,13 | ~60 |
| 17 | Integration Test | all | ~80 |
| 18 | Workspace Wiring | — | ~5 |
| 19 | API Proxy Bridge | 4 | ~80 |
| **Total** | | **33 ACs** | **~1845** |

**Build order (dependency graph):**
```
Task 1 (schema) ─┬→ Task 4 (ledger) ─┬→ Task 7 (router) ──→ Task 15 (main)
                 │                   │
                 ├→ Task 5 (team) ───┤→ Task 14 (gateway) ──→ Task 15
                 │                   │
                 └→ Task 6 (credit) ─┘
                                          Task 17 (integration)
Task 2 (breaker) ────→ Task 7 (router)   Task 18 (wiring)
Task 3 (cost)    ────→ Task 6 (credit)

Task 8 (cli-wrapper) ─┬→ Task 11 (bridge CLI)
Task 9 (session mgr)  ─┤
Task 10 (attestation) ─┘

Task 12 (anthropic) ─→ Task 14 (gateway)
Task 13 (openai)    ─→ Task 14 (gateway)

Task 16 (F013-lite) — independent
Task 19 (api-proxy-bridge) — independent (after Task 1 for schema)
```

**Parallelizable groups:**
- Group A (no deps): Tasks 2, 3, 8, 9, 10, 16
- Group B (after Task 1): Tasks 4, 5, 6, 19
- Group C (after Group A+B): Tasks 7, 12, 13
- Group D (after Group C): Tasks 11, 14, 15
- Group E (after all): Tasks 17, 18
