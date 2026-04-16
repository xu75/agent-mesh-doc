# F019 P0: Multi-Team Governance — Implementation Plan

**Feature:** F019 — `docs/features/F019-multi-team.md`
**Goal:** 支持多 Team 治理：Super-Admin 跨 team 管理 + Team 生命周期（归档/恢复）+ 多 Team 管理面板
**Acceptance Criteria:**
- [ ] AC1: `super_admins` 表存在，super-admin 可用独立 API key 认证
- [ ] AC2: `POST /v1/super/teams` 创建新 team（替代 bootstrap 入口）
- [ ] AC3: `GET /v1/super/teams` 列出所有 team（含成员数、状态、总用量）
- [ ] AC4: `PUT /v1/super/teams/:id/archive` 归档 team，停写不停读
- [ ] AC5: `PUT /v1/super/teams/:id/unarchive` 恢复 team
- [ ] AC6: 归档 team 后：拒绝所有写操作（invoke/register/join + admin 写接口：suspend/borrow-limit/expiry/rate-limit/rotate-key/create-invite），但读操作正常（`/v1/me` + `/v1/admin/members` GET + super-admin 只读查询）
- [ ] AC7: `effectiveStatus` 增加 `team_archived` 枚举值（优先级在 `suspended` 和 `expired` 之间）
- [ ] AC8: `GET /v1/super/teams/:id/members` + `/v1/super/teams/:id/usage` + `/v1/super/teams/:id/bridges` 只读查看 team 详情
- [ ] AC9: `GET /v1/super/usage` 跨 team 用量汇总
- [ ] AC10: `POST /v1/super/bootstrap` 使用 `POOL_BOOTSTRAP_TOKEN` 手动创建 super-admin，且仅允许 0→1（已存在 super-admin 时拒绝，防止重复铸造）
- [ ] AC11: Super-admin 面板页面（`/super`）：team 列表 + 创建 team + 全局用量
- [ ] AC12: 现有 admin.html / member.html 功能不受破坏
**Architecture:** DB 层新增 `super_admins` 表 + `teams` 表扩展（status/archived_at/archived_by），TeamManager 新增 super-admin 方法，api-gateway 新增 `/v1/super/` 路由前缀，前端新增 super.html
**Tech Stack:** TypeScript, better-sqlite3, Fastify, node:test, 原生 HTML/CSS/JS
**前端验证:** Yes — super.html 需 Chrome 实测

---

## Phase A: Data Layer

### Task 1: DB Schema — super_admins table + teams status columns

**Files:**
- Modify: `packages/plan-pool/src/db.ts` (CREATE TABLE + migration)
- Test: `packages/plan-pool/src/db.test.ts`

**Step 1: Write the failing test**

在 `db.test.ts` 末尾添加测试：

```ts
describe("F019: multi-team schema", () => {
  it("super_admins table exists with required columns", () => {
    const cols = db.db.prepare("PRAGMA table_info(super_admins)").all() as Array<{ name: string }>;
    const names = cols.map(c => c.name);
    assert.ok(names.includes("id"), "missing id");
    assert.ok(names.includes("display_name"), "missing display_name");
    assert.ok(names.includes("key_hash"), "missing key_hash");
    assert.ok(names.includes("created_at"), "missing created_at");
  });

  it("teams table has status, archived_at, archived_by columns", () => {
    const cols = db.db.prepare("PRAGMA table_info(teams)").all() as Array<{ name: string }>;
    const names = cols.map(c => c.name);
    assert.ok(names.includes("status"), "missing status column");
    assert.ok(names.includes("archived_at"), "missing archived_at column");
    assert.ok(names.includes("archived_by"), "missing archived_by column");
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/plan-pool && npx tsx --test src/db.test.ts`
Expected: FAIL — table and columns don't exist yet

**Step 3: Implement**

`db.ts` — 在 `invocations` CREATE TABLE 之后、索引之前，添加 `super_admins` 表：

```sql
CREATE TABLE IF NOT EXISTS super_admins (
  id           TEXT PRIMARY KEY,
  display_name TEXT NOT NULL,
  key_hash     TEXT NOT NULL UNIQUE,
  created_at   INTEGER NOT NULL
);
```

`teams` CREATE TABLE 中追加三列：

```sql
status       TEXT NOT NULL DEFAULT 'active',
archived_at  INTEGER,
archived_by  TEXT
```

Migration block（在现有 migration 末尾追加）：

```ts
// Migration: add status/archived_at/archived_by to teams (F019)
const teamCols3 = db.prepare("PRAGMA table_info(teams)").all() as Array<{ name: string }>;
const teamColNames3 = new Set(teamCols3.map((c) => c.name));
if (!teamColNames3.has("status")) {
  db.exec("ALTER TABLE teams ADD COLUMN status TEXT NOT NULL DEFAULT 'active'");
}
if (!teamColNames3.has("archived_at")) {
  db.exec("ALTER TABLE teams ADD COLUMN archived_at INTEGER");
}
if (!teamColNames3.has("archived_by")) {
  db.exec("ALTER TABLE teams ADD COLUMN archived_by TEXT");
}
```

**Step 4: Run test to verify it passes**

Run: `cd packages/plan-pool && npx tsx --test src/db.test.ts`
Expected: PASS

---

### Task 2: TeamManager — Super-Admin CRUD

**Files:**
- Modify: `packages/plan-pool/src/team.ts` (new methods)
- Test: `packages/plan-pool/src/team.test.ts`

**Step 1: Write the failing test**

```ts
describe("F019: super-admin", () => {
  it("createSuperAdmin and authenticateSuperAdmin", () => {
    const result = tm.createSuperAdmin({ displayName: "Root Admin" });
    assert.ok(result.superAdminId);
    assert.ok(result.apiKey.startsWith("sa_"));

    const auth = tm.authenticateSuperAdmin(result.apiKey);
    assert.ok(auth);
    assert.equal(auth.superAdminId, result.superAdminId);
    assert.equal(auth.displayName, "Root Admin");
  });

  it("authenticateSuperAdmin returns null for invalid key", () => {
    const auth = tm.authenticateSuperAdmin("sa_invalid");
    assert.equal(auth, null);
  });

  it("listAllTeams returns all teams with member counts and usage", () => {
    tm.createTeam({ name: "Team-A", providerScope: "claude", ownerDisplayName: "OwnerA", ownerPublicKey: "pkA" });
    tm.createTeam({ name: "Team-B", providerScope: "gpt", ownerDisplayName: "OwnerB", ownerPublicKey: "pkB" });
    const teams = tm.listAllTeams();
    assert.ok(teams.length >= 2);
    const teamA = teams.find(t => t.name === "Team-A");
    assert.ok(teamA);
    assert.equal(teamA.memberCount, 1);
    assert.equal(teamA.status, "active");
    assert.equal(teamA.totalCost, 0);
    assert.equal(teamA.requestCount, 0);
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd packages/plan-pool && npx tsx --test src/team.test.ts`
Expected: FAIL — methods don't exist

**Step 3: Implement**

`team.ts` 新增：

```ts
// ── F019 Super-Admin Methods ──

function generateSuperAdminKey(): string {
  return `sa_${crypto.randomBytes(24).toString("hex")}`;
}

export interface SuperAdminAuth {
  superAdminId: string;
  displayName: string;
}

createSuperAdmin(input: { displayName: string }): { superAdminId: string; apiKey: string } {
  const id = crypto.randomUUID();
  const apiKey = generateSuperAdminKey();
  const now = Date.now();
  this.db.prepare(
    "INSERT INTO super_admins (id, display_name, key_hash, created_at) VALUES (?, ?, ?, ?)"
  ).run(id, input.displayName, sha256(apiKey), now);
  return { superAdminId: id, apiKey };
}

authenticateSuperAdmin(apiKey: string): SuperAdminAuth | null {
  const row = this.db.prepare(
    "SELECT id, display_name FROM super_admins WHERE key_hash = ?"
  ).get(sha256(apiKey)) as { id: string; display_name: string } | undefined;
  if (!row) return null;
  return { superAdminId: row.id, displayName: row.display_name };
}

listAllTeams(): Array<{
  id: string; name: string; providerScope: string; status: string;
  memberCount: number; totalCost: number; requestCount: number; archivedAt: number | null;
}> {
  return this.db.prepare(`
    SELECT t.id, t.name, t.provider_scope AS providerScope, t.status,
      t.archived_at AS archivedAt,
      (SELECT COUNT(*) FROM members m WHERE m.team_id = t.id) AS memberCount,
      COALESCE((SELECT SUM(i.normalized_cost) FROM invocations i WHERE i.team_id = t.id AND i.lease_status IN ('settled','expired')), 0) AS totalCost,
      COALESCE((SELECT COUNT(i.id) FROM invocations i WHERE i.team_id = t.id AND i.lease_status IN ('settled','expired')), 0) AS requestCount
    FROM teams t
    ORDER BY t.created_at ASC
  `).all() as Array<{ id: string; name: string; providerScope: string; status: string; memberCount: number; totalCost: number; requestCount: number; archivedAt: number | null }>;
}
```

**Step 4: Run test to verify it passes**

---

### Task 3: TeamManager — Archive/Unarchive + team_archived effectiveStatus

**Files:**
- Modify: `packages/plan-pool/src/team.ts`
- Test: `packages/plan-pool/src/team.test.ts`

**Step 1: Write the failing test**

```ts
describe("F019: team archive", () => {
  it("archiveTeam sets status to archived", () => {
    const team = tm.createTeam({ name: "Temp", providerScope: "claude", ownerDisplayName: "O", ownerPublicKey: "pk" });
    const saResult = tm.createSuperAdmin({ displayName: "SA" });
    tm.archiveTeam(team.teamId, saResult.superAdminId);

    const teams = tm.listAllTeams();
    const archived = teams.find(t => t.id === team.teamId);
    assert.equal(archived?.status, "archived");
    assert.ok(archived?.archivedAt);
  });

  it("unarchiveTeam restores to active", () => {
    const team = tm.createTeam({ name: "Temp2", providerScope: "gpt", ownerDisplayName: "O2", ownerPublicKey: "pk2" });
    const saResult = tm.createSuperAdmin({ displayName: "SA2" });
    tm.archiveTeam(team.teamId, saResult.superAdminId);
    tm.unarchiveTeam(team.teamId);

    const teams = tm.listAllTeams();
    const restored = teams.find(t => t.id === team.teamId);
    assert.equal(restored?.status, "active");
  });

  it("archived team member getMemberSelfInfo shows team_archived effectiveStatus", () => {
    const team = tm.createTeam({ name: "ArchTest", providerScope: "claude", ownerDisplayName: "O", ownerPublicKey: "pk" });
    const saResult = tm.createSuperAdmin({ displayName: "SA3" });
    tm.archiveTeam(team.teamId, saResult.superAdminId);

    const info = tm.getMemberSelfInfo(team.memberId);
    assert.ok(info);
    assert.equal(info.effectiveStatus, "team_archived");
  });

  it("authenticateApiKey sets canInvoke=false for archived team", () => {
    const team = tm.createTeam({ name: "ArchAuth", providerScope: "claude", ownerDisplayName: "O", ownerPublicKey: "pk" });
    const saResult = tm.createSuperAdmin({ displayName: "SA4" });
    tm.archiveTeam(team.teamId, saResult.superAdminId);

    const auth = tm.authenticateApiKey(team.apiKey);
    assert.ok(auth);
    assert.equal(auth.canInvoke, false);
    assert.equal(auth.canRegisterBridge, false);
    assert.equal(auth.canReadSelf, true);  // 停写不停读
  });
});
```

**Step 2: Run test to verify it fails**

**Step 3: Implement**

`team.ts`:

```ts
archiveTeam(teamId: string, archivedBy: string): void {
  const result = this.db.prepare(
    "UPDATE teams SET status = 'archived', archived_at = ?, archived_by = ? WHERE id = ? AND status = 'active'"
  ).run(Date.now(), archivedBy, teamId);
  if (result.changes === 0) throw new Error("team not found or already archived");
}

unarchiveTeam(teamId: string): void {
  const result = this.db.prepare(
    "UPDATE teams SET status = 'active', archived_at = NULL, archived_by = NULL WHERE id = ? AND status = 'archived'"
  ).run(teamId);
  if (result.changes === 0) throw new Error("team not found or not archived");
}
```

修改 `authenticateApiKey()`：查询中 JOIN teams 获取 `t.status`，当 `t.status === 'archived'` 时 `canInvoke = false, canRegisterBridge = false, canReadSelf = true`。

修改 `getMemberSelfInfo()`：在 effectiveStatus 优先级链中，`team_archived` 插在 `suspended` 之后、`expired` 之前：

```ts
// effectiveStatus priority: suspended > team_archived > expired > quota_exhausted > rate_limited_weekly > rate_limited_5h > active
if (teamStatus === 'archived') {
  effectiveStatus = "team_archived";
  blockingReason = "Team is archived";
}
```

**Step 4: Run test to verify it passes**

**注意点（@gpt52 提醒）：** `team_archived` 是 effectiveStatus 枚举的新增值。前端 admin.html 和 member.html 中显示 effectiveStatus 的逻辑需要覆盖此值（Phase D 处理）。

---

## Phase B: API Layer — Super-Admin Routes

### Task 4: Super-Admin Auth Middleware + Bootstrap

**Files:**
- Modify: `packages/plan-pool/src/api-gateway.ts` (auth helper, bootstrap refactor)
- Modify: `packages/plan-pool/src/config.ts` (config)
- Test: `packages/plan-pool/src/api-gateway.test.ts`

**Step 1: Write the failing test**

```ts
describe("F019: super-admin bootstrap", () => {
  it("first super-admin creation with bootstrap token", async () => {
    const poolDb = createDatabase(":memory:");
    const tm = new TeamManager(poolDb.db);
    const app = await createApiGateway({
      db: poolDb.db, teamManager: tm, bootstrapToken: "boot-secret",
    });
    try {
      const r = await app.inject({
        method: "POST", url: "/v1/super/bootstrap",
        headers: { authorization: "Bearer boot-secret" },
        payload: { displayName: "Root" },
      });
      assert.equal(r.statusCode, 200);
      const body = r.json();
      assert.ok(body.superAdminId);
      assert.ok(body.apiKey.startsWith("sa_"));
    } finally {
      await app.close();
      poolDb.close();
    }
  });

  it("bootstrap rejected without token", async () => {
    const poolDb = createDatabase(":memory:");
    const tm = new TeamManager(poolDb.db);
    const app = await createApiGateway({
      db: poolDb.db, teamManager: tm, bootstrapToken: "boot-secret",
    });
    try {
      const r = await app.inject({
        method: "POST", url: "/v1/super/bootstrap",
        payload: { displayName: "Root" },
      });
      assert.equal(r.statusCode, 401);
    } finally {
      await app.close();
      poolDb.close();
    }
  });
  it("bootstrap rejects duplicate creation (0→1 only)", async () => {
    const poolDb = createDatabase(":memory:");
    const tm = new TeamManager(poolDb.db);
    const app = await createApiGateway({
      db: poolDb.db, teamManager: tm, bootstrapToken: "boot-secret",
    });
    try {
      // First creation succeeds
      await app.inject({
        method: "POST", url: "/v1/super/bootstrap",
        headers: { authorization: "Bearer boot-secret" },
        payload: { displayName: "Root" },
      });
      // Second creation rejected
      const r2 = await app.inject({
        method: "POST", url: "/v1/super/bootstrap",
        headers: { authorization: "Bearer boot-secret" },
        payload: { displayName: "Root2" },
      });
      assert.equal(r2.statusCode, 403);
      assert.ok(r2.json().error.includes("already exists"));
    } finally {
      await app.close();
      poolDb.close();
    }
  });
});
```

**Step 2: Run test to verify it fails**

**Step 3: Implement**

api-gateway.ts 新增 auth helper:

```ts
function authenticateSuperAdmin(req: FastifyRequest): SuperAdminAuth | null {
  const token = extractBearerToken(req);
  if (!token) return null;
  return teamManager.authenticateSuperAdmin(token);
}
```

新增 bootstrap 端点：

```ts
// POST /v1/super/bootstrap — one-time super-admin creation (requires bootstrap token)
// Security: 0→1 only — once a super-admin exists, this endpoint is permanently closed
app.post("/v1/super/bootstrap", async (req, reply) => {
  const token = extractBearerToken(req);
  if (!bootstrapToken || token !== bootstrapToken) {
    reply.status(401);
    return { error: "bootstrap token required" };
  }
  // 0→1 guard: reject if any super-admin already exists
  const existing = db.prepare("SELECT COUNT(*) AS cnt FROM super_admins").get() as { cnt: number };
  if (existing.cnt > 0) {
    reply.status(403);
    return { error: "super-admin already exists — bootstrap closed" };
  }
  const body = req.body as { displayName?: string };
  if (!body?.displayName) {
    reply.status(400);
    return { error: "missing displayName" };
  }
  const result = teamManager.createSuperAdmin({ displayName: body.displayName });
  return result;
});
```

**Step 4: Run test to verify it passes**

---

### Task 5: Super-Admin CRUD Routes — Teams

**Files:**
- Modify: `packages/plan-pool/src/api-gateway.ts`
- Test: `packages/plan-pool/src/api-gateway.test.ts`

**Step 1: Write the failing test**

```ts
describe("F019: super-admin team management", () => {
  let poolDb: PoolDatabase;
  let tm: TeamManager;
  let app: FastifyInstance;
  let saKey: string;

  beforeEach(async () => {
    poolDb = createDatabase(":memory:");
    tm = new TeamManager(poolDb.db);
    app = await createApiGateway({ db: poolDb.db, teamManager: tm, bootstrapToken: "boot" });
    // Create super-admin
    const r = await app.inject({
      method: "POST", url: "/v1/super/bootstrap",
      headers: { authorization: "Bearer boot" },
      payload: { displayName: "Root" },
    });
    saKey = r.json().apiKey;
  });

  afterEach(async () => {
    await app.close();
    poolDb.close();
  });

  it("POST /v1/super/teams creates a team", async () => {
    const r = await app.inject({
      method: "POST", url: "/v1/super/teams",
      headers: { authorization: `Bearer ${saKey}` },
      payload: { name: "Lab-Claude", providerScope: "claude", ownerDisplayName: "Owner1", ownerPublicKey: "pk1" },
    });
    assert.equal(r.statusCode, 200);
    assert.ok(r.json().teamId);
  });

  it("GET /v1/super/teams lists all teams", async () => {
    // Create two teams
    await app.inject({
      method: "POST", url: "/v1/super/teams",
      headers: { authorization: `Bearer ${saKey}` },
      payload: { name: "T1", providerScope: "claude", ownerDisplayName: "O1", ownerPublicKey: "pk1" },
    });
    await app.inject({
      method: "POST", url: "/v1/super/teams",
      headers: { authorization: `Bearer ${saKey}` },
      payload: { name: "T2", providerScope: "gpt", ownerDisplayName: "O2", ownerPublicKey: "pk2" },
    });

    const r = await app.inject({
      method: "GET", url: "/v1/super/teams",
      headers: { authorization: `Bearer ${saKey}` },
    });
    assert.equal(r.statusCode, 200);
    assert.equal(r.json().teams.length, 2);
  });

  it("PUT /v1/super/teams/:id/archive archives a team", async () => {
    const createR = await app.inject({
      method: "POST", url: "/v1/super/teams",
      headers: { authorization: `Bearer ${saKey}` },
      payload: { name: "ToArchive", providerScope: "claude", ownerDisplayName: "O", ownerPublicKey: "pk" },
    });
    const teamId = createR.json().teamId;

    const r = await app.inject({
      method: "PUT", url: `/v1/super/teams/${teamId}/archive`,
      headers: { authorization: `Bearer ${saKey}` },
    });
    assert.equal(r.statusCode, 200);

    // Verify team is archived
    const listR = await app.inject({
      method: "GET", url: "/v1/super/teams",
      headers: { authorization: `Bearer ${saKey}` },
    });
    const team = listR.json().teams.find((t: any) => t.id === teamId);
    assert.equal(team.status, "archived");
  });

  it("PUT /v1/super/teams/:id/unarchive restores a team", async () => {
    const createR = await app.inject({
      method: "POST", url: "/v1/super/teams",
      headers: { authorization: `Bearer ${saKey}` },
      payload: { name: "ToRestore", providerScope: "claude", ownerDisplayName: "O", ownerPublicKey: "pk" },
    });
    const teamId = createR.json().teamId;

    await app.inject({
      method: "PUT", url: `/v1/super/teams/${teamId}/archive`,
      headers: { authorization: `Bearer ${saKey}` },
    });
    const r = await app.inject({
      method: "PUT", url: `/v1/super/teams/${teamId}/unarchive`,
      headers: { authorization: `Bearer ${saKey}` },
    });
    assert.equal(r.statusCode, 200);
  });

  it("rejects super-admin routes with member API key", async () => {
    const r = await app.inject({
      method: "GET", url: "/v1/super/teams",
      headers: { authorization: "Bearer pool_fake" },
    });
    assert.equal(r.statusCode, 401);
  });
});
```

**Step 2: Run test to verify it fails**

**Step 3: Implement**

api-gateway.ts 新增路由：

```ts
// ── Super-Admin API (F019) ──

// POST /v1/super/teams — create team (super-admin only)
app.post("/v1/super/teams", async (req, reply) => {
  const sa = authenticateSuperAdmin(req);
  if (!sa) { reply.status(401); return { error: "super-admin auth required" }; }
  // same validation as old POST /v1/admin/teams
  const body = req.body as { name?: string; providerScope?: string; ownerDisplayName?: string; ownerPublicKey?: string; defaultBorrowLimitUsd?: number };
  if (!body?.name || !body?.providerScope || !body?.ownerDisplayName || !body?.ownerPublicKey) {
    reply.status(400);
    return { error: "missing required fields: name, providerScope, ownerDisplayName, ownerPublicKey" };
  }
  return teamManager.createTeam(body as Parameters<typeof teamManager.createTeam>[0]);
});

// GET /v1/super/teams — list all teams
app.get("/v1/super/teams", async (req, reply) => {
  const sa = authenticateSuperAdmin(req);
  if (!sa) { reply.status(401); return { error: "super-admin auth required" }; }
  return { teams: teamManager.listAllTeams() };
});

// PUT /v1/super/teams/:id/archive
app.put<{ Params: { id: string } }>("/v1/super/teams/:id/archive", async (req, reply) => {
  const sa = authenticateSuperAdmin(req);
  if (!sa) { reply.status(401); return { error: "super-admin auth required" }; }
  try {
    teamManager.archiveTeam(req.params.id, sa.superAdminId);
    // Evict bridges if router is available
    if (router) router.evictTeamBridges(req.params.id);
    return { ok: true };
  } catch (err) {
    reply.status(400);
    return { error: err instanceof Error ? err.message : "archive failed" };
  }
});

// PUT /v1/super/teams/:id/unarchive
app.put<{ Params: { id: string } }>("/v1/super/teams/:id/unarchive", async (req, reply) => {
  const sa = authenticateSuperAdmin(req);
  if (!sa) { reply.status(401); return { error: "super-admin auth required" }; }
  try {
    teamManager.unarchiveTeam(req.params.id);
    return { ok: true };
  } catch (err) {
    reply.status(400);
    return { error: err instanceof Error ? err.message : "unarchive failed" };
  }
});
```

**Step 4: Run test to verify it passes**

---

### Task 6: Super-Admin Read Routes — Team Details + Global Usage

**Files:**
- Modify: `packages/plan-pool/src/api-gateway.ts`
- Modify: `packages/plan-pool/src/team.ts` (getGlobalUsage, getTeamBridges)
- Test: `packages/plan-pool/src/api-gateway.test.ts`

**Step 1: Write the failing test**

```ts
describe("F019: super-admin read routes", () => {
  // (beforeEach/afterEach same as Task 5, create super-admin + one team)

  it("GET /v1/super/teams/:id/members shows team members", async () => {
    const createR = await app.inject({
      method: "POST", url: "/v1/super/teams",
      headers: { authorization: `Bearer ${saKey}` },
      payload: { name: "ReadTest", providerScope: "claude", ownerDisplayName: "Owner", ownerPublicKey: "pk" },
    });
    const teamId = createR.json().teamId;
    const r = await app.inject({
      method: "GET", url: `/v1/super/teams/${teamId}/members`,
      headers: { authorization: `Bearer ${saKey}` },
    });
    assert.equal(r.statusCode, 200);
    assert.equal(r.json().members.length, 1); // owner
  });

  it("GET /v1/super/usage returns cross-team aggregation", async () => {
    const r = await app.inject({
      method: "GET", url: "/v1/super/usage",
      headers: { authorization: `Bearer ${saKey}` },
    });
    assert.equal(r.statusCode, 200);
    assert.ok(Array.isArray(r.json().teams));
  });
});
```

**Step 2: Run test to verify it fails**

**Step 3: Implement**

`team.ts` 新增：

```ts
getGlobalUsage(): Array<{ teamId: string; teamName: string; providerScope: string; totalCost: number; requestCount: number }> {
  return this.db.prepare(`
    SELECT t.id AS teamId, t.name AS teamName, t.provider_scope AS providerScope,
      COALESCE(SUM(i.normalized_cost), 0) AS totalCost,
      COUNT(i.id) AS requestCount
    FROM teams t
    LEFT JOIN invocations i ON i.team_id = t.id AND i.lease_status IN ('settled', 'expired')
    GROUP BY t.id
    ORDER BY totalCost DESC
  `).all() as Array<{ teamId: string; teamName: string; providerScope: string; totalCost: number; requestCount: number }>;
}

getTeamBridges(teamId: string): Array<{ id: string; clientName: string; ownerMemberId: string; shareMode: string; lastActiveAt: number | null }> {
  return this.db.prepare(`
    SELECT id, client_name AS clientName, owner_member_id AS ownerMemberId,
      share_mode AS shareMode, last_owner_active_at AS lastActiveAt
    FROM bridges WHERE team_id = ?
    ORDER BY created_at ASC
  `).all(teamId) as Array<{ id: string; clientName: string; ownerMemberId: string; shareMode: string; lastActiveAt: number | null }>;
}
```

api-gateway.ts 新增路由：

```ts
// GET /v1/super/teams/:id/members
app.get<{ Params: { id: string } }>("/v1/super/teams/:id/members", async (req, reply) => {
  const sa = authenticateSuperAdmin(req);
  if (!sa) { reply.status(401); return { error: "super-admin auth required" }; }
  return teamManager.getTeamMembers(req.params.id);
});

// GET /v1/super/teams/:id/usage
app.get<{ Params: { id: string } }>("/v1/super/teams/:id/usage", async (req, reply) => {
  const sa = authenticateSuperAdmin(req);
  if (!sa) { reply.status(401); return { error: "super-admin auth required" }; }
  return teamManager.getUsage(req.params.id);
});

// GET /v1/super/teams/:id/bridges
app.get<{ Params: { id: string } }>("/v1/super/teams/:id/bridges", async (req, reply) => {
  const sa = authenticateSuperAdmin(req);
  if (!sa) { reply.status(401); return { error: "super-admin auth required" }; }
  return { bridges: teamManager.getTeamBridges(req.params.id) };
});

// GET /v1/super/usage — global usage aggregation
app.get("/v1/super/usage", async (req, reply) => {
  const sa = authenticateSuperAdmin(req);
  if (!sa) { reply.status(401); return { error: "super-admin auth required" }; }
  return { teams: teamManager.getGlobalUsage() };
});
```

**Step 4: Run test to verify it passes**

---

### Task 7: Archive Enforcement — Block ALL writes for archived teams

**Files:**
- Modify: `packages/plan-pool/src/api-gateway.ts` (invoke + register + join + ALL admin write routes)
- Modify: `packages/plan-pool/src/team.ts` (AuthContext 增加 teamStatus 字段)
- Test: `packages/plan-pool/src/api-gateway.test.ts`

**Step 1: Write the failing test**

```ts
describe("F019: archive enforcement", () => {
  // Setup: create super-admin, create team, get owner API key, archive team

  it("invoke returns 403 TEAM_ARCHIVED for archived team", async () => {
    // Try invoke with member key → 403 TEAM_ARCHIVED
  });

  it("bridge register returns 403 for archived team", async () => {
    // Same setup → 403
  });

  it("join returns 403 for invite code of archived team", async () => {
    // Create invite before archiving, archive, try join → error
  });

  // Admin write routes blocked
  it("POST /v1/admin/invites returns 403 for archived team", async () => {
    // Owner tries to create invite → 403 TEAM_ARCHIVED
  });

  it("PUT /v1/admin/members/:id/suspend returns 403 for archived team", async () => {
    // → 403 TEAM_ARCHIVED
  });

  it("PUT /v1/admin/members/:id/borrow-limit returns 403 for archived team", async () => {
    // → 403 TEAM_ARCHIVED
  });

  it("PUT /v1/admin/members/:id/expiry returns 403 for archived team", async () => {
    // → 403 TEAM_ARCHIVED
  });

  it("PUT /v1/admin/members/:id/rate-limit returns 403 for archived team", async () => {
    // → 403 TEAM_ARCHIVED
  });

  it("PUT /v1/admin/team/settings returns 403 for archived team", async () => {
    // → 403 TEAM_ARCHIVED
  });

  it("POST /v1/admin/members/:id/rotate-key returns 403 for archived team", async () => {
    // → 403 TEAM_ARCHIVED
  });

  it("PUT /v1/admin/bridges/:id/approve-attestation returns 403 for archived team", async () => {
    // → 403 TEAM_ARCHIVED
  });

  // Read routes still work
  it("GET /v1/me still works for archived team member", async () => {
    // Archive team, GET /v1/me → 200, effectiveStatus = team_archived
  });

  it("GET /v1/admin/members still works for archived team owner", async () => {
    // Archive team, GET /v1/admin/members → 200 (read-only)
  });

  it("GET /v1/admin/invocations still works for archived team owner", async () => {
    // Archive team, GET /v1/admin/invocations → 200 (read-only)
  });
});
```

**Step 2: Run test to verify it fails**

**Step 3: Implement**

1. `team.ts` — `AuthContext` 增加 `teamStatus` 字段：

```ts
export interface AuthContext {
  // ... existing fields ...
  teamStatus: string;  // 'active' | 'archived'
}
```

`authenticateApiKey()` 查询中已 JOIN teams（Task 3 变更），把 `t.status` 带出来赋给 `teamStatus`。

2. `api-gateway.ts` — 添加统一 write guard 函数：

```ts
/** Returns 403 error if team is archived. Used by all admin write routes. */
function teamWriteGuard(auth: AuthResult): { error: string; statusCode: 403 } | null {
  if (auth.teamStatus === 'archived') {
    return { error: "TEAM_ARCHIVED — team is archived, write operations disabled", statusCode: 403 };
  }
  return null;
}
```

在每个 admin 写路由的认证检查后、业务逻辑前插入：

```ts
const blocked = teamWriteGuard(auth);
if (blocked) { reply.status(403); return { error: blocked.error }; }
```

覆盖的写路由清单（完整 — 10 条）：
- `POST /v1/admin/invites`
- `PUT /v1/admin/members/:id/suspend`
- `PUT /v1/admin/members/:id/borrow-limit`
- `PUT /v1/admin/members/:id/expiry`
- `PUT /v1/admin/members/:id/rate-limit`
- `POST /v1/admin/members/:id/rotate-key`
- `PUT /v1/admin/bridges/:id/approve-attestation`
- `PUT /v1/admin/team/settings`
- `POST /v1/bridges/register`
- `POST /v1/join`

不影响的读路由（保持正常）：
- `GET /v1/admin/members`
- `GET /v1/admin/invocations`
- `GET /v1/admin/team/settings`
- `GET /v1/me`
- `GET /v1/me/usage`
- `GET /v1/me/invocations`

3. `handlePoolRequest()` 里区分归档原因：

```ts
if (!auth.canInvoke) {
  if (auth.teamStatus === 'archived') {
    return { error: "TEAM_ARCHIVED", statusCode: 403, reason: "Team is archived" };
  }
  if (auth.expiresAt != null && auth.expiresAt <= Date.now()) {
    return { error: "EXPIRED", statusCode: 403 };
  }
  return { error: "FORBIDDEN", statusCode: 403 };
}
```

**Step 4: Run test to verify it passes**

---

### Task 8: PoolRouter.evictTeamBridges

**Files:**
- Modify: `packages/plan-pool/src/pool.ts`
- Test: `packages/plan-pool/src/pool.test.ts`

**Step 1: Write the failing test**

```ts
it("evictTeamBridges removes all bridges for a team", () => {
  // Register two bridges for team A, one for team B
  // evictTeamBridges(teamA)
  // Assert team A bridges gone, team B bridge remains
});
```

**Step 2: Run test to verify it fails**

**Step 3: Implement**

`pool.ts`:

```ts
evictTeamBridges(teamId: string): number {
  let count = 0;
  for (const [bridgeId, entry] of this.bridges) {
    if (entry.teamId === teamId) {
      this.bridges.delete(bridgeId);
      count++;
    }
  }
  return count;
}
```

**Step 4: Run test to verify it passes**

---

### Task 9: Legacy bootstrap route — backward compatibility

**Files:**
- Modify: `packages/plan-pool/src/api-gateway.ts`
- Test: `packages/plan-pool/src/api-gateway.test.ts`

**说明**：现有 `POST /v1/admin/teams` 是 bootstrap-protected 的 team 创建入口。F019 后主要入口变为 `POST /v1/super/teams`。需要保持 backward compatibility：

- 如果没有 super-admin 存在（首次部署），`POST /v1/admin/teams` 仍按 first-team-only 逻辑工作（创建首个 team）
- 如果已有 super-admin，`POST /v1/admin/teams` 返回 `410 Gone` + 迁移指引（不用 301/307——POST redirect 语义在各客户端不一致，显式 410 + error message 更安全）

```ts
// Legacy route: after super-admin exists, guide callers to /v1/super/teams
const hasSuperAdmin = db.prepare("SELECT COUNT(*) AS cnt FROM super_admins").get() as { cnt: number };
if (hasSuperAdmin.cnt > 0) {
  reply.status(410);
  return { error: "This endpoint is deprecated. Use POST /v1/super/teams with super-admin credentials.", migration: "/v1/super/teams" };
}
// ...existing first-team-only logic
```

现有的 bootstrap protection 测试必须仍然通过（first-team-only 场景下 super_admins 表为空，走原逻辑）。

---

## Phase C: Frontend

### Task 10: super.html — Super-Admin Dashboard

**Files:**
- Create: `packages/plan-pool/public/super.html`
- Modify: `packages/plan-pool/src/api-gateway.ts` (serve `/super` route)

**实现要点**：

- 登录界面：输入 super-admin API key（`sa_` 前缀）
- 登录后显示：
  - **Team 列表**：卡片/表格式，每个 team 显示名称、provider_scope、成员数、状态（active/archived 用不同颜色标识）
  - **创建 Team 按钮**：弹出表单（name, providerScope, ownerDisplayName, ownerPublicKey）
  - **Team 操作**：归档/恢复按钮
  - **全局用量**：跨 team 汇总（总请求数、总成本）
  - **点击 Team**：展开显示成员列表 + 用量详情（只读）
- 风格与现有 admin.html/member.html 保持一致

### Task 11: admin.html + member.html — team_archived 状态显示

**Files:**
- Modify: `packages/plan-pool/public/admin.html`
- Modify: `packages/plan-pool/public/member.html`

**实现要点**：

- member.html：`effectiveStatus === 'team_archived'` 时显示"Team 已归档"提示，UI 只读
- admin.html：如果 team 已归档，顶部横幅提示"此 Team 已归档，管理操作已禁用"，禁用写操作按钮

---

## Phase D: Integration Testing + 前端验证

### Task 12: 全链路集成测试

**Files:**
- Test: `packages/plan-pool/src/api-gateway.test.ts`

完整场景：
1. Bootstrap super-admin → 创建两个 team → 列表查看
2. Team A 添加成员 → 归档 Team A → 成员 invoke 被拒 → /v1/me 仍可用 → 恢复 Team A → 成员可 invoke
3. 验证现有 bootstrap protection 测试仍通过（backward compat）

### Task 13: 前端实测

浏览器测试：
1. 启动 dev server，访问 `/super`，用 super-admin key 登录
2. 创建两个 team，验证列表显示
3. 归档一个 team，验证状态变化
4. 用 member key 访问 `/member`，验证 team_archived 状态显示
5. 恢复 team，验证功能恢复

---

## Checkpoint Gate

完成 Phase D 后执行 quality-gate：
- [ ] 所有测试通过 `cd packages/plan-pool && npx tsx --test src/*.test.ts`
- [ ] 构建成功 `cd packages/plan-pool && npx tsc --noEmit`
- [ ] 现有 F017/F018 测试无回归
- [ ] 前端 super.html / admin.html / member.html 实测通过
- [ ] spec AC 逐条验证
