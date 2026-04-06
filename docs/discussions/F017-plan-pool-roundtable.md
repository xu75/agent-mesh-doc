---
feature_ids: [F017]
related_features: [F013, F014, F011]
topics: [plan-pool, quota-sharing, subscription, peak-shaving, roundtable]
doc_kind: discussion
created: 2026-04-06
participants: [opus, codex, gemini, antig-opus]
status: draft
---

# F017 圆桌讨论：小组订阅 Plan 通过组内共享实现削峰填谷

> 日期：2026-04-06 | 发起人：铲屎官
> 参与：布偶猫/宪宪(opus)、缅因猫/砚砚(codex)、暹罗猫/烁烁(gemini)、孟加拉猫(antig-opus)
> Sonnet 因额度用尽未参与

## 1. 问题定义

铲屎官提出两个场景：

**场景 A — 号池模式（公司付费）**：多人共享一组 API key/订阅计划，按需分配，避免闲置浪费。"请假休息不用的时候白白浪费，就应该交给同事继续用。"

**场景 B — 个人多 Plan 合并（个人付费）**：一个人的多个订阅在多台设备间无缝切换。"我订阅了 2 个 200 美金的 Coding Plan，希望在一台电脑上就用完 2 个 plan 的额度，不想切换账号。"

**核心诉求**：把"绑定到人/设备的订阅"变成"绑定到池的容量"，按需调度。

## 2. 达成共识

| # | 共识 | 支持者 |
|---|------|--------|
| C1 | 架构路线：pool-node + plan-bridge，不改 Hub 核心 | 全员 |
| C2 | 场景 B（个人多设备）先行验证 | 全员 |
| C3 | MVP 计量用请求次数，不用 token | 全员 |
| C4 | 公平性算法推到 Phase 2 | 全员 |
| C5 | 不用浏览器自动化，CLI 优先 | 全员 |
| C6 | **优先做 non-API Plan 桥接**（铲屎官裁定） | 铲屎官裁定 |

## 3. 分歧与裁定

| 分歧 | opus 主张 | 反方主张 | 铲屎官裁定 |
|------|-----------|---------|-----------|
| MVP 是否包含 non-API Plan | 砍掉，只做有 API key 的 | antig-opus + gemini：non-API 才是差异化价值 | **优先做 non-API** |
| 非 API 桥接方式 | 不进 MVP | CLI 单 session 锁 (antig-opus) | CLI 多 session（铲屎官提示 session-id + resume） |

## 4. 架构方案

### 4.1 整体拓扑

```
Caller → Hub → pool-node → Hub → plan-bridge (CLI/API) → Provider
                 │
            ┌────┴────┐
            │ Ledger  │  (SQLite 记账)
            └─────────┘
```

注：MVP 阶段 pool-node → plan-bridge 也经过 Hub relay（遵循 Phase 1 约束），
双跳延迟相比 LLM 推理时间可忽略。

### 4.2 plan-bridge 类型

| 类型 | 适用 Plan | 技术方案 |
|------|----------|---------|
| API 代理型 | 提供 API key 的 Plan（Claude API、OpenAI API 等） | HTTP 代理转发，bridge 持有 key 不暴露 |
| CLI 封装型 | 不提供 API 的 Plan（Claude Max Coding Plan 等） | `claude -p --session-id <uuid> --output-format stream-json` |

### 4.3 CLI 封装型 plan-bridge 的 session 管理

铲屎官指出 Claude CLI 支持多 session 并行 + `--resume`，因此设计为：

```
caller-alice → session-id: "pool-alice-<uuid>" → CLI process（可常驻或按需启动）
caller-bob   → session-id: "pool-bob-<uuid>"   → CLI process
```

- 每个调用方映射独立的 named session，天然隔离上下文
- `--resume` 支持跨请求保持上下文
- 不需要 session lease / 独占窗口

**CLI 关键参数**：
- `--session-id <uuid>`：指定 session（必须 UUID 格式）
- `-p / --print`：非交互模式（编程化使用的前提）
- `--output-format stream-json`：NDJSON 流式输出
- `--input-format stream-json`：双向流式
- `--max-budget-usd <amount>`：单次成本上限
- `--no-session-persistence`：无状态调用（不保存 session）
- `--permission-mode auto`：自动授权

### 4.4 pool-node 核心逻辑

```typescript
// 伪代码
class PoolNode {
  bridges: PlanBridge[]       // 注册的 plan-bridge 列表
  ledger: SQLiteLedger        // 记账

  async invoke(req: InvokeRequest): Promise<InvokeResponse> {
    const bridge = this.selectBridge()  // round-robin
    try {
      const result = await this.meshClient.invoke(bridge.agentId, req)
      this.ledger.record(req.sourceNodeId, bridge.id, 'success')
      return result
    } catch (e) {
      if (e.code === 'RATE_LIMITED') {
        return this.failover(req, bridge)  // 试下一个
      }
      throw e
    }
  }
}
```

## 5. F013/F014 与本 MVP 的关系

### F013（Open Node Registration）—— 建议部分前置

当前 Hub 静态白名单制要求每加一个 plan-bridge 都要改 config + 重启。
F017 的 plan-bridge 可能动态增减（新加一个订阅 Plan），需要运行时注册能力。

**建议前置的最小切面**：
- HELLO 路径扩展：`open` 模式下未知 nodeId 自动注册（TOFU）
- HELLO body 新增可选 `capabilities` 字段
- Config 新增 `registrationMode` 配置项
- 跳过 `approval` 模式（Phase 2 再做）

**代码改动预估**：~50 行（hub.ts HELLO handler + config schema）

### F014（Hybrid Routing）—— MVP 不需要前置

pool-node 作为普通节点通过 Hub relay 通信即可。直连优化等 F014 成熟后自然获益。

## 6. 业界参考

调研了 one-api、new-api、oai-reverse-proxy 三个主流号池项目：

| 维度 | 业界做法 | F017 借鉴 |
|------|---------|----------|
| 核心抽象 | Channel（渠道）= 一个 key/session | plan-bridge = channel |
| 路由策略 | 加权随机 + 失败 failover | MVP: round-robin + failover |
| 限流 | 全局 IP 限流 + 用户级模型限流 | MVP: sourceNodeId 级请求数限流 |
| 计量 | API 按 token（倍率公式）；非 API 按请求数 | 先按请求数 |
| 健康检查 | 最近 N 次成功率，低阈值自动熔断 | MVP: 429 检测 + failover |
| 公平性 | 用户分组 + 分组倍率 | Phase 2 贡献权重 |
| 非 API 池化 | session token / cookie 池化（fuclaude/oaifree） | 我们走 CLI 封装，更稳定合规 |

**关键差异**：业界号池走的是"cookie 劫持"路线，合规风险大。我们走 CLI 封装 + 本地运行，Plan 主人在自己机器上运行 bridge，不暴露凭证给任何人。

## 7. 风险清单

| # | 风险 | 严重度 | 缓解措施 |
|---|------|--------|---------|
| R1 | CLI 并发 session 数未知——同一 Plan 能跑多少个并行 CLI？ | 高 | **MVP 前必须实测** |
| R2 | CLI session 长期运行的内存/进程管理 | 中 | 按需启动 + TTL 回收 |
| R3 | pool-node → plan-bridge 双跳延迟 | 低 | 相比 LLM 推理可忽略；F014 后可优化 |
| R4 | 重试导致重复消耗 | 中 | invocationId 幂等（砚砚提出） |
| R5 | 非 API Plan 的 TOS 合规性 | 中 | 仅限个人多设备场景（自用），不跨账号 |
| R6 | `enablePublicInvoke` 误开导致无鉴权访问 | 高 | 文档明确禁止 + 配置校验 |

## 8. 待定问题

| # | 问题 | 待谁决定 | 优先级 |
|---|------|---------|--------|
| Q1 | 计量单位最终选请求数还是 token？ | 铲屎官 | P2（MVP 先用请求数） |
| Q2 | 贡献权重公式如何设计？ | 圆桌 Phase 2 | P3 |
| Q3 | F013 open 模式是否前置实现？ | 铲屎官 | P1 |
| Q4 | CLI 并发 session 上限实测结果 | 开发验证 | P0（阻塞 MVP） |
| Q5 | session 生命周期：常驻 vs TTL 回收？ | 技术决策 | P1 |
| Q6 | F017 scope 是否包含场景 A（团队共享）？ | 铲屎官 | P1 |

## 9. 建议的 F017 MVP Scope

```
Feature: F017 — Plan Pool（订阅计划池化）
Phase: 1.5（基于 Phase 1 架构，前置 F013 部分能力）
Owner: TBD

MVP 交付物:
  1. plan-bridge-cli:  CLI 封装型 bridge（多 session 管理 + stream-json 输出解析）
  2. plan-bridge-api:  API 代理型 bridge（HTTP 转发 + key 持有）
  3. pool-node:        虚拟节点（round-robin + failover + SQLite ledger）
  4. F013-lite:        Hub open 注册模式（TOFU，仅 HELLO 路径扩展）

MVP 不包含:
  - 贡献/消耗公平算法
  - 浏览器自动化桥接
  - 跨网络/跨 Region 组网
  - Hub 原生 pool 支持（留 Phase 2）
  - 场景 A 的多人信任/鉴权体系

预计代码量: ~500 行（含 F013-lite）
```

## 10. 演进路线

```
Phase 1.5 (F017 MVP):
  pool-node + plan-bridge(cli/api) + F013-lite + SQLite ledger
  → 验证场景 B（个人多设备无感合并）

Phase 2 (F017 扩展):
  contribution-based fairness + 场景 A（团队共享）
  + F013 full（approval 模式）+ F014（直连优化）

Phase 3 (远期):
  Hub 原生 pool 支持 + 跨 Region + 非 CLI Plan 桥接
```

---

> 下一步：铲屎官确认 scope → 立项 F017 → 实测 CLI 并发 → 开发
