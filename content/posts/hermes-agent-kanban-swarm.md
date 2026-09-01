---
title: "Hermes Agent Kanban Swarm 功能深度解析"
date: 2026-09-01
tags: [hermes, kanban, swarm, multi-agent, ai-agents]
author: yxwuxie
---
# Hermes Agent Kanban Swarm 功能深度解析

你有没有遇到过这样的场景：需要一个 AI 助手帮你完成一个包含调研、写作、审核、发布的长流程任务？传统的单轮对话显然力不从心——它既不能持久跟踪进度，也无法让多个不同角色的"专家"协作。

Hermes Agent 引入了 **Kanban Swarm** 机制，正是为了解决这个问题。今天我们就来深入解析它的架构、配置和使用方法。

---

## 一、Kanban Swarm 是什么？解决什么问题？

Kanban Swarm 是 Hermes Agent 的多智能体协作看板系统。它的核心设计思想是：**将任务视为持久化的工作队列，而非短暂的进程内调用**。

在 v0.16 之前，Hermes 主要依赖 `delegate_task` 来派生子代理。这种方式适合短途、同步的任务——父代理等待子代理返回结果后继续执行。但当任务涉及多个阶段、需要不同角色参与、或者可能因为意外中断需要恢复时，`delegate_task` 就显得力不从心了。

Kanban Swarm 解决了以下痛点：

- **持久化**：任务状态存储在 SQLite 数据库（`~/.hermes/kanban.db`）中，即使 Hermes 进程重启也不会丢失
- **跨会话协作**：多个不同 profile（如 researcher、writer、reviewer）可以在同一个任务上接力工作
- **人工介入**：任务可以进入 `blocked` 状态等待人类决策，之后手动解除阻塞继续执行
- **可审计**：每个任务的所有操作记录在数据库中，可以随时回溯历史

一句话总结：**`delegate_task` 是函数调用，Kanban Swarm 是有状态的工作队列。**

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

### 2.2 数据流

任务的生命周期数据流如下：

1. **创建**：用户或编排代理通过 `kanban_create` 创建任务，指定 `assignee`（执行人）和 `body`（任务描述）
2. **调度**：Gateway 中的 Dispatcher 定期（默认 60 秒）扫描 `ready` 状态的任务
3. **派生**：Dispatcher 启动指定的 profile 作为独立 OS 进程，设置环境变量 `HERMES_KANBAN_TASK=<task_id>`
4. **执行**：Worker 通过 `kanban_*` 工具集与看板交互，完成实际工作
5. **完成**：Worker 调用 `kanban_complete` 提交结果，任务状态变为 `done`

### 2.3 任务状态机

每个任务有六个核心状态，形成有向图：

```
triage → todo → ready → running → done → archived
                 ↓       ↑
              blocked ←┘
                 ↓
               review → done
```

- **triage**：原始想法，等待分解或规格化
- **todo**：已创建但尚未分配，或等待依赖项完成
- **ready**：已分配，等待 Dispatcher 认领
- **running**：Worker 正在执行中
- **blocked**：需要人工输入或遇到了能力限制
- **review**：提交给人工或自动审核
- **done**：已完成
- **archived**：归档（任务完成后可选择归档，从看板隐藏）

### 2.4 依赖关系

任务之间可以通过 `parents` 建立依赖关系。当一个任务有父任务时，它最初处于 `todo` 状态；只有当所有父任务都完成时，它才会自动晋升为 `ready`。这种设计使得复杂的工作流可以自然地表达为 DAG（有向无环图）。

---

## 三、配置指南

### 3.1 基本配置

在 `config.yaml` 中，Kanban 相关配置位于 `kanban` 节：

```yaml
kanban:
  dispatch_in_gateway: true        # 默认值：内置调度器
  dispatch_interval_seconds: 60    # 默认值：每分钟检查一次
  review_dispatch: true            # 默认值：review 状态自动派发带有 sdlc-review 技能的 reviewer profile
  failure_limit: 2                 # 连续失败多少次后自动阻塞
  dispatch_stale_timeout_seconds: 14400  # 4小时：心跳超时判定为卡死
```

### 3.2 关键环境变量

| 变量 | 说明 |
|------|------|
| `HERMES_KANBAN_TASK` | 当前 worker 负责的任务 ID（由 Dispatcher 自动设置）|
| `HERMES_KANBAN_BOARD` | 当前 worker 所在的看板 slug |
| `HERMES_KANBAN_WORKSPACE` | 当前任务的工作空间路径 |

### 3.3 多看板隔离

如果需要为不同项目使用独立的看板，可以通过 `--board` 参数指定：

```bash
# 创建新看板
hermes kanban boards create blog --name "技术博客"

# 在特定看板上操作
hermes kanban --board blog create "写一篇关于 Rust 的文章" --assignee writer
```

每个看板的数据库文件位于 `~/.hermes/kanban/boards/<slug>/kanban.db`，实现完全隔离。

---

## 四、实战案例：博客自动化工作流

让我们通过一个完整的例子，展示如何用 Kanban Swarm 实现博客的自动化生产流程。

### 4.1 场景描述

假设你想建立一个自动化博客系统：收到主题后，自动分解任务、调研、写作、审核、发布。整个过程无需人工干预，只需最后查看成果。

### 4.2 编排流程

首先，编排代理（orchestrator）接收主题并分解任务：

```python
# 编排代理的代码示意
parent_id = kanban_create(
    title="博客自动化工作流",
    assignee="orchestrator"
)

# 创建子任务，形成依赖链
research_task = kanban_create(
    title="调研主题相关资料",
    assignee="researcher",
    parents=[parent_id]
)

writing_task = kanban_create(
    title="撰写博客文章",
    assignee="writer",
    parents=[research_task],
    body="基于调研结果，撰写2000字技术博客"
)

review_task = kanban_create(
    title="审核博客内容",
    assignee="reviewer",
    parents=[writing_task]
)

publish_task = kanban_create(
    title="发布到 GitHub Pages",
    assignee="publisher",
    parents=[review_task]
)
```

### 4.3 Worker 执行流程

每个 Worker 的实际执行过程类似这样：

```python
# researcher worker 的伪代码
def run():
    # 1. 读取任务
    task = kanban_show()
    
    # 2. 执行调研
    results = web_search(task.body)
    kanban_heartbeat(note="调研进行中，已收集 5 份资料")
    
    # 3. 提交结果
    kanban_complete(
        summary="已完成主题调研，整理了架构文档和源码分析",
        metadata={
            "sources_count": 5,
            "key_findings": ["SQLite持久化", "Dispatcher轮询", "依赖提升"]
        }
    )
```

### 4.4 查看进度

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
```

---

## 五、最佳实践与常见问题

### 5.1 何时使用 Kanban Swarm vs delegate_task？

| 场景 | 推荐方案 |
|------|----------|
| 短期推理任务，结果立即使用 | `delegate_task` |
| 多阶段工作流，需要不同角色参与 | Kanban Swarm |
| 任务可能中断，需要恢复 | Kanban Swarm |
| 需要人工中途介入 | Kanban Swarm |
| 并行处理大量独立任务 | Kanban Swarm（Fleet Farming） |

### 5.2 心跳机制

对于长时间运行的任务，Worker 应该定期调用 `kanban_heartbeat` 报告进度。如果超过 `dispatch_stale_timeout_seconds`（默认 4 小时）没有心跳，Dispatcher 会认为 Worker 卡死，将其重新入队。

```python
# 好的做法：定期报告进度
kanban_heartbeat(note="已处理 4/8 个文件")
# ... 执行工作 ...
kanban_heartbeat(note="正在生成报告")
```

### 5.3 结构化交接

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

### 5.4 常见问题

**Q: Dispatcher 没有启动任务？**
检查 Gateway 是否正在运行：`hermes gateway status`。如果没有，运行 `hermes gateway start`。

**Q: 任务卡在 blocked 状态？**
查看任务评论区的阻塞原因，解决后运行 `hermes kanban unblock <task_id>`。

**Q: Worker 执行超时？**
可以通过 `--max-runtime-seconds` 参数设置单个任务的最大运行时间，超时后任务会自动回退到 ready 状态。

---

## 结语

Kanban Swarm 是 Hermes Agent 最具革命性的功能之一。它将 AI 代理从"一次性对话"提升到了"持久化工作流"的层次，使得复杂的多阶段任务有了可靠的执行框架。

无论你是想搭建自动化的内容生产流水线，还是管理跨角色的协作项目，Kanban Swarm 都提供了一个强大而优雅的基础设施。理解它的架构和配置，将帮助你更好地利用 Hermes Agent 的强大能力。

---

*本文基于 Hermes Agent 官方文档编写，更多细节请访问 https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban*
