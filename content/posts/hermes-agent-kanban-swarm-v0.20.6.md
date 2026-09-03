---
title: "Hermes Agent Kanban Swarm 深度解析：从 v0.16 到 v0.20.6 的架构演进"
date: 2026-09-03
tags: [hermes, kanban, swarm, multi-agent, ai-agents, v0.20.6]
author: yxwuxie
---
# Hermes Agent Kanban Swarm 深度解析：从 v0.16 到 v0.20.6 的架构演进

你是否曾面对一个包含调研、写作、审核、发布的复杂任务，却发现单次对话根本无法承载？传统的 AI 助手模式是"一问一答"——你抛出问题，它给出答案，然后一切归零。但当任务涉及多个阶段、需要不同角色接力、或者可能因为意外中断需要恢复时，这种模式就显得力不从心了。

Hermes Agent 的 **Kanban Swarm** 机制正是为了解决这个问题而诞生的。它是一个 durable（持久化）的多智能体协作看板系统，让任务不再是短暂的进程内调用，而是可以跨越会话、等待人工介入、被审计追溯的持久工作队列。

本文基于 Hermes Agent v0.20.6 最新文档与源码，深入解析 Kanban Swarm 的架构设计、配置方法和实战用法，并与上一代方案 `delegate_task` 进行对比。

---

## 一、Kanban Swarm 是什么？解决什么问题？

### 1.1 核心定义

Kanban Swarm 是 Hermes Agent 的**多智能体看板系统**。它的核心设计思想是：**将任务视为有状态的工作队列，而非无状态的函数调用**。

在 v0.16 时代，Hermes 主要依赖 `delegate_task` 来派生子代理。这种方式适合短途、同步的任务——父代理等待子代理返回结果后继续执行。但当任务涉及多个阶段、需要不同角色参与、或者可能因为意外中断需要恢复时，`delegate_task` 就显得力不从心。

Kanban Swarm 解决了以下痛点：

| 痛点 | 传统方案的问题 | Kanban Swarm 的解法 |
|------|----------------|---------------------|
| 任务中断后丢失 | 子代理进程随父进程退出 | SQLite 持久化，重启后可恢复 |
| 多角色协作混乱 | 所有子代理匿名，无法区分角色 | 每个任务指定 `assignee`（profile 名） |
| 需要人工介入 | 无法暂停等待人类决策 | 支持 `blocked` 状态，人工解除阻塞 |
| 审计困难 | 历史被压缩丢失 | 所有操作记录在数据库中，可追溯 |
| 跨进程可见性 | 子代理结果仅限父上下文 | 任何 profile 均可读写任意任务 |

一句话总结：**`delegate_task` 是函数调用，Kanban Swarm 是有状态的工作队列。**

### 1.2 与 delegate_task 的关键区别

```
delegate_task          vs          Kanban Swarm
─────────────────────────────────────────────────────
形状        RPC 调用（fork → join）    持久化消息队列 + 状态机
父代理      阻塞等待子代理返回          创建后即放手（fire-and-forget）
子代理身份  匿名子代理                 命名 profile，带持久记忆
可恢复性   无 — 失败即失败            阻塞→解除→重运行；崩溃→收回
人工介入   不支持                     任何时候可评论/解除阻塞
每任务代理  一次调用 = 一个子代理       任务生命周期内可 N 个代理接力
审计轨迹   随上下文压缩丢失           SQLite 永存行记录
协调方式   层级式（调用者→被调用者）    对等 — 任意 profile 读写任意任务
```

**选择建议：**
- 使用 `delegate_task`：父代理需要短期推理答案后立即继续、无人工介入、结果回注入父上下文
- 使用 Kanban：工作跨越代理边界、需要存活重启、可能需要人工输入、可能被不同角色拾取、事后需要可发现

两者可以共存：Kanban Worker 内部可以调用 `delegate_task`。

---

## 二、核心架构

### 2.1 组件概览

Kanban Swarm 由以下几个核心组件构成：

```
┌─────────────────────────────────────────────────────────┐
│                    User / CLI                            │
│              (hermes kanban <verb>)                      │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                  Gateway (Dispatcher)                    │
│           内置调度器，定期轮询 ready 状态任务            │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│              Kanban DB (SQLite)                          │
│  tasks | task_runs | task_links | boards                │
└──────────────────────┬──────────────────────────────────┘
                       │
    ┌──────────────────┼──────────────────┐
    ▼                  ▼                  ▼
┌───────┐         ┌───────┐         ┌───────┐
│Worker │         │Worker │         │Worker │
│A      │         │B      │         │C      │
└───────┘         └───────┘         └───────┘
```

**Dispatcher（调度器）**：长期运行的循环，默认每 60 秒执行一次：
- 回收过期的认领（stale claims）
- 回收崩溃的 worker（PID 消失但 TTL 未到期）
- 将 `todo` 提升为 `ready`（当所有父任务完成时）
- 原子化认领任务并派生 assigned profile

**关键配置**：
```yaml
kanban:
  dispatch_in_gateway: true        # 默认值：内置调度器
  dispatch_interval_seconds: 60    # 默认值：每分钟检查一次
  review_dispatch: true            # 默认值：review 状态自动派发带有 sdlc-review 技能的 reviewer profile
  failure_limit: 2                 # 连续失败多少次后自动阻塞
  dispatch_stale_timeout_seconds: 14400  # 4小时：心跳超时判定为卡死
  auto_decompose: true             # 默认值：自动分解 triage 任务
  auto_decompose_per_tick: 3       # 每 tick 最多分解 3 个
```

### 2.2 数据流

任务的生命周期数据流如下：

1. **创建**：用户或编排代理通过 `kanban_create` 创建任务，指定 `assignee`（执行人）和 `body`（任务描述）
2. **调度**：Gateway 中的 Dispatcher 定期（默认 60 秒）扫描 `ready` 状态的任务
3. **派生**：Dispatcher 启动指定的 profile 作为独立 OS 进程，设置环境变量 `HERMES_KANBAN_TASK=<task_id>`
4. **执行**：Worker 通过 `kanban_*` 工具集与看板交互，完成实际工作
5. **完成**：Worker 调用 `kanban_complete` 提交结果，任务状态变为 `done`

**关键设计**：Worker 不通过 `hermes kanban` CLI 与看板交互，而是直接使用 `kanban_*` 工具集。这是因为：
- **后端可移植性**：Worker 可能在 Docker/Modal 等远程后端运行，CLI 不可用
- **无 shell 引号问题**：结构化工具参数跳过 shlex + argparse 的脆弱性
- **更好的错误处理**：工具结果是结构化 JSON，模型可以推理

### 2.3 任务状态机（v0.20.6 更新）

每个任务有多个核心状态，形成有向图：

```
triage → todo → ready → running → done → archived
                 ↓       ↑
              blocked ←┘
                 ↓
               review → done
```

**状态详解**：

| 状态 | 含义 | 转换条件 |
|------|------|----------|
| `triage` | 原始想法，等待分解或规格化 | 新建任务默认进入；可由人工放入 |
| `todo` | 已创建但尚未分配，或等待依赖项完成 | 父任务完成时自动晋升为 `ready` |
| `ready` | 已分配，等待 Dispatcher 认领 | Dispatcher 认领后变为 `running` |
| `running` | Worker 正在执行中 | 调用 `kanban_complete` 或 `kanban_block` |
| `blocked` | 需要人工输入或遇到了能力限制 | 手动 `unblock` 回到源阶段 |
| `review` | 提交给人工或自动审核 | 审核通过 → `done`；需要修改 → `ready` |
| `done` | 已完成 | 可选择 `archive` |
| `archived` | 归档（从看板隐藏） | 不可逆，仅用于清理 |

**v0.20.6 新增特性**：

1. **`archived` 状态**：任务完成后可选择归档，从看板隐藏，用于清理已完成的历史任务
2. **`review_dispatch` 配置**：默认 `true`，review 状态自动派发带有 `sdlc-review` 技能的 reviewer profile
3. **`goal_mode`**：Worker 可以在目标循环中运行，辅助 judge 检查输出是否符合验收标准
4. **Per-task 模型覆盖**：可以为单个任务指定特定模型和 provider
5. **Workspace 三种模式**：`scratch`（默认，临时）、`dir:<path>`（持久目录）、`worktree`（git worktree）

### 2.4 依赖关系

任务之间可以通过 `parents` 建立依赖关系。当一个任务有父任务时，它最初处于 `todo` 状态；只有当所有父任务都完成时，它才会自动晋升为 `ready`。这种设计使得复杂的工作流可以自然地表达为 DAG（有向无环图）。

**Orchestrator 模式**：编排代理的职责是分解目标、建立依赖、分配角色，然后退后让 Worker 执行：

```python
# 以下为工具调用示意，非可执行 Python 脚本
# 1. 创建父任务（orchestrator 自己）
parent_id = kanban_create(
    title="博客自动化工作流：Kanban Swarm 深度解析",
    assignee="orchestrator"
)

# 2. 创建子任务，形成依赖链
research_task = kanban_create(
    title="调研 Kanban Swarm v0.20.6 文档和源码",
    assignee="researcher",
    parents=[parent_id],
    body="""调研 Hermes Agent v0.20.6 Kanban Swarm 最新功能：
    1. 官方文档更新
    2. 对比 v0.16 的变化（archived 状态、review_dispatch 等）
    3. 输出功能架构图和配置说明"""
)

writing_task = kanban_create(
    title="撰写博客文章",
    assignee="writer",
    parents=[research_task],
    body="""基于 researcher 的输出，撰写《Hermes Agent Kanban Swarm 深度解析》技术博客。
    要求：1500~3000 字，面向技术读者。"""
)

review_task = kanban_create(
    title="审核博客内容",
    assignee="reviewer",
    parents=[writing_task],
    skills=["sdlc-review"]  # 附加审核技能
)

publish_task = kanban_create(
    title="发布到 GitHub",
    assignee="publisher",
    parents=[review_task]
)

# 3. 完成编排任务
kanban_complete(
    summary="已分解为 4 个子任务：researcher → writer → reviewer → publisher",
    metadata={"children": [research_task, writing_task, review_task, publish_task]}
)
```

**重要原则**：编排决策属于编排者，不属于 Worker。如果两个并行任务需要共享同一命名约定或 Schema，编排者必须提前决定并写入两个任务的 body 中。

---

## 三、配置指南（v0.20.6）

### 3.1 基本配置

在 `~/.hermes/config.yaml` 中，Kanban 相关配置位于 `kanban` 节：

```yaml
kanban:
  dispatch_in_gateway: true        # 默认值：内置调度器
  dispatch_interval_seconds: 60    # 默认值：每分钟检查一次
  review_dispatch: true            # 默认值：review 状态自动派发 reviewer
  failure_limit: 2                 # 连续失败多少次后自动阻塞
  dispatch_stale_timeout_seconds: 14400  # 4小时：心跳超时判定为卡死
  auto_decompose: true             # 自动分解 triage 任务
  auto_decompose_per_tick: 3       # 每 tick 最多分解 3 个
  orchestrator_profile: ""         # 编排任务分配给哪个 profile
  default_assignee: ""             # LLM 选择未知 profile 时的 fallback
  auto_subscribe_on_create: true   # 创建任务时恢复创建者会话
  done_sub_retention_days: 30      # done 状态的订阅保留天数
```

> **为什么重要**：`dispatch_stale_timeout_seconds` 是防止 Worker 卡死的关键配置。默认 4 小时，超过此时长且无心跳信号时，Dispatcher 会回收任务并重新入队。

### 3.2 辅助 LLM 配置

v0.20.6 引入了两个辅助 LLM 槽位：

```yaml
auxiliary:
  kanban_decomposer:               # 分解任务图使用的模型
    model: "gpt-4o"
    provider: "openai"
  profile_describer:               # 自动生成 profile 描述的模型
    model: "gpt-4o-mini"
    provider: "openai"
```

### 3.3 Dashboard 配置

```yaml
dashboard:
  kanban:
    default_tenant: ""             # 默认租户
    lane_by_profile: false         # Running 列按 profile 分组
    include_archived_by_default: false  # 默认显示归档任务
    render_markdown: true          # 渲染 markdown
```

### 3.4 关键环境变量

| 变量 | 说明 |
|------|------|
| `HERMES_KANBAN_TASK` | 当前 worker 负责的任务 ID（由 Dispatcher 自动设置） |
| `HERMES_KANBAN_BOARD` | 当前 worker 所在的看板 slug |
| `HERMES_KANBAN_WORKSPACE` | 当前任务的工作空间路径 |
| `HERMES_KANBAN_DISPATCH_IN_GATEWAY` | 覆盖 `dispatch_in_gateway` 配置（调试用） |
| `HERMES_KANBAN_STOP_NUDGE` | 禁用 worker 停止时的合成提示（默认启用） |

---

## 四、Worker 执行协议

### 4.1 标准执行流程

每个 Worker 的实际执行过程遵循固定协议：

```python
# 以下为工具调用示意，非可执行 Python 脚本
# 1. 读取任务
kanban_show()  # 无参调用，使用 HERMES_KANBAN_TASK

# 2. 进入工作空间
cd $HERMES_KANBAN_WORKSPACE

# 3. 执行工作（长操作时定期报告心跳）
kanban_heartbeat(note="调研进行中，已收集 5 份资料")
# ... 执行工作 ...
kanban_heartbeat(note="正在生成报告")

# 4. 提交结果
kanban_complete(
    summary="已完成主题调研，整理了架构文档和源码分析",
    metadata={
        "sources_count": 5,
        "key_findings": ["SQLite持久化", "Dispatcher轮询", "依赖提升"]
    }
)
```

### 4.2 心跳机制

对于长时间运行的任务，Worker 应该定期调用 `kanban_heartbeat` 报告进度：

```python
# 工具调用示意
kanban_heartbeat(note="已处理 4/8 个文件")
# ... 执行工作 ...
kanban_heartbeat(note="正在生成报告")
```

**重要**：如果任务可能运行超过 1 小时，**必须**每小时至少调用一次心跳。Dispatcher 会在超过 `dispatch_stale_timeout_seconds`（默认 4 小时）没有心跳时，认为 Worker 卡死并重新入队任务。重新入队是良性的（任务回到 `ready` 状态，不增加失败计数），但当前运行进度会丢失。

### 4.3 结构化交接

`kanban_complete` 的 `summary` 和 `metadata` 字段会被传递给下游任务的 Worker。良好的交接格式可以显著减少下游工作的重复劳动：

```python
kanban_complete(
    summary="完成了用户认证模块的数据库迁移，包含 users 和 sessions 两张表",
    metadata={
        "changed_files": ["migrations/001_users.sql", "migrations/002_sessions.sql"],
        "decisions": ["使用 bcrypt 哈希密码", "JWT 有效期 15 分钟"],
        "next_steps": "等待 API 实现 worker 读取此 metadata"
    }
)
```

**Metadata 约定字段**：
- `changed_files`：变更的文件列表
- `verification`：验证命令
- `dependencies`：依赖的任务 ID 或外部 issue
- `blocked_reason`：阻塞原因（如无则为 null）
- `retry_notes`：之前失败的原因
- `residual_risk`：未测试或仍需人工审查的内容

### 4.4 协议违规检测

Worker 必须在使用 `kanban_complete` 或 `kanban_block` 后正常退出。如果 Worker 进程以状态码 0 退出但任务仍为 `running` 状态，Dispatcher 会认为这是协议违规。

Hermes 会注入最多两次合成提示，当检测到模型即将停止但未调用终态工具时：
```
[SYSTEM] Remember to call kanban_complete or kanban_block before finishing.
```

如果提示耗尽或 Worker 在提示前崩溃，Dispatcher 给予**有界重试**（默认连续 3 次违规后自动阻塞任务），防止模型陷入"写纯文本答案然后退出"的循环。

---

## 五、实战案例：博客自动化工作流

让我们通过一个完整的例子，展示如何用 Kanban Swarm 实现博客的自动化生产流程。

### 5.1 场景描述

假设你想建立一个自动化博客系统：收到主题后，自动分解任务、调研、写作、审核、发布。整个过程无需人工干预，只需最后查看成果。

### 5.2 编排流程

首先，编排代理（orchestrator）接收主题并分解任务：

```python
# 工具调用示意（非可执行脚本）
# 1. 创建父任务（orchestrator 自己）
parent_id = kanban_create(
    title="博客自动化工作流：Kanban Swarm 深度解析",
    assignee="orchestrator"
)

# 2. 创建子任务，形成依赖链
research_task = kanban_create(
    title="调研 Kanban Swarm v0.20.6 文档和源码",
    assignee="researcher",
    parents=[parent_id],
    body="""调研 Hermes Agent v0.20.6 Kanban Swarm 最新功能：
    1. 官方文档更新
    2. 对比 v0.16 的变化（archived 状态、review_dispatch 等）
    3. 输出功能架构图和配置说明"""
)

writing_task = kanban_create(
    title="撰写博客文章",
    assignee="writer",
    parents=[research_task],
    body="""基于 researcher 的输出，撰写《Hermes Agent Kanban Swarm 深度解析》技术博客。
    要求：1500~3000 字，面向技术读者。"""
)

review_task = kanban_create(
    title="审核博客内容",
    assignee="reviewer",
    parents=[writing_task],
    skills=["sdlc-review"]  # 附加审核技能
)

publish_task = kanban_create(
    title="发布到 GitHub",
    assignee="publisher",
    parents=[review_task]
)

# 3. 完成编排任务
kanban_complete(
    summary="已分解为 4 个子任务：researcher → writer → reviewer → publisher",
    metadata={"children": [research_task, writing_task, review_task, publish_task]}
)
```

### 5.3 Worker 执行流程

每个 Worker 的实际执行过程类似这样：

**Researcher Worker**：
```python
def run():
    # 1. 读取任务
    task = kanban_show()
    
    # 2. 执行调研
    docs = web_extract(["https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban"])
    kanban_heartbeat(note="已下载官方文档，正在分析 v0.20.6 变化")
    
    # 3. 提交结果
    kanban_complete(
        summary="已完成 v0.20.6 Kanban Swarm 调研，输出功能架构图和配置说明",
        metadata={
            "doc_url": "https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban",
            "new_features": ["archived", "review_dispatch", "goal_mode"]
        }
    )
```

**Writer Worker**：
```python
def run():
    task = kanban_show()
    
    # 读取 parent handoff
    parent_result = task["parents"]["t_177c95d5"]["result"]
    
    # 撰写文章
    article = write_blog(parent_result)
    
    # 保存到工作空间
    write_file("blog-post.md", article)
    
    kanban_complete(
        summary="已完成《Hermes Agent Kanban Swarm 深度解析》博客文章，约 2500 字",
        metadata={
            "artifact": "blog-post.md",
            "word_count": 2500
        }
    )
```

### 5.4 查看进度

作为用户，你可以随时查看看板状态：

```bash
# 列出所有任务
hermes kanban list

# 查看特定任务详情
hermes kanban show <task_id>

# 实时跟踪事件流
hermes kanban watch

# 查看统计信息
hermes kanban stats

# 打开 Dashboard 可视化界面
hermes dashboard
```

Dashboard 的 Kanban 标签页提供直观的可视化：
- 按状态分列显示任务卡片
- 实时 WebSocket 更新
- 拖拽卡片改变状态
- 点击卡片查看详情、评论、依赖关系

---

## 六、高级特性

### 6.1 Goal Mode（目标循环）

v0.20.6 新增的 `goal_mode` 允许 Worker 在目标循环中运行。每次转完后，辅助 judge 检查 Worker 输出是否符合任务的标题和 body（视为验收标准），如果未完成且预算未耗尽，Worker 继续在同一个会话中运行。

```python
kanban_create(
    title="翻译文档网站到法语",
    assignee="linguist",
    body="验收标准：每页翻译完成，无英文残留，链接完整",
    goal_mode=True,
    goal_max_turns=15  # 可选，默认 20
)
```

适用于开放式、多步骤、或"直到 X 为真才停止"的任务。对于廉价的单次任务，不建议使用——每轮 judge 的开销不划算。

### 6.2 附件系统

Worker 可以接收文件附件（PDF、图片、源文档），无需在 body 中粘贴路径：

```python
# 上传附件
kanban_attach(
    filename="source.pdf",
    content_base64="..."
)

# 查看附件
attachments = kanban_attachments()

# 通过 URL 添加
kanban_attach_url(url="https://example.com/doc.pdf")
```

### 6.3 多看板隔离

如果需要为不同项目使用独立的看板：

```bash
# 创建新看板
hermes kanban boards create blog --name "技术博客"

# 在特定看板上操作
hermes kanban --board blog create "写一篇关于 Rust 的文章" --assignee writer

# 切换当前看板
hermes kanban boards switch blog
```

每个看板的数据库文件位于 `~/.hermes/kanban/boards/<slug>/kanban.db`，实现完全隔离。Worker 只能通过 `HERMES_KANBAN_BOARD` 环境变量看到自己的看板。

### 6.4 Block Recurrence Limit（阻塞循环限制）

v0.20.6 新增的防循环机制：如果任务因相同原因反复阻塞，达到 `BLOCK_RECURRENCE_LIMIT`（默认 2 次）后，不再路由回 `blocked`，而是提升到 `triage` 等待人工决策。这是一个确定性的 DB 保护，不是 LLM 判断。

---

## 七、最佳实践与常见问题

### 7.1 何时使用 Kanban Swarm vs delegate_task？

| 场景 | 推荐方案 |
|------|----------|
| 短期推理任务，结果立即使用 | `delegate_task` |
| 多阶段工作流，需要不同角色参与 | Kanban Swarm |
| 任务可能中断，需要恢复 | Kanban Swarm |
| 需要人工中途介入 | Kanban Swarm |
| 并行处理大量独立任务 | Kanban Swarm（Fleet Farming） |
| 需要审计轨迹 | Kanban Swarm |

### 7.2 最佳实践

**1. 良好的心跳习惯**
```python
# 长操作时定期报告（工具调用示意）
kanban_heartbeat(note="已处理 4/8 个文件")
# ... 执行工作 ...
kanban_heartbeat(note="正在生成报告")
```

**2. 结构化的交接元数据**
```python
kanban_complete(
    summary="完成了用户认证模块的数据库迁移",
    metadata={
        "changed_files": ["migrations/001_users.sql"],
        "decisions": ["使用 bcrypt 哈希密码"],
        "next_steps": "等待 API 实现 worker 读取此 metadata"
    }
)
```

**3. 编排者不执行实现工作**
编排者的职责是分解目标、建立依赖、分配角色。不要让编排者同时执行具体工作。

**4. 提前决定共享决策**
如果两个并行任务需要共享命名约定或 Schema，编排者必须提前决定并写入两个任务的 body 中。

### 7.3 常见问题

**Q: Dispatcher 没有启动任务？**
检查 Gateway 是否正在运行：`hermes gateway status`。如果没有，运行 `hermes gateway start`。

**Q: 任务卡在 blocked 状态？**
查看任务评论区的阻塞原因，解决后运行 `hermes kanban unblock <task_id>`。

**Q: Worker 执行超时（任务级）？**
可以通过 `--max-runtime-seconds` 参数设置单个任务的最大运行时间：
```bash
hermes kanban create "长时间任务" --assignee worker --max-runtime 2h
```
超时后任务会自动回退到 `ready` 状态，增加失败计数。

**Q: 如何防止 Worker 卡死（心跳级）？**
这与任务级超时不同。如果 Worker 进程卡死或崩溃，Dispatcher 通过心跳机制检测：
- Worker 需要定期调用 `kanban_heartbeat`（至少每小时一次）
- 超时阈值由 `kanban.dispatch_stale_timeout_seconds` 控制（默认 4 小时）
- 超过阈值且无心跳时，Dispatcher 回收任务并重新入队（**不增加失败计数**）
- 可通过 `HERMES_KANBAN_STOP_NUDGE=0` 禁用停止提示

**Q: 如何恢复中断的 Worker？**
如果 Worker 崩溃，Dispatcher 会在 TTL 到期后重新入队任务。如果任务被自动阻塞，查看评论区了解原因后手动解除阻塞。

---

## 结语

Kanban Swarm 是 Hermes Agent 最具革命性的功能之一。它将 AI 代理从"一次性对话"提升到了"持久化工作流"的层次，使得复杂的多阶段任务有了可靠的执行框架。

从 v0.16 到 v0.20.6，Kanban Swarm 不断演进：新增了 `archived` 状态、`review_dispatch` 自动审核派发、`goal_mode` 目标循环、Per-task 模型覆盖、附件系统、多看板隔离等特性。这些改进让 Kanban Swarm 更加健壮、灵活、适合生产环境使用。

无论你是想搭建自动化的内容生产流水线，还是管理跨角色的协作项目，Kanban Swarm 都提供了一个强大而优雅的基础设施。理解它的架构和配置，将帮助你更好地利用 Hermes Agent 的强大能力。

---

## 参考资料

- [Hermes Agent 官方文档 - Kanban](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- [Hermes Agent 配置参考](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)
- [CLI 命令参考](https://hermes-agent.nousresearch.com/docs/reference/cli-commands)
- [GitHub 仓库](https://github.com/NousResearch/hermes-agent)
- [Kanban 设计规格 PDF](https://github.com/NousResearch/hermes-agent/blob/main/docs/hermes-kanban-v1-spec.pdf)

---

*本文基于 Hermes Agent v0.20.6 官方文档编写，更多内容请访问 https://hermes-agent.nousresearch.com/docs*
