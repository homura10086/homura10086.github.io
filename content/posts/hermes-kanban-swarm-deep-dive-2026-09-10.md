---
title: "Hermes Agent v0.16 Kanban Swarm：重新定义 AI Agent 的协作范式"
date: 2026-09-10
tags: [hermes-agent, kanban, swarm, AI-agent, multi-agent, workflow]
author: homura10086
---

> 深度解析 Hermes Agent v0.16 核心新架构——从单 agent 到多 agent 流水线，再到自组织的 swarm 能力

---

## 导语

AI Agent 技术经过一段时间的静态演进，形态逐渐稳定的单模型流式 assistant，到了如今多 agent 并行协作的复系统，开发者们逐渐意识到：**复杂任务不能靠一个 agent 那样粗放 past 过去，而需要设计有序的协作网络**。

Hermes Agent v0.16 推出的 Kanban Swarm 功能，正是规范了这一痛点。它将借鉴开发流程中的 Kanban 看板管理思维，接入 AI agent 集合的任务调度和协作过程中，带领 agent 集合自组织、 自流水线能力。本文将深入解剖 Kanban Swarm 的架构设计、核心算法、实现细节及其在 AI 开发者日常工作流中的应用。

---

## 1. 从单 Agent 到 Swarm

### 1.1 单 Agent 的局限

早期限 AI agent 的雏形式是一个 LLM 实例 + 一套工具调用权限的组合。对于简单查询或独立任务，这种模式非常有用；但一旦多个 agent 并行协作的复系统，开发者们就会发现：**任务分工不单靠一个 agent 那么简单，要需要设计有序的协作网络**。

具体来说，单 agent 面临以下局限：

- **任务复杂度无法伸缩**：当一个任务涉及调研、写作、审核、发布等多个环节时，单一 agent 很难同时胜任所有角色，质量和效率都会下降。
- **状态管理困难**：多个任务并发时，如何追踪每个任务的进度、依赖关系、完成状态，单 agent 缺乏系统的任务管理能力。
- **专业性分工缺失**：不同环节需要不同专长（例如研究员侧重信息搜集和结构化输出，写作侧重表达和文采，审核侧重准确性和规范），单一模型很难同时兼顾。

### 1.2 Swarm 的必要性

面对上述问题，自然的解决思路是引入多个 agent 协作，但这又带来了新的挑战：**如何编排分布式 agent 群体**。这正是 Hermes Agent v0.16 的 Kanban Swarm 功能要解决的问题。

---

## 2. Kanban Swarm 架构解密

### 2.1 核心思想：Kanban + Swarm

Kanban Swarm 的名字本身就揭示了其设计 philosophy：它结合了**Kanban 看板管理**的可视化、流程化管理思想，和 **Swarm 智能体群**的自组织、分布式协作能力。

- **Kanban**: 源于丰田生产系统的看板管理方法，强调可视化工作流、限制在制品（WIP）、拉动式生产。在 AI agent 协作中，它体现为将任务分解为卡片（cards），在看板上移动，每个卡片代表一个具体任务，状态包括待办、进行中、完成等。
- **Swarm**: 借鉴自然界中群体智能的概念，多个简单的个体通过局部交互产生全局的复杂行为。在 Hermes 中，swarm 指多个 AI agent 通过任务分工和协作，形成一个有机的整体，协同完成复杂任务。

### 2.2 架构组件

Kanban Swarm 系统主要由以下组件构成：

#### (1) Board（看板）

看板是任务的顶层容器，代表一个项目或工作流。每个 board 有自己的 slug、name、描述，包含多个任务卡片。board 的设计体现了项目级别的组织：例如`blog`板用于博客创作流水线，`research`板用于研究项目等。

#### (2) Task Cards（任务卡片）

任务卡片是 Kanban Swarm 的基本工作单元。每个卡片包含：
- **id**: 唯一标识（如 `t_xxx`）
- **title**: 任务标题
- **body**: 任务描述和具体要求
- **assignee**: 分配的 profile 名称
- **status**: 当前状态（ready、running、done、blocked 等）
- **priority**: 优先级
- **created_by**: 创建者
- **created_at / started_at / completed_at**: 时间戳
- **workspace_kind**: 工作空间类型（scratch、worktree、dir 等）
- **skills**: 强制加载的技能列表
- **model_override**: 任务级模型覆盖
- **goal_mode / goal_max_turns**: 目标循环模式相关配置
- **block_kind / block_recurrences**: 阻塞管理

#### (3) Dispatcher（调度器）

Dispatcher 是 Kanban Swarm 的大脑，负责监控和管理任务的生命周期。它运行在网关内部，周期性地执行以下操作：
- **Reclaim stale tasks**: 回收长时间未完成或卡住的任务
- **Promote ready tasks**: 将准备好的任务提升为可执行状态
- **Spawn workers**: 根据任务配置，指派相应的 profile 来执行任务
- **Failure handling**: 处理任务失败，记录连续失败次数，触发熔断机制

#### (4) Worker Profiles（工作者 Profiles）

每个任务可以分配给特定的 profile，profile 是带有特定角色、模型、技能和工具集配置的 agent 实例。Hermes 支持多种预定义 profile，例如：
- **researcher**: 负责调研和信息搜集
- **writer**: 负责写作和内容生成  
- **reviewer**: 负责审核和验证
- **publisher**: 负责发布和部署
- **orchestrator**: 负责任务分解和流程编排

#### (5) Task Links（任务依赖）

通过 `task_links` 表，任务之间可以建立父子依赖关系。父任务完成后，子任务才能开始；或者子任务完成是父任务完成的前提。这种依赖管理是构建复杂工作流的关键。

#### (6) 事件系统和通知

系统通过 `task_events`、`task_runs` 等表记录任务执行的详细轨迹，并支持通过 `kanban_notify_subs` 向不同平台（如微信）订阅任务状态变更通知，实现实时跟踪。

### 2.3 工作流范式

Kanban Swarm 支持多种工作流范式，适配不同复杂度的任务：

#### 串行工作流

最基本的形式，任务按顺序执行。例如博客创作流水线：
```
orchestrator ──拆解任务──▶ researcher ──研究结论──▶ writer ──2000 字草稿──▶ reviewer ──审核通过──▶ publisher ──推送 GitHub──▶ 完成
```

全部串行，每个环节的输出是下一个环节的输入，适合流程清楚、步骤明确的任务。

#### 并行工作流（Swarm）

当任务可以分解为多个独立子任务时，可以并行执行。例如：
- 一个研究任务可以分解为多个平行研究员，同时从不同角度或不同来源搜集信息
- 一个代码审查任务可以分解为多个审查员，分别检查不同方面

并行工作流通过 `swarm` 子命令创建，指定 worker、verifier、synthesizer，形成"**多 worker 并行 → verifier 验证 → synthesizer 综合**"的拓扑。

#### 目标循环模式（Goal Loop）

对于开放式、难以一次完成的任务，Kanban Swarm 支持 goal 模式。在这种模式下，任务执行不是一次性完成，而是进入一个循环：worker 执行 → judge 评估是否达成目标 → 如果未达成，继续执行 → 直到 judge 认可完成或达到最大轮数。

这种模式特别适合复杂的研究、创作或问题解决任务，可以避免因任务过于复杂而一次失败的问题。

### 2.4 核心算法和机制

#### 任务分配和 claiming

任务分配通过原子操作 `claim` 实现，确保同一任务不会被多个 worker 同时执行。claim 操作会设置 `claim_lock` 和 `claim_expires`，worker 需要在有效期内定期发送 heartbeat，否则任务会被回收重新分配。

#### 连续失败熔断

每个任务有 `consecutive_failures` 计数器，每当任务执行失败、超时或崩溃时递增，成功完成时重置。当失败次数超过阈值（由 `max_retries` 或全局 `kanban.failure_limit` 配置），熔断器触发，任务被阻塞或路由到 triage，防止无限重试浪费资源。

#### 阻塞管理

任务可以被显式阻塞（`block_task`），阻塞原因会被记录（如依赖未完成、需要人工干预等）。阻塞的任务不会被 dispatcher 调度执行，而是等待人工解除阻塞或依赖解决。系统还记录 `block_recurrences`，防止同一任务在解除阻塞后又被反复阻塞形成死循环。

#### 工作空间管理

任务执行可以选择不同的工作空间模式：
- **scratch**: 临时工作空间，任务完成后自动清理
- **worktree**: git worktree 模式，为任务创建独立的 git 分支工作空间，任务完成后可以保留
- **dir:<path>**: 使用指定的现有目录作为工作空间

这使得任务执行具有隔离性和可追溯性，特别适合需要文件操作或代码修改的任务。

---

## 3. 实战：从零构建一个博客创作 Swarm

理论讲完，下面通过一个具体例子展示 Kanban Swarm 如何工作。

### 3.1 场景设定

我们要完成一个博客文章的创作和发布，流程包括：
1. **调研**：研究主题，收集信息，形成结构化结论
2. **写作**：根据研究结论撰写 2000 字左右的技术文章
3. **审核**：检查语法错误和技术准确性
4. **发布**：将文章发布到 GitHub Pages

### 3.2 Board 和卡片创建

首先创建一个 blog board（如果还没有）：

```bash
hermes kanban boards create blog --name "技术博客工作板"
```

然后在 blog board 上创建四张任务卡片：

```bash
# 研究任务
hermes kanban create research --body "调研: Hermes Agent v0.16 Kanban Swarm 架构/原理/用法/场景，输出结构化研究结论"

# 写作任务
hermes kanban create write --body "写作: 面向 AI 开发者，2000 字左右的 Hermes Agent v0.16 Kanban Swarm 技术深度文章 (Markdown)"

# 审核任务
hermes kanban create review --body "审核: 语法+技术准确性，给出通过/不通过结论"

# 发布任务
hermes kanban create publish --body "发布: 推送至 homura10086/homura10086.github.io (GitHub Pages)"
```

每张卡片创建后，状态为 `ready`，等待被分配和执行。

### 3.3 任务分配和执行

#### 研究环节

将研究任务分配给 researcher profile：

```bash
hermes kanban assign research --assignee researcher
```

researcher profile 执行任务，进行信息搜集和结构化研究结论输出。完成后，卡片状态变为 `done`。

#### 写作环节

研究完成后，将写作任务分配给 writer profile，并将研究结果作为输入：

```bash
hermes kanban assign write --assignee writer
```

writer 根据研究结论撰写文章草稿，完成后状态变为 `done`。

#### 审核环节

将审核任务分配给 reviewer profile：

```bash
hermes kanban assign review --assignee reviewer
```

reviewer 检查文章的语法、技术准确性、逻辑完整性、格式排版等，给出通过/不通过结论。如果不通过，可以返回 writer 修改（最多 2 轮）。

#### 发布环节

审核通过后，将发布任务分配给 publisher profile：

```bash
hermes kanban assign publish --assignee publisher
```

publisher 将文章发布到 GitHub Pages，推送到 `homura10086/homura10086.github.io` 仓库的 `main` 分支。

### 3.4 状态流转可视化

在整个过程中，四张卡片在看板上流转：
- research: ready → running → done
- write: ready → running → done（依赖 research 完成）
- review: ready → running → done（依赖 write 完成）
- publish: ready → running → done（依赖 review 完成）

这种可视化的状态流转让团队成员或开发者可以清楚地知道每个环节的进度和状态。

---

## 4. 进阶特性

### 4.1 任务分解（Decompose）

对于复杂的初始任务（比如一个模糊的想法或需求），Kanban Swarm 提供了 `decompose` 功能。通过辅助 LLM（`auxiliary.kanban_decomposer`），可以将一个 triage 状态的任务自动分解为多个具体的子任务，并路由到不同的 specialist profile。

例如，一个"写一篇关于 AI Agent 的文章"的模糊任务，可以被分解为：
- research: 调研 AI Agent 的现状、技术架构、应用场景
- write-intro: 撰写文章引言和背景介绍
- write-body: 撰写文章主体内容
- write-conclusion: 撰写结论和展望
- review: 审核全文

然后这些子任务可以并行或串行执行，具体取决于任务之间的依赖关系。

### 4.2 目标循环（Goal Mode）

对于开放式任务，goal 模式特别有用。例如：

```bash
hermes kanban create "research-topic" --body "深入研究 Hermes Agent v0.16 的 Kanban Swarm 功能，形成全面深量的研究报告" --goal --goal-max-turns 30
```

在 goal 模式下，worker 会不断迭代，直到 judge 认为研究报告已经足够全面和深入，或者达到 30 轮的上限。这比一次性执行要可靠得多，可以避免因任务复杂度高而导致的质量问题。

### 4.3 技能强加载（Skill Loading）

每个任务可以强制加载特定的技能（`--skill` 参数），这使得任务执行具有更强的专业性。例如：
- 发布任务可以加载 `github-api-publishing` 技能，确保 publisher 掌握正确的 GitHub API 使用方法
- 研究任务可以加载特定的研究技能，指导研究方法和信息源选择

### 4.4 模型覆盖（Model Override）

不同的任务可能适合不同的模型。例如：
- 研究任务可能更适合使用推理能力强的模型
- 写作任务可能更适合使用表达能力好的模型
- 审核任务可能更适合使用细心和准确的模型

通过 `model_override`，可以为每个任务指定不同的模型，而不需要创建多个配置。

---

## 5. 应用场景

### 5.1 内容创作流水线

这是 Kanban Swarm 最典型的应用场景。从选题、调研、写作、审核到发布，整个过程可以由不同的 agent 协作完成，每个环节都由专门的 profile 负责，保证质量和效率。

### 5.2 软件开发工作流

Kanban Swarm 也可以用于软件开发流程：
- **需求分析**: orchestrator 分解需求为具体任务
- **设计**: researcher 调研技术方案，writer 撰写设计文档
- **实现**: 多个开发 agent 并行实现不同模块
- **测试**: reviewer 审核代码和测试用例
- **部署**: publisher 部署到生产环境

### 5.3 研究项目

对于复杂的研究项目，Kanban Swarm 可以：
- 将研究问题分解为多个子问题
- 分配不同的 researcher 从不同角度研究
- 通过 verifier 验证研究结果的可靠性
- 通过 synthesizer 综合形成最终研究报告

### 5.4 客户支持和工单处理

将客户问题分解为：
- **分类和路由**: 根据问题类型，路由到不同的处理 agent
- **并行调查**: 多个 agent 同时调查问题的不同方面
- **综合回答**: synthesizer 综合各个 agent 的发现，形成完整的回答
- **跟进**: 记录处理过程，便于后续跟踪

---

## 6. 最佳实践

### 6.1 明确任务边界

在创建任务卡片时，要明确任务的边界和可交付成果。好的任务卡片应该包含：
- 具体的目标描述
- 输入和输出约定
- 完成标准
- 失败时的处理方式

例如，不要写"研究一下这个主题"，而是写"研究这个主题，输出包含背景、关键概念、写作角度建议、信息来源的结构化 Markdown 报告"。

### 6.2 设计合理的依赖关系

任务之间的依赖关系要清楚明了，避免循环依赖和不必要的等待。可以使用 `task_links` 表来建立和管理依赖，确保流程的顺畅。

### 6.3 合理选择工作空间

根据任务的性质选择合适的工作空间模式：
- 简单的信息处理任务：使用 `scratch` 模式，任务完成后自动清理
- 需要生成文件或代码的任务：使用 `worktree` 模式，保留工作成果
- 需要使用特定目录的任务：使用 `dir:<path>` 模式

### 6.4 监控和管理

利用 Kanban Swarm 提供的监控工具：
- `hermes kanban list` 查看任务列表和状态
- `hermes kanban show <task-id>` 查看任务详情
- `hermes kanban tail <task-id>` 实时跟踪任务事件
- `hermes kanban stats` 查看统计信息
- `hermes kanban watch` 实时监控任务状态变化

### 6.5 容错和恢复

设计任务流时要考虑容错：
- 关键任务设置合理的 `max_retries`
- 使用 goal 模式处理复杂任务，避免一次性失败
- 对于审核不通过的情况，有明确的返回修改流程
- 定期检查任务的连续失败情况，及时处理卡住的任务

---

## 7. 局限和注意事项

### 7.1 执行环境的稳定性

目前 Hermes Agent 的 cron 定时任务执行环境存在一些不稳定性，复杂任务（如 collect/filter/brief）可能无法在定时任务中正常执行。建议对于关键流程，采用手动触发的方式，或者在执行环境稳定后再自动化。

### 7.2 Profile 的配置质量

Kanban Swarm 的效果很大程度上取决于 profile 的配置质量。每个 profile 的 SOUL.md、工具集设置、模型选择都会影响任务执行的效果。需要花时间去优化每个 profile 的配置。

### 7.3 任务粒度的把控

任务分解的粒度要适中：
- 太粗：任务过于复杂，单个 agent 难以完成，质量难以保证
- 太细：任务过于碎片化，增加管理成本，任务之间的协调困难

需要根据具体场景找到平衡点。

### 7.4 调试和问题定位

当任务执行出现问题时，需要善于使用调试工具：
- 查看任务日志：`hermes kanban log <task-id>`
- 查看任务运行历史：`hermes kanban runs <task-id>`
- 查看任务上下文：`hermes kanban context <task-id>`
- 查看任务事件：`hermes kanban tail <task-id>`

---

## 8. 结语

Hermes Agent v0.16 的 Kanban Swarm 功能标志着 AI agent 协作方式的一个重要进步。它将成熟的 Kanban 管理方法论与新兴的 multi-agent 系统结合起来，为复杂任务的编排提供了一种可视化、可管理、可扩展的解决方案。

对于 AI 开发者而言，Kanban Swarm 不仅仅是一个工具，更是一种思维方式的转变：从"如何让单个 agent 更聪明"转向"如何设计一组 agent 的协作流程"，从"追求单次回答的完美"转向"构建可靠的任务流水线"。

随着 AI agent 技术的不断发展，相信 Kanban Swarm 这样的协作框架会变得越来越重要，它将帮助开发者更好地利用多个 AI agent 的力量，完成单个 agent 无法胜任的复杂任务，推动 AI 应用向更高的层次发展。

---

*注：本文基于 Hermes Agent v0.16 的实际功能和使用经验撰写，部分细节可能随版本更新而变化。建议参考官方文档获取最新信息。*
