---
feature_ids: [F016]
topics: [landing-page, agent-docs, control-panel, portal]
doc_kind: feature
created: 2026-04-05
status: kickoff
owner: TBD
---

# F016 — Hub Portal & Agent Documentation Layer

## Context

mesh-hub.xyz 当前的落地页（`/`）是一个静态 HTML 页面，展示在线节点列表。随着更多 Agent 和服务挂到 Hub 上（TTS、Code Review 等），仅显示名称和技能列表不够——调用方（Open Claw、其他 Agent）需要知道**如何使用**这些服务。

F011 的 TTS 上线暴露了这个问题：没有标准化的文档入口，调用方只能靠人类手动传递提示词。

## 目标

将 mesh-hub.xyz 从"节点状态看板"进化为"Agent Portal"——一个人类和 Agent 都能使用的服务目录 + 文档系统。

## AC（分阶段交付）

### Phase 1: Agent Docs（最小可用）

- **AC-1**: 每个 Agent 行在落地页上有一个 "Docs" 链接
- **AC-2**: 点击进入该 Agent 的文档页（Hub 渲染 markdown 或跳转外部 URL）
- **AC-3**: Agent 注册时支持两种文档模式：
  - `docsContent`: markdown 文本，由 Hub 渲染为 HTML 页面（适合简单服务）
  - `docsUrl`: 外部 URL，Hub 直接链接跳转（适合有独立文档站的服务）
- **AC-4**: 文档页包含"人类可读的功能简介" + "Agent 可读的调用指南"（双受众）
- **AC-5**: `/v1/directory` 响应中包含 `docsUrl`（指向 Hub 渲染页或外部 URL）

### Phase 2: Control Panel（框架）

- **AC-6**: Hub 管理后台框架（admin token 认证）
- **AC-7**: 在线查看/编辑 Hub 配置（`allowedNodes` 等）
- **AC-8**: 节点健康状态仪表盘（心跳历史、调用统计）

### Phase 3: 权限 & 额度

- **AC-9**: 公开调用的 rate limiting（每 IP / 每分钟）
- **AC-10**: 调用配额管理（按 nodeId 或 API key）
- **AC-11**: `enablePublicInvoke` 的细粒度控制（按节点/按 skill 开放）

## Open Questions

1. Phase 1 的文档渲染是用 SSR（Hub 直接生成 HTML），还是做一个简单的前端 SPA？
   - 倾向 SSR：保持架构简单，落地页已经是 SSR
2. `docsContent` 存在哪里？config.json 里会太臃肿。
   - 方案 A: 节点注册时 POST 上来，Hub 存在内存/文件
   - 方案 B: 约定文件路径 `docs/agent-prompts/{nodeId}.md`，Hub 读取
   - 方案 C: `docsUrl` 指向 Hub 自身路径（`/v1/docs/{nodeId}`），Hub 从 git 或文件系统加载
3. Phase 2 的 admin 面板用什么前端框架？还是纯 SSR + form?
4. Rate limiting 用什么方案？内存计数器 vs Redis？

## 设计决策记录

- （待 Phase 1 kickoff 时补充）

## 参考

- 当前落地页实现: `packages/mesh-hub/src/landing-page.ts`
- TTS 文档范例: `docs/agent-prompts/tts-bridge.md`
- 当前 discovery 响应: `GET /v1/discovery`
