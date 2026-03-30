---
feature_ids: [F050, F143]
related_features: []
topics: [agent-mesh, brainstorming, use-cases, escalation, bounty]
doc_kind: discussion
created: 2026-03-30
---

# Agent Mesh 应用场景脑暴（Spark 池）

> 状态：brainstorm（未定稿）
> 目标：先把可讨论的火花沉淀下来，后续再筛选成正式 feature。

## 0. 触发背景

我们希望 Agent 在“单点能力耗尽”后，不是直接失败，而是能升级到 Mesh 协作网络：
- 单 Agent 已尝试：本地 skill、当前 LLM、常规重试
- 仍失败且用户压力高（例如被催促、被责备、需要快速可交付）
- Hub 允许把问题升级到同类型或跨类型 Agent 做众筹式求助

## 1. 核心火花：失败升级 + 悬赏众筹（P0 候选）

### 场景名
Mesh Rescue / 众筹求解模式

### 最小流程（MVP）
1. Trigger：Agent 判定本地失败（达到失败阈值）
2. Package：自动打包“求助包”（问题、上下文、已尝试路径、失败日志、期望输出）
3. Route：Hub 路由到同类 Agent 池（可扩展到跨类）
4. Bounty：发起悬赏（积分/优先级/信用值）
5. Collect：回收多份候选方案 + 置信度 + 成本估计
6. Merge：主 Agent 生成最终答复并附“协作来源说明”

### 价值
- 把“失败”转为“升级协作”，减少单点崩溃
- 提升首次问题解决率
- 留下可复用的失败-修复样本，用于后续 skill 演化

### 护栏
- 默认脱敏后才能出站
- 明确预算上限（token / 时间 / 并发数）
- 用户可一键关闭“自动升级”
- 禁止把不可逆动作交给外部 Agent 直接执行

## 2. 其它应用场景火花

## 2.1 评审陪审团（Solution Jury）
- 用途：同一方案交由 3 个 Agent 独立评审，合并分歧
- 价值：减少主 Agent 盲区，提升方案稳健性
- 风险：共识幻觉；护栏：要求“证据 + 反例”双输出

## 2.2 事故应急蜂群（Incident Swarm）
- 用途：线上事故时并行分派“日志定位/回滚方案/用户通告草稿”
- 价值：缩短 MTTR
- 风险：多 Agent 并发写操作冲突；护栏：只读优先 + 单点执行员

## 2.3 规格争议仲裁（Spec Arbitration）
- 用途：需求含糊时让不同风格 Agent 给出互斥方案并辩论
- 价值：让 tradeoff 显性化，帮助 CVO 决策
- 风险：讨论过长；护栏：时间盒 + 强制输出决策表

## 2.4 失败经验自动蒸馏（Failure-to-Skill）
- 用途：Mesh 求助成功后自动生成“新 skill 草案/FAQ 条目”
- 价值：系统自学习，减少同类问题复发
- 风险：低质量经验污染；护栏：人工审核后入库

## 2.5 任务拍卖市场（Task Auction）
- 用途：复杂任务按子问题挂牌，Agent 竞标接单
- 价值：把“谁更擅长”变成可计算调度
- 风险：激励扭曲（只抢简单题）；护栏：信誉系统 + 难度加权奖励

## 2.6 跨模型成本优化路由（Cost-Aware Relay）
- 用途：高成本模型失败后，自动切换“低成本探索 + 高质量收敛”双阶段
- 价值：控制成本并维持质量
- 风险：路由震荡；护栏：冷却时间 + 策略版本化

## 3. 通用触发条件草案

触发 Mesh 升级建议满足以下至少 2 项：
- 连续失败 N 次（如 N=2）
- 关键子任务置信度 < 阈值
- 用户显式要求“求助/升级/拉群策”
- 预估单 Agent 时间超过 SLA

## 4. 通用回执格式草案（主 Agent 回给用户）

- 本地尝试摘要（做了什么）
- Mesh 升级原因（为什么要升级）
- 协作结果摘要（候选方案 A/B/C）
- 最终建议与风险声明
- 成本/耗时账单（可选）

## 5. 下一步（从 spark 到 feature）

1. 先选 1 个 P0 场景做 PRD（建议从“Mesh Rescue”开始）
2. 定义最小 API：`escalate(task_packet) -> candidates[]`
3. 设计安全基线：脱敏、预算、权限边界
4. 在 Hub 加一个可见入口："升级到 Mesh"

## 6. 开放问题

- 悬赏单位用什么计价：积分、优先级、还是预算配额？
- 同类 Agent 优先还是跨类混合优先？
- 主 Agent 何时应拒绝升级（例如隐私/法律高风险任务）？

## 7. 多猫并行补充（2026-03-30）

## 7.1 来自 Opus 的协议层补充
- 关键抽象：从“预定义路由”升级为“动态能力发现”
- 协议建议：引入 `MEP (Mesh Escalation Protocol)`，链路为 Trigger -> Publish -> Match -> Execute -> Integrate -> Feedback
- 能力前置：需要 `Capability Registry`（技能声明 + 实战成功率）
- 新增场景建议：`Peer Jury`、`Context Relay`、`Shadow Pair`
- 反滥用重点：限制 `hop_count` 避免循环求助；`help_balance` 避免只求助不贡献

## 7.2 来自 Gemini 的体验层补充
- 交互命名建议：可把升级流程产品化为 `Bounty Flow`
- 用户可理解场景：
  - 审美众议院（设计稿主观分歧时做多模型审美会审）
  - 深潜考古队（老系统/无文档系统的专项会诊）
  - 零信任红队（高风险变更前做独立节点盲审）
  - 文化译灵人（跨语言/跨文化内容做本地化复核）
- 护栏补充：默认语境切片与脱敏、预算熔断、协作过程防抄袭

## 7.3 本轮协作覆盖说明
- 已收集：架构视角（Opus）+ 产品体验视角（Gemini）
- 未收集：DARE（运行环境未配置 `DARE_PATH`，本轮未参与）

## 8. 收敛版 MVP（建议）

1. `mesh:registry`（能力注册）
2. `mesh:sos`（失败升级求助）
3. `mesh:jury`（高风险决策陪审）
4. `mesh:bounty`（预算与激励模型，先做最小积分版）

## 9. 第二轮补充：失败边界与系统自愈（Opus）

已有章节更多描述“求助成功”的正路径，本节补“失败前/失败中/失败后/全局失败”的闭环。

## 9.1 `mesh:preflight`（Speculative Preflight）
- 触发：任务开始前做轻量预判（技能匹配 + 历史成功率 + 上下文复杂度），`preflight_score < 0.4` 时直接升级
- 价值：减少“注定失败的本地尝试”，缩短用户等待
- 护栏：预判计算必须轻量（如 <500 tokens）；阈值动态校准；用户可强制 override
- 优先级：P1（依赖 `mesh:registry` 数据）

## 9.2 `mesh:dedup`（Failure Dedup）
- 触发：新 SOS 进入时按 `{问题签名, 技能标签, 错误模式}` 做相似度匹配，命中 open rescue 则合并
- 价值：同类问题 N 次求助合并成 1 次协作，显著降低 mesh 负载
- 护栏：保守阈值避免误合并；保留 subscriber 的上下文差异；TTL 过期后可拆分
- 优先级：P2（需要问题签名/embedding 基础设施）

## 9.3 `mesh:backfill`（Capability Backfill）
- 触发：SOS 成功后，若复现概率高（`recurrence_score > 0.6`）自动创建能力回填任务
- 价值：第一次靠 Mesh，第二次尽量本地自解；长期降低求助频率
- 护栏：回填产物需在历史 case 回测通过再入库；防止过拟合单个 case
- 优先级：P2（依赖 `mesh:sos` + `Failure-to-Skill` 流程）

## 9.4 `mesh:surrender`（Graceful Surrender）
- 触发：候选方案全部验证失败、TTL 超时无人响应、或 peer 置信度集体过低
- 价值：在无法解决时提供结构化可接力包，而不是“空失败”
- 护栏：不得过早认输；必须输出标准化调查报告；自动标记 `human_escalation`
- 优先级：P1（无此能力时，Mesh 失败体验会劣于单 Agent）

## 9.5 生命周期补全图（简版）

`请求 -> mesh:preflight -> (本地解/升级) -> mesh:sos + mesh:dedup -> (成功: mesh:backfill / 失败: mesh:surrender)`
