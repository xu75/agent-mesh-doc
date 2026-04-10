---
feature_ids: [F011]
related_features: [F002, F004, F010, F013, F014]
topics: [tts, bridge, local-model, capability, example]
doc_kind: spec
created: 2026-04-02
layer: application
owner_module: bridge-tts
status: complete
phase: 1
depends_on:
  - { id: F002, type: blocking }
  - { id: F010, type: blocking }
  - { id: F013, type: related }
  - { id: F014, type: related }
evidence:
  - { type: test, ref: "packages/bridge-tts/src/e2e.test.ts", note: "AC-1/2/3/4 e2e" }
  - { type: test, ref: "packages/bridge-tts/src/handler.test.ts", note: "scope/semantic error payload" }
  - { type: test, ref: "packages/bridge-tts/src/synthesizer.test.ts", note: "runtime contract + trace header + retry" }
  - { type: example, ref: "examples/tts-bridge/README.md", note: "one-command start + one-command invoke" }
---

# F011: Local TTS Bridge Example

> Status: complete | Owner: 砚砚

## Why

当前 mesh 已经验证了通用调用链（F010），但还缺一个“真实业务能力”的示例。  
把本地 TTS（Qwen3-TTS Base）作为第一批能力开放出去，可以验证：

1. 本地模型如何在 mesh 内被安全复用
2. 桥接包与本地运行时包装层如何分层
3. 远端 Agent 调用音频能力的端到端体验

## What

交付一个可复现的 TTS 示例链路：

`Caller Agent -> Mesh Hub -> TTS Bridge -> Local TTS Runtime -> Audio Result`

目标产物分三层：

1. `packages/bridge-tts/`：可发布桥接能力包（TTS invoke contract + Mesh 注册/心跳/handler 生命周期）
2. `bridges/qwen3-tts/`：本地 TTS 运行时包装层（Python 进程、依赖管理、启动脚本、配置）
3. `examples/`：最小调用演示（含调用命令与验收步骤）

边界约束：
- `packages/bridge-tts/` 不扩展 `packages/mesh-bridge` 的 Clowder-first 语义组合层
- `bridges/qwen3-tts/` 不承载 Mesh 协议逻辑，只暴露本地 HTTP 推理端点
- 两层之间通过本地 HTTP 调用（`bridge-tts -> qwen3-tts`），不是 Mesh 协议互调
- 本 feature 采用 **BYO Runtime**：默认前提是本地 Qwen3-TTS 服务已安装并可运行，agent-mesh 不负责模型安装

## Scope

In scope:
- 单能力 `tts.synthesize`（文本转语音）
- 同步返回可消费结果（`audioUrl` 或 `base64+format+durationMs`）
- 与现有 Hub identity/scope/audit 机制集成

Out of scope:
- 多音色市场化能力编排
- 流式音频分片
- 第三方账单计费系统
- 模型权重下载、CUDA/驱动安装、conda/venv 环境初始化

## Acceptance Criteria

- [x] AC-1: `bridge-tts` 通过标准 `HELLO + heartbeat` 路径完成注册，Hub `CAPS` 可见 `tts.synthesize` 且状态在线。
- [x] AC-2: 远端 Agent 通过 mesh invoke 成功拿到可播放音频结果（URL 或 base64）。
- [x] AC-3: scope 不足时返回可审计拒绝事件（非 500，语义化错误码）。
- [x] AC-4: 端到端 traceId 可追踪（caller/hub/bridge/runtime 全链路可对齐）。
- [x] AC-5: 文档提供“一条命令启动 + 一条命令调用”的最小复现步骤。

## Dependencies

- F002（Plugin Adapter）提供 runtime 语义映射基础
- F004（Node Liveness）提供在线状态语义

参考（非阻塞）：
- F010（Mesh-Cat Cafe Integration）提供实战调用链和 sidecar 经验

## Registration Path

本 feature 采用 **标准节点路径**：

- `HELLO` 完成身份握手与注册
- `heartbeat` 维持 liveness
- `INVOKE` handler 调用本地 `qwen3-tts` HTTP 推理接口

不采用 `/v1/adapters/register` 作为主注册路径。

## Risk

- 本地模型资源压力（CPU/GPU/内存）导致调用抖动
- 音频结果格式不统一导致调用方接入成本上升
- 本地运行时包装层与可发布桥接层边界不清，造成目录治理回退

## Delivery Evidence (2026-04-10)

- `pnpm --filter @agent-mesh/bridge-tts build`
- `pnpm --filter @agent-mesh/bridge-tts test`
- `pnpm --filter @agent-mesh/bridge-tts lint`
- `packages/bridge-tts/src/e2e.test.ts` 新增端到端用例，覆盖 AC-1/2/3/4：
  - HELLO + heartbeat 后 CAPS 可见 `tts.synthesize` 且状态 `online`
  - caller invoke `tts.synthesize` 成功返回音频 payload
  - 非 `tts` scope 返回语义化错误 `SCOPE_INSUFFICIENT`（非 500）
  - runtime 收到的 `x-trace-id` 与 invoke result `traceId` 对齐

## Post-Delivery TODO (Ops Hardening)

- [ ] TODO-H1: `bridge-tts` 在 Hub 重启或连接中断后自动完成 re-HELLO（无需人工重启 bridge 进程）
  - 现状：Hub 重启后，bridge 进程可能仍在运行，但 Hub 侧节点状态会回到 offline，调用返回 503
  - 目标：bridge 检测到 heartbeat/INVOKE 通道失效时自动重握手并恢复在线状态
- [ ] TODO-H2: 增加远端可发现的音色目录能力（`tts.listVoices` 或 CAPS metadata 扩展）
  - 现状：远端只能通过文档/约定知道 `voice` 可选值，无法动态探测
  - 目标：调用方可在调用 `tts.synthesize` 前先查询可用音色、默认值与参数约束

## Open Questions

1. TTS 结果主输出应优先 `audioUrl` 还是 `base64`？
2. 本地桥接包装层首版用 Python 进程还是 Node 子进程管理？
3. 失败重试策略是在 bridge 层处理，还是调用方自行处理？
4. 首版是否需要固定 voice/profile，还是开放可选参数（voice/style/speed）？

## 需求点 Checklist

- [ ] 需求是否清晰区分了“可发布桥接包”和“本地运行时包装层”？
- [ ] AC 是否都可通过日志/输出/命令复现实证？
- [ ] 安全边界（scope/revoke/audit）是否有明确验收条目？
- [ ] 示例是否足够小（5 分钟内可跑通）？
