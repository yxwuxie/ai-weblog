---
title: "AI Agent Skills Ecosystems: From Tool Chains to Collaborative Networks"
date: 2026-09-01
tags: [ai-agents, hermes, skills, mcp, claude-code, codex]
author: yxwuxie
---
# AI Agent Skills Ecosystems: From Tool Chains to Collaborative Networks

还记得第一次用AI写代码的场景吗？输入提示，模型吐出代码，你手动复制粘贴。那是"单工具Agent"的时代——每个任务都是孤立的，每次交互都要重新开始。

今天，情况变了。

Agent不再只是工具调用者，它们成为了技能的使用者、创造者和编排者。技能系统让Agent能够在多次会话中积累知识，将复杂工作流分解为可重用的组件，并与多个Agent协同工作。

这就是AI Agent技能生态系统的故事——从简单的工具链到复杂的协作网络。

## 什么是AI Agent技能？

在深入讨论之前，我们需要明确定义。这里所说的"技能"指的是什么？

以Hermes Agent为例，技能是Agent在执行任务时可加载的**程序性知识库**。每个技能是一个标记文件（SKILL.md），包含结构化元数据和指令，描述何时使用、如何使用，以及相关的脚本和模板。

技能与工具的区别至关重要：

- **工具**是Agent可以调用的具体操作（如`terminal`执行命令、`write_file`写入文件）
- **技能**是高级抽象，指导Agent**如何组合工具完成特定类型的工作**

Hermes的技能架构采用**渐进式披露**模式，分为四个层级：

1. **Tier 0：** 类别摘要（名称 + 技能数量）
2. **Tier 1：** 技能列表（名称 + 描述）
3. **Tier 2：** 完整技能内容（SKILL.md正文）
4. **Tier 3：** 关联文件（引用、模板、脚本）

这种设计让Agent在对话开始时只加载元数据，按需深入，避免上下文膨胀问题。

## 技能如何工作：加载、注册与上下文注入

*注：本文分析基于 Hermes Agent v0.20.x 版本。*

技能的生命周期始于文件系统发现。Agent启动时，会扫描指定目录下的所有SKILL.md文件，解析前4000个字符提取元数据，构建技能索引。

```python
# 简化的技能发现逻辑
def _find_all_skills():
    skills = []
    for skill_dir in skills_path.rglob("SKILL.md"):
        if any(skip in skill_dir.parts for skip in ['.git', '.github', '.hub']):
            continue
        metadata = parse_frontmatter(skill_dir.read_text()[:4000])
        skills.append(metadata)
    return deduplicate_by_name(skills)
```

技能的激活条件由元数据控制：

- `requires_tools`：列出的工具不可用时，技能隐藏
- `fallback_for_tools`：列出的工具可用时，技能隐藏（工作区模式）
- `platforms`：限制在特定操作系统加载
- `required_environment_variables`：自动注册为沙箱环境传递

当Agent决定使用某个技能时，技能内容被注入到系统提示中。这包括：
1. 技能的`When to Use`部分——触发条件
2. 技能的`Procedure`部分——执行步骤
3. 技能的`Pitfalls`部分——常见陷阱
4. 关联文件路径（如果需要）

工具注册是另一个关键机制。Hermes的工具采用**自注册模式**——开发者只需在`tools/*.py`中调用`registry.register()`，框架通过AST解析自动发现并注册。

```python
registry.register(
    name="terminal",
    toolset="terminal",
    schema={"command": {"type": "string"}, "timeout": {"type": "integer"}},
    handler=handle_terminal,
    requires_env=["SHELL"],
)
```

这种设计消除了手动维护工具列表的需要，使系统更具可扩展性。

## 从工具链到协作网络：演化故事

早期的AI Agent遵循简单的**顺序工具链**模式：

```
搜索 → 提取 → 总结
```

每个步骤依赖前一步的输出，失败就会中断整个流程。这种模式适合简单任务，但无法处理复杂工作流。

技能系统的出现带来了范式转变：**从线性链到网络化协作**。

### 协作模式的演变

| 模式 | 描述 | 示例 |
|------|------|------|
| **顺序链** | 技能按顺序执行 | 研究→写作→评审 |
| **回退链** | 主要技能不可用时切换 | `web_search` → `duckduckgo-search` |
| **条件激活** | 基于工具可用性加载 | `requires_tools: [web_search]` |
| **平台门控** | 限制特定OS | `platforms: [macos]` |
| **并行分解** | 多Agent同时执行 | 分发给多个worker |

### 真实案例：博客编写自动化工作流

让我们看一个完整的技能协作案例：

```
微信接收主题
  ↓
Kanban Swarm 启动
  ↓
Orchestrator 分解任务
  ├── researcher（调研）
  ├── writer（撰写2000字文章）
  ├── reviewer（语法检查）
  └── publisher（GitHub发布）
```

这个工作流的核心创新在于：

1. **技能组合**：每个角色由专门技能定义
2. **持久状态**：Kanban板提供任务跟踪和依赖管理
3. **并行执行**：子任务可独立运行
4. **质量门禁**：reviewer阶段确保输出质量

技能系统使这种复杂编排成为可能，而无需硬编码工作流——只需要正确的技能定义和协调机制。

## 多技能协作的架构模式

### 1. GitHub工作流

```
github-auth（设置认证）
  → github-repo-management（克隆/分支）
    → github-issues（创建/分类问题）
      → github-code-review（审查PR）
        → github-pr-workflow（合并）
```

每个技能处理一个离散阶段，`hermes-agent`技能提供编排上下文。

### 2. MLOps流水线

```
huggingface-hub（模型发现）
  → llama-cpp（本地推理）
    → weights-and-biases（实验追踪）
      → arxiv（文献回顾）
```

跨类别技能集成：模型发现→本地部署→实验日志→研究基础。

### 3. Kanban Swarm（看板协同）

```
Orchestrator（任务分解）
  → researcher（调查）
    → writer（文档）
      → reviewer（质检）
        → publisher（部署）
```

技能定义工作流，看板板提供持久状态管理。

## Hermes Agent技能生态系统现状

截至2026年9月，Hermes Agent已安装**75个技能**，覆盖**12个类别**：

| 类别 | 数量 | 示例技能 |
|------|------|----------|
| 自主AI Agent | 6 | hermes-agent, claude-code, codex, computer-use |
| 创意 | 17 | ai-comic-production, manim-video, ascii-art |
| 生产力 | 17 | notion, airtable, docx, xlsx, pdf |
| 研究 | 5 | arxiv, blogwatcher, competitor-news-monitor |
| GitHub | 6 | github-auth, github-code-review, github-pr-workflow |
| MLOps | 3 | huggingface-hub, llama-cpp, weights-and-biases |
| 媒体 | 3 | youtube-content, gif-search, songsee |
| 软件开发 | 7 | test-driven-development, systematic-debugging |

**最活跃的类别**：生产力（17）、创意（17）、自主AI Agent（6）、GitHub（6）。

**跨类别集成**在研究工作流中最为常见：研究 + MLOps + 生产力。

*注：工具数量为近似值，实际数量随版本和配置变化。*

## 技能设计最佳实践

基于对Hermes技能生态系统的分析，以下是技能设计的最佳实践：

### 1. 描述要精确

技能描述是系统提示的一部分，每个对话都会加载。遵循硬标准：

- **≤ 60个字符**
- 一句话，以句号结束，无营销词汇
- 说明能力，而非实现

**好：** `追踪指定公司的重大新闻，提供引用摘要。`
**坏：** `用于监控命名竞争对手或公司获取产品发布、定价变更、融资等信息...`（240字符）

### 2. 平台门控要审计

不要复制相邻技能的`platforms`字段。根据技能实际调用的脚本和命令进行审计：

| 技能仅使用... | platforms: |
|--------------|------------|
| Hermes工具 + 标准库Python + 跨平台CLI | `[linux, macos, windows]` |
| bash管道、grep/awk/sed | `[linux, macos]` |
| osascript, defaults, pmset | `[macos]` |
| apt/systemctl//proc | `[linux]` |

### 3. 使用条件激活

`requires_tools`和`fallback_for_tools`防止技能杂乱：

```yaml
---
name: advanced-web-search
description: 高级Web搜索，使用多种引擎获取更全面结果。
requires_tools: [web_search, web_extract]
fallback_for_tools: [web_search]
---
```

### 4. 将规则与概念并列

不要重复解释环境变量——如果技能需要特定环境变量，直接在`Prerequisites`部分说明，并在脚本中使用它们。

### 5. 测试跨平台兼容性

Windows路径处理是常见问题。使用`pathlib.Path`而非硬编码路径，使用`os.sep`或通用分隔符。

## 与Claude Code和OpenAI Codex的对比

技能生态系统不是Hermes独有的。让我们看看主要竞争者如何实现类似功能。

### 工具系统架构

| 维度 | Hermes Agent | Claude Code | OpenAI Codex |
|------|--------------|-------------|--------------|
| **注册方式** | 文件系统发现（rglob SKILL.md） | TypeScript自注册 | 技能包（专有） |
| **扩展性** | MCP服务器、插件、自定义工具 | MCP服务器、Hooks、Agent Teams | 技能、函数调用 |
| **工具数量** | 47+内置 | ~50 | 25+ |
| **协议** | OpenAI兼容 + 自定义 | OpenAI兼容 | OpenAI兼容 |

### 安全模型

| 方面 | Hermes Agent | Claude Code | OpenAI Codex |
|------|--------------|-------------|--------------|
| **审批模式** | 3种（手动、智能、关闭） | 多种（含ML分类器） | 沙箱层级 |
| **逐动作审查** | 是（4个界面渲染） | 是（ML分类器） | 是（沙箱执行） |
| **进程隔离** | 可选（Docker/Modal） | 否（本地执行） | 是（云沙箱） |
| **危险模式** | `DANGEROUS_PATTERNS`正则列表 | `bashSecurity.ts` + `destructiveCommandWarning.ts` | OS级执行 |
| **信任模型** | 模型与执行之间 | 模型到机器边界 | 网关外围 |

### 上下文管理

| 功能 | Hermes Agent | Claude Code | OpenAI Codex |
|------|--------------|-------------|--------------|
| **压缩** | 渐进式（v3） | 4层管道 | Rust-powered截断 |
| **会话恢复** | 完整转录 | 边界标记 | 紧凑边界重链接 |
| **记忆** | 跨会话持久 | 有限 | 有限 |
| **窗口大小** | 依赖提供商（1M+ token） | 1M（beta） | 依赖提供商 |

### 多Agent能力

| 能力 | Hermes Agent | Claude Code | OpenAI Codex |
|------|--------------|-------------|--------------|
| **生成** | `delegate_task` | Agent Teams（beta） | `spawn_agent` |
| **隔离** | 独立会话 | 子Agent `QueryEngine` | 隔离容器 |
| **协调** | Kanban看板 | 通过聊天手动 | `SendMessageTool` |
| **持续时间** | 小时/天（后台） | 分钟 | 小时 |

### 关键差异点

**Hermes的优势：**
- 跨平台（Linux、macOS、Windows、Android/Termux、Nix）
- 多表面部署（CLI、TUI、桌面、Web、Telegram、Discord、WhatsApp等）
- 自改进：Agent可以编写自己的技能
- 跨会话持久记忆
- 开源（MIT许可）

**Claude Code的优势：**
- 最深入的工具生态（MCP有数百个服务器）
- 最复杂的权限系统（多种模式 + ML分类器）
- 最好的上下文管理（4层压缩）
- 最强的多Agent编排（Swarms）

**Codex的优势：**
- OS级沙箱（真正隔离）
- 云原生并行（提交任务，关闭笔记本）
- GitHub原生集成（PR、问题、CI）
- 语音输入支持

## 未来方向：技能生态系统的下一步

### 趋势1：技能市场基础设施

当前的技能分发依赖GitHub仓库。未来的生态系统可能会看到：
- 官方技能市场（类似VS Code扩展市场）
- 版本控制和依赖管理
- 社区评分和评论系统
- 一键安装（`hermes skill install marketplace/skill-name`）

### 趋势2：标准化工作流模板

目前的工作流依赖于手工编排。未来可能出现：
- 预定义工作流模板（"博客发布"、"代码审查"、"数据分析"）
- 可视化工作流编辑器
- 工作流共享和版本控制

### 趋势3：使用分析和反馈循环

当前的技能系统缺乏使用跟踪。未来可能：
- 技能使用统计（安装量、调用频率、成功率）
- 用户反馈回路（"这个技能有帮助吗？"）
- 自动技能更新（基于使用情况的热度）

### 趋势4：增强的安全分类

Claude Code的ML驱动安全分类器是一个有趣的模式。Hermes可以：
- 集成CVE数据库进行依赖扫描
- 为工具调用添加行为预测
- 实施基于风险的动态审批

### 趋势5：跨平台技能可移植性

随着技能数量增长，开发者会遇到平台特定问题。未来的解决方案包括：
- 统一的跨平台抽象层
- 自动平台检测和功能回退
- 技能兼容性测试套件

## 结论

AI Agent技能生态系统代表了从"工具调用者"到"知识使用者和创造者"的范式转变。Hermes Agent通过渐进式披露、自注册工具和Kanban协作系统，实现了灵活且可扩展的技能架构。

当前生态系统的75个技能分布在12个类别中，展示了跨领域集成的潜力。从博客编写工作流到MLOps流水线，技能组合使复杂自动化成为可能。

然而，竞争并不停滞。Claude Code拥有更深入的MCP生态系统和更成熟的安全分类器，Codex提供真正的进程隔离和云原生并行。Hermes的独特优势在于跨平台部署、自改进能力和开源许可。

未来，技能生态系统可能会向市场基础设施、标准化工作流、使用分析和增强安全分类发展。对于开发者和AI工程师来说，理解这些模式不仅有助于使用现有工具，还能指导未来Agent架构的设计。

Agent不再是孤立的工具调用者。它们是技能的消费者、创造者和编排者——而技能生态系统将是这一变革的核心。

---

*这篇文章基于对Hermes Agent技能生态系统的研究，分析了75个已安装技能，比较了与Claude Code和OpenAI Codex的架构差异，并探讨了技能系统的未来发展方向。*

## 参考资料

1. **Hermes Agent Documentation** — https://hermes-agent.nousresearch.com/docs/
2. **Creating Skills Guide** — https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills
3. **Tools Runtime** — https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime
4. **Skill System Pattern Analysis** — https://kenhuangus.substack.com/p/chapter-12-the-skill-system-pattern
5. **Claude Code vs Codex Architecture** — https://gist.github.com/Haseeb-Qureshi/2213cc0487ea71d62572a645d7582518
6. **Dive into Claude Code Paper** — https://arxiv.org/html/2604.14228
