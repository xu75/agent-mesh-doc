---
feature_ids: [F017]
related_features: [F017]
topics: [plan-bridge, cli-output, api-purity, response-filter, competitive-analysis, roundtable]
doc_kind: discussion
created: 2026-04-09
participants: [opus]
status: draft
---

# F017 专题讨论：无 --bare 时如何让 CLI 输出更像 Pure API

> 日期：2026-04-09 | 发起人：铲屎官
> 参与：布偶猫/宪宪(opus)
> 议题：plan-bridge-cli 在无法使用 --bare 模式的前提下，如何从 stream-json 输出中提取干净的 API 级响应

## 1. 问题定义

plan-bridge-cli 将 Claude CLI 封装为 chat-only bridge，通过 `--output-format stream-json` 获取结构化输出。当前参数已做了基础抑制：

```
--tools ""               # 禁用所有工具
--setting-sources ""     # 禁用 .claude 设置
--strict-mcp-config      # 严格 MCP 配置
```

**但 stream-json 输出仍然包含大量非文本噪声**：

| 噪声类型 | 典型事件 | 占比估计 | 能否在源头消除 |
|----------|---------|---------|-------------|
| system_prompt 注入 | CLI 自带系统提示 | 一次性 | 不能（非 bare 模式必有） |
| thinking 块 | `type: "thinking"` | 可达 30-50% | 可通过参数关闭 |
| tool_use 幻觉 | 模型"想"调工具 | 偶发但扰人 | --tools="" 未必 100% 阻止 |
| content_block 元数据 | start/stop/delta 事件 | 频繁 | 可在解析层过滤 |
| result 元数据 | session/版本/成本信息 | 一次性 | 有用，不算噪声 |

**核心问题**：消费者（caller 节点）期望收到类似 `POST /v1/messages` 的干净 API 响应，而不是 CLI 的原始事件流。

---

## 2. 竞品全景：业界如何解决 "CLI/订阅 → 干净 API" 问题

### 2.1 API Key 池化网关（传统路线）

#### songquanpeng/one-api (~20k stars)

| 维度 | 做法 |
|------|------|
| 语言 | Go (Gin) |
| 核心思路 | 多 Provider API key 统一为 OpenAI 兼容接口 |
| 响应处理 | SSE StreamHandler 逐行解析，按 relayMode 分发 |
| Claude 适配 | `ResponseClaude2OpenAI`: tool_use → function_call，stop_reason 映射 |
| 去噪策略 | **无主动过滤** — 纯协议转换，不处理 thinking/空块 |
| Tool 处理 | 累积 `input_json_delta`，补齐空参数 `"{}"` |
| 局限 | 只处理 API 响应，不涉及 CLI 输出 |

**评价**：one-api 面对的是已经干净的 API 响应，不存在 CLI 噪声问题。它的价值在于**协议转换**（Claude → OpenAI 格式），而非响应清洗。

#### Calcium-Ion/new-api (~20k stars, one-api 增强分支)

| 维度 | 做法 |
|------|------|
| 语言 | Go (Gin) |
| 核心增强 | 比 one-api 多了 reasoning/thinking 处理 |
| thinking 处理 | `reasonmap` 模块：Claude thinking → OpenAI reasoning_content 字段映射 |
| 拒绝检测 | `maybeMarkClaudeRefusal()`: 检测 stop_reason="refusal" 并标记上下文 |
| Web Search | 支持 Claude web_search tool 的 max_uses 配置 (Low/Medium/High) |
| 协议转换 | 更完整的 Claude ↔ OpenAI 双向转换，支持 beta query 参数 |
| 局限 | 同 one-api — 只处理 API 响应 |

**事件映射表**（`StreamResponseClaude2OpenAI` 中的显式映射）：

| Claude 事件 | OpenAI 输出 | 处理 |
|-------------|-----------|------|
| `message_start` | 空 delta + `role: assistant` | 转换 |
| `content_block_start` (text) | 空/初始 text delta | 转换 |
| `content_block_start` (tool_use) | tool_calls entry + name | 转换 |
| `content_block_delta` (text_delta) | content delta | 转换 |
| `content_block_delta` (input_json_delta) | tool_calls argument chunk | 转换 |
| `content_block_delta` (thinking_delta) | `reasoning_content` 字段 | **降级而非丢弃** |
| `content_block_delta` (signature_delta) | `reasoning_content: "\n"` | signature 丢弃 |
| `message_delta` | `finish_reason` | 转换 |
| `message_stop` | **nil — 事件丢弃** | 丢弃 |
| 其他 | **nil — 事件丢弃** | 丢弃 |

**额外能力**：自动 `-thinking` 后缀检测 → `extended_thinking` 注入；effort-level 映射；`patchClaudeMessageDeltaUsageData` 回填 Bedrock 缺失的 usage 字段。

**评价**：new-api 的事件映射表是我们设计 filter pipeline 的**最佳参考** — 它不是简单丢弃 thinking，而是**转换为下游协议的等价字段**（`reasoning_content`）。这个思路可以借鉴：对于 thinking 块，与其丢弃不如转换为 caller 可理解的格式。

#### nai-degen/oai-reverse-proxy (ORP)

| 维度 | 做法 |
|------|------|
| 语言 | TypeScript (Express + http-proxy-middleware) |
| 核心思路 | 四阶段 Transform 管道 + 协议转换 |
| 托管 | GitLab (gitgud.io)，GitHub 有镜像 (cg-dot/oai-reverse-proxy) |
| **流式处理** | `proxyRes → Decoder → SSEStreamAdapter → SSEMessageTransformer → client` |
| 事件过滤 | `asAnthropicChatDelta()` **只透传** `content_block_start` 和 `content_block_delta`，其余全部丢弃 |
| Tool/Thinking | **静默丢弃** — tool_use/thinking 块落入 else 分支，仅 log warning |
| 非文本内容 | 替换为 `[Omitted multimodal content of type X]` |
| 请求净化 | Zod schema 校验 + `.strip()` 去除未知字段 + `max_tokens` 上限裁剪 |
| Header 净化 | 移除 origin/referer/x-forwarded-for/cf-connecting-ip 防止身份泄露 |

**ORP 的事件过滤表**（`asAnthropicChatDelta` 中的隐式白名单）：

| Claude 事件 | 处理 |
|-------------|------|
| `content_block_start` | 透传（提取 text） |
| `content_block_delta` | 透传（提取 delta.text） |
| `message_start` | **丢弃** |
| `message_delta` | **丢弃** |
| `message_stop` | **丢弃** |
| `content_block_stop` | **丢弃** |
| `ping` | **丢弃** |
| tool_use / thinking | **丢弃**（log warning） |

**评价**：ORP 表面看是"反代"，实际有**相当激进的过滤逻辑** — 只保留文本 delta，其他一律丢弃。这和 sub2api 的显式 5 重过滤形成对比：ORP 用**白名单隐式过滤**（只声明保留什么），sub2api 用**黑名单显式过滤**（声明删除什么）。白名单更安全——未知事件默认丢弃。

### 2.2 订阅 → API 转换（CLI Proxy 新兴路线）

这类项目才是我们的**直接竞品**——面临的问题几乎一模一样。

#### Wei-Shaw/sub2api (~11k stars) — 最成熟的方案

| 维度 | 做法 |
|------|------|
| 语言 | Go (Gin + Ent ORM + Wire DI) |
| 核心思路 | 服务端网关，订阅 → API key 池，统一 OpenAI 兼容接口 |
| **去噪体系** | **5 重过滤**（业界最完整） |

**5 重过滤体系详解**：

| # | 过滤层 | 函数 | 做法 |
|---|-------|------|------|
| 1 | Thinking Block | `FilterThinkingBlocks()` | 未启用 thinking → 删除；重试 → 降级为 text；redacted → 删除 |
| 2 | 空内容块 | `stripEmptyTextBlocksFromSlice()` | 递归清理 `{"type":"text","text":""}` 含 tool_result 内嵌 |
| 3 | Tool Schema | `CodexToolCorrector` | call_id/id 矫正，fc_ 前缀归一化 |
| 4 | System Prompt | `sanitizeSystemText()` | 非官方身份 prompt → 替换为标准 Claude Code banner |
| 5 | Response Header | `compileResponseHeaderFilter()` | 白名单透传，隐藏内部状态 |

**额外技术**：
- 检测 Claude Code 客户端（`claudeCodePromptPrefixes` 匹配 system 开头）
- 系统提示迁移（v0.1.109: system 字段 → messages 数组）
- 大型注入指令精简（v0.1.93: 移除 /response 路径的冗余指令）
- OpenCode bridge prompt 注入（工具名映射 apply_patch → edit 等）

**评价**：sub2api 是**去噪做得最深**的项目，但它走的是"伪装流量"路线（替换 system prompt、伪造 User-Agent），合规风险大。其 thinking 块三策略（删除/降级/清理）非常实用。

#### horselock/claude-code-proxy — OAuth PKCE 路线

| 维度 | 做法 |
|------|------|
| 语言 | Python |
| 认证 | OAuth PKCE 2.0 直连 claude.ai + 自动 token 刷新 |
| 降级认证 | 读取 `~/.claude/.credentials.json`（Claude Code 本地凭证） |
| **请求净化** | header 注入 + cache_control 中 ttl 字段移除 + system prompt 前置 "You are Claude Code..." |
| 参数过滤 | `filter_sampling_params`: 阻止 Sonnet 4.5 同时收到 temperature 和 top_p |
| 响应处理 | **无主动过滤** — 聚焦请求改写而非响应清洗 |

**评价**：这个项目的思路是**从请求侧入手**——通过伪装请求头让 Anthropic 后端认为是合法 Claude Code 客户端，从而直接获得 API 级响应。不走 CLI，直接调 API 端点。缺点是**完全依赖 OAuth token**，一旦 Anthropic 封堵就失效。

#### Meridian — SDK query() 路线

| 维度 | 做法 |
|------|------|
| 语言 | TypeScript |
| 核心思路 | 通过 Claude Code SDK 的 `query()` 函数发起请求 |
| 认证 | SDK 内置 OAuth，自动刷新 |
| 协议转换 | `openai.ts` 模块：Anthropic → OpenAI 双向转换 |
| Tool 处理 | OpenCode adapter 内部执行 / ForgeCode passthrough 模式 |
| 去噪 | **无主动过滤** — "Play nice with the SDK, let Anthropic optimize" |
| Session | 指纹哈希 + 文件持久化 + SDK compaction |

**评价**：Meridian 代表了**最合规的路线** — 直接用 SDK 编程接口，不封装 CLI 进程。但这条路要求 SDK 暴露了 query() 函数，并且 SDK 的输出本身就比 CLI 干净。

#### Claude Max API Proxy — Rust CLI Wrapper

| 维度 | 做法 |
|------|------|
| 语言 | Rust |
| 核心思路 | CLI subprocess + NDJSON 解析 + 协议转换 |
| 架构 | `openai_to_cli.rs` → CLI → `cli_to_openai.rs` |
| Session | `~/.claude-code-cli-sessions.json` 持久化 |
| 模型映射 | 灵活：`claude-opus-4` / `opus` / 带日期后缀均可 |
| 去噪 | **通过协议转换隐式过滤** — cli_to_openai 只提取有效字段 |

**评价**：和我们的 plan-bridge-cli 最接近——都是 CLI subprocess 方式。它的去噪策略是**转换时自然丢弃**：cli_to_openai 转换器只映射已知字段，未知事件自动忽略。

#### CLIProxyAPI — 多 Provider CLI 统一

| 维度 | 做法 |
|------|------|
| 语言 | TypeScript |
| 核心思路 | Claude Code / Gemini CLI / OpenAI Codex → 统一 API |
| 认证 | 各 Provider 的 OAuth 流程 |
| 路由 | Round-robin 多账号负载均衡 |
| 协议 | /v1/messages, /v1/chat/completions, /v1beta/models/... |
| 去噪 | 通过格式翻译层隐式过滤 |

**评价**：多 Provider 支持是独特卖点。去噪策略和 Claude Max API Proxy 类似——格式翻译层自然丢弃非标准事件。

#### OpenClaw Claude Bridge — XML Tool Call 解析

| 维度 | 做法 |
|------|------|
| 核心思路 | OpenAI 格式 ↔ Claude CLI，OpenClaw 是上游 agent |
| **Tool 分离** | Bridge 不执行工具，只翻译。检测 `<tool_call>` XML 块 → OpenAI `tool_calls` 格式 |
| **Claude 原生工具禁用** | `--tools ""` — 和我们一样 |
| **System Prompt 注入** | 动态注入工具调用指令，基于上游传入的 tools 数组 |
| 响应分类 | 三类：纯文本 / tool_call XML / thinking 块，分别处理 |

**评价**：OpenClaw Bridge 的**三类响应分离**策略最值得借鉴——它不是粗暴过滤，而是把混合输出拆分为三个语义通道，各自走不同的处理路径。

### 2.3 其他 CLI-to-API 项目（补充覆盖）

#### sethschnrt/claude-max-api-proxy 系列（JS/Rust，多个变种）

| 维度 | 做法 |
|------|------|
| 变种 | sethschnrt (JS), theserverlessdev (JS), thhuang (Rust), mattschwen (JS) |
| 核心思路 | CLI subprocess + NDJSON 解析 + 协议转换（同 "Max API Proxy" 小节） |
| 认证 | **零凭证处理** — CLI 自己管理 OS keychain 中的 OAuth token |
| 重要发现 | **2026-04-04 Anthropic 政策变化**：对第三方 CLI wrapper 的订阅额度增加了额外计费 |

#### zacdcook/openclaw-billing-proxy — 7 步请求伪装（反面教材）

| 步骤 | 操作 | 目的 |
|------|------|------|
| 1 | 替换 auth token 为 Claude Code OAuth token | 身份伪装 |
| 2 | 清洗 ~30 个触发词（"OpenClaw"→无害替代） | 指纹消除 |
| 3 | 重命名 29 个工具名为 PascalCase（"exec"→"Bash"） | 格式对齐 |
| 4 | 裁剪 ~28,000 字符系统提示 → ~500 字符自然语言 | 减小请求体 |
| 5 | 移除工具描述 | 降低指纹 |
| 6 | 重命名 Schema 属性 | 格式对齐 |
| 7 | 注入 84 字符 Claude Code billing identifier | 计费伪装 |

**评价**：技术上最激进，**合规上完全不可取**。但它暴露了一个事实：Anthropic 有服务端检测机制（system prompt 前缀匹配、工具名特征），这意味着我们不走伪装路线是正确的。

#### starbaser/ccproxy — 规则引擎路由

| 维度 | 做法 |
|------|------|
| 核心思路 | LiteLLM + hook 管道 (rule_evaluator → model_router → forward_oauth) |
| 路由规则 | MatchModelRule / ThinkingRule / TokenCountRule / MatchToolRule |
| 去噪 | **无** — 聚焦请求路由，响应透传 |

**评价**：有趣的正交能力——按请求特征智能路由到不同模型。这对 pool-node 的路由策略有借鉴。

#### shinglokto/openclaw-claude-bridge — Extended Thinking 排序修复

| 维度 | 做法 |
|------|------|
| Tool Call | `<tool_call>` XML 块 → OpenAI `tool_calls` JSON |
| Thinking | `reasoning_effort` → Claude `--effort` flag 映射 |
| **关键修复** | thinking 块必须在 assistant 消息最前面（否则 API 400） |

> **⚠️ 已知约束**：当 Extended Thinking 启用时，assistant 消息**必须**以 `thinking` 或 `redacted_thinking` 块开头。多个 bridge 项目（含 opencode #9364）因消息重组时顺序错误而出 bug。修复方式：分离 reasoning parts 和 other parts，重组时 thinking 放最前。

**对 F017 的启示**：如果我们的 filter pipeline 未来需要重组消息（而非简单丢弃），必须注意这个排序约束。

### 2.4 Stream-JSON 解析器生态

#### udhaykumarbala/claude-code-parser — 最佳 TypeScript 解析器

| 维度 | 做法 |
|------|------|
| 语言 | TypeScript，零依赖，11 kB |
| 核心 | `parseLine()` 将 NDJSON → 类型化 `ClaudeEvent` |
| **去重** | `Translator` 类处理 `--verbose` 模式的累积快照问题（每条消息包含完整历史） |
| **归一化** | 输出 7 种 relay 事件：`text_delta` / `thinking_delta` / `tool_use` / `tool_result` / `session_meta` / `turn_complete` / `error` |
| 多 agent | 通过内容指纹识别处理 sub-agent 交错输出 |

**评价**：几乎是我们需要的 drop-in 解决方案。它的 7 种 relay 事件类型定义比我们当前的 `ParsedStreamLine` 更精细。

#### shibuido/claude-stream-json-parser (Rust)

明确的 filter 示例 `filter_messages.rs`，5 级 verbosity 控制。

#### Khan/format-claude-stream

最简 CLI 管道过滤器：`claude ... | format-claude-stream`。

#### 官方 jq 模式（text-only streaming）

```bash
claude -p "..." --output-format stream-json --verbose | \
  jq -rj 'select(.type == "stream_event" and .event.delta.type? == "text_delta") | .event.delta.text'
```

### 2.5 输出清洗工具

#### Clean Clode (cleanclode.com)

Web 工具，将 Claude Code 终端输出（含 ANSI、管道、空白）清洗为干净文本。面向人类阅读，非程序化场景。

---

## 3. 技术策略对比矩阵

| 策略 | sub2api | claude-code-proxy | Meridian | Max API Proxy | OpenClaw Bridge | **F017 当前** |
|------|---------|-------------------|----------|---------------|-----------------|--------------|
| **方法** | 服务端 5 重过滤 | 请求侧伪装 | SDK query() | CLI→协议转换 | XML 解析+分离 | --tools="" |
| Thinking 处理 | 删除/降级/清理 | 不处理 | SDK 透传 | 转换时丢弃 | 单独通道 | **未处理** |
| Tool 幻觉 | Schema 矫正 | 不涉及 | SDK 管理 | 转换时丢弃 | XML 解析 | 依赖 --tools="" |
| 空内容块 | 递归清理 | 不处理 | SDK 管理 | 转换时丢弃 | 不处理 | **未处理** |
| System Prompt | 替换/伪装 | 注入标准 banner | SDK 透传 | 不处理 | 动态注入 | 环境变量清洗 |
| 协议输出 | OpenAI 兼容 | 原生 Anthropic | 双协议 | 双协议 | OpenAI 兼容 | 原始 stream-json |
| 合规性 | 低（伪装流量） | 低（OAuth 劫持） | 中（SDK 合法调用） | 中（CLI 封装） | 中（CLI 封装） | **高（自用 CLI）** |

---

## 4. 官方现状与社区呼声

**Anthropic 官方至今（2026-04）未提供 `--quiet` 或 `--text-only` flag**。

社区需求追踪：
- **anthropics/claude-code#9340** — `--quiet` flag 请求，25+ upvotes，6 个重复 issue（#9512, #11685, #12544, #12728, #13065, #14684）
- **anthropics/claude-code#21246** — 请求 `--verbosity quiet` / `--diff=summary`
- **anthropics/claude-code#24311** — 复杂 turn 中文本被 tool call 淹没

**已知的官方替代方案**：
| 方案 | 效果 | 局限 |
|------|------|------|
| `--output-format json` + `.result` | **零噪声** — 直接拿纯文本 | 无 streaming，只有最终结果 |
| `--output-format stream-json` + jq 过滤 | 可实时获取 text_delta | 需要自己写过滤逻辑 |
| `--system-prompt "..."` | 完全替换默认系统提示 | 可能影响 CLI 行为 |
| `--bare` | 跳过 hooks/MCP/skills | 需要 ANTHROPIC_API_KEY |
| `DISABLE_NON_ESSENTIAL_MODEL_CALLS=1` | 减少非必要调用 | 仅边际改善 |

**对 F017 的关键启示**：对于不需要 streaming 的场景（如简单 Q&A），`--output-format json` + `.result` 是**零成本的终极解**。

## 5. 归纳：四大技术流派

### 流派 A：源头抑制（Source Suppression）
> 在 CLI 参数/环境层面减少噪声产生

- `--tools ""` 禁用工具定义
- `--bare` 跳过 hooks/plugins/MCP
- 清洁环境变量（防止子进程继承父 context）
- System prompt 覆盖（注入纯 chat 指令）

**我们当前位置**：已做到 A 流派的基础水平。`--output-format json` 可作为非 streaming 场景的终极方案。

### 流派 B：解析层过滤（Parse-time Filtering）
> 在 stream-json 解析阶段主动过滤/转换噪声事件

- Thinking 块：删除 / 降级为 text / 转为 reasoning_content
- 空内容块：递归清理
- Tool use 事件：丢弃 / 转换为 caller 可理解的格式
- content_block_start/stop：只保留有效 delta

**业界标杆**：sub2api（5 重过滤）、new-api（reasonmap）

### 流派 C：协议转换隐式丢弃（Protocol Translation）
> 在输出格式转换时自然丢弃非标准事件

- CLI stream-json → Anthropic Messages API 格式
- CLI stream-json → OpenAI Chat Completions 格式
- 只映射已知字段，未知事件自动忽略

**业界标杆**：Claude Max API Proxy（cli_to_openai.rs）、OpenClaw Bridge（三通道分离）

### 流派 D：XML Tool Passthrough（工具语义重定向）
> 禁用原生 tool_use，通过系统提示注入 XML 格式的工具调用指令

- `--tools ""` 禁用原生工具
- 系统提示动态注入工具定义（基于 caller 传入的 tools 数组）
- 模型输出 `<tool_call>` XML 块而非原生 tool_use content block
- Bridge 解析 XML → 转为 caller 期望的格式

**业界标杆**：OpenClaw Claude Bridge

**评价**：彻底消除 tool_use/tool_result 噪声。代价是放弃 Claude 原生 tool use，改用 prompt-based tool calling。对 chat-only bridge（我们的用例），这反而是优势。

---

## 6. F017 plan-bridge-cli 的策略建议

### 推荐路线：A + B 组合，Phase 2 考虑 C/D

#### Phase 1（立即可做）— 完善源头抑制

| 动作 | 难度 | 收益 |
|------|------|------|
| 验证 `--tools ""` 是否 100% 阻止 tool_use 输出 | 低 | 确认假设 |
| 探索 `--no-thinking` 或等价参数 | 低 | 消除最大噪声源 |
| 对非 streaming 场景改用 `--output-format json` + `.result` | **极低** | **零噪声，零代码改动** |
| 探索 `--system-prompt` 完全替换（而非 `-a` 追加） | 低 | 从源头消除 CLI 默认系统提示 |

#### Phase 2（短期）— 实现解析层过滤

```
Stream Event Filter Pipeline:

  stream-json line
       │
       ├─ type: "system" ──────────► 丢弃（或存为 metadata）
       ├─ type: "thinking" ────────► 丢弃（或降级为 text，由 caller 控制）
       ├─ type: "assistant" ───────► 提取 text content blocks
       │    ├─ content[].type: "text" ───► 保留
       │    ├─ content[].type: "tool_use" ► 丢弃（或降级为 text 描述）
       │    └─ content[].type: 其他 ─────► 丢弃
       ├─ type: "result" ─────────► 保留（usage + cost）
       └─ 其他 ───────────────────► 丢弃
```

实现位置：`cli-wrapper.ts` 新增 `filterStreamEvents()` 函数，在 `parseStreamJsonLine` 之后、`extractUsageFromLines` 之前。

**开源参考**：`udhaykumarbala/claude-code-parser`（TypeScript/0 依赖/11kB）已实现完整的去重 + 归一化，可作为设计参考或直接引入。

#### Phase 3（中期）— 协议转换输出

将 bridge 输出从原始 `lines[]` 升级为结构化 API 响应：

```typescript
interface PurifiedResponse {
  text: string;          // 纯文本结果
  usage: UsageInfo;      // token 用量
  model: string;         // 使用的模型
  thinking?: string;     // 可选：thinking 内容（caller 可选择是否要）
  metadata?: {           // 可选：调试信息
    sessionId: string;
    durationMs: number;
  };
}
```

### 不推荐的做法

| 做法 | 原因 |
|------|------|
| 伪装 System Prompt（sub2api 路线） | 合规风险，违背 P1 终态原则 |
| OAuth token 劫持（claude-code-proxy 路线） | 脆弱，Anthropic 可随时封堵 |
| 完全透传不过滤 | caller 体验差，噪声传播（注：ORP 实际并非透传，而是白名单过滤） |

---

## 7. 讨论问题

| # | 问题 | 影响 | 建议 |
|---|------|------|------|
| Q1 | `--tools ""` 是否完全阻止了 tool_use 事件？还是模型仍可能输出 tool_use content block？ | 决定是否需要解析层兜底 | 实测验证 |
| Q2 | Claude CLI 是否有 `--no-thinking` 或等价参数？ | 消除最大噪声源 | 查 CLI help |
| Q3 | thinking 内容对 caller 是否有价值？（用于推理链展示） | 决定是删除还是降级 | 由铲屎官定 |
| Q4 | 解析层过滤应该放在 bridge 侧还是 caller 侧？ | 架构决策 | bridge 侧过滤更合理（单点处理） |
| Q5 | 是否需要支持 OpenAI 兼容输出格式？（Phase 3 scope） | 扩大 caller 生态 | Phase 2 再议 |
| Q6 | `-a` 系统提示注入是否对 plan-bridge chat-only 场景有效？ | 源头抑制效果 | 实测验证 |

---

## 8. 竞品对比总结表（供回填圆桌主文档第 6 节）

| 项目 | Stars | 路线 | 去噪深度 | CLI 支持 | 合规性 | 借鉴价值 |
|------|-------|------|---------|---------|--------|---------|
| one-api | ~20k | API key 池化 | 无（纯协议转换） | 无 | 中 | 协议转换参考 |
| new-api | ~20k | API key 池化 | 低（reasoning 映射） | 无 | 中 | thinking→reasoning 映射 |
| oai-reverse-proxy | — | API 反代 | **中（白名单过滤）** | 无 | 中 | **白名单 > 黑名单** 设计哲学 |
| **sub2api** | ~11k | 订阅→API 网关 | **高（5 重过滤）** | 间接 | **低** | thinking 三策略、空块清理 |
| claude-code-proxy | ~1k | OAuth PKCE 代理 | 中（请求侧净化） | 间接 | 低 | cache_control 清理 |
| Meridian | — | SDK query() | 无（SDK 透传） | 通过 SDK | 中 | SDK 路线参考 |
| Max API Proxy | — | CLI Rust Wrapper | 中（转换时丢弃） | 直接 | 中 | NDJSON→API 转换 |
| CLIProxyAPI | — | 多 CLI 统一 | 中（翻译层过滤） | 直接 | 中 | 多 Provider 架构 |
| OpenClaw Bridge | — | CLI 翻译桥 | 中（三通道分离） | 直接 | 中 | **XML→tool_calls 分离** |
| openclaw-billing-proxy | — | 7 步请求伪装 | 高（全面重写） | 间接 | **极低** | 反面教材：Anthropic 有检测 |
| ccproxy (starbaser) | — | 规则引擎路由 | 无（响应透传） | 间接 | 中 | 智能路由策略 |
| openclaw-claude-bridge | — | XML→JSON 翻译 | 中（三通道） | 直接 | 中 | **thinking 排序约束** |
| **F017 bridge** | — | 本地 CLI 封装 | **低（仅源头抑制）** | 直接 | **高** | — |

---

## 9. 结论

1. **CLI subprocess 已成为 2025-2026 主流模式** — 至少 8 个独立项目走了相同路线，验证了 plan-bridge-cli 的方向正确
2. **业界尚无"标准答案"** — 每个项目都在摸索，说明这是真实痛点
3. **F017 的合规优势不可放弃** — 不走 OAuth 劫持/流量伪装路线（Anthropic 有服务端检测，且 2026-04-04 已开始对第三方 CLI wrapper 额外计费）
4. **白名单过滤优于黑名单** — ORP 的经验：声明"只保留什么"比"删除什么"更安全，未来新事件类型默认丢弃
5. **Sub2API 的 5 重过滤值得借鉴但需简化** — 我们只需 thinking + 空块 + tool_use 三项
6. **OpenClaw 的三通道分离是最优雅的思路** — 把混合输出拆为 text/tool/thinking 三路
7. **Extended Thinking 排序约束** — 若未来需要重组消息，thinking 块必须在 assistant 消息最前面（多个项目踩坑）
8. **短期 ROI 最高的动作**：实测 `--tools ""` 效果 + 实现白名单式解析层 filter
