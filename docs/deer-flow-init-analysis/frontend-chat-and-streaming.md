# 前端聊天与实时流链路

## 1. 聊天页面的主入口

默认聊天页入口是 `frontend/src/app/workspace/chats/[thread_id]/page.tsx`。页面本身并不直接实现流逻辑，而是把职责分给三个核心部分：

- `useThreadChat()`: 解析当前 thread id。
- `useThreadStream()`: 维护 LangGraph stream 生命周期。
- `MessageList` 与 `InputBox`: 把运行态映射成 UI。

## 2. thread id 如何管理

`useThreadChat()` 的规则是：

1. 如果路径是 `/new`，先在前端生成临时 UUID。
2. 真正创建线程后，由 `onCreated(meta.thread_id)` 用服务端 thread id 覆盖。
3. 页面用 `history.replaceState()` 而不是 Next.js router 切换 URL，避免组件重挂载导致流状态丢失。

这个设计是为了保证“新建会话”期间也能先输入内容，但最终仍以服务端线程标识为准。

## 3. `useThreadStream()` 是整个前端聊天链路的中心

`frontend/src/core/threads/hooks.ts` 中的 `useThreadStream()` 同时做了几件事：

- 创建 LangGraph SDK `useStream` 订阅。
- 管理 optimistic messages。
- 在提交前执行文件上传。
- 维护 run resume 信息。
- 把 stream 的 update/custom/langchain event 转换成前端状态。

这是前端最值得优先阅读的单个文件。

## 4. 用户发送消息时会发生什么

### 4.1 无附件场景

1. `InputBox` 调用 `sendMessage(threadId, message)`。
2. `useThreadStream()` 先写入 optimistic human message。
3. 调用 `thread.submit(...)`，把消息发给 LangGraph。
4. 同时附带 `context`：
   - `thinking_enabled`
   - `is_plan_mode`
   - `subagent_enabled`
   - `reasoning_effort`
   - `thread_id`
5. 后端开始流式执行，前端持续消费 SSE。

### 4.2 有附件场景

1. 先显示 optimistic “上传中” UI。
2. 前端把 `PromptInput` 的附件转换成 `File`。
3. 调用 Gateway 上传接口。
4. 上传成功后，把 `virtual_path` 写入 human message 的 `additional_kwargs.files`。
5. 再向 LangGraph 提交消息。

这意味着文件上传不是 run 的一部分，而是 run 之前的独立 REST 操作。

## 5. 前端如何接收 SSE

`useStream` 订阅时启用了几个关键能力：

- `reconnectOnMount`: 通过 `sessionStorage` 保存 run id，支持页面重新挂载后的 resumable stream。
- `fetchStateHistory: { limit: 1 }`: 首次加载时取最近一次状态快照。
- `streamSubgraphs: true`
- `streamResumable: true`

因此，前端并不是只消费“纯 token 流”，而是消费 LangGraph 的多类型事件。

## 6. 各类事件分别更新什么 UI

| 事件来源 | 前端处理位置 | 用途 |
| --- | --- | --- |
| `onCreated` | `useThreadStream()` | 绑定真实 thread id，修正 URL |
| `onUpdateEvent` | `useThreadStream()` | 更新线程标题等 state values |
| `onCustomEvent(task_running)` | `useThreadStream()` | 更新 subtask 的 `latestMessage` |
| `onLangChainEvent(on_tool_end)` | `useThreadStream()` | 暴露工具结束钩子 |
| `onFinish` | `useThreadStream()` | 收尾、刷新线程列表、必要时发系统通知 |

## 7. `MessageList` 如何把消息流映射成不同组件

`frontend/src/components/workspace/messages/message-list.tsx` 不会把所有 message 当作同一种气泡，而是先分组，然后按组类型渲染：

- `human` / `assistant`: 普通消息气泡
- `assistant:clarification`: 单独 clarification 内容块
- `assistant:present-files`: artifact 文件列表
- `assistant:subagent`: subtask 卡片群组
- 其他 group: 通用 `MessageGroup`

这意味着 DeerFlow 的消息区本质上是“对 LangGraph 消息结构的解释器”。

## 8. 实时 subtask UI 为什么能一边跑一边变

这是前端最有代表性的实时 UI 之一。机制分两步：

### 8.1 过程态

`task_tool` 在后端发出 `task_running` custom event，前端 `onCustomEvent` 把最新 AI message 填进 subtask context。

### 8.2 终态

同一个 subagent 完成后，主线程里会出现对应 tool message。`MessageList` 解析其文本前缀：

- `Task Succeeded. Result:`
- `Task failed.`
- `Task timed out`

再决定 subtask 的 `completed / failed` 状态。

所以 `SubtaskCard` 的变化过程是：

1. 主模型先产生 `task` tool call。
2. UI 生成卡片，状态为 `in_progress`。
3. custom event 持续刷新“最新动作”。
4. tool message 到达后，卡片切到完成或失败态。

## 9. reasoning、todo、artifact 的 UI 更新方式

### reasoning

当 AI message 只有 reasoning，没有正式回答时，`MessageListItem` 会渲染 `Reasoning` 组件，并根据 `isLoading` 决定是否处于流式展开状态。

### todo

聊天页头部上方的 `TodoList` 直接依赖 `thread.values.todos`。因此 todo 的渲染来源是 `values` 状态流，不是独立 API。

### artifact

artifact 主要通过消息中的 `/mnt/...` 路径和 Gateway artifact URL 映射来展示。图片、文件链接和 `present_files` 都会走 artifact 解析逻辑。

## 10. Stop 按钮如何生效

聊天页的 stop 操作调用 `thread.stop()`。前端本身不直接拼取消接口，而是依赖 LangGraph SDK 的停止逻辑；后端在 `thread_runs.py` 中实现了兼容的 cancel-then-stream 行为，确保前端停止后仍能看到剩余缓冲事件和干净的终态。

## 11. 前端在聊天链路中的几个容易忽略的点

1. optimistic message 只是在“服务端消息数量增加后”才清空，因此消息列表不是纯服务端真值。
2. 附件上传和 run 提交是两步，不要把上传失败误以为模型报错。
3. subtask 实时状态依赖 custom event，而不是普通 message delta。
4. 标题更新来自 `values` 更新事件，不是前端本地生成。
5. 新线程 URL 切换故意避开 Next.js router，以保护流式状态。

## 12. 接手时建议的阅读顺序

1. `frontend/src/app/workspace/chats/[thread_id]/page.tsx`
2. `frontend/src/core/threads/hooks.ts`
3. `frontend/src/components/workspace/messages/message-list.tsx`
4. `frontend/src/components/workspace/messages/subtask-card.tsx`
5. `frontend/src/components/workspace/input-box.tsx`

按这个顺序读，可以先建立“发送 -> 流 -> 消息区”的主链，再回头补模式切换、artifact、followups 等细节。
