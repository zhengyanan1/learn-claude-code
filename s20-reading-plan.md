# s20 阅读理解计划

目标：按执行链路理解 `s20_comprehensive/code.py`，不要从第 1 行顺读到最后。

## 总体思路

`s20` 是综合章，把前面章节的机制重新装进一个完整 harness：

- tool dispatch
- permission hooks
- todo
- subagent
- skill loading
- context compaction
- memory
- error recovery
- task graph
- background tasks
- cron
- teams and protocols
- autonomous teammates
- worktrees
- MCP tools

阅读时优先抓主循环，再按主循环调用到的模块回看。

## 第 1 轮：建立地图

先看：

- 文件头：`s20_comprehensive/code.py:3`
- 模块边界：`rg -n "^# ──" s20_comprehensive/code.py`
- 工具表：`s20_comprehensive/code.py:1725`
- handler 表：`s20_comprehensive/code.py:1871`

这一轮只回答：

- s20 暴露了哪些工具？
- 哪些能力来自前面章节？
- 哪些能力最终通过 `tool_use` 被模型调用？

不要深入每个模块实现。

## 第 2 轮：读主循环

重点看：

- `prepare_context()`：`s20_comprehensive/code.py:1916`
- `build_user_content()`：`s20_comprehensive/code.py:1926`
- `call_llm()`：`s20_comprehensive/code.py:1942`
- `agent_loop()`：`s20_comprehensive/code.py:1955`

按这个顺序理解：

1. cron 任务注入 messages
2. background notification 注入 messages
3. todo reminder 注入 messages
4. 压缩上下文
5. 更新 context
6. 组装 builtin tools + MCP tools
7. 调 LLM
8. 处理 error recovery 和 max_tokens
9. 没有 `tool_use` 就触发 Stop hook 并返回
10. 有 `tool_use` 就执行工具
11. 工具结果作为 user message 回灌给模型

## 第 3 轮：按主循环回看模块

按 `agent_loop` 调用顺序看模块：

- Context Compaction：`s20_comprehensive/code.py:1055`
- Error Recovery：`s20_comprehensive/code.py:1206`
- Hooks + Permission：`s20_comprehensive/code.py:874`
- Background Tasks：`s20_comprehensive/code.py:1259`
- Cron Scheduler：`s20_comprehensive/code.py:1330`
- MCP System：`s20_comprehensive/code.py:1531`

每个模块只看两个问题：

- 它什么时候被 `agent_loop` 调用？
- 它改变了什么状态？例如 `messages`、`tools`、`handlers`、文件系统或后台线程。

## 第 4 轮：看周边能力

最后看不在主循环最核心、但构成完整产品形态的模块：

- Task System：`s20_comprehensive/code.py:71`
- Worktree System：`s20_comprehensive/code.py:172`
- Skill Loading：`s20_comprehensive/code.py:285`
- MessageBus：`s20_comprehensive/code.py:490`
- Protocol State：`s20_comprehensive/code.py:523`
- Autonomous Agent：`s20_comprehensive/code.py:568`
- Teammate Thread：`s20_comprehensive/code.py:622`
- Subagent Tool：`s20_comprehensive/code.py:962`

重点区分：

- `task` 工具：启动一次性 subagent，最后只返回 summary。
- `spawn_teammate`：启动后台 teammate，会持续 idle/work。
- `create_task/list_tasks/claim_task`：持久任务系统。
- `create_worktree/remove_worktree`：隔离工作目录。
- `connect_mcp`：动态扩展工具池。

## 主线图

```text
用户输入 / cron / background / teammate inbox
  ↓
agent_loop
  ↓
prepare_context 压缩上下文
  ↓
update_context 更新运行时状态
  ↓
assemble_tool_pool 组装 builtin + MCP tools
  ↓
call_llm + with_retry
  ↓
response.content
  ↓
没有 tool_use：Stop hook + 返回
  ↓
有 tool_use：PreToolUse hook / permission
  ↓
后台执行 or 直接执行 handler
  ↓
PostToolUse hook
  ↓
tool_result + background notification
  ↓
回到下一轮 LLM
```

## 后续带读方式

后续按轮次阅读：

1. 先读第 1 轮的结构和工具表。
2. 再逐句读 `agent_loop()`。
3. 然后沿着 `agent_loop()` 调用链拆模块。
4. 最后看 teammate、worktree、MCP 这些周边系统如何接进主循环。

每一轮都先讲“这一段解决什么问题”，再讲“代码怎么做”，最后讲“它在总循环里的位置”。
