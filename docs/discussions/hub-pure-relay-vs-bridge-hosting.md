---
topics: [architecture, hub, bridge, relay, coupling]
doc_kind: discussion
created: 2026-04-05
status: open
participants: [铲屎官, 宪宪]
---

# 讨论：Hub 应该是纯中继还是需要托管 Bridge？

## 背景

F011（TTS Bridge）上线后暴露的架构问题：

当前部署拓扑：
```
本地 Mac                          Hub 服务器 (mesh-hub.xyz)
┌──────────────┐    SSH tunnel    ┌─────────────────────────────────┐
│ tts-api.py   │ ◄──────────────► │ qwen3-tts/server.py  (port 5001)│
│ (port 9879)  │                  │ bridge-tts           (port 3008)│
│              │                  │ mesh-hub             (port 3004)│
└──────────────┘                  └─────────────────────────────────┘
```

问题：每挂一个新 Agent 服务，Hub 服务器上要新增一个定制 bridge 后台 + 一个 runtime wrapper。这导致：
- Hub 从通用 relay 退化成"托管平台"
- 功能拓扑不干净，模块耦合紧
- 无法 scale（每加一个服务都要动 Hub 服务器）
- ref_audio 等资源文件应该在服务本地，却跨机器传路径

## 铲屎官原话

> bridge 如果是运行在 hub 上的，那不应该有这些 ref 文件。也牵涉到另一个问题，就是我们的 hub 按理说是通用的，不能因为有一个新的 Agent 服务，就增加一个后台，功能拓扑就不干净了，模块之间耦合太紧。是我们 hub 层面没做好通用 relay 必须定制，还是功能本来应该放在本地而不是 hub？

## 方向 A：Bridge 本地化（推荐）

```
本地 Mac                                    Hub 服务器
┌──────────────────────────┐                ┌──────────────┐
│ tts-api.py    (port 9879)│                │ mesh-hub     │
│ bridge-tts    (port 3008)│ ──HELLO/HB──►  │ (pure relay) │
│ qwen3-tts     (port 5001)│                │              │
│ ref_audio/               │                │              │
└──────────────────────────┘                └──────────────┘
```

- Bridge 和服务跑在同一台机器
- Bridge 作为 mesh node 主动连到 Hub（HELLO + heartbeat）
- Hub 只做 relay，不跑任何业务代码
- 资源文件（ref_audio）留在本地，没有跨机器路径问题
- 新服务接入不需要碰 Hub 服务器

需要解决：本地无固定 IP → bridge 需要保持长连接或反向隧道

## 方向 B：Hub 通用 HTTP 适配层

```
任意机器                                    Hub 服务器
┌──────────────────────┐                    ┌──────────────────────┐
│ tts-api.py (有公网IP) │ ◄───HTTP relay──  │ mesh-hub             │
│                      │                    │ + generic HTTP proxy │
└──────────────────────┘                    └──────────────────────┘
```

- Hub 内置通用 HTTP 适配器：把 INVOKE 翻译成标准 HTTP 调用
- 不需要 bridge 代码，config 里配 `callbackUrl` + 一个 payload 映射规则即可
- 适合有公网可达地址的服务

需要解决：payload 格式转换规则、错误码映射、scope 语义如何在 HTTP 层面表达

## 方向 C：混合（A + B）

- 有公网地址的服务 → 方向 B（Hub 直接 relay HTTP）
- 本地无固定 IP 的服务 → 方向 A（bridge 本地化 + 反向隧道/长连接）
- Hub 不托管任何业务代码

## 待讨论

1. F011 的 TTS 现在要不要迁移到方向 A？还是先记录，等下一个服务接入时一并重构？
2. 方向 B 的"通用 HTTP 适配"和 F014（混合路由）是否是同一件事？
3. Hub 上现有的 bridge-tts / qwen3-tts systemd 服务是否要清理？

## 结论

（待讨论后填写）
