# DeerFlow 初始化源码分析

## 文档目的

这组文档面向第一次接手 DeerFlow 2.x 源码的开发者，目标是帮助读者在不依赖 README 营销描述的前提下，直接从仓库实现理解系统分层、模块边界、调用链和扩展点。分析范围以当前仓库源码为准，重点覆盖 `backend/` 与 `frontend/`，其中后端深度高于前端。

## 建议阅读顺序

| 文档 | 适合解决的问题 |
| --- | --- |
| `backend-overview.md` | 后端分几层、每层负责什么、请求如何进入系统 |
| `backend-agent-runtime.md` | lead agent 如何被组装、middleware 顺序是什么、prompt 如何注入 |
| `backend-tools-skills-sandbox-subagents.md` | tools、skills、sandbox、subagent、ACP agent 如何协作与隔离 |
| `frontend-architecture.md` | 前端目录结构、页面壳层、与后端的接口边界 |
| `frontend-chat-and-streaming.md` | 聊天提交、SSE、实时 UI、subtask 卡片如何变化 |

## 一页结论

1. DeerFlow 的核心不是“一个 FastAPI 应用”，而是三层协作：Nginx 统一入口、LangGraph lead agent runtime、Gateway API 辅助面。
2. 后端 agent 不是手写多节点业务图，而是 `langchain.agents.create_agent()` 加上一条严格排序的 middleware pipeline。
3. `skills`、`tools`、`sandbox`、`subagents` 都不是独立服务，而是围绕 lead agent 运行时动态拼装的能力层。
4. 前端主聊天链路优先直连 LangGraph 协议面 `/api/langgraph`；Gateway 主要承接上传、模型、技能、内存、artifact 与线程清理等非 agent 操作。
5. 线程级持久化并不依赖数据库，核心会话附件和产物默认落在 `backend/.deer-flow/threads/{thread_id}/`。

## 关键源码入口

- `backend/packages/harness/deerflow/agents/lead_agent/agent.py`
- `backend/packages/harness/deerflow/runtime/runs/worker.py`
- `backend/packages/harness/deerflow/tools/tools.py`
- `backend/packages/harness/deerflow/tools/builtins/task_tool.py`
- `backend/packages/harness/deerflow/subagents/executor.py`
- `frontend/src/core/threads/hooks.ts`
- `frontend/src/components/workspace/messages/message-list.tsx`

## 额外观察

- 仓库同时支持经典 LangGraph server 模式和 `--gateway` / `*-pro` 实验模式，接手时不要把两套运行面混为一谈。
- DeerFlow 内建的 `subagent` 与 ACP agent 是两条不同扩展路径：前者共享 thread 上下文，后者拥有独立 ACP workspace。
- 部分仓库文档对 middleware 数量与顺序的描述已经滞后；本文档统一以源码实现为准。

## 启动与验证入口

从仓库根目录常用的入口命令如下：

- `make check`: 检查 Python、Node.js、pnpm、uv、nginx 等依赖。
- `make install`: 安装 backend 与 frontend 依赖。
- `make dev`: 启动完整开发环境。
- `cd backend && make test`: 运行后端 pytest。
- `cd frontend && pnpm test`: 运行前端 vitest。

## 待补充信息

- 生产环境部署拓扑、真实域名和告警链路：未在仓库中发现。
- 线上模型配额、第三方服务密钥托管方式：需业务方确认。
- 团队对 `--gateway` 实验模式的正式支持范围：需维护者确认。
