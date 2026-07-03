# s20 第 4 轮阅读理解：周边能力

这一轮看的是：`s20` 除了主循环以外，怎么把“任务、隔离目录、技能、团队通信、协议、长期 teammate、一次性 subagent”这些能力接到 agent 产品里。

核心判断标准是：这些模块大多不是自己主动跑，而是被模型通过工具调用触发。

## 1. Task System

入口：[s20_comprehensive/code.py:71](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:71)

这里定义的是持久任务系统。任务不是内存变量，而是写到 `.tasks/task_xxx.json` 里。

`Task` 结构：[s20_comprehensive/code.py:80](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:80)

字段含义：

- `id`：任务唯一 ID
- `subject`：任务标题
- `description`：任务详情
- `status`：`pending / in_progress / completed`
- `owner`：谁领取了任务
- `blockedBy`：依赖哪些任务完成后才能开始
- `worktree`：绑定的隔离工作目录

关键逻辑：

- `create_task()` 创建任务并落盘：[s20_comprehensive/code.py:95](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:95)
- `can_start()` 判断依赖任务是否都 completed：[s20_comprehensive/code.py:124](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:124)
- `claim_task()` 把任务从 `pending` 改成 `in_progress`，并写入 owner：[s20_comprehensive/code.py:136](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:136)
- `complete_task()` 把任务标记为 completed，并检查是否解锁了其他任务：[s20_comprehensive/code.py:157](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:157)

所以 Task System 的本质是：给 agent/team 一个可持久化的任务看板。

## 2. Worktree System

入口：[s20_comprehensive/code.py:172](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:172)

它解决的是：多个 teammate 同时工作时，不要都改同一个目录。

`create_worktree()`：[s20_comprehensive/code.py:211](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:211)

流程：

1. 校验 worktree 名字
2. 如果传了 `task_id`，先确认任务存在
3. 执行 `git worktree add`
4. 如果绑定任务，就把 `task.worktree = name`
5. 记录事件

绑定逻辑：[s20_comprehensive/code.py:235](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:235)

删除逻辑：[s20_comprehensive/code.py:254](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:254)

注意：`remove_worktree()` 默认会检查是否有未提交文件或新 commit。有内容时不会直接删，除非 `discard_changes=true`。

这个模块的本质是：把任务和独立工作目录绑起来。

## 3. Skill Loading

入口：[s20_comprehensive/code.py:285](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:285)

它做的是本地 skill 扫描和加载。

`scan_skills()`：[s20_comprehensive/code.py:303](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:303)

流程：

1. 遍历 `SKILLS_DIR`
2. 找每个目录下的 `SKILL.md`
3. 解析 frontmatter
4. 注册到 `SKILL_REGISTRY`

`load_skill()`：[s20_comprehensive/code.py:335](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:335)

模型调用 `load_skill` 工具时，返回对应 skill 的完整内容。

所以这里的本质是：skill 不直接改变 `agent_loop`，而是作为工具暴露给模型，由模型决定什么时候加载。

## 4. MessageBus

入口：[s20_comprehensive/code.py:490](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:490)

这是 teammate 通信系统。

`send()`：[s20_comprehensive/code.py:498](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:498)

它把消息追加写入 `.mailboxes/{agent}.jsonl`。

`read_inbox()`：[s20_comprehensive/code.py:510](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:510)

它读取某个 agent 的 mailbox，读完后 `unlink()` 删除邮箱文件。也就是说 inbox 是“消费型”的：读一次就清空。

这解释了之前 demo 里看到的现象：`check_inbox` 读过一次后，再读就是 empty。

## 5. Protocol State

入口：[s20_comprehensive/code.py:523](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:523)

它解决的是：普通消息无法表达“请求-响应-审批”这种协议状态。

`ProtocolState` 记录：

- `request_id`
- `type`
- `sender`
- `target`
- `status`
- `payload`

`pending_requests`：[s20_comprehensive/code.py:536](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:536)

这是内存里的请求表。

`match_response()`：[s20_comprehensive/code.py:543](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:543)

它按 `request_id` 匹配响应，避免一个 response 错误批准另一个 request。

lead 主循环外层会消费 lead inbox：[s20_comprehensive/code.py:2111](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2111)

这里会把 teammate 发给 lead 的消息读出来，并追加进 `history`，让下一轮模型能看到。

## 6. Autonomous Agent

入口：[s20_comprehensive/code.py:568](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:568)

这块是 teammate 为什么能“没任务时 idle，有任务时自动 claim”。

`scan_unclaimed_tasks()`：[s20_comprehensive/code.py:574](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:574)

它扫描 `.tasks`，找：

- `status == pending`
- 没有 owner
- 依赖已满足

`idle_poll()`：[s20_comprehensive/code.py:585](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:585)

优先级是：

1. 先看 inbox
2. 如果有 shutdown，直接响应并退出
3. 如果有普通消息，塞进 teammate 自己的 messages，返回 `work`
4. 如果没有消息，再扫描可领取任务
5. 有任务就 claim，并把任务信息塞进 messages
6. 60 秒都没有，就 timeout

所以 teammate 的 idle 不是“线程空转什么都不做”，而是周期性检查 mailbox 和 task board。

## 7. Teammate Thread

入口：[s20_comprehensive/code.py:622](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:622)

`spawn_teammate_thread()` 启动的是长期 teammate。

关键点一：它有自己的 `messages` 和 `system`。

[s20_comprehensive/code.py:631](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:631)

关键点二：它有自己的工具表。

[s20_comprehensive/code.py:693](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:693)

teammate 工具包括：

- `bash`
- `read_file`
- `write_file`
- `send_message`
- `submit_plan`
- `list_tasks`
- `claim_task`
- `complete_task`

关键点三：如果任务绑定了 worktree，teammate 的文件工具会自动切到 worktree 目录。

[s20_comprehensive/code.py:656](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:656)

关键点四：plan approval 是硬门禁。

[s20_comprehensive/code.py:762](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:762)

如果 `waiting_plan` 存在，teammate 只轮询协议回复，不继续执行任务。

关键点五：summary 只在 teammate 退出时发一次。

[s20_comprehensive/code.py:813](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:813)

也就是说：长期 teammate 可以中途 `send_message` 给 lead，但最终 `result summary` 只有退出时统一发一次。

## 8. Subagent Tool

入口：[s20_comprehensive/code.py:962](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:962)

`task` 工具对应的是一次性 subagent。

工具定义：[s20_comprehensive/code.py:1763](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1763)

handler 绑定：[s20_comprehensive/code.py:1874](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1874)

真正执行：[s20_comprehensive/code.py:1023](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1023)

它和 teammate 最大区别是：

- `task`：启动一次，最多跑 30 轮，最后只返回最终 summary
- `spawn_teammate`：启动后台线程，长期存在，可以收消息、等审批、自动 claim 任务、idle、shutdown

## 这一轮的主线

可以把 s20 的周边能力理解成三层：

第一层：持久状态

- `Task System`
- `Worktree System`
- `Skill Registry`
- `Mailbox`

第二层：协调协议

- `MessageBus`
- `ProtocolState`
- `request_plan`
- `review_plan`
- `request_shutdown`

第三层：执行者

- `task` 是一次性 subagent
- `spawn_teammate` 是长期 teammate

所以 s20 的关键不是 `agent_loop` 变复杂，而是：主循环外面挂了一批“产品级能力”，然后通过工具表暴露给模型，让模型自己决定什么时候调用。
