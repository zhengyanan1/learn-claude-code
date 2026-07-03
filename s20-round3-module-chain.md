# s20 第三轮阅读理解：沿 agent_loop 拆模块

目标：沿着 `agent_loop` 的调用链读模块。重点是：模块什么时候接入主循环、它改变了什么状态。

## 1. Context Compaction

模块入口：

[s20_comprehensive/code.py:1055](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1055)

主循环接入点：

[s20_comprehensive/code.py:1916](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1916)

每轮 LLM 前都会执行：

```python
messages[:] = tool_result_budget(messages)
messages[:] = snip_compact(messages)
messages[:] = micro_compact(messages)
if estimate_size(messages) > CONTEXT_LIMIT:
    messages[:] = compact_history(messages)
```

它分四层。

控制最新 tool_result 体积：

[s20_comprehensive/code.py:1109](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1109)

消息太多时保留头尾：

[s20_comprehensive/code.py:1133](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1133)

压缩较旧的长 tool_result：

[s20_comprehensive/code.py:1152](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1152)

仍然超限时，写 transcript 并总结历史：

[s20_comprehensive/code.py:1183](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1183)

理解重点：

**compaction 是每轮调模型前的常规预处理，不是等报错才做。**

## 2. Reactive Compact

入口：

[s20_comprehensive/code.py:1190](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1190)

接入点：

[s20_comprehensive/code.py:1983](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1983)

如果 LLM 报 prompt too long：

```python
if is_prompt_too_long_error(e) and not state.has_attempted_reactive_compact:
    messages[:] = reactive_compact(messages)
    state.has_attempted_reactive_compact = True
    continue
```

区别：

- `prepare_context()` 是预防
- `reactive_compact()` 是报错后的急救

它会保留最近几条消息，把更早的内容摘要掉。

## 3. Error Recovery

模块入口：

[s20_comprehensive/code.py:1206](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1206)

LLM 调用入口：

[s20_comprehensive/code.py:1942](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1942)

核心状态：

[s20_comprehensive/code.py:1208](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1208)

`RecoveryState` 记录：

- 是否已经升过 `max_tokens`
- 已经续写恢复几次
- 连续 529 次数
- 是否已经 reactive compact
- 当前使用哪个 model

重试逻辑：

[s20_comprehensive/code.py:1222](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1222)

处理：

- 429 rate limit：等待后重试
- 529 overloaded：等待后重试，多次后可切 fallback model

`max_tokens` 恢复逻辑：

[s20_comprehensive/code.py:1992](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1992)

逻辑：

1. 第一次 `max_tokens`：把 `max_tokens` 从 8000 升到 16000，重新请求。
2. 如果已经升过：保存部分 response，再追加 `CONTINUATION_PROMPT` 让模型继续。
3. 超过恢复次数就 return。

所以 error recovery 分两层：

- `with_retry()` 处理 API 异常
- `agent_loop()` 处理 response 正常但被截断

## 4. Hooks + Permission

模块入口：

[s20_comprehensive/code.py:874](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:874)

hook 注册表：

[s20_comprehensive/code.py:878](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:878)

触发函数：

[s20_comprehensive/code.py:886](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:886)

接入点如下。

用户提交 prompt：

[s20_comprehensive/code.py:2103](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2103)

工具执行前：

[s20_comprehensive/code.py:2026](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2026)

工具执行后：

[s20_comprehensive/code.py:2044](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2044)

无 tool_use 结束时：

[s20_comprehensive/code.py:2008](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2008)

权限逻辑：

[s20_comprehensive/code.py:898](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:898)

它能拦截：

- 危险 bash：`rm -rf /`、`sudo`、`shutdown`
- destructive bash：需要用户确认
- `write_file/edit_file` 写出 workspace
- deploy 类 MCP 工具

理解重点：

**权限不写在每个 tool handler 里，而是在 `PreToolUse` 统一拦截。**

## 5. Background Tasks

模块入口：

[s20_comprehensive/code.py:1259](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1259)

判断慢任务：

[s20_comprehensive/code.py:1279](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1279)

主循环接入点：

[s20_comprehensive/code.py:2033](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2033)

如果是慢 `bash`，流程如下。

创建 `bg_0001` 这种 ID：

[s20_comprehensive/code.py:1285](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1285)

存进 `background_tasks`：

[s20_comprehensive/code.py:1299](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1299)

起 daemon thread：

[s20_comprehensive/code.py:1305](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1305)

立即给模型返回占位 tool_result：

[s20_comprehensive/code.py:2035](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:2035)

后台 worker 完成后，会执行真实 handler：

[s20_comprehensive/code.py:1291](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1291)

触发 `PostToolUse`：

[s20_comprehensive/code.py:1294](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1294)

写入 `background_results`：

[s20_comprehensive/code.py:1295](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1295)

结果收集：

[s20_comprehensive/code.py:1310](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1310)

它会生成 `<task_notification>`。

主循环有两个地方接收后台结果。

每轮开始注入：

[s20_comprehensive/code.py:1935](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1935)

tool_result 回灌时追加：

[s20_comprehensive/code.py:1926](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1926)

所以后台任务不是阻塞等待，而是靠 notification 回到模型上下文。

## 6. Cron Scheduler

模块入口：

[s20_comprehensive/code.py:1330](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1330)

核心数据结构：

[s20_comprehensive/code.py:1337](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1337)

创建 cron：

[s20_comprehensive/code.py:1451](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1451)

后台 scheduler：

[s20_comprehensive/code.py:1477](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1477)

启动 scheduler：

[s20_comprehensive/code.py:1527](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1527)

主循环消费 cron queue：

[s20_comprehensive/code.py:1496](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1496)

接入点：

[s20_comprehensive/code.py:1964](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1964)

本质是：

```text
cron job 触发
  ↓
进入 cron_queue
  ↓
agent_loop 消费
  ↓
变成 user message: [Scheduled] ...
  ↓
让同一个 LLM loop 处理
```

所以 cron 不是另起一套 agent，它只是定时注入 prompt。

## 7. MCP System

模块入口：

[s20_comprehensive/code.py:1531](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1531)

核心类：

[s20_comprehensive/code.py:1535](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1535)

连接表：

[s20_comprehensive/code.py:1558](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1558)

连接入口：

[s20_comprehensive/code.py:1614](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1614)

工具池组装：

[s20_comprehensive/code.py:1629](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1629)

主循环每轮都会重新组装工具池：

[s20_comprehensive/code.py:1979](/Users/zhengyanan18/Desktop/work-git-ai/learn-claude-code/s20_comprehensive/code.py:1979)

流程：

```text
connect_mcp("docs")
  ↓
mcp_clients["docs"] = MCPClient(...)
  ↓
assemble_tool_pool()
  ↓
mcp__docs__search / mcp__docs__get_version 加入 tools
```

关键理解：

**MCP 工具不是写死在 `BUILTIN_TOOLS` 里，而是连接后动态加入。**

真实 agent 里，MCP server 通常通过配置或插件预先接入；s20 用 `connect_mcp` 做成工具，是为了教学演示动态工具发现。

## 8. 第三轮总结

把模块和主循环对应起来：

```text
prepare_context()
  -> Context Compaction

call_llm()
  -> Error Recovery

trigger_hooks("PreToolUse")
  -> Permission

should_run_background()
  -> Background Tasks

consume_cron_queue()
  -> Cron Scheduler

assemble_tool_pool()
  -> MCP dynamic tools
```

核心理解：

**s20 的复杂度不是 `agent_loop` 里塞满业务，而是每个能力模块都挂在固定接入点上。**
