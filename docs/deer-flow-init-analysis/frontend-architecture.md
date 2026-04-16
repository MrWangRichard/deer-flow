# 前端架构

## 1. 前端的角色

DeerFlow 前端不是传统“把所有业务都代理到自家 REST API”的页面层，而是一个同时面对两类后端面的工作台：

- LangGraph 协议面：负责 threads、runs、streaming。
- Gateway REST 面：负责 uploads、models、skills、mcp、memory、artifacts、thread cleanup。

因此，理解前端时必须先接受一个事实：它不是只连一个后端 base URL。

## 2. 目录结构与职责

| 目录 | 职责 |
| --- | --- |
| `frontend/src/app/` | Next.js App Router 页面与布局 |
| `frontend/src/components/` | 页面组件、workspace 组件、AI elements、通用 UI |
| `frontend/src/core/` | 前端业务逻辑层，含 API client、threads、uploads、models、skills、memory、todos、artifacts |
| `frontend/src/server/` | server-only 代码，目前主要是 `better-auth` |
| `frontend/tests/unit/` | 单元测试，重点覆盖 core 逻辑 |
| `frontend/public/demo/` | demo 线程样例与静态产物 |

`core/` 是前端的关键层。它承担了：

- 与 LangGraph SDK 交互的适配。
- 与 Gateway REST 的请求封装。
- thread、upload、artifact、settings 等状态逻辑。
- 对 SSE 流和消息结构的前端解释。

## 3. 页面壳层如何组织

工作台布局由 `frontend/src/app/workspace/layout.tsx` 提供：

- `QueryClientProvider`
- `SidebarProvider`
- `WorkspaceSidebar`
- `CommandPalette`
- `Toaster`

这说明 workspace 层不是单纯页面，而是一个带全局查询缓存、侧边栏和命令面板的应用壳。

## 4. 主要页面入口

| 路由 | 入口文件 | 用途 |
| --- | --- | --- |
| `/workspace/chats/[thread_id]` | `src/app/workspace/chats/[thread_id]/page.tsx` | 默认聊天页面 |
| `/workspace/agents/[agent_name]/chats/[thread_id]` | `src/app/workspace/agents/[agent_name]/chats/[thread_id]/page.tsx` | 自定义 agent 聊天页面 |
| `/workspace/agents` | `src/app/workspace/agents/page.tsx` | agent 列表与管理 |
| `/workspace` | `src/app/workspace/page.tsx` | workspace 首页 |

默认聊天页与 agent 聊天页结构相似，差别主要在上下文里的 `agent_name`。

## 5. 前端如何区分两类后端

### LangGraph 面

`frontend/src/core/config/index.ts` 和 `frontend/src/core/api/api-client.ts` 负责生成 LangGraph SDK client：

- 默认地址是当前 origin 下的 `/api/langgraph`
- mock 模式走 `/mock/api`
- 对 `runs.stream` 和 `runs.joinStream` 做了 `streamMode` 兼容清洗

### Gateway 面

Gateway base URL 由 `getBackendBaseURL()` 提供，默认走同源 `/api/*`。它主要用于：

- 文件上传
- 模型列表
- memory 设置与状态
- skill 管理
- artifact 获取
- 删除本地 thread 数据

## 6. 业务模块分工

| 模块 | 目录 | 责任 |
| --- | --- | --- |
| 线程与流 | `core/threads/` | `useThreadStream`、thread 查询、删除线程 |
| API 适配 | `core/api/` | LangGraph SDK client 包装、stream mode 兼容 |
| 上传 | `core/uploads/` | 文件校验、上传、消息附件格式转换 |
| 产物 | `core/artifacts/` | artifact URL 解析与加载 |
| 模型 | `core/models/` | 获取模型列表和能力标记 |
| 技能 | `core/skills/` | skill 列表与管理调用 |
| 内存 | `core/memory/` | memory 查询、配置、状态 |
| 本地设置 | `core/settings/` | thread 上下文与 UI 设置缓存 |

## 7. 关键 UI 组件

| 组件 | 作用 |
| --- | --- |
| `InputBox` | 统一消息输入、模式切换、模型选择、文件附件和 followups |
| `MessageList` | 负责把 stream 中的 message 分组并渲染成多种消息块 |
| `SubtaskCard` | 展示 subagent 的实时执行状态 |
| `TodoList` | plan mode 下的 todo 展示 |
| `ArtifactTrigger` / `ArtifactFileList` | 展示 thread 产物 |
| `ThreadTitle` | 展示与更新线程标题 |

## 8. mode 不是纯前端 UI 概念

`InputBox` 中的 mode 有四种：

- `flash`
- `thinking`
- `pro`
- `ultra`

这些 mode 会被映射成真正的后端上下文字段：

- `thinking_enabled`
- `is_plan_mode`
- `subagent_enabled`
- `reasoning_effort`

因此 mode 是前后端协同协议的一部分，不只是按钮样式。

## 9. mock 与真实后端的双实现

前端保留了 `src/app/mock/api/*` 用于 demo / mock 场景。这意味着：

- 页面层通常可以在 mock 与真实后端之间复用。
- 如果某个 UI 只在真实环境出问题，需要先确认它是否依赖 Gateway 或 LangGraph 的特定返回格式。

## 10. 当前可见的自动化测试覆盖

前端测试集中在 `frontend/tests/unit/`，当前仓库里可见的重点包括：

- `core/api/stream-mode`
- `core/uploads/file-validation`
- `core/uploads/prompt-input-files`
- `core/threads/utils`

这说明团队目前更关注“前端业务逻辑正确性”，而不是大规模页面快照测试。

## 11. 接手建议

1. 先看 `src/app/workspace/.../page.tsx`，理解页面壳层和聊天主页面。
2. 再看 `core/threads/hooks.ts`，这是前端与 LangGraph run/stream 的交汇点。
3. 然后读 `components/workspace/messages/`，理解消息分组、artifact、reasoning、subtask 的 UI 映射。
4. 最后再看 `core/uploads/`、`core/skills/`、`core/memory/` 这些 REST 辅助模块。
