# Tools、Skills、Sandbox 与 Subagents

## 1. Tool 系统如何装配

tool 总入口是 `backend/packages/harness/deerflow/tools/tools.py` 中的 `get_available_tools()`。它会把工具来源拆成四层：

| 来源 | 说明 |
| --- | --- |
| `config.yaml` 工具 | 通过 `tool.use` 反射得到的基础工具 |
| built-in 工具 | 如 `present_file`、`ask_clarification`、`view_image`、`task` |
| MCP 工具 | 从启用的 MCP server 动态加载 |
| ACP agent 工具 | 当配置了 ACP agents 时，额外暴露 `invoke_acp_agent` |

装配过程还有几个条件判断：

- 当使用 `LocalSandboxProvider` 且未显式允许 `allow_host_bash` 时，host bash 工具不会暴露。
- 只有模型支持视觉时才会加入 `view_image_tool`。
- 只有 subagent 模式开启时才会加入 `task_tool`。
- 只有配置了启用的 ACP agent 时才会加入 `invoke_acp_agent`。

## 2. deferred tools 与 tool search

当 `tool_search` 打开时，MCP tools 不会全部直接 bind 给模型，而是先注册到 deferred registry，然后向模型暴露一个 `tool_search` 工具。

这套设计的作用是：

1. 控制初始 prompt / tool schema 体积。
2. 避免一次性把大量 MCP schema 全塞进上下文。
3. 让模型先通过工具搜索，再逐步获取真正需要的工具。

这是一种“按需暴露工具”的策略，不是 MCP 的额外协议层。

## 3. Skills 是如何被发现和启用的

`backend/packages/harness/deerflow/skills/loader.py` 负责扫描：

- `skills/public/`
- `skills/custom/`

扫描规则是递归查找包含 `SKILL.md` 的目录，并通过 `parse_skill_file()` 解析元数据。技能的最终启用态不由文件夹决定，而由 `extensions_config.json` 中的 skill state 决定。

因此，skill 的真实模型是：

- `SKILL.md`: 定义技能内容与 frontmatter。
- `extensions_config.json`: 决定是否对 runtime 生效。
- prompt cache: 把启用中的 skills 转成 `<available_skills>` 提示块。

## 4. Skills 如何进入 system prompt

`backend/packages/harness/deerflow/agents/lead_agent/prompt.py` 中的 `get_skills_prompt_section()` 会：

1. 读取当前启用的 skills。
2. 解析 skills 的容器路径，默认是 `/mnt/skills`。
3. 生成 `<available_skills>` 清单。
4. 在 `apply_prompt_template()` 中拼回 system prompt。

这意味着 skills 当前主要是“prompt-level workflow injection”，而不是 Python 插件执行框架。

## 5. Skills 的管理面

Gateway `backend/app/gateway/routers/skills.py` 负责：

- 列表与启用态管理。
- 从 `.skill` 压缩包安装技能。
- 读取、编辑、删除 custom skill。
- 记录 custom skill 历史。
- 回滚 custom skill 历史版本。

对 custom skill 编辑时，后端还会做两层保护：

- frontmatter 校验，确保 `name`、`description`、命名规则等合法。
- security scanner 扫描，阻断危险内容。

## 6. Sandbox 是如何隔离线程的

thread 目录由 `backend/packages/harness/deerflow/config/paths.py` 和 `ThreadDataMiddleware` 共同管理，默认布局是：

- `backend/.deer-flow/threads/{thread_id}/user-data/workspace`
- `backend/.deer-flow/threads/{thread_id}/user-data/uploads`
- `backend/.deer-flow/threads/{thread_id}/user-data/outputs`
- `backend/.deer-flow/threads/{thread_id}/acp-workspace`

对 agent 而言，这些目录在 sandbox 内表现为：

- `/mnt/user-data/workspace`
- `/mnt/user-data/uploads`
- `/mnt/user-data/outputs`
- `/mnt/acp-workspace`

这里的隔离单位是 thread，而不是用户全局目录。

## 7. LocalSandboxProvider 的特点

`LocalSandboxProvider` 的设计重点是“路径映射”，不是容器边界。它会：

- 把 `/mnt/skills` 映射到仓库 `skills/` 目录，并强制只读。
- 根据 `sandbox.mounts` 增加自定义 mount。
- 在本地直接执行命令、读写文件、列目录。
- 自动做 container path 与 host path 的双向转换，尽量让模型始终看到 `/mnt/...` 风格路径。

这套实现对开发调试很高效，但安全边界弱于容器沙箱。

## 8. 为什么默认禁止 LocalSandbox 的 host bash

`backend/packages/harness/deerflow/sandbox/security.py` 明确规定：

- 如果使用 `LocalSandboxProvider`，默认禁止 host bash。
- 只有显式设置 `sandbox.allow_host_bash: true` 才允许。
- `bash` subagent 也会同步受这一限制影响。

原因很直接：本地 provider 不是安全沙箱边界，直接开放 shell 会把宿主机暴露给 agent。

## 9. Subagent 的真实实现方式

内建 subagent 配置在 `backend/packages/harness/deerflow/subagents/builtins/`：

- `general-purpose`
- `bash`

但它们不是预先常驻的后台服务，而是由 `task_tool` 动态触发。

执行链如下：

1. lead agent 决定调用 `task(...)`。
2. `task_tool` 从 runtime 中继承 `sandbox`、`thread_data`、`thread_id`、父模型信息。
3. 它重新调用 `get_available_tools(..., subagent_enabled=False)`，显式禁掉 task 嵌套。
4. `SubagentExecutor` 创建独立 agent，并用 `agent.astream()` 后台执行。
5. 执行状态被写入后台任务表，并通过 `get_stream_writer()` 发出 `task_started`、`task_running`、`task_completed` 等 custom event。
6. 同时，最终 tool message 仍会写回主对话，作为确定性的终态结果。

## 10. 为什么前端同时依赖 custom event 和 tool message

这是 DeerFlow 一个很重要、也很容易忽略的设计点：

- custom event 用于“实时过程态”，例如 subagent 最新一步做了什么。
- tool message 用于“最终确定态”，例如成功、失败、超时。

所以前端 `SubtaskCard` 的状态不是由单一路径驱动，而是双通道合并：

- `task_running` 更新 `latestMessage`
- `tool` 消息解析结果字符串，决定 `completed / failed / timed_out`

## 11. Subagent 的并发与超时控制

并发控制分两层：

| 层 | 作用 |
| --- | --- |
| Prompt 规则 | 在 system prompt 中要求模型按批次分解任务 |
| `SubagentLimitMiddleware` | 在运行时强制截断超额 task 调用 |

默认 timeout 和 max turns 定义在 `SubagentConfig`，但 `subagents_config` 允许在 `config.yaml` 中覆盖。执行器内部还使用多个 `ThreadPoolExecutor` 管理调度、执行和隔离 event loop。

## 12. ACP agent 与 built-in subagent 的区别

DeerFlow 还有一条平行扩展路径：ACP agent。

区别如下：

| 能力 | built-in subagent | ACP agent |
| --- | --- | --- |
| 入口 | `task` 工具 | `invoke_acp_agent` 工具 |
| 工作目录 | 共享当前 thread 的 `/mnt/user-data` | 独立 `/mnt/acp-workspace` |
| 目的 | 上下文隔离的子任务 | 调外部 agent runtime，如 codex、claude_code |
| UI 反馈 | 通过 subtask 卡片展示 | 更偏结果导向 |

接手时不要把 ACP agent 误认成内建 subagent 的另一种配置项。

## 13. 接手时最值得优先看的源码

1. `backend/packages/harness/deerflow/tools/tools.py`
2. `backend/packages/harness/deerflow/skills/loader.py`
3. `backend/packages/harness/deerflow/tools/builtins/task_tool.py`
4. `backend/packages/harness/deerflow/subagents/executor.py`
5. `backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py`
6. `backend/packages/harness/deerflow/sandbox/security.py`

## 14. 风险与注意事项

1. Skills 目前主要通过 prompt 注入生效，能力边界依赖提示词纪律，而不是强制插件沙箱。
2. LocalSandboxProvider 强调开发便利，不等同于安全隔离。
3. subagent 过程态依赖 custom event，若只看最终 tool message，会误判 UI 为什么能实时刷新。
4. ACP agent 使用独立 workspace，生成的文件不能默认假设出现在 `/mnt/user-data/outputs`。
