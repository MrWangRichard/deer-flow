# 后端总览

## 1. 后端由哪几层组成

DeerFlow 后端不是单一应用，而是三层协作：

| 层 | 主要入口 | 主要职责 |
| --- | --- | --- |
| 统一入口层 | Nginx，默认对外 `:2026` | 统一暴露 Web、Gateway 和 LangGraph 两类 API |
| Agent Runtime 层 | `backend/packages/harness/deerflow/agents/lead_agent/agent.py` | 负责线程状态、模型调用、tool 执行、流式输出、checkpoint |
| Gateway API 层 | `backend/app/gateway/app.py` | 提供 models、skills、mcp、memory、uploads、artifacts、thread cleanup、runs 兼容接口 |

Nginx 的职责分界在仓库文档和代码里是一致的：

- `/api/langgraph/*` 进入 LangGraph server，供 `@langchain/langgraph-sdk` 和 `useStream` 使用。
- 其他 `/api/*` 进入 Gateway FastAPI。
- `/` 及非 API 请求进入 Next.js frontend。

## 2. 后端目录与模块职责

| 模块 | 目录 | 责任边界 |
| --- | --- | --- |
| 配置层 | `backend/packages/harness/deerflow/config/` | 从 `config.yaml` 和 `extensions_config.json` 加载模型、tools、sandbox、skills、stream bridge、checkpointer 等配置 |
| Agent 层 | `backend/packages/harness/deerflow/agents/` | 组装 lead agent、定义 `ThreadState`、挂接 middleware、memory、title、todo、clarification |
| Runtime 层 | `backend/packages/harness/deerflow/runtime/` | 管理 run lifecycle、SSE stream bridge、run manager、序列化和 store |
| Tool 与扩展层 | `backend/packages/harness/deerflow/tools/`、`mcp/`、`community/` | 装配内建工具、MCP 工具、社区工具与 ACP agent 工具 |
| 隔离执行层 | `backend/packages/harness/deerflow/sandbox/`、`subagents/` | 负责 thread 级工作目录、沙箱 provider、本地/容器执行、subagent 调度 |
| API 层 | `backend/app/gateway/` | FastAPI 路由、运行时单例注入、与前端非 agent 场景对接 |

## 3. 请求是如何进入系统的

### 3.1 聊天 run 主链路

1. 前端调用 LangGraph SDK，请求 `/api/langgraph/threads/{thread_id}/runs/stream`。
2. LangGraph server 通过 `make_lead_agent(config)` 创建 agent。
3. agent 执行 middleware 链，准备 thread 目录、uploads、sandbox、memory、todo 等状态。
4. 模型生成回复，并在需要时调用 tools 或 subagents。
5. `runtime/runs/worker.py` 用 `agent.astream()` 将 `values`、`messages`、`custom` 等事件转为 SSE。
6. 前端 `useStream` 接收事件，更新消息列表、标题、todo 和 subtask UI。

### 3.2 文件上传链路

1. 前端先调用 Gateway `POST /api/threads/{thread_id}/uploads`。
2. Gateway 把文件写入 `backend/.deer-flow/threads/{thread_id}/user-data/uploads/`。
3. 对 PDF / PPT / Excel / Word 等可转换文件，Gateway 会生成 Markdown 衍生文件。
4. 下一个 agent run 中，`UploadsMiddleware` 读取这些文件并注入对话上下文。

### 3.3 删除线程链路

1. 前端先通过 LangGraph 删除线程状态。
2. 随后调用 Gateway `DELETE /api/threads/{thread_id}` 删除 DeerFlow 本地文件目录。
3. 这一拆分意味着“会话状态删除”和“本地文件清理”不是同一个接口完成的。

## 4. 后端的主要设计模式

| 模式 | 体现位置 | 说明 |
| --- | --- | --- |
| Factory | `models/factory.py`、`agents/lead_agent/agent.py` | 配置驱动创建模型与 agent |
| Middleware Pipeline | `agents/middlewares/*` | 把 thread data、sandbox、memory、title、clarification 等横切能力串成固定顺序 |
| Provider | `sandbox/*provider.py`、guardrail provider、stream bridge provider | 用统一接口屏蔽本地与容器、不同 guardrail 后端等差异 |
| Registry | `subagents/registry.py`、deferred tool registry | 统一管理可见 subagent 和 deferred tool |
| Runtime Singleton | `app/gateway/deps.py` | StreamBridge、checkpointer、store、RunManager 放到 `app.state` |

## 5. 配置系统如何装配整个后端

`backend/packages/harness/deerflow/config/app_config.py` 是后端配置总入口。它负责：

- 解析 `config.yaml`，优先级为显式路径、`DEER_FLOW_CONFIG_PATH`、仓库根目录/`backend/` 默认位置。
- 解析 `$ENV_VAR` 形式的环境变量引用。
- 顺带刷新 `subagents`、`tool_search`、`guardrails`、`checkpointer`、`stream_bridge`、`acp_agents` 等子配置。
- 从独立文件 `extensions_config.json` 读取 MCP server 开关和 skill 启用态。

这意味着 DeerFlow 的“配置”并不是单一 YAML，而是：

- `config.yaml`: 运行时主体配置。
- `extensions_config.json`: MCP 与 skills 的动态开关。
- `.env` / 环境变量: API key 与外部服务密钥。

## 6. 运行与测试入口

### 根目录

- `make check`
- `make install`
- `make dev`
- `make docker-start`
- `make up`

### backend 目录

- `make dev`: `uv run langgraph dev`
- `make gateway`: `uv run uvicorn app.gateway.app:app`
- `make test`: `pytest tests/ -v`
- `make lint`

## 7. 接手时应优先注意的风险

1. 仓库里既有 LangGraph server 路径，也有 Gateway runs 兼容层；如果只盯着 FastAPI，很容易误判聊天主链路。
2. `AppConfig` 和 MCP tools 使用缓存与热重载，调试配置问题时要区分“磁盘已改”和“运行态是否已刷新”。
3. `--gateway` / `*-pro` 模式在 Makefile 中被标成 experimental，接手时不应默认把它视为主路径。
4. 后端本地状态默认写入 `backend/.deer-flow/`，不是数据库；排查线程附件问题时应优先看文件系统。
