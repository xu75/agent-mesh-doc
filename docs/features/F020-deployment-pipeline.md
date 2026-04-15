---
feature_ids: [F020]
related_features: [F017]
topics: [deployment, ci-cd, packaging, bridge-install]
doc_kind: spec
created: 2026-04-15
layer: infrastructure
owner_module: scripts, plan-pool
status: draft
---

# F020: 一键部署 + Bridge 安装包发布

> Status: **draft** | 来源: 铲屎官 2026-04-15 提出

## 1. 问题

当前部署和安装存在以下痛点：

| 问题 | 现状 | 预期 |
|---|---|---|
| 服务端部署 | 手动 rsync + ssh restart，容易遗漏 | 服务器上一条命令升级到 GitHub 最新版 |
| Bridge 安装包 | download page 存在但 tar.gz 不存在（404） | 用户从 download page 下载即可运行 |
| 环境隔离 | 只有 mesh-hub.xyz（正式），无测试环境 | 正式域名 + 独立 staging 部署脚本 |
| 版本管理 | 无版本号、无 changelog | 语义版本 + 部署记录 |

**架构决策（CVO 2026-04-15）**：
- Bridge 采用**统一包 + 多 provider 配置**，后续支持 Codex 等其他 plan 时不新建包
- Bridge 防篡改（签名验证）作为独立 feature，不阻塞本 spec

## 2. 架构

```
本地开发 → git push → GitHub (main)
                          │
            ┌─────────────┼─────────────┐
            ↓             ↓             ↓
      test.mesh-hub.xyz  mesh-hub.xyz  download page
      (staging/QA)       (production)  (bridge 安装包)
```

## 3. 服务端部署（已完成 MVP）

### 3.1 一键部署脚本

`scripts/deploy-prod.sh` — 已实现并验证。

```bash
# 从本地触发：
ssh mesh-hub 'sudo bash /opt/mesh-hub/scripts/deploy-prod.sh'

# 或在服务器上直接执行：
sudo bash /opt/mesh-hub/scripts/deploy-prod.sh
```

流程：git pull → pnpm install → build → restart → health check。

### 3.2 环境隔离（待实现，CVO 决策：正式域名 + 独立脚本）

| 环境 | 域名 | 部署目录 | 脚本 | 用途 |
|---|---|---|---|---|
| Production | mesh-hub.xyz / api.mesh-hub.xyz | /opt/mesh-hub | deploy-prod.sh | 正式服务 |
| Staging | 正式域名，独立端口 | /opt/mesh-hub-staging | deploy-staging.sh | 版本验证、回归测试 |

**Staging 环境规则**：
- 独立的 systemd service（`mesh-hub-staging`、`plan-pool-staging`）
- 独立的端口（Hub: 3020, Pool: 3021）
- 独立的 SQLite 数据库
- 不使用独立域名，通过端口区分
- 部署不影响 production

**部署流程**：
1. 先部署到 staging：`ssh mesh-hub 'sudo bash /opt/mesh-hub-staging/scripts/deploy-staging.sh'`
2. 在 staging 验证（手动或自动 smoke test）
3. 确认后部署到 production：`ssh mesh-hub 'sudo bash /opt/mesh-hub/scripts/deploy-prod.sh'`

## 4. Bridge 安装包（待实现）

### 4.1 问题

download page（`/download`）已有完整 UI，但实际的 tar.gz 包不存在。用户下载到 404 HTML，tar 解压失败。

### 4.2 包格式（方案 B：轻量包 + install 脚本引导，CVO 决策）

tar.gz 只含编译产物和配置，用户通过 install 脚本引导安装依赖：

```
plan-bridge-oauth-{version}-{platform}-{arch}.tar.gz
├── package.json          # 依赖声明（用户 npm install）
├── dist/                 # 编译产物
│   ├── bridge.js
│   ├── proxy.js
│   ├── streaming-proxy.js
│   ├── token-manager.js
│   ├── env-normalizer.js
│   └── index.js
├── install.sh            # 引导安装脚本（检查环境 + npm install + 配置提示）
├── start.sh              # 启动脚本（设置环境变量 + node dist/bridge.js）
└── README.md             # 快速入门
```

### 4.3 支持平台

| 平台 | 架构 | 文件名 |
|---|---|---|
| macOS | arm64 | plan-bridge-oauth-{ver}-darwin-arm64.tar.gz |
| macOS | x64 | plan-bridge-oauth-{ver}-darwin-x64.tar.gz |
| Linux | x64 | plan-bridge-oauth-{ver}-linux-x64.tar.gz |

### 4.4 打包流程

`scripts/package-bridge.sh`：

1. `pnpm build`（编译 TypeScript）
2. 组装 staging 目录（dist/ + package.json + install.sh + start.sh + README.md）
3. 生成 tar.gz（每平台一个，平台差异仅在 install.sh 提示文案）
4. 产物放到 `packages/plan-pool/public/download/`（.gitignore，不提交）
5. `deploy-prod.sh` 执行时自动调用此脚本，打包并同步到服务器

### 4.5 start.sh 模板

```bash
#!/usr/bin/env bash
# Plan Bridge OAuth — Quick Start
# Prerequisites: Node.js >= 20, Claude Code with OAuth login

set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

# Check Node.js
command -v node >/dev/null || { echo "Error: Node.js not found. Install from https://nodejs.org"; exit 1; }

# Startup Gate G2: no API key
if [ -n "${ANTHROPIC_API_KEY:-}" ]; then
  echo "Error: ANTHROPIC_API_KEY is set. Bridge requires OAuth mode."
  exit 1
fi

# Prompt for Pool connection
: "${MESH_HUB_URL:?Set MESH_HUB_URL (e.g. https://api.mesh-hub.xyz)}"
: "${POOL_API_KEY:?Set POOL_API_KEY (get from team admin)}"

export MESH_HUB_URL
export POOL_NODE_URL="${POOL_NODE_URL:-$MESH_HUB_URL}"
export POOL_API_KEY
export PLAN_BRIDGE_OAUTH_IDENTITY_DIR="${PLAN_BRIDGE_OAUTH_IDENTITY_DIR:-$HOME/.plan-bridge-oauth-identity}"
export PLAN_BRIDGE_OAUTH_FINGERPRINT_POLICY="${PLAN_BRIDGE_OAUTH_FINGERPRINT_POLICY:-warn}"

exec node "$SCRIPT_DIR/dist/bridge.js"
```

### 4.6 install.sh（一键安装）

download page 已引用 `/download/install.sh`，需要实现：

```bash
curl -fsSL https://api.mesh-hub.xyz/download/install.sh | bash
```

脚本逻辑：
1. 检测 OS + arch
2. 下载对应 tar.gz
3. 解压到 `~/.plan-bridge-oauth/`
4. 提示设置环境变量
5. 提示运行 `./start.sh`

## 5. 版本管理

### 5.1 版本号

遵循语义版本（SemVer）：`{major}.{minor}.{patch}`

- `0.1.0` — 当前首个可用版本（caller-agnostic + 启动门控）
- version 写在 `package.json` 和 download page 的 `VERSION` 变量中

### 5.2 部署记录

每次 production 部署自动记录：
- deploy 脚本在成功后追加一行到 `/var/log/mesh-hub/deploy.log`
- 格式：`{timestamp} {commit} {user} {message}`

## 6. 实施优先级

| 优先级 | 任务 | 依赖 |
|---|---|---|
| P0 | Bridge tar.gz 打包 + 上传到 download 目录 | 无 |
| P0 | install.sh 实现 | tar.gz 就绪 |
| P1 | Staging 环境搭建（test.mesh-hub.xyz） | Caddy 配置 |
| P1 | deploy-staging.sh | staging 目录结构 |
| P2 | 版本号自动化（package.json → download page） | 无 |
| P2 | 部署日志 | 无 |
