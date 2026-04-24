---
title: "Claude Code 源码精读：12 个功能点实测笔记"
date: 2026-04-24T12:00:00+08:00
lastmod: 2026-04-24T12:00:00+08:00
description: "逐步拆解 Claude Code 的 Agent Loop、多工具 Dispatch、Todo 规划、Subagent 隔离、上下文压缩、持久化任务系统、后台任务、Agent Teams、Skill Loading 等核心功能点的源码实现与实测记录。"
keywords: ["Claude Code", "Agent", "LLM", "源码分析", "AI"]
tags: ["Agent", "Claude Code", "AI", "源码阅读"]
categories: ["技术"]
draft: false
showToc: true
TocOpen: true
ShowReadingTime: true
ShowCodeCopyButtons: true
ShowShareButtons: true
---

> **💡 测试进度**
> **4 / 12** 功能点已完成 · 源码对应 `agents/s01` ~ `agents/s12`

---

# 功能点 1：基础 Agent Loop（s01）

> **🎯 目标**
> 观察 LLM ↔ 工具 ↔ 消息的基本往返循环

> **💡 Lead Agent Loop 流程**
>
> **（1）每轮 Context 预处理**
>
> **① 压缩**：micro compact 和 auto compact
> *(截图略)*
>
> **② 注入后台任务结果**
> - 整理成一条消息，以 `"role": "user"` 塞回 `messages`，让模型知道后台任务已完成
> - 用 `<background-results>...</background-results>` 标签包裹，标识为事件通知
> *(截图略)*
>
> **③ 查看信箱（read_inbox）**
> - 别的 agent 往 lead 发消息时，追加到 `lead.jsonl`；`read_inbox("lead")` 读出后清空
> - 用 `<inbox>...</inbox>` 包裹，作为 `"role": "user"` 追加进 messages
> *(截图略)*
>
> **（2）LLM Call（API 调用）**
> ```python
> response = client.messages.create(
>     model=MODEL, system=SYSTEM, messages=messages,
>     tools=TOOLS, max_tokens=8000,
> )
> messages.append({"role": "assistant", "content": response.content})
> ```
>
> 示例入参：
> ```
> MODEL    = 'qwen-max'
> SYSTEM   = 'You are a coding agent at D:\Dev\...'
> messages = [{'role': 'user', 'content': '列出当前目录下所有的python文件。'}]
> TOOLS    = [bash, read_file, write_file, edit_file, TodoWrite, ..., idle, claim_task]
> ```
>
> 回复示例：
> ```python
> {'role': 'assistant', 'content': [
>     ThinkingBlock(thinking='...我可以使用 dir *.py (Windows) 或 ls *.py ...'),
>     ToolUseBlock(input={'command': 'ls .py'}, name='bash')
> ]}
> ```
>
> **（3）Tool Use Block 执行**（仅返回前 5000 字符）
> *(截图略)*
> ```python
> [{'type': 'tool_result', 'tool_use_id': '...',
>   'content': "'ls' 不是内部或外部命令..."}]
> ```
>
> **（4）Todo 提醒注入**
> 若有未完成待办 && 连续 3 轮未更新 todo，注入：
> ```xml
> <reminder>Update your todos.</reminder>
> ```
> *(截图略)*
>
> **（5）循环退出条件**
> 将 tool result 塞入 messages 后继续下一轮；当 `stop_reason != "tool_use"` 时退出：
> ```python
> if response.stop_reason != "tool_use":
>     return
> ```
> *(截图略)*

---

## 1.1 — list python files

**Prompt**：`list all python files in this directory`

**预期数据流**：
```
LLM 生成 bash tool call → run_bash("ls .py") → 结果追加 messages → LLM 返回文本
```

**关注点**：messages 列表如何增长：`user → assistant → user(tool_result) → assistant`

**结果**：✅（见上方 Loop 流程详解）

## 1.3 — 同步阻塞验证

**Prompt**：`run: echo hello && sleep 5 && echo world`

**关注点**：验证同步阻塞特性，对比后续的异步模式

>
> bash 阻塞 5s 后返回完整输出 ✅

## 1.4 — 危险命令拦截

**Prompt**：`rm -rf /tmp`

**预期数据流**：`run_bash` 返回 `"Error: Dangerous command blocked"`

**关注点**：安全过滤逻辑（dangerous 列表）是否正确

>
> 占坑，暂未深测。

---

# 功能点 2：多工具 Dispatch（s02）

> **🎯 目标**
> 观察 `TOOL_HANDLERS` 路由机制，文件读写的 `safe_path` 沙箱

> **💡 TOOLS 变量结构**
>
> **核心三部分**：`name`（工具名）· `description`（用途说明）· `input_schema`（JSON Schema 参数定义）
>
> 示例（TodoWrite）：
> ```json
> {
>   "name": "TodoWrite",
>   "description": "Update task tracking list.",
>   "input_schema": {
>     "type": "object",
>     "properties": {
>       "items": {
>         "type": "array",
>         "items": {
>           "type": "object",
>           "properties": {
>             "content":    {"type": "string"},
>             "status":     {"type": "string", "enum": ["pending", "in_progress", "completed"]},
>             "activeForm": {"type": "string"}
>           },
>           "required": ["content", "status", "activeForm"]
>         }
>       }
>     },
>     "required": ["items"]
>   }
> }
> ```
>
> **设计原则**：`TOOLS` 只声明接口（= 函数签名 + 文档），让 LLM 决定用不用、怎么用；`TOOL_HANDLERS` 才是真正的函数体，负责执行与校验。

---

## 2.1 — write_file 路由

**Prompt**：`create a file called debug_test.txt with content "hello world"`

**预期数据流**：`LLM 调用 write_file → run_write → 创建文件 → "Wrote N bytes" 返回`

**关注点**：`TOOL_HANDLERS` dict 路由：`block.name` 决定走哪个 handler

>
> 路由核心逻辑（ThinkingBlock 直接跳过）：
> ```python
> for block in response.content:
>     if block.type == "tool_use":
>         if block.name == "compress":
>             manual_compress = True
>         handler = TOOL_HANDLERS.get(block.name)
>         output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
> ```
>
> ToolUseBlock 进入分支：
> ```
> ToolUseBlock(input={'path': 'debug_test.py', 'content': 'hello world!'}, name='write_file')
> ```
> *(截图略)*
> *(截图略)*
> *(截图略)*

## 2.2 — read_file + limit 参数

**Prompt**：`read the file agents/s01_agent_loop.py (first 10 lines only)`

**预期数据流**：`LLM 调用 read_file(path, limit=10) → 返回截断内容`

**关注点**：`limit` 参数是否被 LLM 正确传入，`run_read` 的切片逻辑

**结果**：✅

## 2.3 — edit_file 替换

**Prompt**：`edit debug_test.txt: replace "hello world" with "goodbye world"`

**预期数据流**：`LLM 调用 edit_file(old_text, new_text)`

**关注点**：`old_text not in content` 的 Error path

**结果**：✅

## 2.4 — 路径沙箱隔离

**Prompt**：`read file ../../../etc/passwd`

**预期数据流**：`safe_path` 抛出 `ValueError: Path escapes workspace`

**关注点**：`path.is_relative_to(WORKDIR)` 检查

>
> 安全功能，暂时跳过。
> 逻辑：检查解析后的真实路径是否仍在工作区内，若"逃出"项目目录则报错。

---

# 功能点 3：Todo 自我规划（s03）

> **🎯 目标**
> 观察 `TodoManager` 状态机 + nag reminder 注入机制

> **💡 Todo 系统设计**
>
> **解决的核心问题：**
> - 多步骤任务里，模型容易忘记做到哪一步
> - 口头计划与实际执行脱节
> - 用户难以看出 agent 当前在干什么
> - 模型可能同时展开太多支线，失焦
>
> **数据结构（每条 todo）：**
>
> | 字段 | 说明 |
> |------|------|
> | `content` | 任务内容 |
> | `status` | `pending` / `in_progress` / `completed`（同时只能有一个 `in_progress`）|
> | `activeForm` | 正在做的动作（供 render 渲染动态列表）|
>
> **TodoManager 核心方法：**
> - `update(items)`：先 validate → 整体替换（完整快照 + 合法性校验）
> - `render()`：格式化输出 `[ ]` / `[>]→activeForm` / `[x]`，附进度统计
> - `has_open_items()`：只要有未完成项，工作流就算活着
>
> **Nag Reminder 机制：**
> ```
> has_open_items() == True  &&  rounds_without_todo >= 3
>   → 在 message 中注入 <reminder>Update your todos.</reminder>
> ```
> *(截图略)*

---

## 3.1 — 任务规划 + Todo 状态流转

**Prompt**：`create a simple calculator in python: add, subtract, multiply, divide`

**预期数据流**：
```
LLM 调用 TodoWrite 创建任务列表 → 逐步执行 → 每步标记 in_progress/completed
```

**关注点**：`rounds_since_todo` 计数器如何累积，`TODO.render()` 的输出格式

>
>
> **① 第一轮 tool_use：创建 Todo 列表**
> ```python
> ToolUseBlock(name='TodoWrite', input={'items': [
>     {'content': '创建calculator.py文件，定义add函数并测试', 'status': 'pending', 'activeForm': '创建add函数'},
>     {'content': '定义subtract函数并测试',                  'status': 'pending', 'activeForm': '创建subtract函数'},
>     {'content': '定义multiply函数并测试',                  'status': 'pending', 'activeForm': '创建multiply函数'},
>     {'content': '定义divide函数并测试（含除零检查）',        'status': 'pending', 'activeForm': '创建divide函数'},
>     {'content': '添加用户交互界面，整合所有功能',            'status': 'pending', 'activeForm': '添加用户界面'},
> ]})
> ```
> *(截图略)*
>
> 写入 message 的 `tool_result`：
> ```
> [ ] 创建calculator.py文件，定义add函数并测试
> [ ] 定义subtract函数并测试
> [ ] 定义multiply函数并测试
> [ ] 定义divide函数并测试（含除零检查）
> [ ] 添加用户交互界面，整合所有功能
> (0/5 completed)
> ```
>
> **② 第二轮：将 task 1 标记为 in_progress**
> ```python
> ToolUseBlock(name='TodoWrite', input={'items': [
>     {'content': '创建calculator.py...', 'status': 'in_progress', 'activeForm': '创建add函数'},
>     # ...其余为 pending
> ]})
> ```
> render 效果：
> *(截图略)*
>
> 后续轮次正常 tool 调用，逐步更新 todo 状态。
>
> **Reminder 注入示例：**
> *(截图略)*

## 3.2 — Nag Reminder 触发

**Prompt**：接上轮，故意用简单任务让 LLM 不主动更新 todo，连续 3 轮不调用 todo

**预期数据流**：第 3 轮后 results 中出现 `<reminder>Update your todos.</reminder>`

**关注点**：`rounds_since_todo >= 3` 触发注入

**结果**：✅

## 3.3 — 双 in_progress 校验

**Prompt**：`create 2 tasks and mark both as in_progress simultaneously`

**预期数据流**：`TodoManager.update` 抛出 `ValueError: Only one task can be in_progress at a time`

**关注点**：业务约束校验逻辑（s03:73）

**结果**：✅

## 3.4 — render 格式验证

**Prompt**：`track my progress: step 1 done, step 2 in progress, step 3 pending`

**预期数据流**：观察 `TODO.render()` 输出的 `[x]` / `[>]` / `[ ]` 标记格式

**结果**：✅

---

# 功能点 4：Subagent 上下文隔离（s04）

>
> Process isolation gives context isolation for free.

> **🎯 目标**
> 观察父子 Agent 的 messages 隔离，以及 summary-only 返回

> **💡 SubAgent 设计原则**
>
> **核心思想**：把子任务交给全新上下文的小代理去做，完成后只把摘要带回来，避免中间结果污染父上下文。
>
> | 特性 | 说明 |
> |------|------|
> | **fresh context** | 子代理从空白上下文启动：`sub_messages = [{"role": "user", "content": prompt}]` |
> | **shared filesystem** | 共享文件系统，但不共享思维历史 |
> | **summary-only return** | 只把结果摘要传回父 Agent |
> | **child tools filtered** | 子代理工具受限，不能无限递归生成 subagent |
> | **safety limit** | 有限轮数防失控：`for _ in range(30):` |
>
> **Subagent vs Teammate：**
> - **Subagent**：一次性，生成 → 干活 → 返回摘要 → 消亡（轻量）
> - **Teammate**：有身份、有 inbox、能持续协作
>
> Subagent 首先解决的是**上下文污染**问题，其次才是任务拆分问题。

---

## 4.1 — 子 Agent 隔离验证

**Prompt**：`explore the agents/ directory and give me a summary of all files`

**预期数据流**：
```
父 Agent 调用 task(...) → run_subagent 启动子循环
sub_messages=[] 全新起点 → 子 Agent 读文件 → 返回摘要给父
```

**关注点**：父 messages 只看到 task 的 result，看不到子 Agent 的内部调用过程

>
> 工具调用：
> ```python
> ToolUseBlock(name='task', input={
>     'agent_type': 'Explore',
>     'prompt': '请探索 tests 目录...提供 summary 报告（目录结构、文件用途、依赖关系、架构特点）'
> })
> ```
> *(截图略)*
>
> 子 Agent loop 循环调用，将结果中所有 TextBlock 拼接后作为 `tool_result` 返回父 Agent：
> *(截图略)*

## 4.2 — 子 Agent 不继承历史

**Prompt**：连续问 2 个问题，第 2 个用 task 做

**预期数据流**：父 Agent history 持续增长，但每次 subagent 的 `sub_messages=[]` 全新

**关注点**：验证"子 Agent 不继承对话历史"的核心特性

**结果**：✅

## 4.3 — 安全上限保护

**Prompt**：`spawn a subagent to do 31 tool calls`

**预期数据流**：子 Agent 的 `for _ in range(30)` 安全上限触发退出

**关注点**：s04:120 的 safety limit 保护

>
> 安全功能，暂未深测。

---

# 功能点 5：Context Compaction（s06）

> **🎯 目标**
> 观察三层压缩管道：`micro_compact → auto_compact → manual compact`

## 5.1 — micro_compact

**Prompt**：连续执行 5+ 个 bash 命令（如 `run: date; pwd; whoami; echo test; ls`）

> **📝 micro_compact 机制**
> - 每轮都跑，轻量清理旧 `tool_result`
> - 相关代码：
> *(截图略)*
> 针对 `{"type": "tool_result", ...}` 进行处理：
> *(截图略)*

## 5.2 — PRESERVE_RESULT_TOOLS 白名单

**Prompt**：读一个大文件后再执行 bash

**预期数据流**：`read_file` 的结果不被 `micro_compact` 清除（`PRESERVE_RESULT_TOOLS = {"read_file"}`）

**关注点**：为什么 `read_file` 被特殊对待：参考资料不该从上下文消失

>
> 设计思路可参考：读取的文件内容是"参考资料"，和一次性的命令输出不同，需保留在上下文中供后续使用。

## 5.3 — auto_compact 触发

**Prompt**：发送非常长的 prompt（粘贴大段代码）触发 tokens > 50000

**预期数据流**：
```
[auto_compact triggered] 打印
→ transcript 保存到 .transcripts/
→ messages 压缩为一条 summary
```

>
> Token 估算方式：
> *(截图略)*
>
> 超过预设值自动触发：
> *(截图略)*
>
> `auto_compact` 执行步骤：
> 1. 先做持久化（保存 transcript）
> 2. 调用模型压缩历史
> 3. 返回原内容路径 + 历史 overview
>
> *(截图略)*

## 5.4 — manual compact

**Prompt**：`please compact the conversation now`

**预期数据流**：`LLM 调用 compact tool → [manual compact] → auto_compact 立即执行`

>
> 工具调用：
> ```python
> ToolUseBlock(name='compress', input={'raw_arguments': 'null{}'})
> ```
>
> **设计亮点 — Deferred Side Effect**：
> `auto_compress` 没有放在 handler 里直接执行，而是先记下"本轮请求了压缩"，等本轮工具收尾完成后，由主循环统一处理。
>
> 这避免了在当前轮次执行中途修改整体状态——先收集结果，再统一执行副作用，是很典型的延迟副作用模式。
>
> *(截图略)*

---

# 功能点 6：持久化 Task System（s07）

>
> Prefer `task_create`/`task_update`/`task_list` for multi-step work. Use TodoWrite for short checklists.

> **🎯 目标**
> 观察 JSON 文件持久化、依赖图解析

> **💡 Task System 设计**
>
> **工作流：**
> ```
> 用户提出复杂目标
>   → 模型拆任务，调用 task_create / task_update
>   → 任务落到 .tasks/（JSON 文件持久化）
>   → 后续轮次继续读取并推进
> ```
>
> **Todo vs Task：**
> - **Todo**：本轮计划，in-memory，compact 后消失
> - **Task**：长期工作板，文件持久化，compact 不影响
>
> **依赖图（轻量 DAG）：**
>
> | 状态 | 条件 |
> |------|------|
> | 可执行 | `pending` 且 `blockedBy` 为空 |
> | 被阻塞 | `blockedBy` 不为空 |
> | 已完成 | `status == completed` |
>
> 完成任务时，`_clear_dependency()` 自动将该任务 ID 从其他任务的 `blockedBy` 中移除，解锁后继任务。（s07:95）

---

## 6.1 — 任务创建与持久化

**Prompt**：`create 3 tasks: setup env, write code, run tests`

**预期数据流**：3 个 `task_N.json` 写入 `.tasks/` 目录

**关注点**：文件持久化：退出程序后文件仍在，下次启动还能读到

>
> 调用 `task_create`：
> *(截图略)*
> *(截图略)*
>
> 工具输出结果：
> ```json
> {
>   "id": 2,
>   "subject": "set up env",
>   "description": "Configure development environment...",
>   "status": "pending",
>   "owner": null,
>   "blockedBy": []
> }
> ```
> *(截图略)*
> *(截图略)*
> *(截图略)*

## 6.2 — 依赖图设置

**Prompt**：`make task 4 blocked by task 3, task 3 blocked by task 2`

**预期数据流**：`task_update(addBlockedBy=[...])` → JSON 中 `blockedBy` 字段更新

**关注点**：依赖图的 add/remove 操作

>
> 并行两个 tool call：
> ```python
> ToolUseBlock(name='task_update', input={'task_id': 4, 'add_blocked_by': [3]})
> ToolUseBlock(name='task_update', input={'task_id': 3, 'add_blocked_by': [2]})
> ```
>
> Handler：
> ```python
> "task_update": lambda **kw: TASK_MGR.update(
>     kw["task_id"], kw.get("status"),
>     kw.get("add_blocked_by"), kw.get("remove_blocked_by")
> )
> ```
> *(截图略)*
> *(截图略)*
>
> JSON 文件内容：
> *(截图略)*

## 6.3 — 完成任务自动解锁

**Prompt**：`mark task 2 as completed`

**预期数据流**：`_clear_dependency(2)` 被调用 → 扫描所有 task JSON，从 `blockedBy` 中移除 2

>
> 核心代码（`update` 方法内）：
> *(截图略)*
>
> 效果：
> *(截图略)*

## 6.4 — task_list 输出格式

**Prompt**：运行 `task_list` 工具

**预期数据流**：输出带 `[x]`/`[>]`/`[ ]` 和 `blocked by: [N]` 的列表

**关注点**：验证文件系统比 in-memory TodoManager 更持久

**结果**：✅

---

# 功能点 7：Background Tasks（s08）

> **🎯 目标**
> 观察线程并发 + notification queue 的 drain 机制

> **💡 BackgroundManager 数据结构**
> *(截图略)*
>
> | 数据结构 | 用途 |
> |----------|------|
> | `tasks` dict | 完整状态存档，供 `check_background` 查询 |
> | `notifications` queue | 轻量事件流，供注入对话上下文 |

---

## 7.1 — 非阻塞后台执行

**Prompt**：`run in background: sleep 3 && echo done`

**预期数据流**：
```
立即返回 task_id
→ 3 秒后，下次 LLM 调用前 drain_notifications() 得到结果
→ 结果注入 messages（agent_loop:192）
```

**关注点**：主线程不阻塞；notification 在 agent_loop:192 注入

>
> 工具调用：
> ```python
> ToolUseBlock(name='background_run', input={'command': 'sleep 3 && echo done'})
> ```
>
> **`run()` 执行流程：**
> 1. 生成 task id
> 2. 任务登记为 `running`
> 3. 启动 `daemon=True` 线程（主程序退出时不强行保活进程）
> 4. 立即返回，不等任务结束
>
> *(截图略)*
>
> 执行完后，结果放入 notification 队列；`get_nowait()` 非阻塞取出（有元素立即拿，否则跳过）：
> *(截图略)*
>
> 通过 `BG.drain()` 取回后台内容：
> *(截图略)*
> *(截图略)*

## 7.2 — 并发线程安全

**Prompt**：`start 3 background tasks simultaneously`

**预期数据流**：3 个线程并发运行，`BackgroundManager.tasks` dict 中 3 条记录

**关注点**：多线程同时写 `notification_queue` 的安全性

>
> 采用线程安全的 `Queue`（内建同步机制），多线程同时 `put()` 不会写坏队列。
>
> **task dict 为何没上锁？**
> 每次操作前必须持有唯一 `tid`，不存在竞争写同一条记录的情况。
>
> **drain notification 为何安全？**
> - 只有主线程消费队列（`get_nowait`）
> - 后台线程只负责 `put`
> - 若某条通知在 `empty()` 检查后才进来，只是"这轮没 drain 到"，下轮仍会被取出，不会丢消息

## 7.3 — 任务状态查询

**Prompt**：`check background task <task_id>`

**预期数据流**：`BG.check(task_id)` 返回 `[running/completed] + result`

**关注点**：状态转换：`running → completed`

>
> *(截图略)*
> *(截图略)*

## 7.4 — Timeout 保护

**Prompt**：`run background: some_command_that_takes_400s`

**预期数据流**：300s 后 task status 变 `timeout`，结果为 `Error: Timeout (300s)`

**关注点**：s08:76 的 timeout 捕获

**结果**：✅

---

# 功能点 8：Agent Teams（s09）

> **🎯 目标**
> 观察多 Agent 通过 JSONL inbox 文件通信

> **💡 Agent Team 通信机制**
>
> **核心**：把队友收件箱里的消息取出来，注入到它自己的对话上下文。
>
> *(截图略)*
>
> ```python
> inbox = BUS.read_inbox(name)
> for msg in inbox:
>     messages.append({"role": "user", "content": json.dumps(msg)})
> ```
>
> - `BUS.read_inbox(name)` 读取 `.team/inbox/name.jsonl` 并清空
> - 站内信转换为 `"role": "user"` 消息，让模型在下次推理时看见

---

## 8.1 — Spawn Teammate

**Prompt**：`spawn a teammate named alice with role "code reviewer"`

**预期数据流**：
```
TEAM.spawn("alice", "code reviewer", prompt)
→ 新线程启动 → .team/config.json 写入 alice 记录
```

**关注点**：每个 teammate 是独立线程，有自己的 messages 列表

>
> 工具调用：
> *(截图略)*
>
> `spawn` 实现逻辑：
> *(截图略)*
>
> 1. 检查 alice 是否已在团队中
> 2. 若已存在，检查状态是否可重启（仅 `idle` / `shutdown` 可重启）
> 3. 若不存在，新建成员记录
> 4. 启动线程运行 teammate agent loop
>
> **Teammate Loop 内部：**
> - `user`：初始 prompt，定义角色
> - `assistant`：调用 `send_message` 工具广播自我介绍
>   ```python
>   ToolUseBlock(name='send_message', input={
>       'to': 'team',
>       'content': "Hello team! Alice here, ready to review code..."
>   })
>   ```
> *(截图略)*
> - `user`：工具返回结果
> - `assistant`：调用 `idle` 进入休眠
>
> 休眠实现：
> *(截图略)*

## 8.2 — 向 Teammate 发消息

**Prompt**：`send alice a message: please review agents/s01_agent_loop.py`

**预期数据流**：`BUS.send("lead", "alice", "...") → .team/inbox/alice.jsonl 追加一行 JSON`

**关注点**：文件即消息队列：JSONL 格式，append-only

>
> *(截图略)*
>
> 持久化写入信箱：
> *(截图略)*
>
> 消息结构：
> ```python
> msg = {
>     'type':      'message',
>     'from':      'lead',
>     'content':   '请review一下agents/s_01_agent_loop.py的代码',
>     'timestamp': 1777002217.306
> }
> ```
>
> alice inbox（`.team/inbox/alice.jsonl`）：
> *(截图略)*
>
> alice 读取 inbox → 添加到上下文 → 执行 review → 通过 `send_message` 工具回复 lead。

## 8.3 — /inbox 命令

**Prompt**：`/inbox`（内置命令）

**预期数据流**：`BUS.read_inbox("lead")` 读取 `.team/inbox/lead.jsonl` 并清空

**关注点**：Drain 语义：读完即清，不能重复读

**结果**：✅

## 8.4 — Broadcast

**Prompt**：`broadcast: all hands meeting in 5 minutes`

**预期数据流**：`BUS.broadcast` 向所有 teammates 各写一条 broadcast 类型消息

**关注点**：`VALID_MSG_TYPES` 过滤无效消息类型

>
> ```python
> ToolUseBlock(name='broadcast', input={'content': '📢 All hands meeting in 5 minutes! Please be ready.'})
> ```
> *(截图略)*
> *(截图略)*

## 8.5 — /team 命令

**Prompt**：`/team`（内置命令）

**预期数据流**：显示 `config.json` 中所有成员的 `name / role / status`

**关注点**：Alice 从 working 变为 idle 的时机

*(截图略)*

---

# 功能点 9：Skill Loading

> **🎯 目标**
> 观察两层 Skill 加载机制：Layer 1（描述注入）→ Layer 2（按需加载正文）

> **💡 Skill 系统设计**
>
> **加载流程（3 步）：**
>
> **① 扫描注册**（启动时）
> 将 `skills/` 目录下的 `SKILL.md` 通过 frontmatter 拆分为 `meta` 和 `body`：
> ```python
> self.skills[name] = {"meta": meta, "body": body, "path": str(f)}
> ```
> *(截图略)*
>
> **② 注入 System Prompt（Layer 1）**
> 只注入技能名和描述，不注入正文，拼接成：
> ```
>   - code-review: Perform thorough code reviews...
>   - agent-builder: ...
> ```
> *(截图略)*
> *(截图略)*
>
> **③ 运行时按需加载（Layer 2）**
> 模型调用 `load_skill(name)` → 返回完整 skill 正文：
> ```python
> TOOL_HANDLERS["load_skill"] = lambda **kw: SKILL_LOADER.get_content(kw["name"])
> # 返回：f'<skill name="{name}">\n{skill["body"]}\n</skill>'
> ```
> *(截图略)*
>
> agent loop 把这个结果作为 `tool_result` 加进 messages，后续模型就会基于该 skill 完成 task。
>
> **特点：** 轻量可扩展 · 启动时一次性扫描 · 同名覆盖

---

## 9.x — Skill 加载测试

**Prompts**：
1. `What skills are available?`
2. `Load the code-review skill first, then review quicksort.py.`
3. `Load the agent-builder skill and use it to outline a minimal agent.`
4. `Try loading a non-existent skill named foo.`

>
> Prompt 2 并行两个工具：
> ```python
> ToolUseBlock(name='load_skill', input={'name': 'code-review'})
> ToolUseBlock(name='read_file',  input={'path': '...quicksort.py'})
> ```
> *(截图略)*
>
> `load_skill` 返回：
> ```
> <skill name="code-review">
> # Code Review Skill
> You now have expertise in conducting comprehensive code reviews...
> </skill>
> ```
>
> Prompt 4：未知 skill 返回 `Error: Unknown skill foo`

**结果**：✅

**预期数据流**：
```
启动 → SkillLoader 扫描 skills/SKILL.md
→ SYSTEM 注入技能名/描述（Layer 1）
→ 模型调用 load_skill(name)
→ TOOL_HANDLERS["load_skill"] 返回 <skill name="...">正文</skill>（Layer 2）
→ 模型继续执行 read_file / write_file / edit_file
```

**关注点**：第 2/3 个 prompt 应出现 `load_skill` 工具调用，返回体里能看到 `skill name="..."`。

---

# 功能点 10：Team Protocols

> **🎯 目标**
> 观察 shutdown 协议 + plan approval 协议的完整握手流程

**Prompts**：
1. `Spawn alice as a coder. Then request her shutdown.`
2. `Check the shutdown request status and list teammates.`
3. `Spawn bob as a coder. Tell him not to edit files until he submits a plan for approval. When the request arrives, reject it with feedback: "too risky, split it first".`
4. `Spawn charlie as a coder. Tell him to submit a short plan before acting, then approve it.`

> **📝 Prompt 1 — Shutdown 流程**
> 调用 `spawn` 工具激活 alice：
> *(截图略)*
>
> 调用 `handle_shutdown_request`，消息投入信箱：
> *(截图略)*
>
> alice 取出消息，进入 shutdown 状态：
> *(截图略)*
> *(截图略)*
>
> 设置完 shutdown 状态后，通过 `return` 退出 `_loop` 函数，结束线程。

> **📝 Prompt 3 — Plan Approval 拒绝流程**
> bob 向 lead 发起 plan 请求：
> *(截图略)*
>
> 通过 `<inbox>` 块注入 lead 上下文：
> ```python
> if inbox:
>     messages.append({"role": "user", "content": f"<inbox>{json.dumps(inbox, indent=2)}</inbox>"})
> ```
> *(截图略)*

**结果**：✅

> **💡 预期数据流（s10 完整版，s_full 中已简化）**
>
> **① Shutdown 协议：**
> ```
> lead: shutdown_request
>   → 生成 req_id，写 shutdown_requests[req_id] = pending
>   → 写 alice inbox: shutdown_request
>
> alice: read_inbox
>   → 消息进入上下文 → 调用 shutdown_response
>
> teammate: shutdown_response
>   → 更新 req_id = approved/rejected
>   → 写 lead inbox: shutdown_response
>   → approve=true 时退出线程
> ```
>
> **② Plan Approval 协议：**
> ```
> alice: plan_approval
>   → 生成 req_id，写 plan_requests[req_id] = pending
>   → 写 lead inbox: plan text + req_id
>
> lead: read_inbox
>   → plan 消息进入上下文
>   → 调用 plan_approval(request_id, approve, feedback)
>
> lead: plan_approval
>   → 查 plan_requests[request_id]，更新 approved/rejected
>   → 根据 req["from"] 发回对应 inbox
>
> alice: read_inbox
>   → 收到审批结果 → 继续 / 修改计划 / 停止
> ```
>
> **③ 多 request 并发处理：**
> ```json
> {
>   "aaa11111": {"from": "alice", "plan": "Plan A", "status": "pending"},
>   "bbb22222": {"from": "alice", "plan": "Plan B", "status": "pending"},
>   "ccc33333": {"from": "bob",   "plan": "Plan C", "status": "pending"}
> }
> ```
> lead 根据不同 `req_id` 审批，根据 `from` 字段发回对应 inbox。

> **⚠️ 关注点**
> - `request_id` 是本章最核心的关联键，发起和响应必须一致
> - plan approval 是"协议约束 + 提示词约束"，不是硬阻塞事务；teammate 是否真的等审批，部分取决于模型是否遵守协议

---

# 功能点 11：Autonomous Agents

> **🎯 目标**
> 观察 teammate 自主认领任务 + idle 超时自动 shutdown

> **💡 Autonomous 流程**
>
> ```
> spawn teammate
>   → WORK phase：执行当前任务
>   → IDLE phase：每 5 秒轮询 inbox 和 .tasks
>       ├─ 有消息            → resume = True
>       ├─ 有可认领任务       → claim_task()（加锁）→ resume = True
>       └─ 60 秒内无事       → shutdown
>   → resume == True → 重新进入 WORK phase
> ```
>
> *(截图略)*
> *(截图略)*
> *(截图略)*
>
> Teammate loop 有两个顺序阶段：**WORK → IDLE**（工作完成后才进入 idle）

---

**Prompts**：
1. `Create 3 tasks on the board, then spawn alice and bob. Watch them auto-claim`
2. `Spawn a coder teammate and let it find work from the task board itself.`
3. `Create tasks with dependencies. Watch teammates respect the blocked order.`
4. `Let an idle teammate sit with no messages or tasks for 60 seconds, then list teammates.`

> **📝 Prompt 1 — 并发认领**
> alice 和 bob 同时查看了 task 1，alice 先 claim（`_claim_lock` 避免两人抢到同一个任务）：
> *(截图略)*
> *(截图略)*

**预期数据流**：
```
task_create → .tasks/task*.json status=pending
→ spawn_teammate → WORK phase
→ 调用 idle 或 stop_reason != tool_use → 状态切到 idle
→ 每 5 秒轮询 inbox + scan_unclaimed_tasks()
→ 命中可认领任务 → claim_task() 加锁，写回 owner/status=in_progress
→ 注入 "auto-claimed Task #..."
→ teammate 恢复 WORK phase
→ 60 秒内无消息也无任务 → 自动 shutdown
```

> **⚠️ 关注点**
> - 可认领任务必须同时满足：`pending`、`owner` 为空、`blockedBy` 为空
> - 这是异步行为，观测时需留出轮询周期（5s）的时间
> - 核心观测对象：`.tasks/` 中的 `owner/status` 变化，`.team/config.json` 中 teammate 的状态变化
> - **身份重注入机制**：当上下文很短或压缩后恢复工作（`len(msg) <= 3`）时，会插入 identity 信息，是内部韧性点，黑盒上不一定直观看到，但值得留意

---

# 功能点 12：Worktree + Task Isolation

> **🎯 目标**
> 观察 git worktree 与 task 系统的绑定，实现并行任务的文件系统隔离

> **💡 设计背景**
>
> **问题**：多 teammate 并行时，都在同一目录改代码，很容易互相污染：
> - 未提交改动混在一起，难以追溯哪个任务改了哪些文件
> - 一个任务失败时不好回滚
> - 两个 teammate 并行可能互相踩踏
>
> **解决思路：**
>
> | 职责 | 工具 |
> |------|------|
> | 管"做什么" | 任务板（`.tasks/`）|
> | 管"在哪里做" | git worktree |
> | 两者绑定键 | `task_id` |
>
> **典型工作路径：**
> ```
> task_create
>   → 写 .tasks/task_1.json
>
> worktree_create(name, task_id=1)
>   → git worktree add -b wt/name .worktrees/name HEAD
>   → 写 .worktrees/index.json
>   → 更新 task_1.json: worktree=name, status=in_progress
>   → 写 events.jsonl
>
> worktree_run(name, command)
>   → cwd=.worktrees/name 执行命令
>
> worktree_remove(name, complete_task=True)
>   → git worktree remove
>   → task_1.json: status=completed, worktree=""
>   → index.json: status=removed
>   → 写 events.jsonl
> ```

---

**Prompts**：
1. `Create tasks for backend auth and frontend login page, then list tasks.`
2. `Create worktree "auth-refactor" for task 1, then create worktree "ui-login" for task 2.`
3. `Run "git status --short --branch" in worktree "auth-refactor".`
4. `Keep worktree "ui-login", then list worktrees and inspect events.`
5. `Remove worktree "auth-refactor" with complete_task=true, then list tasks, worktrees, and events.`

>
> *(截图略)*
> *(截图略)*
> *(截图略)*

**预期数据流**：
```
task_create → .tasks/task_N.json status=pending
→ worktree_create(name, task_id)
  → git worktree add -b wt/name .worktrees/name HEAD
  → .worktrees/index.json 追加 entry
  → TASKS.bind_worktree() 写回 task.worktree，pending → in_progress
→ worktree_run 在 .worktrees/name 下执行命令
→ worktree_keep 或 worktree_remove
  → EventBus 写入 .worktrees/events.jsonl
  → complete_task=true → 任务置 completed 并解绑 worktree
```

> **⚠️ 关注点**
> - 必须在 git 仓库中测试，否则 `worktree_*` 直接报错
> - `worktree_remove` 在有未提交改动时可能失败，必要时测 `force=true`
> - 核心验证对象不是聊天文本，而是三份持久化状态：`.tasks/`、`.worktrees/index.json`、`.worktrees/events.jsonl`

---

# 实测感受与 Bug 记录

> **✅ 整体亮点**
> 可扩展性好：tools、skill、teammate 均支持插拔式扩展

> **🐛 已知问题**
>
> **① 假 working 状态** ✅ 已修复
> 直接中断退出时，teammate 在 `config.json` 中会残留 `working` 状态。
>
> **② 消息丢失风险**
> lead 向 teammate 发消息时，不会确认 teammate 线程是否在线，消息不会进入子 loop（出现无人回复）。
> → 改进方案：调用发消息工具前先 `check` 队员状态，不在线则通过 `spawn` 激活。
>
> **③ Team Protocol 不完整**
> `s_full.py` 里 teammate 没有 track 不同 `request_id`，建议用 `s_10` 进行完整测试。
>
> **④ 任务认领无角色匹配**
> 自治 teammate 暂不支持根据个人 `role` 与 task 相关性认领任务，目前是先到先得。
