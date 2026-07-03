# s20 第二轮阅读理解：agent_loop 主循环

目标：理解 `s20` 每一轮怎么跑。不要深入每个模块内部，先掌握主循环如何串起 messages、context、tools、LLM、tool_result。

## 1. update_context：每轮给 system prompt 准备运行时状态

入口：

[s20_comprehensive/code.py:1899](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1899)

它每轮收集三类状态：

- `memories`：`.memory/MEMORY.md` 的前 2000 字
- `connected_mcp`：当前连接了哪些 MCP server
- `active_teammates`：当前有哪些 teammate 活着

这些状态会进入 `assemble_system_prompt()`，让模型知道当前环境状态。

## 2. prepare_context：每次调模型前先压缩上下文

入口：

[s20_comprehensive/code.py:1916](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1916)

核心逻辑：

```python
messages[:] = tool_result_budget(messages)
messages[:] = snip_compact(messages)
messages[:] = micro_compact(messages)
if estimate_size(messages) > CONTEXT_LIMIT:
    messages[:] = compact_history(messages)
```

注意这里用的是 `messages[:] = ...`，表示原地修改同一个 history 列表。

这样外层 `history` 也同步被压缩。

压缩顺序：

1. 控制工具结果体积
2. 裁剪过长历史
3. 微压缩旧 tool_result
4. 如果整体仍然太大，做完整 compact

## 3. build_user_content：工具结果和后台通知一起回灌

入口：

[s20_comprehensive/code.py:1926](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1926)

它会把普通工具结果和已完成后台任务通知合并：

```python
content = list(results)
for note in collect_background_results():
    content.append({"type": "text", "text": note})
return content
```

所以最后塞回 messages 的 user content 里可能同时有：

- 本轮工具结果
- 之前后台任务完成通知

这延续了 background task 的机制。

## 4. call_llm：统一组装 system prompt + retry 调模型

入口：

[s20_comprehensive/code.py:1942](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1942)

它做两件事：

1. 每次调用前重新 `assemble_system_prompt(context)`
2. 实际 LLM 调用包在 `with_retry()` 里

也就是说，error recovery 不散落在主循环各处，而是集中在 `with_retry()` 和后面的异常处理里。

## 5. agent_loop 初始化

入口：

[s20_comprehensive/code.py:1955](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1955)

开头：

```python
tools, handlers = assemble_tool_pool()
state = RecoveryState()
max_tokens = DEFAULT_MAX_TOKENS
```

重点：

- `tools` 不是只来自 `BUILTIN_TOOLS`
- `assemble_tool_pool()` 会合并 builtin tools 和已连接 MCP server 的动态工具
- `handlers` 同样包括 builtin handlers 和 MCP dynamic handlers

## 6. 每一轮开始：先注入外部事件

位置：

[s20_comprehensive/code.py:1961](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1961)

每轮开始先处理 cron：

[s20_comprehensive/code.py:1964](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1964)

命中的 cron job 会变成：

```python
{"role": "user", "content": f"[Scheduled] {job.prompt}"}
```

然后注入后台任务完成通知：

[s20_comprehensive/code.py:1970](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1970)

再处理 todo reminder：

[s20_comprehensive/code.py:1972](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1972)

所以模型看到的不只是用户输入，还可能看到系统注入的：

- `[Scheduled] ...`
- `<task_notification>...`
- `<reminder>Update your todos.</reminder>`

## 7. 调模型前：压缩、更新 context、重新组装工具池

位置：

[s20_comprehensive/code.py:1977](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1977)

核心：

```python
prepare_context(messages)
context = update_context(context, messages)
tools, handlers = assemble_tool_pool()
```

为什么每轮都重新做：

- messages 可能变长，需要压缩
- memory / MCP / teammate 状态可能变化，需要更新 context
- MCP 连接可能新增，需要更新工具池

## 8. 调 LLM + 异常恢复

位置：

[s20_comprehensive/code.py:1981](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1981)

核心调用：

```python
response = call_llm(messages, context, tools, state, max_tokens)
```

如果异常是 prompt too long，并且还没 reactive compact：

[s20_comprehensive/code.py:1984](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1984)

就压缩后重来一轮：

```python
messages[:] = reactive_compact(messages)
state.has_attempted_reactive_compact = True
continue
```

否则把错误作为 assistant text 放进 messages，然后 return。

## 9. max_tokens 处理

位置：

[s20_comprehensive/code.py:1992](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1992)

如果模型因为 `max_tokens` 停了：

第一次：

```python
max_tokens = ESCALATED_MAX_TOKENS
continue
```

如果已经升过上限：

```python
messages.append({"role": "assistant", "content": response.content})
messages.append({"role": "user", "content": CONTINUATION_PROMPT})
continue
```

含义：

- 第一次先提高输出上限重试
- 后续保留部分输出，再让模型继续

## 10. 正常 response：追加 assistant 消息

位置：

[s20_comprehensive/code.py:2005](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2005)

正常拿到 response 后：

```python
max_tokens = DEFAULT_MAX_TOKENS
state.has_escalated = False
messages.append({"role": "assistant", "content": response.content})
```

一旦拿到正常 response，就把它加入 history。

## 11. 没有 tool_use：本轮结束

位置：

[s20_comprehensive/code.py:2008](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2008)

```python
if not has_tool_use(response.content):
    trigger_hooks("Stop", messages)
    return
```

含义：

- 如果模型没有工具调用，说明它已经给出最终回答
- 触发 Stop hook
- 返回外层主循环

外层负责打印 assistant 文本。

打印逻辑：

[s20_comprehensive/code.py:2061](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2061)

外层调用位置：

[s20_comprehensive/code.py:2106](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2106)

## 12. 有 tool_use：进入工具执行阶段

位置：

[s20_comprehensive/code.py:2012](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2012)

遍历 response 里的 tool_use block：

```python
for block in response.content:
    if block.type != "tool_use":
        continue
```

## 13. compact 是特殊工具

位置：

[s20_comprehensive/code.py:2019](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2019)

`compact` 不走普通 handler：

```python
messages[:] = compact_history(messages)
messages.append({"role": "user",
                 "content": "[Compacted. Continue with summarized context.]"})
compacted_now = True
break
```

它直接压缩 history，然后进入下一轮 LLM。

## 14. 其他工具先过 PreToolUse hook

位置：

[s20_comprehensive/code.py:2026](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2026)

```python
blocked = trigger_hooks("PreToolUse", block)
if blocked:
    results.append({"type": "tool_result", ...})
    continue
```

如果 permission hook 拒绝工具执行，不会真的执行工具，而是把拒绝原因作为 `tool_result` 返回给模型。

## 15. 判断是否后台执行

位置：

[s20_comprehensive/code.py:2033](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2033)

如果工具适合后台执行：

```python
bg_id = start_background_task(block, handlers)
output = "[Background task ... started]"
results.append({"type": "tool_result", "content": output})
continue
```

真正结果后面通过 background notification 注入。

## 16. 普通工具执行

位置：

[s20_comprehensive/code.py:2042](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2042)

```python
handler = handlers.get(block.name)
output = call_tool_handler(handler, block.input, block.name)
trigger_hooks("PostToolUse", block, output)
print(str(output)[:300])
```

这里的 `handlers` 包括：

- builtin handlers
- MCP dynamic handlers

所以模型调用 builtin 工具和 MCP 工具，最终都走同一条 dispatch 路径。

## 17. todo reminder 计数

位置：

[s20_comprehensive/code.py:2047](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2047)

```python
if block.name == "todo_write":
    rounds_since_todo = 0
else:
    rounds_since_todo += 1
```

如果模型一直不用 `todo_write`，几轮后会被 reminder 提醒。

## 18. 工具结果回灌

位置：

[s20_comprehensive/code.py:2052](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2052)

每个工具执行后追加：

```python
results.append({"type": "tool_result",
                "tool_use_id": block.id,
                "content": output})
```

最后：

[s20_comprehensive/code.py:2058](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2058)

```python
messages.append({"role": "user", "content": build_user_content(results)})
```

这是 agent loop 的核心闭环：

```text
assistant tool_use
  ↓
Python 执行工具
  ↓
user tool_result
  ↓
下一轮 LLM
```

## 19. 外层主循环

入口：

[s20_comprehensive/code.py:2088](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2088)

用户输入后：

[s20_comprehensive/code.py:2103](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2103)

```python
trigger_hooks("UserPromptSubmit", query)
turn_start = len(history)
history.append({"role": "user", "content": query})
with agent_lock:
    agent_loop(history, context)
    context = update_context(context, history)
    print_turn_assistants(history, turn_start)
```

这里再次说明：

`agent_loop` 只负责 append assistant response，不直接打印最终回答。

最终打印由 `print_turn_assistants()` 负责。

## 第二轮结论

`agent_loop` 可以理解成：

```text
每一轮开始
  ↓
注入 cron / background / todo reminder
  ↓
压缩 messages
  ↓
更新 context
  ↓
动态组装 tools / handlers
  ↓
call_llm + retry
  ↓
如果没有 tool_use：Stop hook，返回
  ↓
如果有 tool_use：
  compact 特殊处理
  PreToolUse 权限检查
  后台任务判断
  handler 执行
  PostToolUse
  tool_result 回灌
  ↓
回到下一轮
```

这是 s20 的总装核心。
