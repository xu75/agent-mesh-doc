---
feature_ids: [F011]
related_features: [F002, F004, F010]
topics: [tts, bridge, local-model, capability, example]
doc_kind: spec
created: 2026-04-02
---

# F011: Local TTS Bridge Example

> Status: spec | Owner: 砚砚

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

1. `packages/`：可发布桥接能力包（TTS invoke contract + bridge lifecycle wiring）
2. `bridges/`：本地 TTS 运行时包装层（进程/配置/启动脚本）
3. `examples/`：最小调用演示（含调用命令与验收步骤）

## Scope

In scope:
- 单能力 `tts.synthesize`（文本转语音）
- 同步返回可消费结果（`audioUrl` 或 `base64+format+durationMs`）
- 与现有 Hub identity/scope/audit 机制集成

Out of scope:
- 多音色市场化能力编排
- 流式音频分片
- 第三方账单计费系统

## Acceptance Criteria

- [ ] AC-1: Hub `CAPS` 可见 `tts.synthesize`，并能稳定显示在线状态。
- [ ] AC-2: 远端 Agent 通过 mesh invoke 成功拿到可播放音频结果（URL 或 base64）。
- [ ] AC-3: scope 不足时返回可审计拒绝事件（非 500，语义化错误码）。
- [ ] AC-4: 端到端 traceId 可追踪（caller/hub/bridge/runtime 全链路可对齐）。
- [ ] AC-5: 文档提供“一条命令启动 + 一条命令调用”的最小复现步骤。

## Dependencies

- F010（Mesh-Cat Cafe Integration）提供实战调用链和 sidecar 经验
- F002（Plugin Adapter）提供 runtime 语义映射基础
- F004（Node Liveness）提供在线状态语义

## Risk

- 本地模型资源压力（CPU/GPU/内存）导致调用抖动
- 音频结果格式不统一导致调用方接入成本上升
- 本地运行时包装层与可发布桥接层边界不清，造成目录治理回退

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

