# s20 第一轮阅读理解：结构地图和工具表

目标：先建立 `s20_comprehensive/code.py` 的整体地图，不深入每个模块实现。

## 1. s20 是什么

入口说明在：

[s20_comprehensive/code.py:3](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:3)

文件头说明 `s20` 是综合章：

- dispatch
- permission
- hooks
- todo
- subagent
- skills
- compaction
- memory
- prompt assembly
- error recovery
- task graph
- background tasks
- cron
- teams
- protocols
- autonomous agents
- worktrees
- MCP

所以 `s20` 不是新增一个单点功能，而是把前面章节拆开的机制重新装进一个完整 agent harness。

阅读时重点问题不是“新增了什么”，而是：

**这些系统最后是怎么被一个 agent loop 串起来的？**

## 2. 模块地图

模块边界可以用：

```bash
rg -n "^# ──" s20_comprehensive/code.py
```

当前模块地图：

```text
71   Task System
172  Worktree System
285  Skill Loading
343  Prompt Assembly
377  Basic Tools
490  MessageBus
523  Protocol State
568  Autonomous Agent
622  Teammate Thread
843  Lead Protocol Tools
874  Hooks + Permission Pipeline
962  Subagent Tool
1055 Context Compaction
1206 Error Recovery
1259 Background Tasks
1330 Cron Scheduler
1531 MCP System
1648 Lead Worktree Tools
1660 Basic tool handlers
1721 Tool Definitions
1893 Context
1910 Agent Loop
```

可以分成三类理解。

## 3. 底层状态类

这些模块主要负责持久状态、共享状态或外部连接：

- Task System：`.tasks/task_xxx.json`
- Worktree System：`.worktrees/`
- MessageBus：`.mailboxes/`
- Protocol State：plan/shutdown 这类 request/response 的内存状态
- MCP System：动态外部工具源

这类模块本身不是主循环，但给主循环和工具 handler 提供状态基础。

## 4. 能力模块类

这些模块是 agent 的能力：

- Skill Loading
- Basic Tools
- Hooks + Permission
- Subagent Tool
- Context Compaction
- Error Recovery
- Background Tasks
- Cron Scheduler
- Autonomous Agent / Teammate Thread

这些能力最终大多通过工具、hook 或 agent loop 接入。

## 5. 总装入口类

真正把能力暴露给模型、接进循环的是：

- Tool Definitions
- Context
- Agent Loop

可以先记成：

```text
前面都是零件
Tool Definitions + Context + Agent Loop 是总装
```

## 6. 工具表：模型能调用什么

工具表入口：

[s20_comprehensive/code.py:1725](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1725)

`BUILTIN_TOOLS` 是模型可见的工具 schema。

也就是说，模型能不能调用某个能力，先看这里有没有注册。

## 7. 工具分组

### 基础操作工具

- `bash`
- `read_file`
- `write_file`
- `edit_file`
- `glob`

位置：

[s20_comprehensive/code.py:1726](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1726)

这组是 coding agent 最基础的观察和行动接口。

### 会话任务工具

- `todo_write`

位置：

[s20_comprehensive/code.py:1752](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1752)

这是当前会话内的 todo，不是 `.tasks` 持久任务系统。

### 子 agent / skill / compact

- `task`
- `load_skill`
- `compact`

位置：

[s20_comprehensive/code.py:1763](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1763)

含义：

- `task`：启动一次性 subagent，只拿最终 summary。
- `load_skill`：加载技能内容。
- `compact`：让模型主动要求压缩上下文。

### 持久任务系统

- `create_task`
- `list_tasks`
- `get_task`
- `claim_task`
- `complete_task`

位置：

[s20_comprehensive/code.py:1778](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1778)

这是 `.tasks/task_xxx.json` 那套持久任务系统。

### Cron 定时任务

- `schedule_cron`
- `list_crons`
- `cancel_cron`

位置：

[s20_comprehensive/code.py:1799](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1799)

这是后台调度能力。后续触发后会进入 `agent_loop` 注入消息。

### 团队协作 / Teammate

- `spawn_teammate`
- `send_message`
- `check_inbox`
- `request_shutdown`
- `request_plan`
- `review_plan`

位置：

[s20_comprehensive/code.py:1815](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1815)

这是 team + protocol + autonomous teammate 相关能力。

### Worktree

- `create_worktree`
- `remove_worktree`
- `keep_worktree`

位置：

[s20_comprehensive/code.py:1847](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1847)

这是隔离工作区能力。

### MCP

- `connect_mcp`

位置：

[s20_comprehensive/code.py:1864](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1864)

注意：这里只注册了 `connect_mcp`。

真正的 MCP 工具不是写死在 `BUILTIN_TOOLS` 里，而是在连接 MCP server 后动态加入工具池。

## 8. Handler 表

Handler 表入口：

[s20_comprehensive/code.py:1871](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1871)

`BUILTIN_HANDLERS` 回答的问题是：

**模型调用工具名后，Python 实际执行哪个函数？**

例如：

```python
"bash": run_bash
"todo_write": run_todo_write
"task": spawn_subagent
"load_skill": load_skill
"schedule_cron": run_schedule_cron
"spawn_teammate": run_spawn_teammate
"connect_mcp": run_connect_mcp
```

## 9. 第一轮结论

第一轮只需要记住这张图：

```text
s20_comprehensive.py
  ├─ 状态系统
  │   ├─ tasks
  │   ├─ worktrees
  │   ├─ mailboxes
  │   ├─ protocol state
  │   └─ MCP clients
  │
  ├─ 能力模块
  │   ├─ basic tools
  │   ├─ hooks / permission
  │   ├─ skills
  │   ├─ subagent
  │   ├─ compaction
  │   ├─ recovery
  │   ├─ background tasks
  │   ├─ cron
  │   └─ teammate
  │
  └─ 总装
      ├─ BUILTIN_TOOLS
      ├─ BUILTIN_HANDLERS
      ├─ update_context
      └─ agent_loop
```

核心理解：

**`BUILTIN_TOOLS` 是模型可见接口；`BUILTIN_HANDLERS` 是 harness 执行接口。**

模型看不到 handler 函数，只看到 schema。

Python 不理解 schema 的业务语义，只按工具名查 handler 执行。

第一轮不要纠结每个函数怎么实现。先抓住：**s20 的能力入口集中在工具表，所有工具最后都会被 `agent_loop` 调度。**
