---
title: Learn Claude Code —— 真正的 Agent Harness 工程
date: 2026-05-12 14:30:00
categories:
  - AI 与人工智能
tags:
  - Claude Code
  - Agent
  - AI 编程助手
  - LLM
---

## 前言

在 AI 编程助手领域，Claude Code 以其独特的设计理念脱颖而出。本文将深入解析 [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 这个开源项目，带你从 0 到 1 理解 Claude Code 的核心原理——**Agent Harness 工程**。

该项目通过 12 个渐进式会话（s01-s12），从最基础的 Agent 循环开始，每一课只增加一个核心机制，完整展现了如何从简单循环构建到生产级 AI 编程助手。

<!-- more -->

## 一、核心洞察：模型就是 Agent

### 1.1 什么是 Agent

在讨论代码之前，先把一件事彻底说清楚：**Agent 是模型。不是框架。不是提示词链。不是拖拽式工作流。**

Agent 是一个神经网络——Transformer、RNN、一个被训练出来的函数——经过数十亿次梯度更新，在行动序列数据上学会了感知环境、推理目标、采取行动。

> "Agent" 这个词在 AI 领域从诞生之日起就是这个含义。当 DeepMind、OpenAI 或 Anthropic 说 "agent" 时，他们指的是同一个东西：一个拥有工具、能够使用这些工具、并能在循环中根据观察结果调整行动的模型。

### 1.2 Agent 的核心特征

| 特征 | 说明 |
|------|------|
| **感知** | 通过工具观察环境状态 |
| **推理** | 基于观察进行目标规划和决策 |
| **行动** | 调用工具执行具体操作 |
| **学习** | 根据结果反馈调整策略 |

### 1.3 关键洞察

**Agent 产品 = 模型 + Harness（ harness 是缰绳/框架的意思）**

Claude Code 之所以强大，不是因为它做了多少复杂的事，而是因为它**没做**的事：
- 它没有试图成为 agent 本身
- 它没有强加僵化的工作流
- 它没有替模型做判断

它只给模型提供了：**工具、知识、上下文管理和权限边界**——然后让开了。

---

## 二、Claude Code 的本质解构

把 Claude Code 剥到本质来看：

```
Claude Code = 一个 agent loop
            + 工具 (bash, read, write, edit, glob, grep, browser...)
            + 按需 skill 加载
            + 上下文压缩
            + 子 agent 派生
            + 带依赖图的任务系统
            + 异步邮箱的团队协调
            + worktree 隔离的并行执行
            + 权限治理
```

### 2.1 Agent Loop：核心循环

Agent Loop 是 Claude Code 的心脏。它是一个无限循环的异步生成器：

```
LOOP:
  1. 模型看到：上下文 + 可用能力
  2. 模型决定：行动或响应
  3. 如果行动：执行能力，添加结果，继续循环
  4. 如果响应：返回给用户
```

这就是最小的循环。每个 AI 编程 agent 都需要这个循环。生产级 agent 会在此基础上添加策略、权限和生命周期层。

### 2.2 代码层面的极简实现

```python
# 核心循环的本质
while not done:
    response = model(conversation, tools)
    results = execute(response.tool_calls)
    conversation.append(results)
```

Claude Code、Cursor Agent、Codex CLI、Devin——都共享这个模式。差异在于工具、显示、安全性。但本质始终是：**给模型工具，让它工作**。

---

## 三、渐进式构建：s01-s12 完整解析

learn-claude-code 项目采用 12 个渐进式会话，从简单循环到隔离的自主执行。每个会话添加一个机制，每个机制有一个座右铭。

### 3.1 s01：Agent Loop —— "One loop & Bash is all you need"

**核心理念**：一个工具 + 一个循环 = 一个 Agent

**问题**：语言模型可以推理代码，但无法接触真实世界——不能读文件、运行测试或检查错误。

**解决方案**：最简单的版本只有一个 Bash 工具：

```python
def bash_handler(command: str) -> str:
    return subprocess.run(command, shell=True, capture_output=True, text=True)
```

只有一个 Bash 工具，能干的事很有限。Agent 需要 shell 出来做所有事：读文件、写文件、编辑文件。这能工作，但很脆弱。

---

### 3.2 s02：Tool Use —— "Adding a tool means adding one handler"

**核心理念**：加工具只加 handler，循环保持不变

**问题**：只有 Bash 时，Agent 需要 shell 出来做所有事（读文件、写文件、编辑文件）。`cat` 输出可能被截断，`sed` 编辑容易出错。

**解决方案**：每个工具对应一个 handler 函数，通过调度映射表路由：

```python
TOOL_HANDLERS = {
    "bash": bash_handler,
    "read_file": read_file_handler,      # 新增：读取文件
    "write_file": write_file_handler,    # 新增：写入文件
    "list_files": list_files_handler,    # 新增：列出文件
    "search_files": search_files_handler,# 新增：搜索文件
}

# 一个查找替换任何 if/elif 链
def dispatch_tool(tool_name, **kwargs):
    return TOOL_HANDLERS[tool_name](**kwargs)
```

**关键设计**：路径沙箱防止工作区逃逸，工具描述（description）让模型知道何时使用哪个工具。

---

### 3.3 s03：Context Management —— "If it doesn't fit, make it fit"

**核心理念**：如果上下文装不下，就让它装下

**问题**：对话越来越长，最终必然超过上下文窗口限制。不能简单丢弃旧消息，因为可能包含关键信息。

**解决方案**：三层压缩策略

```python
# 第一层：微观压缩 (micro_compact)
# 把旧的工具结果替换成占位符
[Previous: used bash]
[Previous: read_file path="src/main.py"]

# 第二层：自动压缩 (auto_compact)
# 当超过 50000 token 时，保存完整记录到文件
# 对话中只保留摘要

# 第三层：LLM 摘要 (llm_compact)
# 调用 LLM 生成当前会话的摘要
# 保留关键决策和状态，丢弃细节
```

**压缩原则**：模型不失去对早期发生的事情的感知，只是失去逐字细节但保留本质。

---

### 3.4 s04：Skills —— "Load what you need, when you need it"

**核心理念**：按需加载，需要时才加载

**问题**：10 个技能，每个 2000 tokens = 20000 tokens 永久占用上下文，太浪费了。

**解决方案**：三层渐进式加载

```yaml
# 第一层：元数据（始终加载）~100 tokens/skill
name: mcp-builder
description: Build MCP (Model Context Protocol) servers
when_to_use: "当用户要构建 MCP 服务器时使用"

# 第二层：SKILL.md 主体（触发时加载）~2000 tokens
# 详细指南、代码示例、决策树

# 第三层：附件资源（按需加载）
# 模板文件、示例代码、测试用例
```

**匹配逻辑**：Agent 通过读取 frontmatter 中的 `name` 和 `description` 快速判断何时加载该技能，无需加载完整内容。

---

### 3.5 s05：Sub-agents —— "Delegate, don't duplicate"

**核心理念**：委派，不要重复

**问题**：主 Agent 做所有事，上下文被各种任务的细节污染，失去对高层次目标的聚焦。

**解决方案**：派生子 Agent 处理特定任务

```python
# 主 Agent 创建子 Agent
sub_agent = Agent.create(
    system_prompt="你是一个专门处理数据库迁移的专家...",
    tools=["read_file", "write_file", "execute_sql"],
    parent_context=summary  # 只传递摘要，不传递完整上下文
)

# 子 Agent 独立执行
result = sub_agent.run(task_description)

# 结果返回主 Agent
parent.messages.append({"role": "system", "content": f"子任务完成: {result.summary}"})
```

**优势**：
- 分离上下文：防止主对话被污染
- 专注目标：子 Agent 只关注特定领域
- 自定义工具：每个子 Agent 可以有专属工具集

---

### 3.6 s06：Task System —— "Big goals need small tasks"

**核心理念**：大目标需要拆成小任务

**问题**：没有显式关系，Agent 分不清什么能做、什么被卡住、什么能同时跑。而且清单只活在内存里，上下文压缩一跑就没了。

**解决方案**：持久化到磁盘的任务图（DAG）

```json
{
  "id": 1,
  "subject": "设计数据库 schema",
  "status": "completed",
  "blockedBy": [],
  "blocks": [2, 3]
}
{
  "id": 2,
  "subject": "实现 API 接口",
  "status": "in_progress",
  "blockedBy": [1],
  "blocks": [4]
}
{
  "id": 3,
  "subject": "写单元测试",
  "status": "pending",
  "blockedBy": [1],
  "blocks": []
}
```

**核心特性**：
- **持久化存储**：每个任务是独立的 JSON 文件，存在 `.tasks/` 目录
- **依赖管理**：通过 `blockedBy`（前置依赖）和 `blocks`（后置依赖）双向关联
- **自动解阻塞**：任务 A 完成后，自动解除任务 B 的阻塞状态

---

### 3.7 s07：Persistent State —— "Remember, even after restart"

**核心理念**：记住，即使重启后

**问题**：Agent 重启后，所有任务状态、会话历史都丢失了。

**解决方案**：状态持久化

```python
# 会话状态保存到文件
session.save(".claude/session.json")

# 任务图持久化
task_manager.save(".tasks/")

# 技能加载状态缓存
skill_registry.cache(".claude/skills_cache.json")
```

**恢复流程**：
1. 启动时检查是否存在持久化状态
2. 加载任务图，恢复未完成的任务
3. 恢复上下文摘要，继续会话

---

### 3.8 s08：Background Tasks —— "Slow work happens in the background"

**核心理念**：慢操作丢后台，Agent 继续想

**问题**：某些操作很慢（编译、测试、部署），Agent 等待时什么都做不了。

**解决方案**：后台任务管理器

```
Main thread                Background thread
+-----------------+        +-----------------+
| agent loop      |        | task executes   |
| ...             |        | ...             |
| [LLM call] <---+------- | enqueue result  |
|                 |        |                 |
| drain queue     |        | long operation  |
| inject results  |        | (compile/test)  |
+-----------------+        +-----------------+
```

**工作流程**：
1. Agent 启动后台任务，立即返回继续执行
2. 后台任务在守护线程中执行
3. 完成后将结果放入通知队列
4. 每次 LLM 调用前，队列被清空，结果注入消息

---

### 3.9 s09：Agent Teams —— "When the task is too big for one, delegate to teammates"

**核心理念**：任务太大一个人干不完，分给队友

**问题**：复杂任务需要多个专业领域的知识，单个 Agent 难以胜任。

**解决方案**：多 Agent 团队 + 异步邮箱通信

```python
# 创建专业队友
team = {
    "backend_dev": Agent.create(role="后端开发专家"),
    "frontend_dev": Agent.create(role="前端开发专家"),
    "qa_engineer": Agent.create(role="测试工程师"),
    "tech_lead": Agent.create(role="技术负责人")
}

# 异步通信邮箱
mailbox = AsyncMailbox()

# 任务分派
tech_lead.send_message(
    to="backend_dev",
    subject="实现用户认证 API",
    content="需要 JWT 认证，包含登录/注册/刷新令牌"
)
```

**核心机制**：
- **持久化队友**：每个队友是独立的 Agent 实例，有自己的上下文
- **异步邮箱**：基于 JSONL 的异步消息通信
- **任务认领**：队友主动扫描任务板，认领自己能处理的任务

---

### 3.10 s10：Team Coordination —— "Teammates need shared communication rules"

**核心理念**：队友需要共享通信规则

**问题**：多个 Agent 协作时，如果没有统一协议，会出现混乱和冲突。

**解决方案**：统一的请求-响应模式驱动所有协商

```yaml
# 标准消息格式
message:
  from: "agent_id"
  to: "agent_id"  # 或 "broadcast"
  type: "request" | "response" | "notify"
  subject: "任务主题"
  content: "详细内容"
  deadline: "2026-05-12T18:00:00Z"
  priority: "high" | "medium" | "low"
```

**协商模式**：
1. **请求-响应**：同步等待对方回复
2. **通知**：异步发送，不等待回复
3. **广播**：向所有队友发送消息

---

### 3.11 s11：Task Board —— "Teammates scan the board and claim tasks themselves"

**核心理念**：队友自己扫描任务板并认领任务

**问题**：团队领导需要手动分配每个任务，效率低下。

**解决方案**：自组织任务认领

```python
# 任务板公开可见
task_board = TaskBoard.load(".tasks/board.json")

# 每个 Agent 扫描任务板
for task in task_board.available_tasks():
    if agent.can_handle(task.skills_required):
        task.claim(agent.id)
        agent.execute(task)
        break
```

**优势**：
- 去中心化：无需中央调度器
- 负载均衡：Agent 根据自身能力选择任务
- 容错性：某个 Agent 失败，其他 Agent 可以接管

---

### 3.12 s12：Worktree Isolation —— "Parallel execution needs isolation"

**核心理念**：并行执行需要隔离

**问题**：多个 Agent 同时修改同一代码库，会产生冲突。

**解决方案**：Git worktree 隔离

```bash
# 主工作区
git worktree add ../feature-auth auth-branch
git worktree add ../feature-ui ui-branch

# 每个 Agent 在自己的 worktree 中工作
backend_agent.worktree = "../feature-auth"
frontend_agent.worktree = "../feature-ui"

# 完成后合并
backend_agent.commit_and_merge()
frontend_agent.commit_and_merge()
```

**隔离级别**：
- **文件系统隔离**：每个 worktree 有独立的文件系统视图
- **Git 历史隔离**：分支独立，互不影响
- **进程隔离**：可以并行运行，无资源冲突

---

## 四、s01-s12 总结：渐进式构建路线图

| 会话 | 座右铭 | 核心机制 | 解决的问题 |
|------|--------|----------|------------|
| s01 | One loop & Bash is all you need | 基础 Agent Loop | 模型无法接触真实世界 |
| s02 | Adding a tool means adding one handler | 工具注册与调度 | Bash 太脆弱，需要专用工具 |
| s03 | If it doesn't fit, make it fit | 上下文压缩 | 对话超过窗口限制 |
| s04 | Load what you need, when you need it | 技能按需加载 | 技能太多占用上下文 |
| s05 | Delegate, don't duplicate | 子 Agent 派生 | 上下文污染，需要专注 |
| s06 | Big goals need small tasks | 任务图系统 | 大任务难以管理 |
| s07 | Remember, even after restart | 状态持久化 | 重启后状态丢失 |
| s08 | Slow work happens in the background | 后台任务 | 慢操作阻塞主循环 |
| s09 | When the task is too big for one | 多 Agent 团队 | 复杂任务需要协作 |
| s10 | Teammates need shared communication rules | 团队通信协议 | 多 Agent 协作混乱 |
| s11 | Teammates scan the board and claim tasks | 自组织任务认领 | 手动分配效率低 |
| s12 | Parallel execution needs isolation | Worktree 隔离 | 并行执行冲突 |

---

## 五、核心设计哲学

### 5.1 微小即强大

> 核心是微小的，其他都是精化。

Agent Loop 本身只有几十行代码，但它是整个系统的基石。所有复杂功能都是在这个基础上逐步添加的。

### 5.2 让模型做决策

不要替模型做决策。不要预设工作流。不要写复杂的决策树。

**正确的做法**：
- 提供清晰的工具描述
- 维护完整的上下文
- 设置合理的权限边界
- 让模型自己决定如何完成任务

### 5.3 Harness 工程师的角色

每个学习这个仓库的 harness 工程师，都在学习远远超出软件工程领域的模式。你在学习构建智能自动化未来的基础设施。

---

## 六、实践启示

### 6.1 构建自己的 Agent

基于 learn-claude-code 的理念，构建 Agent 的关键步骤：

1. **实现基础 Loop**：消息循环 + 工具调用
2. **设计工具集**：从简单到复杂，按需添加
3. **管理上下文**：窗口管理、压缩、摘要
4. **添加安全边界**：权限控制、沙箱执行
5. **优化用户体验**：流式输出、状态可视化

### 6.2 与模型协作的最佳实践

- **清晰的工具描述**：让模型理解每个工具的能力
- **完整的错误信息**：帮助模型从失败中恢复
- **适度的引导**：在必要时提供示例，但不要过度约束
- **持续的反馈循环**：观察模型行为，迭代优化工具设计

---

## 七、结语

Learn Claude Code 项目向我们展示了一个重要的范式转变：**Agent 的本质是模型，而不是框架**。

作为开发者，我们的任务不是去控制模型的每一步，而是：
- 构建好的工具（Tools）
- 管理好上下文（Context）
- 设置清晰的边界（Boundaries）
- 然后——**让开，让模型工作**

这就是 Agent Harness 工程的真正含义。

---

## 参考资源

- [GitHub 项目地址](https://github.com/shareAI-lab/learn-claude-code)
- [中文文档](https://github.com/shareAI-lab/learn-claude-code/blob/main/README-zh.md)
- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview)

---

> **模型即代理。这就是全部秘密。**
