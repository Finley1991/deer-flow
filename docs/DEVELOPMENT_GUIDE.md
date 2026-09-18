# DeepHourAI 开发指南

面向在本仓库进行二次开发的工程师。架构与启动入口见 [README.md](../README.md)；模块级深度文档见各目录 `AGENTS.md`（backend、frontend、harness 各子系统均有）。

## 1. 开发环境

| 组件 | 要求 |
|------|------|
| Python | 3.12+，使用 [uv](https://docs.astral.sh/uv/) 管理依赖 |
| Node.js | 20+，使用 pnpm（仓库通过 `scripts/pnpm.py` 封装，Corepack 锁版本） |
| Docker | 可选 — 沙箱、生产部署、IM 渠道联调时需要 |

首次环境检查：

```bash
make check       # 检查必需工具
make doctor      # 检查配置与系统要求（更全面）
make install     # backend: uv sync --locked；frontend: pnpm install；pre-commit 钩子
```

## 2. 模型接入

模型在 `config.yaml` 的 `models:` 列表中声明，每项含 `use:` 字段指向实现类（如 `deerflow.models.patched_deepseek:PatchedChatDeepSeek`）。支持 OpenAI 兼容协议（DeepSeek、豆包/Ark、vLLM 等）与 Coding Plan 网关。API Key 通过 `.env` 注入：

```bash
# .env 示例
DEEPSEEK_API_KEY=sk-...
VOLCENGINE_API_KEY=...
```

完整字段说明见 [backend/docs/CONFIGURATION.md](../backend/docs/CONFIGURATION.md)。

## 3. 代码结构速查

### 后端（Python）

```
backend/
├── packages/harness/deerflow/   # 核心框架（import: deerflow.*）
│   ├── agents/                  # lead agent、子智能体、middleware 链
│   ├── tools/                   # 内置工具（bash/文件/glob/grep/搜索…）
│   ├── sandbox/                 # 沙箱 provider（local / Docker AIO / E2B / opensandbox）
│   ├── memory/                  # 持久化记忆 backends（deermem/mem0/honcho…）
│   ├── skills/                  # 技能加载、扫描、审查
│   ├── extensions/              # 扩展管理器与贡献契约
│   ├── mcp/                     # MCP 客户端与 OAuth
│   ├── persistence/             # Postgres 持久化 + Alembic 迁移
│   ├── config/                  # 配置 schema 与解析
│   └── tui/                     # 终端客户端（DeerFlowTUI）
├── app/gateway/                 # FastAPI Gateway（import: app.*）
│   ├── auth/ authz.py           # 认证（含 SSO/OIDC）与 RBAC 授权
│   ├── routers/                 # REST 路由
│   └── github/                  # GitHub 渠道集成
├── app/channels/                # IM 渠道适配（飞书/Slack/TG/Discord/钉钉/企微/微信）
└── packages/extension-api/      # 扩展公开 API（import: deerflow_extension_api.*）
```

### 前端（Next.js App Router）

```
frontend/src/
├── app/                         # 路由：/(auth) 登录、/workspace 主界面、/landing、docs、blog
├── core/                        # API client、i18n（zh-CN/en-US）、threads、artifacts
└── components/                  # UI 组件（workspace、landing）
```

## 4. 测试与质量

```bash
# 后端
cd backend && make test                    # 默认套件（排除 live 与 blocking-io）
cd backend && python -m pytest tests/test_xxx.py::test_func -q   # 单测
cd backend && make test-blocking-io        # 阻塞 IO 严格套件
cd backend && make lint && make format     # ruff

# 前端
cd frontend && pnpm check                  # eslint + tsc --noEmit（提交前必跑）
cd frontend && pnpm test                   # 单元测试 (rstest)
cd frontend && pnpm test:e2e               # Playwright e2e
```

## 5. 扩展开发

五种贡献类型：中间件、任务生命周期钩子、系统模型观察者、Gateway 服务、FastAPI HTTP 路由。

```bash
make extension-install SOURCE=./my-extension/   # 本地目录即可
# 改动后重启 Gateway 生效
```

从 [examples/deerflow-extension-example/](../examples/deerflow-extension-example/) 开始，它演示了全部五种贡献方式。扩展 API 契约见 `backend/packages/extension-api/`。

> 安全边界：`config.yaml` 顶层 `plugins:` 列表会引入代码执行，仅允许操作员控制的受信来源。

## 6. 持久化与迁移

```bash
cd backend && uv run alembic -c app... upgrade head   # 具体命令见 persistence/AGENTS.md
```

迁移文件在 `backend/packages/harness/deerflow/persistence/migrations/versions/`。新增表/字段的流程见 [backend/packages/harness/deerflow/persistence/migrations/AGENTS.md](../backend/packages/harness/deerflow/persistence/migrations/AGENTS.md)。

## 7. 遗留命名说明

代码内部保留 `deerflow` / `deer-flow` 作为技术标识符（**不要随意重命名**）：

| 标识符 | 位置 | 原因 |
|--------|------|------|
| `deerflow.*` Python 包名 / import | 后端全量 | 改名涉及数百处 import、扩展契约、pyproject |
| `deerflow.*` localStorage key | 前端 | 影响已登录用户本地数据兼容 |
| `X-DeerFlow-Internal-Token` 等请求头 | Gateway 内部认证 | API 契约，前后端联动 |
| `ghcr.io/bytedance/deer-flow-sandbox-network-proxy` | 沙箱网络代理默认镜像 | 功能性镜像源，改名会导致拉取失败 |
| `DEER_FLOW_*` 环境变量 | 运行时 | 与配置系统耦合 |

对外呈现（UI 标题、文案、文档）统一使用 **DeepHourAI**。如未来要彻底改名，需一次完整迁移并跑全量测试。
