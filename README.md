# DeepHourAI

企业级 AI 智能体平台。基于 LangGraph 的 Super-Agent 系统，具备沙箱执行、持久化记忆、子智能体委派和可扩展工具（内置 / MCP / 社区）能力，前端为 Next.js 聊天界面，支持飞书、Slack、Telegram、Discord、钉钉等 IM 渠道接入。

## 架构概览

一次 `make dev` / Docker 栈会启动四个协作服务：

| 服务 | 端口 | 角色 |
|------|------|------|
| **Nginx** | `2026` | 统一反向代理入口 — 浏览器访问这里 |
| **Gateway API** | `8001` | FastAPI REST API + 内嵌 LangGraph 智能体运行时 |
| **Frontend** | `3000` | Next.js Web 界面 |
| **Provisioner** | `8002` | 可选 — 沙箱配置为 provisioner/K8s 模式时启用 |

Nginx 是唯一公开入口：`/api/*` 代理到 Gateway（`/api/langgraph/*` 重写到 Gateway 原生路由），同时托管前端静态资源。

## 目录结构

```
deep-hour-ai/
├── Makefile                        # 根编排：驱动整个技术栈（dev/start/stop、docker、setup）
├── config.example.yaml             # 模板 → 复制为 config.yaml（gitignore）
├── extensions_config.example.json  # 模板 → 复制为 extensions_config.json（MCP 服务器 + 技能）
├── backend/                        # Python 后端
│   ├── app/gateway/                # FastAPI Gateway + 认证/授权
│   ├── app/channels/               # IM 渠道（飞书/Slack/Telegram/Discord/钉钉/企微/微信）
│   ├── packages/harness/deerflow/  # 智能体框架（agents/tools/sandbox/memory/skills/extensions）
│   └── packages/extension-api/     # 扩展公开契约包
├── frontend/                       # Next.js 前端 (pnpm)
├── docker/                         # docker-compose、nginx 配置、provisioner
├── skills/                         # 智能体技能：public/（已提交）、custom/（gitignore）
├── examples/deerflow-extension-example/  # 扩展开发参考示例
└── docs/                           # 设计文档与实施计划
```

各模块的详细开发指南见 [backend/AGENTS.md](backend/AGENTS.md) 与 [frontend/AGENTS.md](frontend/AGENTS.md)。

## 快速开始

前置要求：Python 3.12+（`uv`）、Node.js 20+（`pnpm`）、可选 Docker。

```bash
make config      # 1. 生成 config.yaml + extensions_config.json（必须先做）
make install     # 2. 安装前后端依赖 + pre-commit 钩子
make dev         # 3. 开发模式启动全栈（热重载），浏览器打开 http://localhost:2026
```

常用命令：

```bash
make doctor      # 检查环境与配置问题
make stop        # 停止所有服务
make start       # 生产模式本地启动（SKIP_FRONTEND_BUILD=1 复用已有前端构建）
make up / down   # Docker 生产栈（localhost:2026）
```

## 配置

- `config.yaml`（仓库根目录）：主应用配置 — 模型接入、工具、沙箱、调度器等。所有字段支持环境变量（如 `api_key: $OPENAI_API_KEY`）。
- `extensions_config.json`：MCP 服务器与技能清单，可通过 Gateway API 运行时修改。
- `.env`：密钥类环境变量（模型 API Key、JWT secret 等），**禁止提交**。

配置字段变更后可用 `make config-upgrade` 合并新增字段。配置体系详见 [backend/docs/CONFIGURATION.md](backend/docs/CONFIGURATION.md)。

## 二次开发

**单模块开发**（改代码时更常用）：

```bash
cd backend && make dev        # 只起 Gateway（8001，带 reload）
cd backend && make test       # 默认后端测试套件
cd backend && make lint       # ruff check + format check
cd frontend && pnpm dev       # 只起前端（默认 Webpack，DEER_FLOW_DEV_BUNDLER=turbo 可切）
cd frontend && pnpm check     # 提交前运行：lint + 类型检查
```

**扩展机制**：通过 Python 扩展贡献中间件、任务生命周期钩子、Gateway 服务和 FastAPI 路由，无需修改核心代码：

```bash
make extension-install SOURCE=<包名|git-url|目录>
make extension-list / enable NAME=... / disable NAME=... / remove NAME=...
```

参考实现：[examples/deerflow-extension-example/](examples/deerflow-extension-example/)。扩展契约见 [backend/packages/harness/deerflow/extensions/](backend/packages/harness/deerflow/extensions/)。

**注意事项**：

- `config.yaml`、`extensions_config.json`、`.env` 均为 gitignore 文件，不要提交
- 修改扩展后需要重启 Gateway 才能生效
- 环境变量 `DEER_FLOW_PROJECT_ROOT` / `DEER_FLOW_CONFIG_PATH` / `DEER_FLOW_HOME` 可覆盖默认路径
