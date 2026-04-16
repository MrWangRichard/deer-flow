# Lead Agent Runtime

## 1. lead agent 的真实入口

DeerFlow 默认对话 agent 的入口是 `backend/packages/harness/deerflow/agents/lead_agent/agent.py` 中的 `make_lead_agent(config)`。它不是手写的 LangGraph 多节点业务图，而是围绕 `langchain.agents.create_agent()` 做运行时装配：

- 解析本次 run 的配置。
- 选择模型。
- 装配 tool 列表。
- 拼接 system prompt。
- 组装 middleware 链。
- 指定 `ThreadState` 作为状态模式。

因此，理解 DeerFlow 的关键不是找“复杂 DAG”，而是看这几个拼装步骤如何协同。

## 2. 模型解析顺序

模型优先级来自 `make_lead_agent()` 和 `_resolve_model_name()`：

1. 请求上下文里的 `model_name` / `model`
2. 自定义 agent 配置中的模型
3. `config.yaml` 第一项模型作为默认值

`create_chat_model()` 由 `backend/packages/harness/deerflow/models/factory.py` 提供，负责：

- 通过 `use` 字段反射出 LangChain 模型类。
- 合并 thinking / reasoning_effort 的开关。
- 针对 Codex Responses API 做额外兼容。
- 给模型实例挂 tracing callbacks。

## 3. prompt 是如何拼出来的

`apply_prompt_template()` 不是静态模板替换，而是按运行态拼装多个片段：

- `SOUL.md` 对应的 agent personality。
- skills 提示块 `<available_skills>`。
- deferred tools 提示块 `<available-deferred-tools>`。
- memory 注入片段。
- subagent orchestration 规则。
- ACP agent 说明。
- 自定义 sandbox mounts 说明。
- 当天日期 `<current_date>`。

这说明 DeerFlow 的 prompt 设计是“固定骨架 + 运行态补丁”，而不是把所有规则硬编码在一个超大常量里。

## 4. `ThreadState` 中保存了什么

`backend/packages/harness/deerflow/agents/thread_state.py` 扩展了 LangChain 的 `AgentState`，关键字段包括：

| 字段 | 作用 |
| --- | --- |
| `messages` | 对话消息主状态 |
| `sandbox` | 当前 sandbox 标识 |
| `thread_data` | workspace / uploads / outputs 目录 |
| `artifacts` | 产物路径聚合 |
| `title` | 自动标题 |
| `todos` | plan mode 的任务列表 |
| `uploaded_files` | 当前线程上传文件摘要 |
| `viewed_images` | 供视觉模型使用的图片数据缓存 |

这里最关键的设计点是：DeerFlow 把“线程运行现场”显式放进状态，而不是散落在全局变量中。

## 5. middleware 的真实顺序

以源码为准，lead agent 的 middleware 顺序是：

1. `ThreadDataMiddleware`
2. `UploadsMiddleware`
3. `SandboxMiddleware`
4. `DanglingToolCallMiddleware`
5. `LLMErrorHandlingMiddleware`
6. `GuardrailMiddleware`（仅启用时）
7. `SandboxAuditMiddleware`
8. `ToolErrorHandlingMiddleware`
9. `DeerFlowSummarizationMiddleware`（仅启用时）
10. `TodoMiddleware`（仅 plan mode）
11. `TokenUsageMiddleware`（仅启用时）
12. `TitleMiddleware`
13. `MemoryMiddleware`
14. `ViewImageMiddleware`（仅模型支持视觉时）
15. `DeferredToolFilterMiddleware`（仅 tool_search 启用时）
16. `SubagentLimitMiddleware`（仅 subagent 模式）
17. `LoopDetectionMiddleware`
18. `ClarificationMiddleware`

这条链路体现了 DeerFlow 的运行时哲学：先准备环境和容错，再做上下文裁剪、任务跟踪、标题/记忆增强，最后再用 clarification 作为出口闸门。

## 6. 为什么这种 pipeline 设计有效

### 6.1 环境准备前置

`ThreadDataMiddleware` 与 `SandboxMiddleware` 靠前，保证后续 tools 和 uploads 都有稳定的 thread 上下文与目录挂载。

### 6.2 错误变成上下文，而不是直接崩溃

`LLMErrorHandlingMiddleware` 与 `ToolErrorHandlingMiddleware` 的职责都是把异常转换成 agent 还能继续理解的上下文消息，而不是让 run 直接终止。

### 6.3 plan mode 是运行时特性，不是独立 agent

Todo 只是同一 lead agent 上的可选中间件，不意味着系统切换到另一套 agent 架构。

### 6.4 clarification 被放在最后

这样可以让前面的工具执行、图像处理、todo 管理先发生，再由 clarification 决定是否真正中断并向用户追问。

## 7. Memory、Title、Summarization 的位置说明

### Summarization

`_create_summarization_middleware()` 会在启用时创建独立模型实例，用于削减长对话上下文。若 memory 功能打开，summarization 前还会触发 memory flush hook。

### Title

`TitleMiddleware` 在首轮交换后自动生成标题，前端会通过 `onUpdateEvent` 把 `title` 同步到线程列表缓存。

### Memory

`MemoryMiddleware` 在 title 之后挂入，用当前 agent 名称区分 per-agent memory 存储。它的角色更接近“异步写回队列入口”，而不是直接在每轮推理时计算复杂记忆。

## 8. subagent 模式如何接进 lead agent

lead agent 本身并不变成多 agent 图。`subagent_enabled` 只是带来两类变化：

- 在 prompt 中加入“分解、分批、并发上限”的系统规则。
- 在工具列表中加入 `task` 工具，并通过 `SubagentLimitMiddleware` 限制单次响应里的并发 task 数量。

换句话说，subagent 是 tool-based delegation，不是把主图替换成另一张 orchestration graph。

## 9. 与常见“多 agent 编排系统”的区别

| 常见方案 | DeerFlow 当前实现 |
| --- | --- |
| 预先定义多个业务节点图 | 一个 lead agent，必要时动态调用 tools / subagents |
| 每种能力是单独 agent 流程 | 多数能力通过 middleware 或 tool 注入 |
| 线程状态分散在数据库和服务之间 | 线程核心状态聚合在 `ThreadState` 和 `.deer-flow` 目录 |

## 10. 接手建议

1. 先读 `make_lead_agent()`，理解运行时配置如何被吃进去。
2. 再顺着 middleware 顺序看 `thread_data -> sandbox -> uploads -> summarization/title/memory -> clarification`。
3. 最后再看 `tools/`、`subagents/`、`skills/`，否则容易把“能力层”误认为“主流程层”。
