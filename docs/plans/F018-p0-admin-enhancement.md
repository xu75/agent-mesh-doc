# F018 P0: Plan Pool Admin Enhancement — Implementation Plan

**Feature:** F018 — `docs/features/F018-plan-pool-admin.md`
**Goal:** 为 Plan Pool 增加时间+额度+速率限制三维管理，并提供成员自服务面板
**Acceptance Criteria:**
- [x] AC1: 成员有 `expires_at` 字段，到期后 invoke/register 被拒（403 EXPIRED）
- [x] AC2: 过期 ≠ suspend：过期成员 status 不变、key 不 revoke，仍可访问 `/v1/me`
- [x] AC3: Owner 可在 admin 面板设置/修改成员过期时间和额度
- [x] AC4: 时间、额度、速率限制是 AND 关系：全部满足才可调用
- [x] AC5: 5H/1W 滚动窗口速率限制：按金额控制，超限返回 429 + Retry-After
- [x] AC6: Admin 端可设每个成员的 5H/1W 限额（金额），也可设团队默认值
- [x] AC7: suspend/expire 时立即从路由表摘除成员 bridge（不等心跳超时）
- [x] AC8: 成员用 API key 可登录 `/member` 查看 effectiveStatus
- [x] AC9: Member 端显示 quota $（available/total）+ per-call $，5H/7D 窗口显示百分比
- [x] AC10: 成员可看到自己的调用历史和用量统计
**Architecture:** DB 层增加 `expires_at` + rate limit 列，auth 层从 `AuthResult` 重构为 `AuthContext`（含 `canInvoke/canReadSelf/canAdmin/canRegisterBridge` 权限方法），credit 层新增 `checkRateLimit()` UTC 整点桶实现，前端分 admin.html（owner）和 member.html（成员）两套面板
**Tech Stack:** TypeScript, better-sqlite3, Fastify, node:test, 原生 HTML/CSS/JS
**前端验证:** Yes — admin.html + member.html 均需 Chrome 实测

---

## Phase A: Data Layer

### Task 1: DB Migration — expires_at + rate limit columns

**Files:**
- Modify: `packages/plan-pool/src/db.ts:14-122` (CREATE TABLE + migration)
- Test: `packages/plan-pool/src/db.test.ts`

**Step 1: Write the failing test**

在 `db.test.ts` 末尾添加测试：

```ts
it("members table has expires_at column", () => {
  const cols = db.db.prepare("PRAGMA table_info(members)").all() as Array<{ name: string }>;
  const names = cols.map(c => c.name);
  assert.ok(names.includes("expires_at"), "missing expires_at column");
});

it("members table has rate_limit_5h_usd and rate_limit_weekly_usd columns", () => {
  const cols = db.db.prepare("PRAGMA table_info(members)").all() as Array<{ name: string }>;
  const names = cols.map(c => c.name);
  assert.ok(names.includes("rate_limit_5h_usd"), "missing rate_limit_5h_usd");
  assert.ok(names.includes("rate_limit_weekly_usd"), "missing rate_limit_weekly_usd");
});

it("teams table has default_rate_limit_5h_usd and default_rate_limit_weekly_usd columns", () => {
  const cols = db.db.prepare("PRAGMA table_info(teams)").all() as Array<{ name: string }>;
  const names = cols.map(c => c.name);
  assert.ok(names.includes("default_rate_limit_5h_usd"), "missing default_rate_limit_5h_usd");
  assert.ok(names.includes("default_rate_limit_weekly_usd"), "missing default_rate_limit_weekly_usd");
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/plan-pool && npx tsx --test src/db.test.ts`
Expected: FAIL — columns don't exist yet

**Step 3: Implement — add columns to CREATE TABLE + migration**

`db.ts` — `members` CREATE TABLE 增加三列：

```sql
expires_at                INTEGER,
rate_limit_5h_usd         REAL,
rate_limit_weekly_usd     REAL,
```

`teams` CREATE TABLE 增加两列：

```sql
default_rate_limit_5h_usd    REAL DEFAULT 5.0,
default_rate_limit_weekly_usd REAL DEFAULT 40.0,
```

Migration block（在现有 migration 后追加）：

```ts
// Migration: add expires_at + rate limit columns to members
const memberCols = db.prepare("PRAGMA table_info(members)").all() as Array<{ name: string }>;
const memberColNames = new Set(memberCols.map(c => c.name));
if (!memberColNames.has("expires_at")) {
  db.exec("ALTER TABLE members ADD COLUMN expires_at INTEGER");
}
if (!memberColNames.has("rate_limit_5h_usd")) {
  db.exec("ALTER TABLE members ADD COLUMN rate_limit_5h_usd REAL");
}
if (!memberColNames.has("rate_limit_weekly_usd")) {
  db.exec("ALTER TABLE members ADD COLUMN rate_limit_weekly_usd REAL");
}

// Migration: add default rate limit columns to teams
const teamCols2 = db.prepare("PRAGMA table_info(teams)").all() as Array<{ name: string }>;
const teamColNames2 = new Set(teamCols2.map(c => c.name));
if (!teamColNames2.has("default_rate_limit_5h_usd")) {
  db.exec("ALTER TABLE teams ADD COLUMN default_rate_limit_5h_usd REAL DEFAULT 5.0");
}
if (!teamColNames2.has("default_rate_limit_weekly_usd")) {
  db.exec("ALTER TABLE teams ADD COLUMN default_rate_limit_weekly_usd REAL DEFAULT 40.0");
}
```

**Step 4: Run test to verify it passes**

Run: `cd packages/plan-pool && npx tsx --test src/db.test.ts`
Expected: ALL PASS

**Step 5: Commit**

```bash
git add packages/plan-pool/src/db.ts packages/plan-pool/src/db.test.ts
git commit -m "feat(plan-pool): add expires_at + rate limit columns to members/teams schema"
```

---

### Task 2: credit.ts — Model ID prefix matching (R6)

**Files:**
- Modify: `packages/plan-pool/src/credit.ts:27-29` (normalizeModelId)
- Test: `packages/plan-pool/src/credit.test.ts`

> R6 风险：bridge 回报的 model ID 可能带版本后缀（如 `claude-sonnet-4-20250514`），
> 当前 `normalizeModelId` 仅去 `[...]`。解法：去后缀后再做前缀匹配。

**Step 1: Write the failing test**

```ts
it("matches versioned model ID by prefix (claude-sonnet-4-20250514)", () => {
  const cost = calculateNormalizedCost({
    costUsd: null,
    model: "claude-sonnet-4-20250514",
    inputTokens: 1_000_000,
    outputTokens: 100_000,
  });
  assert.equal(cost.source, "token_pricing");
  assert.ok(Math.abs(cost.amount - 4.5) < 0.001);
});

it("matches claude-haiku-3-5-sonnet variant by prefix", () => {
  const cost = calculateNormalizedCost({
    costUsd: null,
    model: "claude-haiku-4-5",
    inputTokens: 1_000_000,
    outputTokens: 1_000_000,
  });
  // haiku should have a pricing entry
  assert.notEqual(cost.source, "fallback");
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/plan-pool && npx tsx --test src/credit.test.ts`
Expected: FAIL — versioned IDs fall through to fallback

**Step 3: Implement — add haiku pricing + prefix matching**

`credit.ts` — 扩展 OFFICIAL_PRICING + 改 `calculateNormalizedCost`：

```ts
export const OFFICIAL_PRICING: Record<string, { input: number; output: number }> = {
  "claude-sonnet-4": { input: 3.0 / 1e6, output: 15.0 / 1e6 },
  "claude-sonnet-4-6": { input: 3.0 / 1e6, output: 15.0 / 1e6 },
  "claude-opus-4": { input: 15.0 / 1e6, output: 75.0 / 1e6 },
  "claude-opus-4-6": { input: 15.0 / 1e6, output: 75.0 / 1e6 },
  "claude-haiku-4-5": { input: 0.8 / 1e6, output: 4.0 / 1e6 },
  "claude-haiku-3-5": { input: 0.8 / 1e6, output: 4.0 / 1e6 },
  "gpt-4o": { input: 2.5 / 1e6, output: 10.0 / 1e6 },
  "gpt-4o-mini": { input: 0.15 / 1e6, output: 0.6 / 1e6 },
  "o1": { input: 15.0 / 1e6, output: 60.0 / 1e6 },
  "o3-mini": { input: 1.1 / 1e6, output: 4.4 / 1e6 },
};

/** Strip CLI suffix, then try exact match → prefix match against known models. */
function resolveModelPricing(raw: string): { input: number; output: number } | undefined {
  const normalized = normalizeModelId(raw);
  // Exact match first
  if (OFFICIAL_PRICING[normalized]) return OFFICIAL_PRICING[normalized];
  // Prefix match: "claude-sonnet-4-20250514" → "claude-sonnet-4"
  for (const key of Object.keys(OFFICIAL_PRICING)) {
    if (normalized.startsWith(key)) return OFFICIAL_PRICING[key];
  }
  return undefined;
}
```

在 `calculateNormalizedCost` 中用 `resolveModelPricing(usage.model)` 替换原来的 `OFFICIAL_PRICING[normalized]`。

**Step 4: Run test to verify it passes**

Run: `cd packages/plan-pool && npx tsx --test src/credit.test.ts`
Expected: ALL PASS

**Step 5: Commit**

```bash
git add packages/plan-pool/src/credit.ts packages/plan-pool/src/credit.test.ts
git commit -m "feat(plan-pool): expand pricing table + prefix matching for model IDs (R6)"
```

---

## Phase B: Business Logic

### Task 3: checkRateLimit() — UTC bucket rate limiting

**Files:**
- Modify: `packages/plan-pool/src/credit.ts` (新增 checkRateLimit 函数)
- Test: `packages/plan-pool/src/credit.test.ts`

**Step 1: Write the failing test**

```ts
import { createDatabase, type PoolDatabase } from "./db.js";

describe("checkRateLimit", () => {
  let poolDb: PoolDatabase;

  beforeEach(() => {
    poolDb = createDatabase(":memory:");
    const now = Date.now();
    // Seed team + member
    poolDb.db.prepare(
      `INSERT INTO teams (id, name, provider_scope, owner_member_id, default_rate_limit_5h_usd, default_rate_limit_weekly_usd, created_at)
       VALUES ('t1', 'Lab', 'claude', 'm1', 5.0, 40.0, ?)`
    ).run(now);
    poolDb.db.prepare(
      `INSERT INTO members (id, team_id, display_name, public_key, role, status, joined_at, last_active_at)
       VALUES ('m1', 't1', 'Alice', 'pk-a', 'owner', 'active', ?, ?)`
    ).run(now, now);
  });

  afterEach(() => { poolDb.close(); });

  it("returns allowed=true when no invocations", () => {
    const result = checkRateLimit(poolDb.db, "m1", "t1");
    assert.equal(result.allowed, true);
    assert.equal(result.exceeded5h, false);
    assert.equal(result.exceeded7d, false);
  });

  it("returns exceeded5h=true when 5H window exceeded", () => {
    const now = Date.now();
    poolDb.db.prepare(
      `INSERT INTO invocations (idempotency_key, team_id, source_member_id, bridge_id, bridge_owner_id,
       provider, normalized_cost, lease_status, status, created_at)
       VALUES (?, 't1', 'm1', 'b1', 'm2', 'claude', 6.0, 'settled', 'success', ?)`
    ).run("k1", now);
    const result = checkRateLimit(poolDb.db, "m1", "t1");
    assert.equal(result.allowed, false);
    assert.equal(result.exceeded5h, true);
    assert.ok(result.retryAfterSeconds5h > 0);
    // retryAfter should point to next UTC hour boundary
    assert.ok(result.retryAfterSeconds5h <= 3600);
  });

  it("returns exceeded7d=true when ONLY 7D window exceeded (5H still OK)", () => {
    // Set member 5H limit high ($100) so only 7D triggers
    poolDb.db.prepare("UPDATE members SET rate_limit_5h_usd = 100.0 WHERE id = 'm1'").run();
    const now = Date.now();
    poolDb.db.prepare(
      `INSERT INTO invocations (idempotency_key, team_id, source_member_id, bridge_id, bridge_owner_id,
       provider, normalized_cost, lease_status, status, created_at)
       VALUES (?, 't1', 'm1', 'b1', 'm2', 'claude', 50.0, 'settled', 'success', ?)`
    ).run("k2", now);
    const result = checkRateLimit(poolDb.db, "m1", "t1");
    assert.equal(result.allowed, false);
    assert.equal(result.exceeded5h, false); // $50 < $100 override
    assert.equal(result.exceeded7d, true);  // $50 > $40 default
    assert.ok(result.retryAfterSeconds7d > 0);
    assert.ok(result.retryAfterSeconds7d <= 86400);
  });

  it("returns both exceeded flags when both windows exceeded", () => {
    const now = Date.now();
    poolDb.db.prepare(
      `INSERT INTO invocations (idempotency_key, team_id, source_member_id, bridge_id, bridge_owner_id,
       provider, normalized_cost, lease_status, status, created_at)
       VALUES (?, 't1', 'm1', 'b1', 'm2', 'claude', 50.0, 'settled', 'success', ?)`
    ).run("k2b", now);
    const result = checkRateLimit(poolDb.db, "m1", "t1");
    assert.equal(result.allowed, false);
    assert.equal(result.exceeded5h, true);  // $50 > $5
    assert.equal(result.exceeded7d, true);  // $50 > $40
    // rateLimitPrimaryReason should pick weekly (canonical priority)
    const primary = rateLimitPrimaryReason(result);
    assert.equal(primary.reason, "RATE_LIMITED_WEEKLY");
  });

  it("UTC bucket boundary: fake clock 10:32 UTC → bucketStart5h = 06:00", () => {
    // Fake time: 2026-04-11 10:32:15 UTC = 1744364135000 (approximate)
    const fakeNow = Date.UTC(2026, 3, 11, 10, 32, 15); // month is 0-indexed
    const expectedBucketStart = Date.UTC(2026, 3, 11, 6, 0, 0); // floor(10) - 4 = 6
    const expectedNextHour = Date.UTC(2026, 3, 11, 11, 0, 0);
    const expectedRetryAfter = Math.ceil((expectedNextHour - fakeNow) / 1000); // ~1665s

    // Insert invocation at the fake time
    poolDb.db.prepare(
      `INSERT INTO invocations (idempotency_key, team_id, source_member_id, bridge_id, bridge_owner_id,
       provider, normalized_cost, lease_status, status, created_at)
       VALUES (?, 't1', 'm1', 'b1', 'm2', 'claude', 6.0, 'settled', 'success', ?)`
    ).run("k-clock", fakeNow);

    // Insert old invocation BEFORE bucket start — should NOT count
    poolDb.db.prepare(
      `INSERT INTO invocations (idempotency_key, team_id, source_member_id, bridge_id, bridge_owner_id,
       provider, normalized_cost, lease_status, status, created_at)
       VALUES (?, 't1', 'm1', 'b1', 'm2', 'claude', 100.0, 'settled', 'success', ?)`
    ).run("k-old", expectedBucketStart - 1);

    // checkRateLimit uses Date.now() internally — we need to inject time.
    // Implementation note: refactor checkRateLimit to accept optional `nowMs` param for testability.
    const result = checkRateLimit(poolDb.db, "m1", "t1", fakeNow);
    assert.equal(result.exceeded5h, true); // $6 > $5
    assert.equal(result.retryAfterSeconds5h, expectedRetryAfter);
  });

  it("UTC 7D bucket boundary: retryAfter points to next UTC day 00:00", () => {
    // Fake time: 2026-04-11 15:00:00 UTC
    const fakeNow = Date.UTC(2026, 3, 11, 15, 0, 0);
    const expectedNextDay = Date.UTC(2026, 3, 12, 0, 0, 0);
    const expectedRetryAfter7d = Math.ceil((expectedNextDay - fakeNow) / 1000); // 32400s = 9h

    poolDb.db.prepare("UPDATE members SET rate_limit_5h_usd = 100.0 WHERE id = 'm1'").run();
    poolDb.db.prepare(
      `INSERT INTO invocations (idempotency_key, team_id, source_member_id, bridge_id, bridge_owner_id,
       provider, normalized_cost, lease_status, status, created_at)
       VALUES (?, 't1', 'm1', 'b1', 'm2', 'claude', 50.0, 'settled', 'success', ?)`
    ).run("k-7d", fakeNow);

    const result = checkRateLimit(poolDb.db, "m1", "t1", fakeNow);
    assert.equal(result.exceeded7d, true);
    assert.equal(result.retryAfterSeconds7d, expectedRetryAfter7d);
  });

  it("respects per-member override over team default", () => {
    poolDb.db.prepare("UPDATE members SET rate_limit_5h_usd = 10.0 WHERE id = 'm1'").run();
    const now = Date.now();
    poolDb.db.prepare(
      `INSERT INTO invocations (idempotency_key, team_id, source_member_id, bridge_id, bridge_owner_id,
       provider, normalized_cost, lease_status, status, created_at)
       VALUES (?, 't1', 'm1', 'b1', 'm2', 'claude', 6.0, 'settled', 'success', ?)`
    ).run("k3", now);
    const result = checkRateLimit(poolDb.db, "m1", "t1");
    assert.equal(result.allowed, true); // $6 < $10 override
  });

  it("excludes rolled_back from window calculation", () => {
    const now = Date.now();
    poolDb.db.prepare(
      `INSERT INTO invocations (idempotency_key, team_id, source_member_id, bridge_id, bridge_owner_id,
       provider, normalized_cost, lease_status, status, created_at)
       VALUES (?, 't1', 'm1', 'b1', 'm2', 'claude', 6.0, 'rolled_back', 'error', ?)`
    ).run("k4", now);
    const result = checkRateLimit(poolDb.db, "m1", "t1");
    assert.equal(result.allowed, true); // rolled_back not counted
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/plan-pool && npx tsx --test src/credit.test.ts`
Expected: FAIL — `checkRateLimit` not exported

**Step 3: Implement checkRateLimit**

`credit.ts` — 新增：

```ts
export interface RateLimitResult {
  allowed: boolean;
  /** Both exceeded flags — callers apply canonical priority to pick the primary reason. */
  exceeded5h: boolean;
  exceeded7d: boolean;
  retryAfterSeconds5h: number;
  retryAfterSeconds7d: number;
  used5hUsd: number;
  limit5hUsd: number;
  used7dUsd: number;
  limit7dUsd: number;
}

/**
 * Apply spec canonical effectiveStatus priority to pick the primary blocking reason.
 * Priority: rate_limited_weekly > rate_limited_5h (weekly is harder to recover from).
 */
export function rateLimitPrimaryReason(r: RateLimitResult): {
  reason: "RATE_LIMITED_5H" | "RATE_LIMITED_WEEKLY" | null;
  retryAfterSeconds: number;
} {
  if (r.exceeded7d) return { reason: "RATE_LIMITED_WEEKLY", retryAfterSeconds: r.retryAfterSeconds7d };
  if (r.exceeded5h) return { reason: "RATE_LIMITED_5H", retryAfterSeconds: r.retryAfterSeconds5h };
  return { reason: null, retryAfterSeconds: 0 };
}

/** @param nowMs — injectable clock for testing (defaults to Date.now()). */
export function checkRateLimit(
  db: Database.Database,
  memberId: string,
  teamId: string,
  nowMs?: number,
): RateLimitResult {
  const empty: RateLimitResult = { allowed: true, exceeded5h: false, exceeded7d: false, retryAfterSeconds5h: 0, retryAfterSeconds7d: 0, used5hUsd: 0, limit5hUsd: 0, used7dUsd: 0, limit7dUsd: 0 };

  // Resolve effective limits: member override > team default
  const row = db.prepare(`
    SELECT
      COALESCE(m.rate_limit_5h_usd, t.default_rate_limit_5h_usd) AS limit_5h,
      COALESCE(m.rate_limit_weekly_usd, t.default_rate_limit_weekly_usd) AS limit_7d
    FROM members m JOIN teams t ON m.team_id = t.id
    WHERE m.id = ? AND m.team_id = ?
  `).get(memberId, teamId) as { limit_5h: number | null; limit_7d: number | null } | undefined;

  if (!row) return empty;

  const now = nowMs ?? Date.now();

  // 5H bucket: floor(now_hour) - 4h — 含当前 UTC 小时共 5 个整点小时
  const nowHour = Math.floor(now / 3600_000) * 3600_000;
  const bucketStart5h = nowHour - 4 * 3600_000;

  // 7D bucket: floor(now_day) - 6d — 含当天共 7 个 UTC 自然天
  const nowDay = Math.floor(now / 86400_000) * 86400_000;
  const bucketStart7d = nowDay - 6 * 86400_000;

  const used5h = (db.prepare(
    `SELECT COALESCE(SUM(normalized_cost), 0) AS total FROM invocations
     WHERE source_member_id = ? AND created_at >= ?
     AND lease_status IN ('pending','settled','expired')`
  ).get(memberId, bucketStart5h) as { total: number }).total;

  const used7d = (db.prepare(
    `SELECT COALESCE(SUM(normalized_cost), 0) AS total FROM invocations
     WHERE source_member_id = ? AND created_at >= ?
     AND lease_status IN ('pending','settled','expired')`
  ).get(memberId, bucketStart7d) as { total: number }).total;

  const limit5h = row.limit_5h ?? Infinity;
  const limit7d = row.limit_7d ?? Infinity;

  const exceeded5h = limit5h !== Infinity && used5h >= limit5h;
  const exceeded7d = limit7d !== Infinity && used7d >= limit7d;

  // Retry-After: 距下一个桶边界（UTC 整点 / UTC 日 00:00）
  const retryAfterSeconds5h = exceeded5h ? Math.ceil(((nowHour + 3600_000) - now) / 1000) : 0;
  const retryAfterSeconds7d = exceeded7d ? Math.ceil(((nowDay + 86400_000) - now) / 1000) : 0;

  return {
    allowed: !exceeded5h && !exceeded7d,
    exceeded5h,
    exceeded7d,
    retryAfterSeconds5h,
    retryAfterSeconds7d,
    used5hUsd: used5h,
    limit5hUsd: limit5h === Infinity ? 0 : limit5h,
    used7dUsd: used7d,
    limit7dUsd: limit7d === Infinity ? 0 : limit7d,
  };
}
```

**Step 4: Run test to verify it passes**

Run: `cd packages/plan-pool && npx tsx --test src/credit.test.ts`
Expected: ALL PASS

**Step 5: Commit**

```bash
git add packages/plan-pool/src/credit.ts packages/plan-pool/src/credit.test.ts
git commit -m "feat(plan-pool): add checkRateLimit with UTC bucket windows (5H/7D)"
```

---

### Task 4: AuthContext refactor

**Files:**
- Modify: `packages/plan-pool/src/team.ts:50-55,190-216` (AuthResult → AuthContext)
- Test: `packages/plan-pool/src/team.test.ts`

**Step 1: Write the failing test**

```ts
describe("authenticateApiKey → AuthContext", () => {
  it("active member canInvoke/canReadSelf/canRegisterBridge", () => {
    const { teamId, memberId, apiKey } = tm.createTeam({
      name: "Lab", providerScope: "claude",
      ownerDisplayName: "Alice", ownerPublicKey: "pk-a",
    });
    const auth = tm.authenticateApiKey(apiKey);
    assert.ok(auth);
    assert.equal(auth.canReadSelf, true);
    assert.equal(auth.canAdmin, true); // owner
  });

  it("expired member canReadSelf but NOT canInvoke", () => {
    const { teamId, memberId, apiKey } = tm.createTeam({
      name: "Lab", providerScope: "claude",
      ownerDisplayName: "Alice", ownerPublicKey: "pk-a",
    });
    // Set expires_at to past
    poolDb.db.prepare("UPDATE members SET expires_at = ? WHERE id = ?")
      .run(Date.now() - 10_000, memberId);
    const auth = tm.authenticateApiKey(apiKey);
    assert.ok(auth);
    assert.equal(auth.canReadSelf, true);
    assert.equal(auth.canInvoke, false);
    assert.equal(auth.canRegisterBridge, false);
  });

  it("suspended member returns null (key revoked)", () => {
    const owner = tm.createTeam({
      name: "Lab", providerScope: "claude",
      ownerDisplayName: "Alice", ownerPublicKey: "pk-a",
    });
    // Create invite + join as member
    const { code } = tm.createInvite({
      teamId: owner.teamId, createdBy: owner.memberId, maxUses: 1, ttlHours: 24,
    });
    const joined = tm.joinTeam({ code, displayName: "Bob", publicKey: "pk-b" });
    tm.suspendMember(owner.teamId, joined.memberId);
    const auth = tm.authenticateApiKey(joined.apiKey);
    assert.equal(auth, null); // keys revoked on suspend
  });

  it("null expires_at means never expires (canInvoke=true)", () => {
    const { apiKey } = tm.createTeam({
      name: "Lab", providerScope: "claude",
      ownerDisplayName: "Alice", ownerPublicKey: "pk-a",
    });
    const auth = tm.authenticateApiKey(apiKey);
    assert.ok(auth);
    assert.equal(auth.canInvoke, true);
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/plan-pool && npx tsx --test src/team.test.ts`
Expected: FAIL — `canInvoke`, `canReadSelf` etc. don't exist on AuthResult

**Step 3: Implement AuthContext**

`team.ts` — 替换 `AuthResult`：

```ts
export interface AuthContext {
  teamId: string;
  memberId: string;
  providerScope: string;
  role: string;
  /** active + not expired (quota + rate limits checked by caller) */
  canInvoke: boolean;
  /** active + not expired */
  canRegisterBridge: boolean;
  /** active + role=owner */
  canAdmin: boolean;
  /** active (even if expired — preserves self-service read access) */
  canReadSelf: boolean;
  /** null = never expires */
  expiresAt: number | null;
}
```

修改 `authenticateApiKey` 查询，额外取 `m.expires_at`，计算权限：

```ts
authenticateApiKey(apiKey: string): AuthContext | null {
  const keyHash = sha256(apiKey);
  const row = this.db.prepare(`
    SELECT k.team_id, k.member_id, t.provider_scope, m.role, m.expires_at
    FROM api_keys k
    JOIN teams t ON k.team_id = t.id
    JOIN members m ON k.member_id = m.id
    WHERE k.key_hash = ? AND k.status = 'active' AND m.status = 'active'
  `).get(keyHash) as {
    team_id: string; member_id: string; provider_scope: string;
    role: string; expires_at: number | null;
  } | undefined;

  if (!row) return null;

  const now = Date.now();
  const expired = row.expires_at != null && row.expires_at <= now;

  return {
    teamId: row.team_id,
    memberId: row.member_id,
    providerScope: row.provider_scope,
    role: row.role,
    canInvoke: !expired, // quota + rate limits checked by caller
    canRegisterBridge: !expired,
    canAdmin: row.role === "owner",
    canReadSelf: true, // active status already confirmed by query
    expiresAt: row.expires_at,
  };
}
```

> 注意：旧的 `AuthResult` 接口保留为 alias `export type AuthResult = AuthContext;` 确保 api-gateway.ts 编译通过（后续 Task 7 再清理引用）。

**Step 4: Run test to verify it passes**

Run: `cd packages/plan-pool && npx tsx --test src/team.test.ts`
Expected: ALL PASS

**Step 5: Run full test suite to verify no regression**

Run: `cd packages/plan-pool && npx tsx --test src/**/*.test.ts`
Expected: ALL PASS

**Step 6: Commit**

```bash
git add packages/plan-pool/src/team.ts packages/plan-pool/src/team.test.ts
git commit -m "feat(plan-pool): refactor authenticateApiKey → AuthContext with canInvoke/canReadSelf"
```

---

### Task 5: PoolRouter.evictMemberBridges()

**Files:**
- Modify: `packages/plan-pool/src/pool.ts:70-76` (新增 evictMemberBridges)
- Test: `packages/plan-pool/src/pool.test.ts`

**Step 1: Write the failing test**

在 `pool.test.ts` 中添加：

```ts
describe("evictMemberBridges", () => {
  it("removes all bridges owned by the given member", () => {
    const router = new PoolRouter();
    router.addBridge(makeBridge({ id: "b1", ownerMemberId: "m1", teamId: "t1" }));
    router.addBridge(makeBridge({ id: "b2", ownerMemberId: "m1", teamId: "t1" }));
    router.addBridge(makeBridge({ id: "b3", ownerMemberId: "m2", teamId: "t1" }));
    const evicted = router.evictMemberBridges("m1");
    assert.equal(evicted, 2);
    assert.equal(router.getBridge("b1"), undefined);
    assert.equal(router.getBridge("b2"), undefined);
    assert.ok(router.getBridge("b3")); // m2 unaffected
  });

  it("returns 0 when member has no bridges", () => {
    const router = new PoolRouter();
    assert.equal(router.evictMemberBridges("nonexistent"), 0);
  });
});
```

> 注：`makeBridge` 是测试 helper，如 pool.test.ts 中已有类似的构造。如无则创建。

**Step 2: Run test to verify it fails**

Run: `cd packages/plan-pool && npx tsx --test src/pool.test.ts`
Expected: FAIL — `evictMemberBridges` not a function

**Step 3: Implement evictMemberBridges**

`pool.ts` — 在 `removeBridge` 后添加：

```ts
/** Remove all bridges owned by a member. Returns count evicted. */
evictMemberBridges(memberId: string): number {
  let count = 0;
  for (const [id, bridge] of this.bridges) {
    if (bridge.ownerMemberId === memberId) {
      this.bridges.delete(id);
      count++;
    }
  }
  return count;
}
```

**Step 4: Run test to verify it passes**

Run: `cd packages/plan-pool && npx tsx --test src/pool.test.ts`
Expected: ALL PASS

**Step 5: Commit**

```bash
git add packages/plan-pool/src/pool.ts packages/plan-pool/src/pool.test.ts
git commit -m "feat(plan-pool): add evictMemberBridges to PoolRouter (R1 bridge leak fix)"
```

---

### Task 6: TeamManager — new admin methods

**Files:**
- Modify: `packages/plan-pool/src/team.ts` (新增 setExpiry, setMemberRateLimit, updateTeamSettings, getTeamSettings, getMemberSelfInfo)
- Test: `packages/plan-pool/src/team.test.ts`

**Step 6a: setExpiry**

**Test:**
```ts
describe("setExpiry", () => {
  it("sets expires_at on a member", () => {
    const { teamId, memberId } = tm.createTeam({
      name: "Lab", providerScope: "claude",
      ownerDisplayName: "Alice", ownerPublicKey: "pk-a",
    });
    const future = Date.now() + 7 * 86400_000;
    tm.setExpiry(teamId, memberId, future);
    const row = poolDb.db.prepare("SELECT expires_at FROM members WHERE id = ?")
      .get(memberId) as { expires_at: number };
    assert.equal(row.expires_at, future);
  });

  it("clears expires_at with null (never expires)", () => {
    const { teamId, memberId } = tm.createTeam({
      name: "Lab", providerScope: "claude",
      ownerDisplayName: "Alice", ownerPublicKey: "pk-a",
    });
    tm.setExpiry(teamId, memberId, Date.now() + 1000);
    tm.setExpiry(teamId, memberId, null);
    const row = poolDb.db.prepare("SELECT expires_at FROM members WHERE id = ?")
      .get(memberId) as { expires_at: number | null };
    assert.equal(row.expires_at, null);
  });
});
```

**Implementation:**
```ts
setExpiry(teamId: string, memberId: string, expiresAt: number | null): void {
  const result = this.db
    .prepare("UPDATE members SET expires_at = ? WHERE id = ? AND team_id = ?")
    .run(expiresAt, memberId, teamId);
  if (result.changes === 0) throw new Error("member not found");
}
```

**Step 6b: setMemberRateLimit**

**Test:**
```ts
describe("setMemberRateLimit", () => {
  it("sets per-member rate limit overrides", () => {
    const { teamId, memberId } = tm.createTeam({
      name: "Lab", providerScope: "claude",
      ownerDisplayName: "Alice", ownerPublicKey: "pk-a",
    });
    tm.setMemberRateLimit(teamId, memberId, { rateLimit5hUsd: 10.0, rateLimitWeeklyUsd: 80.0 });
    const row = poolDb.db.prepare("SELECT rate_limit_5h_usd, rate_limit_weekly_usd FROM members WHERE id = ?")
      .get(memberId) as { rate_limit_5h_usd: number; rate_limit_weekly_usd: number };
    assert.equal(row.rate_limit_5h_usd, 10.0);
    assert.equal(row.rate_limit_weekly_usd, 80.0);
  });

  it("clears overrides with null (use team defaults)", () => {
    const { teamId, memberId } = tm.createTeam({
      name: "Lab", providerScope: "claude",
      ownerDisplayName: "Alice", ownerPublicKey: "pk-a",
    });
    tm.setMemberRateLimit(teamId, memberId, { rateLimit5hUsd: null, rateLimitWeeklyUsd: null });
    const row = poolDb.db.prepare("SELECT rate_limit_5h_usd, rate_limit_weekly_usd FROM members WHERE id = ?")
      .get(memberId) as { rate_limit_5h_usd: number | null; rate_limit_weekly_usd: number | null };
    assert.equal(row.rate_limit_5h_usd, null);
    assert.equal(row.rate_limit_weekly_usd, null);
  });
});
```

**Implementation:**
```ts
setMemberRateLimit(teamId: string, memberId: string, limits: { rateLimit5hUsd: number | null; rateLimitWeeklyUsd: number | null }): void {
  const result = this.db
    .prepare("UPDATE members SET rate_limit_5h_usd = ?, rate_limit_weekly_usd = ? WHERE id = ? AND team_id = ?")
    .run(limits.rateLimit5hUsd, limits.rateLimitWeeklyUsd, memberId, teamId);
  if (result.changes === 0) throw new Error("member not found");
}
```

**Step 6c: updateTeamSettings + getTeamSettings**

**Test:**
```ts
describe("updateTeamSettings", () => {
  it("updates team default rate limits", () => {
    const { teamId } = tm.createTeam({
      name: "Lab", providerScope: "claude",
      ownerDisplayName: "Alice", ownerPublicKey: "pk-a",
    });
    tm.updateTeamSettings(teamId, {
      defaultBorrowLimitUsd: 50.0,
      defaultRateLimit5hUsd: 8.0,
      defaultRateLimitWeeklyUsd: 60.0,
    });
    const settings = tm.getTeamSettings(teamId);
    assert.equal(settings.defaultBorrowLimitUsd, 50.0);
    assert.equal(settings.defaultRateLimit5hUsd, 8.0);
    assert.equal(settings.defaultRateLimitWeeklyUsd, 60.0);
  });
});
```

**Implementation:**
```ts
updateTeamSettings(teamId: string, settings: {
  defaultBorrowLimitUsd?: number;
  defaultRateLimit5hUsd?: number | null;
  defaultRateLimitWeeklyUsd?: number | null;
}): void {
  const updates: string[] = [];
  const params: unknown[] = [];
  if (settings.defaultBorrowLimitUsd !== undefined) {
    updates.push("default_borrow_limit_usd = ?");
    params.push(settings.defaultBorrowLimitUsd);
  }
  if (settings.defaultRateLimit5hUsd !== undefined) {
    updates.push("default_rate_limit_5h_usd = ?");
    params.push(settings.defaultRateLimit5hUsd);
  }
  if (settings.defaultRateLimitWeeklyUsd !== undefined) {
    updates.push("default_rate_limit_weekly_usd = ?");
    params.push(settings.defaultRateLimitWeeklyUsd);
  }
  if (updates.length === 0) return;
  params.push(teamId);
  const result = this.db
    .prepare(`UPDATE teams SET ${updates.join(", ")} WHERE id = ?`)
    .run(...params);
  if (result.changes === 0) throw new Error("team not found");
}

getTeamSettings(teamId: string): {
  defaultBorrowLimitUsd: number;
  defaultRateLimit5hUsd: number | null;
  defaultRateLimitWeeklyUsd: number | null;
} {
  const row = this.db.prepare(
    `SELECT default_borrow_limit_usd, default_rate_limit_5h_usd, default_rate_limit_weekly_usd
     FROM teams WHERE id = ?`
  ).get(teamId) as {
    default_borrow_limit_usd: number;
    default_rate_limit_5h_usd: number | null;
    default_rate_limit_weekly_usd: number | null;
  } | undefined;
  if (!row) throw new Error("team not found");
  return {
    defaultBorrowLimitUsd: row.default_borrow_limit_usd,
    defaultRateLimit5hUsd: row.default_rate_limit_5h_usd,
    defaultRateLimitWeeklyUsd: row.default_rate_limit_weekly_usd,
  };
}
```

**Step 6d: getMemberSelfInfo**

**Test:**
```ts
describe("getMemberSelfInfo", () => {
  it("returns effectiveStatus=active for healthy member", () => {
    const { teamId, memberId } = tm.createTeam({
      name: "Lab", providerScope: "claude",
      ownerDisplayName: "Alice", ownerPublicKey: "pk-a",
    });
    const info = tm.getMemberSelfInfo(memberId);
    assert.ok(info);
    assert.equal(info.effectiveStatus, "active");
    assert.equal(info.teamName, "Lab");
  });

  it("returns effectiveStatus=expired for past expires_at", () => {
    const { teamId, memberId } = tm.createTeam({
      name: "Lab", providerScope: "claude",
      ownerDisplayName: "Alice", ownerPublicKey: "pk-a",
    });
    poolDb.db.prepare("UPDATE members SET expires_at = ? WHERE id = ?")
      .run(Date.now() - 10_000, memberId);
    const info = tm.getMemberSelfInfo(memberId);
    assert.ok(info);
    assert.equal(info.effectiveStatus, "expired");
  });
});
```

**Implementation:**
```ts
getMemberSelfInfo(memberId: string): {
  memberId: string;
  displayName: string;
  effectiveStatus: string;
  blockingReason: string | null;
  nextAvailableAt: number | null;
  retryAfterSeconds: number | null;
  quotaPercent: number;
  rateLimitPercent5h: number;
  rateLimitPercent7d: number;
  expiresAt: number | null;
  credentialStatus: string;
  credentialLastRotatedAt: number | null;
  teamName: string;
} | null {
  const row = this.db.prepare(`
    SELECT m.id, m.display_name, m.team_id, m.status, m.expires_at,
           t.name AS team_name,
           COALESCE(mb.available_usd, 0) AS available,
           COALESCE(mb.borrow_limit_usd, 30.0) AS borrow_limit,
           COALESCE(mb.contributed_usd, 0) AS contributed,
           COALESCE(mb.consumed_usd, 0) AS consumed,
           (SELECT MAX(k.created_at) FROM api_keys k WHERE k.member_id = m.id AND k.status = 'active') AS key_created_at
    FROM members m
    JOIN teams t ON m.team_id = t.id
    LEFT JOIN member_balances mb ON mb.member_id = m.id
    WHERE m.id = ? AND m.status = 'active'
  `).get(memberId) as Record<string, unknown> | undefined;

  if (!row) return null;

  const now = Date.now();
  const expiresAt = row.expires_at as number | null;
  const expired = expiresAt != null && expiresAt <= now;
  const available = row.available as number;
  const borrowLimit = row.borrow_limit as number;
  const totalQuota = (row.contributed as number) + borrowLimit;
  const quotaPercent = totalQuota > 0 ? Math.round(((totalQuota - (row.consumed as number)) / totalQuota) * 100) : 0;

  // Rate limit check
  const rateLimit = checkRateLimit(this.db, memberId, row.team_id as string);
  const rlPrimary = rateLimitPrimaryReason(rateLimit);

  // effectiveStatus priority: suspended > expired > quota_exhausted > rate_limited_weekly > rate_limited_5h > active
  // (suspended already filtered by query — m.status = 'active')
  let effectiveStatus = "active";
  let blockingReason: string | null = null;
  let nextAvailableAt: number | null = null;
  let retryAfterSeconds: number | null = null;

  if (expired) {
    effectiveStatus = "expired";
    blockingReason = "Membership expired";
    nextAvailableAt = null; // owner must extend
  } else if (available <= 0) {
    effectiveStatus = "quota_exhausted";
    blockingReason = "Quota exhausted";
  } else if (rlPrimary.reason === "RATE_LIMITED_WEEKLY") {
    effectiveStatus = "rate_limited_weekly";
    blockingReason = "Weekly rate limit exceeded";
    retryAfterSeconds = rlPrimary.retryAfterSeconds;
    nextAvailableAt = now + rlPrimary.retryAfterSeconds * 1000;
  } else if (rlPrimary.reason === "RATE_LIMITED_5H") {
    effectiveStatus = "rate_limited_5h";
    blockingReason = "5-hour rate limit exceeded";
    retryAfterSeconds = rlPrimary.retryAfterSeconds;
    nextAvailableAt = now + rlPrimary.retryAfterSeconds * 1000;
  }

  // Rate limit percentages
  const rl5hPct = rateLimit.limit5hUsd > 0 ? Math.round((rateLimit.used5hUsd / rateLimit.limit5hUsd) * 100) : 0;
  const rl7dPct = rateLimit.limit7dUsd > 0 ? Math.round((rateLimit.used7dUsd / rateLimit.limit7dUsd) * 100) : 0;

  return {
    memberId: row.id as string,
    displayName: row.display_name as string,
    effectiveStatus,
    blockingReason,
    nextAvailableAt,
    retryAfterSeconds,
    quotaPercent: Math.max(0, Math.min(100, quotaPercent)),
    rateLimitPercent5h: Math.min(100, rl5hPct),
    rateLimitPercent7d: Math.min(100, rl7dPct),
    expiresAt,
    credentialStatus: "active",
    credentialLastRotatedAt: row.key_created_at as number | null,
    teamName: row.team_name as string,
  };
}
```

> 注：`getMemberSelfInfo` 内调用 `checkRateLimit` + `rateLimitPrimaryReason`，需要 `import { checkRateLimit, rateLimitPrimaryReason } from "./credit.js";`

**Run tests + commit:**

Run: `cd packages/plan-pool && npx tsx --test src/team.test.ts`
Expected: ALL PASS

```bash
git add packages/plan-pool/src/team.ts packages/plan-pool/src/team.test.ts
git commit -m "feat(plan-pool): add setExpiry, setMemberRateLimit, updateTeamSettings, getMemberSelfInfo"
```

---

## Phase C: API Layer

### Task 7: Wire auth checks into handlePoolRequest + bridge register

**Files:**
- Modify: `packages/plan-pool/src/api-gateway.ts:69-73,76-177,326-389,458-475`
- Test: `packages/plan-pool/src/api-gateway.test.ts`

**Step 1: Update authenticate helper**

Replace auth helper usage across all routes. The `authenticate()` function already returns `AuthContext` (which has `canInvoke`/`canAdmin`/`canReadSelf`/`canRegisterBridge`).

Changes:
1. `handlePoolRequest` — 在 quota check 之前加 `canInvoke` + `checkRateLimit` 检查：

```ts
if (!auth.canInvoke) {
  if (auth.expiresAt != null && auth.expiresAt <= Date.now()) {
    return { error: "EXPIRED", statusCode: 403 };
  }
  return { error: "FORBIDDEN", statusCode: 403 };
}

// Rate limit check (same rateLimitPrimaryReason as getMemberSelfInfo — single canonical priority)
const rateLimit = checkRateLimit(db, auth.memberId, auth.teamId);
if (!rateLimit.allowed) {
  const primary = rateLimitPrimaryReason(rateLimit);
  return {
    error: primary.reason!,
    statusCode: 429,
    retryAfterSeconds: primary.retryAfterSeconds,
  };
}
```

2. Bridge register route — check `canRegisterBridge`:

```ts
if (!auth.canRegisterBridge) {
  reply.status(403);
  return { error: "expired or suspended — cannot register bridge" };
}
```

3. Admin routes — check `canAdmin` instead of `role !== 'owner'`:

```ts
if (!auth.canAdmin) {
  reply.status(403);
  return { error: "admin access required" };
}
```

4. `handlePoolRequest` 的 429 返回需要加 `Retry-After` header：

```ts
if ('retryAfterSeconds' in result && result.retryAfterSeconds) {
  reply.header('Retry-After', String(result.retryAfterSeconds));
}
```

**Step 2: Wire suspendMember to evict bridges**

在 `PUT /v1/admin/members/:id/suspend` 路由中，`suspendMember` 调用后添加：

```ts
if (router) {
  const evicted = router.evictMemberBridges(req.params.id);
  if (evicted > 0) console.log(`[plan-pool] evicted ${evicted} bridge(s) for suspended member ${req.params.id}`);
}
```

**Step 3: Write integration test**

在 `api-gateway.test.ts` 中添加：

```ts
it("returns 403 EXPIRED for expired member invoke", async () => {
  // Set member as expired
  db.prepare("UPDATE members SET expires_at = ? WHERE id = ?")
    .run(Date.now() - 10_000, memberId);
  const res = await app.inject({
    method: "POST", url: "/v1/messages",
    headers: { authorization: `Bearer ${apiKey}`, "content-type": "application/json" },
    body: JSON.stringify({ messages: [{ role: "user", content: "hi" }] }),
  });
  assert.equal(res.statusCode, 403);
  const body = JSON.parse(res.body);
  assert.equal(body.error, "EXPIRED");
});
```

**Step 4: Run tests**

Run: `cd packages/plan-pool && npx tsx --test src/api-gateway.test.ts`
Expected: ALL PASS

**Step 5: Commit**

```bash
git add packages/plan-pool/src/api-gateway.ts packages/plan-pool/src/api-gateway.test.ts
git commit -m "feat(plan-pool): wire AuthContext + rate limits + bridge eviction into request flow"
```

---

### Task 8: Admin API — new endpoints

**Files:**
- Modify: `packages/plan-pool/src/api-gateway.ts` (新增 routes)
- Test: `packages/plan-pool/src/api-gateway.test.ts`

**8a: PUT /v1/admin/members/:id/expiry**

```ts
app.put<{ Params: { id: string } }>("/v1/admin/members/:id/expiry", async (req, reply) => {
  const auth = authenticate(req);
  if (!auth?.canAdmin) { reply.status(auth ? 403 : 401); return { error: auth ? "admin access required" : "unauthorized" }; }
  const body = req.body as { expiresAt?: number | null };
  try {
    teamManager.setExpiry(auth.teamId, req.params.id, body?.expiresAt ?? null);
    // Evict bridges if now expired
    if (body?.expiresAt != null && body.expiresAt <= Date.now() && router) {
      router.evictMemberBridges(req.params.id);
    }
    return { updated: true };
  } catch (err) {
    reply.status(400);
    return { error: err instanceof Error ? err.message : "update failed" };
  }
});
```

**8b: PUT /v1/admin/members/:id/rate-limit**

```ts
app.put<{ Params: { id: string } }>("/v1/admin/members/:id/rate-limit", async (req, reply) => {
  const auth = authenticate(req);
  if (!auth?.canAdmin) { reply.status(auth ? 403 : 401); return { error: auth ? "admin access required" : "unauthorized" }; }
  const body = req.body as { rateLimit5hUsd?: number | null; rateLimitWeeklyUsd?: number | null };
  try {
    teamManager.setMemberRateLimit(auth.teamId, req.params.id, {
      rateLimit5hUsd: body?.rateLimit5hUsd ?? null,
      rateLimitWeeklyUsd: body?.rateLimitWeeklyUsd ?? null,
    });
    return { updated: true };
  } catch (err) {
    reply.status(400);
    return { error: err instanceof Error ? err.message : "update failed" };
  }
});
```

**8c: PUT /v1/admin/team/settings**

```ts
app.put("/v1/admin/team/settings", async (req, reply) => {
  const auth = authenticate(req);
  if (!auth?.canAdmin) { reply.status(auth ? 403 : 401); return { error: auth ? "admin access required" : "unauthorized" }; }
  const body = req.body as {
    defaultBorrowLimitUsd?: number;
    defaultRateLimit5hUsd?: number | null;
    defaultRateLimitWeeklyUsd?: number | null;
  };
  try {
    teamManager.updateTeamSettings(auth.teamId, body ?? {});
    return { updated: true };
  } catch (err) {
    reply.status(400);
    return { error: err instanceof Error ? err.message : "update failed" };
  }
});
```

**8d: GET /v1/admin/team/settings**

```ts
app.get("/v1/admin/team/settings", async (req, reply) => {
  const auth = authenticate(req);
  if (!auth?.canAdmin) { reply.status(auth ? 403 : 401); return { error: auth ? "admin access required" : "unauthorized" }; }
  try {
    return teamManager.getTeamSettings(auth.teamId);
  } catch (err) {
    reply.status(400);
    return { error: err instanceof Error ? err.message : "fetch failed" };
  }
});
```

**8e: Update GET /v1/admin/members to include expires_at + rate limits**

在 `getTeamMembers` 返回的 members 中增加 `expiresAt`、`rateLimit5hUsd`、`rateLimitWeeklyUsd` 字段。修改 `team.ts` 中的 `getTeamMembers` 查询：

```sql
SELECT m.id AS memberId, m.display_name AS displayName, m.role, m.status,
  m.expires_at AS expiresAt,
  m.rate_limit_5h_usd AS rateLimit5hUsd,
  m.rate_limit_weekly_usd AS rateLimitWeeklyUsd,
  COALESCE(mb.borrow_limit_usd, 30.0) AS borrowLimit,
  ...
```

**Tests:**

```ts
it("PUT /v1/admin/members/:id/expiry sets expiry", async () => {
  const future = Date.now() + 86400_000;
  const res = await app.inject({
    method: "PUT", url: `/v1/admin/members/${memberId}/expiry`,
    headers: { authorization: `Bearer ${ownerApiKey}`, "content-type": "application/json" },
    body: JSON.stringify({ expiresAt: future }),
  });
  assert.equal(res.statusCode, 200);
});

it("PUT /v1/admin/team/settings updates defaults", async () => {
  const res = await app.inject({
    method: "PUT", url: "/v1/admin/team/settings",
    headers: { authorization: `Bearer ${ownerApiKey}`, "content-type": "application/json" },
    body: JSON.stringify({ defaultRateLimit5hUsd: 8.0, defaultRateLimitWeeklyUsd: 60.0 }),
  });
  assert.equal(res.statusCode, 200);
});
```

**Run + Commit:**

Run: `cd packages/plan-pool && npx tsx --test src/api-gateway.test.ts`

```bash
git add packages/plan-pool/src/api-gateway.ts packages/plan-pool/src/team.ts packages/plan-pool/src/api-gateway.test.ts
git commit -m "feat(plan-pool): add admin APIs for expiry, rate limits, team settings"
```

---

### Task 9: Member self-service API

**Files:**
- Modify: `packages/plan-pool/src/api-gateway.ts` (新增 /v1/me routes)
- Test: `packages/plan-pool/src/api-gateway.test.ts`

**9a: GET /v1/me**

```ts
app.get("/v1/me", async (req, reply) => {
  const auth = authenticate(req);
  if (!auth) { reply.status(401); return { error: "unauthorized" }; }
  if (!auth.canReadSelf) { reply.status(403); return { error: "access denied" }; }
  const info = teamManager.getMemberSelfInfo(auth.memberId);
  if (!info) { reply.status(404); return { error: "member not found" }; }
  return info;
});
```

**9b: GET /v1/me/usage**

```ts
app.get("/v1/me/usage", async (req, reply) => {
  const auth = authenticate(req);
  if (!auth?.canReadSelf) { reply.status(auth ? 403 : 401); return { error: auth ? "access denied" : "unauthorized" }; }
  const rows = db.prepare(`
    SELECT observed_model AS model,
      CAST(created_at / 86400000 AS INTEGER) * 86400000 AS day,
      SUM(normalized_cost) AS totalCost,
      SUM(input_tokens) AS totalInputTokens,
      SUM(output_tokens) AS totalOutputTokens,
      COUNT(*) AS requestCount
    FROM invocations
    WHERE source_member_id = ? AND created_at >= ?
    AND lease_status IN ('settled','expired')
    GROUP BY model, day
    ORDER BY day DESC
  `).all(auth.memberId, Date.now() - 7 * 86400_000);
  return { usage: rows };
});
```

**9c: GET /v1/me/invocations**

```ts
app.get("/v1/me/invocations", async (req, reply) => {
  const auth = authenticate(req);
  if (!auth?.canReadSelf) { reply.status(auth ? 403 : 401); return { error: auth ? "access denied" : "unauthorized" }; }
  const query = req.query as { limit?: string };
  const limit = Math.min(Math.max(parseInt(query.limit ?? "50") || 50, 1), 200);
  const rows = db.prepare(`
    SELECT observed_model AS model, provider, input_tokens AS inputTokens,
      output_tokens AS outputTokens, normalized_cost AS normalizedCost,
      status, created_at AS createdAt
    FROM invocations
    WHERE source_member_id = ?
    ORDER BY created_at DESC LIMIT ?
  `).all(auth.memberId, limit);
  return { invocations: rows };
});
```

**Tests:**

```ts
it("GET /v1/me returns member info with effectiveStatus", async () => {
  const res = await app.inject({
    method: "GET", url: "/v1/me",
    headers: { authorization: `Bearer ${memberApiKey}` },
  });
  assert.equal(res.statusCode, 200);
  const body = JSON.parse(res.body);
  assert.equal(body.effectiveStatus, "active");
  assert.ok("quotaPercent" in body);
  assert.ok("rateLimitPercent5h" in body);
});

it("GET /v1/me works for expired member (canReadSelf)", async () => {
  db.prepare("UPDATE members SET expires_at = ? WHERE id = ?")
    .run(Date.now() - 10_000, memberId);
  const res = await app.inject({
    method: "GET", url: "/v1/me",
    headers: { authorization: `Bearer ${memberApiKey}` },
  });
  assert.equal(res.statusCode, 200);
  const body = JSON.parse(res.body);
  assert.equal(body.effectiveStatus, "expired");
});
```

**Run + Commit:**

Run: `cd packages/plan-pool && npx tsx --test src/api-gateway.test.ts`

```bash
git add packages/plan-pool/src/api-gateway.ts packages/plan-pool/src/api-gateway.test.ts
git commit -m "feat(plan-pool): add /v1/me, /v1/me/usage, /v1/me/invocations member self-service APIs"
```

---

## Phase D: Frontend

### Task 10: Admin UI — Team Settings + 1200px + dialog modal

**Files:**
- Modify: `packages/plan-pool/public/admin.html`
- Test: Chrome 手动验证

**Step 1: Layout changes**

- `.container` max-width: `800px` → `1200px`
- Members table 增加列：Expires、Rate 5H、Rate 7D
- 移除 `prompt()` 调用，替换为 `<dialog>` 模态框

**Step 2: Team Settings card（邀请码区块下方）**

在 Generate Invite card 和 Members card 之间插入：

```html
<div class="card">
  <h2>Team Settings</h2>
  <div class="row">
    <div>
      <label>Default Borrow Limit (USD)</label>
      <input type="number" id="ts-borrow" step="1" style="width:120px">
    </div>
    <div>
      <label>Default 5H Rate Limit (USD)</label>
      <input type="number" id="ts-5h" step="0.5" style="width:120px">
    </div>
    <div>
      <label>Default 7D Rate Limit (USD)</label>
      <input type="number" id="ts-7d" step="1" style="width:120px">
    </div>
    <div>
      <button onclick="saveTeamSettings()">Save</button>
    </div>
  </div>
  <div id="ts-msg" class="hidden"></div>
</div>
```

JS：

```js
async function loadTeamSettings() {
  const res = await fetch(BASE + '/v1/admin/team/settings', {
    headers: { 'Authorization': 'Bearer ' + apiKey },
  });
  if (!res.ok) return;
  const data = await res.json();
  document.getElementById('ts-borrow').value = data.defaultBorrowLimitUsd ?? 30;
  document.getElementById('ts-5h').value = data.defaultRateLimit5hUsd ?? 5;
  document.getElementById('ts-7d').value = data.defaultRateLimitWeeklyUsd ?? 40;
}

async function saveTeamSettings() {
  const body = {
    defaultBorrowLimitUsd: parseFloat(document.getElementById('ts-borrow').value),
    defaultRateLimit5hUsd: parseFloat(document.getElementById('ts-5h').value),
    defaultRateLimitWeeklyUsd: parseFloat(document.getElementById('ts-7d').value),
  };
  const res = await fetch(BASE + '/v1/admin/team/settings', {
    method: 'PUT',
    headers: { 'Authorization': 'Bearer ' + apiKey, 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });
  const msg = document.getElementById('ts-msg');
  msg.textContent = res.ok ? 'Saved!' : 'Error saving';
  msg.className = res.ok ? 'msg ok' : 'msg err';
}
```

**Step 3: Member edit dialog**

替换 `editLimit()` 和 `suspendMember()` 为统一的 `<dialog>` 模态框：

```html
<dialog id="member-dialog">
  <h3>Edit Member: <span id="md-name"></span></h3>
  <form method="dialog">
    <!-- Access Group -->
    <fieldset>
      <legend>Access</legend>
      <label>Expires At</label>
      <input type="datetime-local" id="md-expires">
      <label><input type="checkbox" id="md-never-expires"> Never expires</label>
    </fieldset>
    <!-- Budget Group -->
    <fieldset>
      <legend>Budget</legend>
      <label>Borrow Limit (USD)</label>
      <input type="number" id="md-borrow" step="1">
    </fieldset>
    <!-- Rate Windows Group -->
    <fieldset>
      <legend>Rate Windows</legend>
      <label><input type="checkbox" id="md-use-defaults" checked> Use Team Defaults</label>
      <div id="md-rate-custom" class="hidden">
        <label>5H Limit (USD)</label>
        <input type="number" id="md-5h" step="0.5">
        <label>7D Limit (USD)</label>
        <input type="number" id="md-7d" step="1">
      </div>
    </fieldset>
    <div style="margin-top:1rem;display:flex;gap:0.5rem;justify-content:flex-end">
      <button type="button" onclick="document.getElementById('member-dialog').close()">Cancel</button>
      <button type="button" onclick="saveMemberEdit()">Save</button>
    </div>
  </form>
</dialog>
```

JS：

```js
let editingMemberId = null;

function openMemberEdit(m) {
  editingMemberId = m.memberId;
  document.getElementById('md-name').textContent = m.displayName;
  // Expires
  const neverExpires = !m.expiresAt;
  document.getElementById('md-never-expires').checked = neverExpires;
  document.getElementById('md-expires').disabled = neverExpires;
  if (m.expiresAt) {
    document.getElementById('md-expires').value = new Date(m.expiresAt).toISOString().slice(0, 16);
  } else {
    document.getElementById('md-expires').value = '';
  }
  // Budget
  document.getElementById('md-borrow').value = m.borrowLimit || 30;
  // Rate limits
  const useDefaults = m.rateLimit5hUsd == null && m.rateLimitWeeklyUsd == null;
  document.getElementById('md-use-defaults').checked = useDefaults;
  document.getElementById('md-rate-custom').classList.toggle('hidden', useDefaults);
  document.getElementById('md-5h').value = m.rateLimit5hUsd ?? '';
  document.getElementById('md-7d').value = m.rateLimitWeeklyUsd ?? '';
  document.getElementById('member-dialog').showModal();
}

document.getElementById('md-never-expires').addEventListener('change', (e) => {
  document.getElementById('md-expires').disabled = e.target.checked;
});
document.getElementById('md-use-defaults').addEventListener('change', (e) => {
  document.getElementById('md-rate-custom').classList.toggle('hidden', e.target.checked);
});

async function saveMemberEdit() {
  const id = editingMemberId;
  // Expiry
  const neverExpires = document.getElementById('md-never-expires').checked;
  const expiresVal = document.getElementById('md-expires').value;
  const expiresAt = neverExpires ? null : (expiresVal ? new Date(expiresVal).getTime() : null);
  await fetch(BASE + '/v1/admin/members/' + id + '/expiry', {
    method: 'PUT',
    headers: { 'Authorization': 'Bearer ' + apiKey, 'Content-Type': 'application/json' },
    body: JSON.stringify({ expiresAt }),
  });
  // Borrow limit
  const limitUsd = parseFloat(document.getElementById('md-borrow').value);
  await fetch(BASE + '/v1/admin/members/' + id + '/borrow-limit', {
    method: 'PUT',
    headers: { 'Authorization': 'Bearer ' + apiKey, 'Content-Type': 'application/json' },
    body: JSON.stringify({ limitUsd }),
  });
  // Rate limits
  const useDefaults = document.getElementById('md-use-defaults').checked;
  await fetch(BASE + '/v1/admin/members/' + id + '/rate-limit', {
    method: 'PUT',
    headers: { 'Authorization': 'Bearer ' + apiKey, 'Content-Type': 'application/json' },
    body: JSON.stringify({
      rateLimit5hUsd: useDefaults ? null : (parseFloat(document.getElementById('md-5h').value) || null),
      rateLimitWeeklyUsd: useDefaults ? null : (parseFloat(document.getElementById('md-7d').value) || null),
    }),
  });
  document.getElementById('member-dialog').close();
  loadMembers();
}
```

**Step 4: Members table — 增加 Expires 列 + Edit 按钮**

```js
function renderMembers(members) {
  // ... header includes: Name, Role, Status, Expires, Contributed, Consumed, Balance, Actions
  // Actions: Edit button (openMemberEdit) + Suspend button
  // Expires column: format date or "Never"
}
```

**Step 5: Auto-load team settings on auth**

在 `tryAuth()` 成功后调用 `loadTeamSettings()`。

**Step 6: 验证**

- Chrome 打开 `/admin`
- 登录后看到 Team Settings 卡片
- 点 Edit 弹出 `<dialog>` modal
- 设置 expires_at + rate limits
- 确认数据持久化

**Commit:**

```bash
git add packages/plan-pool/public/admin.html
git commit -m "feat(plan-pool): admin UI — team settings card + member edit dialog + 1200px layout"
```

---

### Task 11: Member UI — member.html

**Files:**
- Create: `packages/plan-pool/public/member.html`
- Modify: `packages/plan-pool/src/api-gateway.ts` (添加 /member 路由)
- Modify: `packages/plan-pool/public/index.html` (添加 /member 链接)

**Step 1: 添加 /member 静态路由**

`api-gateway.ts` 的 static pages 区域：

```ts
app.get("/member", async (_req, reply) => serveFile("member.html", reply));
```

**Step 2: 创建 member.html**

页面结构（参考 spec 信息层次设计）：

```
┌─────────────────────────────────────────────────┐
│ Auth gate: API key input + Login button         │
├─────────────────────────────────────────────────┤
│ L1 Hero Card: effectiveStatus + blockingReason  │
│   大进度条 = 最紧迫限制（木桶效应）              │
│   + 预计恢复时间                                 │
├────────┬────────┬───────────────────────────────┤
│ L2 小卡 │ 小卡   │ 小卡                         │
│ 总额度%  │ 5H%   │ 7D%                          │
├─────────────────────────────────────────────────┤
│ L3 Credential: status + last rotated + docs link│
├─────────────────────────────────────────────────┤
│ L4 Usage trends: 7-day CSS bar chart            │
├─────────────────────────────────────────────────┤
│ L5 Recent calls: last 50 invocations            │
└─────────────────────────────────────────────────┘
```

关键设计：
- Auth gate 用 sessionStorage 存 API key，登录后调用 `GET /v1/me`
- Hero Card 根据 `effectiveStatus` 显示颜色（active=green, expired=red, rate_limited=yellow, quota_exhausted=orange）
- 进度条显示最紧迫限制（`Math.max(quotaPercent, rateLimitPercent5h, rateLimitPercent7d)` 中最高使用率的那个）
- L2 三个小卡独立展示 quotaPercent / rateLimitPercent5h / rateLimitPercent7d
- L4 用 CSS 柱状图（div 高度按比例），数据来自 `GET /v1/me/usage`
- L5 调用列表来自 `GET /v1/me/invocations`
- 所有百分比展示，不暴露金额

**Step 3: index.html — 添加 Member Dashboard 链接**

在 Resources card 中添加：

```html
<p><a href="/member">Member Dashboard</a> (view your usage & limits)</p>
```

**Step 4: 验证**

- Chrome 打开 `/member`
- 用成员 API key 登录
- 确认 Hero Card、小卡、credential、用量趋势、调用列表均正确渲染
- 模拟 expired / rate limited 场景确认状态展示

**Commit:**

```bash
git add packages/plan-pool/public/member.html packages/plan-pool/public/index.html packages/plan-pool/src/api-gateway.ts
git commit -m "feat(plan-pool): add member self-service dashboard (member.html)"
```

---

## Phase E: Integration + Lint + Final Verification

### Task 12: Full integration test + lint

**Step 1: Run full test suite**

```bash
cd packages/plan-pool && npx tsx --test src/**/*.test.ts
```

Expected: ALL PASS

**Step 2: Type check**

```bash
cd packages/plan-pool && npx tsc --noEmit
```

Expected: no errors

**Step 3: Build**

```bash
cd packages/plan-pool && npm run build
```

Expected: clean build

**Step 4: 端到端验证 checklist**

用 Chrome 测试以下场景：

1. [ ] Owner 登录 admin，看到 Team Settings 卡片（5H=$5, 7D=$40）
2. [ ] Owner 编辑成员：设置 expires_at = 明天，保存成功
3. [ ] Owner 编辑成员：设置自定义 rate limit（5H=$10），保存成功
4. [ ] Owner 编辑成员：勾选 "Use Team Defaults"，保存后 rate_limit 列显示 "-"
5. [ ] Member 登录 `/member`，看到 Hero Card 状态 = active (green)
6. [ ] Member 看到三个小卡（额度%、5H%、7D%）
7. [ ] 模拟过期：owner 设置 expires_at = 过去时间 → member 刷新看到 expired (red)
8. [ ] 过期 member 调用 /v1/messages → 403 EXPIRED
9. [ ] 过期 member 仍可访问 /v1/me → 200，effectiveStatus=expired
10. [ ] 模拟 5H 限速：大量调用后 → member 看到 rate_limited_5h (yellow)
11. [ ] 5H 限速调用 /v1/messages → 429 + Retry-After header（Retry-After ≤ 3600）
12. [ ] 模拟 7D 限速（设 7D=$1 低阈值）→ member 看到 rate_limited_weekly (yellow)
13. [ ] 7D 限速调用 /v1/messages → 429 + Retry-After header（Retry-After ≤ 86400）
14. [ ] 双超限场景（5H+7D 同时超）→ 429 报 RATE_LIMITED_WEEKLY（canonical priority: weekly > 5h）

**Commit:**

```bash
git add -A
git commit -m "feat(plan-pool): F018 P0 — admin enhancement + member self-service (complete)"
```

---

## Task Summary

| Phase | Task | Files | AC Coverage |
|-------|------|-------|-------------|
| A | DB migration | db.ts, db.test.ts | AC1 |
| A | credit.ts R6 | credit.ts, credit.test.ts | (prerequisite) |
| B | checkRateLimit | credit.ts, credit.test.ts | AC4, AC5 |
| B | AuthContext | team.ts, team.test.ts | AC1, AC2, AC4 |
| B | evictMemberBridges | pool.ts, pool.test.ts | AC7 |
| B | TeamManager methods | team.ts, team.test.ts | AC3, AC6 |
| C | Wire auth+rateLimit | api-gateway.ts, api-gateway.test.ts | AC1, AC4, AC5, AC7 |
| C | Admin APIs | api-gateway.ts, api-gateway.test.ts | AC3, AC6 |
| C | Member APIs | api-gateway.ts, api-gateway.test.ts | AC8, AC10 |
| D | Admin UI | admin.html | AC3, AC6 |
| D | Member UI | member.html, index.html | AC8, AC9, AC10 |
| E | Integration | all | all |

## Not Building (explicit exclusions)

- P1: unsuspend/reactivate, invite management, inline edit (side drawer)
- P2: audit log, charts (Chart.js), pagination, alerts, bridge panel
- Grace period / burst token bucket
- Bridge contributor rate limit bonus (x1.5)
- Idle-time transfer / market-based fairness
