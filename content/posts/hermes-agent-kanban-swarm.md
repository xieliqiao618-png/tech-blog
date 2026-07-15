---
title: "Hermes Agent v0.16 Kanban Swarm：多 Agent 协作的工程化实践"
date: 2026-07-16
tags: ["hermes-agent", "multi-agent", "kanban", "swarm", "AI-framework", "NousResearch"]
author: "xieliqiao618-png"
description: "深度解析 Hermes Agent v0.16 的 Kanban Swarm 功能——一个基于 SQLite 的持久化多 Agent 协作系统，从架构设计到实战配置的全方位指南。"
---

# Hermes Agent v0.16 Kanban Swarm：多 Agent 协作的工程化实践

## 引言

2026 年 6 月 5 日，Nous Research 发布了 Hermes Agent v0.16.0 "The Surface Release"——一个包含 874 次提交、542 个合并 PR、社区 170 位贡献者参与的重大版本。虽然 Desktop 原生应用和简体中文支持是这次版本最耀眼的名片，但真正值得 AI 开发者深入研究的，是隐藏在 Kanban 系统中的多 Agent 协作能力，它在 v0.15 的 Swarm 基础上进一步成熟，形成了完整的工程化方案。

如果你接触过 AutoGPT 的 Agent 循环、CrewAI 的角色编排、或 LangGraph 的有向图执行，你会发现 Hermes 的 Kanban Swarm 走出了一条不同的路——**它不是函数调用，不是 RPC 调用，而是一个持久的 SQLite-backed 任务队列 + 状态机**。每个 Agent 是独立的 OS 进程，每个交接是一条任何人都能读写的数据行，每个任务的状态可审计、可恢复、可人工介入。

这篇文章将从架构设计、核心概念、Swarm 图模型、实战配置和框架对比五个维度，为你解析这套系统。

---

## 一、Kanban 是什么？它解决了什么问题？

### 1.1 从 delegate_task 到 Kanban

在了解 Kanban 之前，先理解它的前身 `delegate_task`。这是 Hermes 内置的子 Agent 机制——主 Agent 将一个子任务派发给匿名子 Agent，**阻塞等待**其返回结果。它的调用方式是 RPC 风格的：`fork → join`。

这对于"查个文档、写个函数"这种短任务足够用，但当工作流变成这样时，瓶颈出现了：

- 多 Agent 并行调研，然后合并结论
- 跨 session 的长期任务（需要一周时间完成）
- 需要人工审批的流程
- 任务执行过程中 Agent 崩溃了需要恢复

**Kanban 把"远程调用"变成了"消息队列"**。一张对比表就可以看清差异：

| 维度 | `delegate_task` | Kanban |
|------|----------------|--------|
| 形态 | RPC 调用（fork → join） | 持久消息队列 + 状态机 |
| 父进程 | 阻塞等待子进程返回 | 创建后即忘（fire-and-forget） |
| 子 Agent 身份 | 匿名子进程 | 有名字的 Profile + 持久化记忆 |
| 可恢复性 | 无——失败即失败 | Block → unblock → 重跑；崩溃 → 回收 |
| 人工介入 | 不支持 | 任一时间点评论 / unblock |
| 审计追溯 | 上下文压缩后丢失 | SQLite 持久化存储，终身可查 |
| 协作模式 | 层级化（调用者 → 被调用者） | 对等——任何 Profile 可读写任何任务 |

### 1.2 核心架构

Hermes Kanban 的核心是**三件套**：

```
┌─────────────────────────────────────────┐
│              SQLite Board               │
│  ~/.hermes/kanban/boards/<slug>/        │
│  ├── kanban.db        (任务 + 事件)      │
│  ├── workspaces/      (工作目录)         │
│  └── logs/            (执行日志)         │
└─────────────────────────────────────────┘
          ▲         ▲          ▲
          │         │          │
    ┌─────┴──┐ ┌───┴────┐ ┌───┴──────┐
    │Dispatcher│ │ Worker │ │  Human   │
    │(Gateway) │ │(Profile)│ │  (CLI)  │
    └─────────┘ └────────┘ └──────────┘
```

**Board（看板）**：一个隔离的任务队列。每个 Board 有自己的 SQLite 数据库、工作目录和日志目录。不同项目可以用不同的 Board 隔离（如 `blog`、`ops`、`project-x`），互不干扰。

**Task（任务）**：一条包含标题、描述、assignee、状态机的数据行。任务状态流转为：

```
triage → todo → ready → running → done
                          ↓
                       blocked → ready (unblock 后)
```

**Dispatcher（调度器）**：默认运行在 Gateway 进程中，每 60 秒一次心跳检测。它做三件事：
1. **回收**：检查 `running` 状态的任务，如果 worker 的 PID 已不存在（崩溃），重新标记为 `ready`
2. **推进**：检查 `todo` 状态的任务，如果所有父依赖都已完成，推进为 `ready`
3. **派发**：原子性地 claim 一个 `ready` 任务，spawn 对应 Profile 的 Hermes 实例

**Worker（工作者）**：一个独立的 Hermes Agent 进程，运行在指定 Profile 下。它通过 `kanban_show`、`kanban_complete`、`kanban_heartbeat` 等工具与 Board 交互。

**Comment（评论）**：Agent 之间、Agent 与人类之间的通信协议。Worker 启动时读取完整评论线程作为上下文的一部分。

---

## 二、Kanban Swarm v1：图执行模型

v0.15 "The Velocity Release" 引入的重磅能力——`hermes kanban swarm` 命令——让你通过一行 CLI 命令创建一个完整的多 Agent 执行图。这个图模型在 v0.16 中得到了进一步强化。

### 2.1 Swarm 拓扑结构

一个标准的 Swarm v1 图包含五个角色：

```
                ┌─────────────────┐
                │    Blackboard   │
                │  (共享上下文)    │
                └────────┬────────┘
                         │
                ┌────────▼────────┐
                │      Root       │
                │  (分解目标)      │
                └────────┬────────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
   ┌──────▼──────┐ ┌────▼─────┐ ┌──────▼──────┐
   │  Worker 1   │ │ Worker 2 │ │ Worker N... │
   │  (并行调研)  │ │ (并行编码)│ │ (并行写作)  │
   └──────┬──────┘ └────┬─────┘ └──────┬──────┘
          │              │              │
          └──────────────┼──────────────┘
                         │
                ┌────────▼────────┐
                │    Verifier     │
                │ (门控：全部完成 → 验证)│
                └────────┬────────┘
                         │
                ┌────────▼────────┐
                │   Synthesizer   │
                │ (合并 → 输出)   │
                └─────────────────┘
```

**执行流程**：

1. **Root**：接收人类输入的模糊目标（如"重构 auth 模块并添加测试"），使用辅助 LLM 自动分解为具体子任务，写入 Board
2. **N × Workers**：并行执行子任务，每个 Worker 是独立的 Hermes 进程，运行在各自的 git worktree 中以避免冲突
3. **Verifier**：**门控在全部 Worker 完成后被唤醒**，检查每个 Worker 输出的质量和正确性。被检查出问题的 Worker 会被标记为 blocked，等待修复后重新运行
4. **Synthesizer**：Verifier 通过后，Synthesizer 合并所有 Worker 的产出，生成最终交付物
5. **Blackboard**：贯穿始终的共享上下文存储，所有 Agent 都可以读写

### 2.2 CLI 实战用法

```bash
# 最简单的用法——4 个 Worker + 验证 + 合成
hermes kanban swarm --goal "将认证模块重构为 OAuth 2.0，添加单元测试和集成测试" \
  --workers 4 \
  --verifier \
  --synthesizer

# 高级用法——不同 Worker 用不同模型
hermes kanban swarm --goal "编写一篇关于 RAG 系统的深度技术文章" \
  --workers 3 \
  --verifier \
  --synthesizer \
  --worker-model openai/gpt-4o-mini \
  --verifier-model anthropic/claude-sonnet-4 \
  --synthesizer-model deepseek/deepseek-v4-flash

# 指定 Skill 和 Workspace 类型
hermes kanban swarm --goal "修复 CI pipeline 中的 5 个 flaky test" \
  --workers 2 \
  --verifier \
  --synthesizer \
  --workspace worktree \
  --skill github-pr-workflow \
  --skill test-driven-development
```

### 2.3 单任务手动的等价操作

如果你不想用 `swarm` 命令自动生成图，也可以手动创建任务并建立依赖关系：

```bash
# 1. 创建根任务
hermes kanban create "重构 auth 模块" --body "..." --assignee orchestrator

# 2. 创建子任务（自动处于 todo 状态）
hermes kanban create "添加 OAuth 2.0 支持" --assignee developer --parent t_xxxxx
hermes kanban create "添加 JWT refresh 逻辑" --assignee developer --parent t_xxxxx
hermes kanban create "更新单元测试" --assignee developer --parent t_xxxxx

# 3. 子任务完成后，父任务自动 ready → dispatcher 派发给 orchestrator
# 4. orchestrator 验证后 complete
```

---

## 三、v0.16 对 Kanban 的强化

虽然 Swarm 图模型在 v0.15 首次亮相，v0.16 在以下方面做了关键性强化：

### 3.1 Board 多项目隔离

v0.16 正式引入了 Board 系统。一个 Hermes 实例可以有多个 Board：

```bash
hermes kanban boards create blog --name "技术博客工作板" --switch
hermes kanban boards create ops --name "运维看板"
hermes kanban boards create research --name "AI 调研看板"

# 查看所有 Board
hermes kanban boards list

# 切换当前 Board
hermes kanban boards switch blog
```

每个 Board 的隔离是绝对的——不同的 SQLite 数据库、不同的工作目录、不同的日志。Worker spawn 时通过环境变量 `HERMES_KANBAN_BOARD` 锁定，看不到其他 Board 的任务。

### 3.2 事件系统与审计

v0.16 丰富的事件系统让任务全过程可追溯：

| 事件类型 | 触发时机 | 业务含义 |
|---------|---------|---------|
| `created` | 任务创建 | 初始状态 |
| `claimed` | worker 获取任务 | 开始执行 |
| `spawned` | worker 进程启动 | 进程 ID 记录 |
| `heartbeat` | worker 定期心跳 | 长任务存活信号 |
| `completed` | 正常完成 | 成功交付 |
| `crashed` | worker 崩溃 | PID 消失，TTL 未过期 |
| `reclaimed` | 声明超时 | 任务回到 ready，等待重试 |
| `timed_out` | 超时 | max_runtime 超限，SIGTERM → SIGKILL |
| `gave_up` | 多次失败 | 断路器触发，自动 blocked |
| `protocol_violation` | 违反协议 | Worker 退出但未调用 complete/block |

通过 `hermes kanban tail <task_id>` 可以实时跟踪事件流，`hermes kanban watch` 可以全板范围监听。

### 3.3 智能重试与断路器

Dispatcher 在 v0.16 中引入了多重保护：

- **`respawning_guarded`**：当检测到上一次失败是认证/限额错误时，等待冷却窗口后再重试
- **`consecutive_failures`**：连续失败达到 `failure_limit`（默认 2 次）后自动 blocked，防止空转
- **`protocol_violation`**：Worker 退出但未正确标记完成/阻塞的，有独立的违规计数（默认 3 次）
- **`recent_success`**：如果 1 小时内已有成功运行，不会盲目重跑

这些机制让 Kanban 系统能够在生产环境中稳定运行，而不会在单点故障上无限循环。

---

## 四、与其他框架的对比

### 4.1 AutoGPT

AutoGPT 的核心是**单 Agent 循环**——一个 Agent 反复执行"思考 → 行动 → 观察"的循环。它通过 `block` 命令实现了简单的任务队列，但缺乏持久化状态、跨 Agent 协作和人工审批。

| | Hermes Kanban | AutoGPT |
|--|--------------|---------|
| Agent 数量 | 多 Agent（不同 Profile） | 单 Agent |
| 持久化 | SQLite 永久存储 | 内存中 |
| 协作模式 | 对等、层级均可 | 不支持 |
| 人工介入 | 评论 / unblock 任意时刻 | 仅暂停 |
| 图执行 | ✅ Swarm v1 图模型 | ❌ 线性循环 |

### 4.2 CrewAI

CrewAI 是我认为距离 Hermes Kanban **形态最近**的框架。两者都支持角色定义、任务委派和顺序/并行执行。但关键差异在于**进程模型**：

CrewAI 的多 Agent 在**同一个 Python 进程**内运行，共享内存空间。这在小型任务中简单高效，但在以下场景成为瓶颈：

- Agent 崩溃导致整个 Crew 崩溃
- 任务无法保存状态，进程结束即丢失
- 无法与外部系统（CI/CD、Slack bot）集成

而 Hermes Kanban 的**每个 Worker 是独立 OS 进程**，拥有独立的内存、文件系统和 Profile 身份。这意味着：

- Worker A 崩溃了 → Dispatcher 自动回收重试
- Worker B 需要 2 小时执行 → 不受其他 Worker 影响
- 人类可以随时`kanban comment`插入指令

### 4.3 LangGraph / LangChain

LangGraph 提供了有向图执行引擎，可以构建复杂的 Agent 工作流。它比 Hermes Kanban 更灵活——你可以定义任意拓扑，而 Kanban 的 Swarm 是一个固定的"并行 Worker → Verifier → Synthesizer"模式。

但 LangGraph 是**库级别的抽象**，你需要自己处理持久化、错误恢复、进程管理和人工审批。Kanban 把这些都内置了——出箱即用，一个 `hermes kanban swarm` 命令就拿到了一个完整的、带断路器、带审计、可人工介入的多 Agent 系统。

### 4.4 Claude Code / Codex CLI

Anthropic 的 Claude Code 和 OpenAI 的 Codex CLI 是目前最流行的单体 Agent 编码工具。它们专注于"一个强大的 Agent + 文件系统访问"模式，在多 Agent 协作方面是空白。Hermes 的 Kanban 恰好填补了这个生态位——你可以用 Claude Code 作为 Worker Profile 来执行具体编码任务，而 Kanban Board 负责协调编排。

---

## 五、最佳实践与配置建议

### 5.1 何时使用 Swarm？

| 场景 | 推荐模式 | 原因 |
|------|---------|------|
| 单文件代码修改 | 直接对话 | Swarm 太 heavyweight |
| 跨模块重构（3+ 文件） | Swarm + worktree | 并行 Worker 不冲突 |
| 技术调研 + 报告 | Swarm + verifier | Verifier 保证质量 |
| 博客文章撰写 | Orchestrator → Researcher → Writer → Reviewer → Publisher | 经典多角色管线 |
| 运维巡检（每日） | Cron + Kanban | Cron 触发，Kanban 执行，结果持久化 |

### 5.2 Per-Task 模型配置

Kanban Swarm 最强大的能力之一是**每个节点使用不同模型**：

```bash
hermes kanban swarm --goal "..." \
  --workers 4 \
  --worker-model openai/gpt-4o-mini \    # 便宜模型做执行
  --verifier-model anthropic/claude-sonnet-4 \  # 强模型做验证
  --synthesizer-model deepseek/deepseek-v4-flash  # 快速模型做合成
```

这是成本优化的关键：将 80% 的 token 消耗放在便宜的 Worker 模型上，只在关键的 Verifier 门控处使用最强（最贵）的模型。

### 5.3 Tenant 多租户隔离

如果你运营多个业务线或客户，Tenant 机制提供了 Board 之内的软隔离：

```bash
hermes kanban create "处理客户 A 的数据迁移" \
  --assignee data-ops \
  --tenant customer-a

hermes kanban create "客户 B 的周报生成" \
  --assignee writer \
  --tenant customer-b
```

Tenant 通过 workspace 路径和 memory key 前缀实现隔离，同一个 Profile 可以服务于多个 Tenant。

### 5.4 Workspace 类型选择

| 类型 | 语法 | 生命周期 | 适用场景 |
|------|------|---------|---------|
| scratch | 默认 | 任务完成即删除 | 调研、一次性分析 |
| dir:/path | `--workspace dir:/obsidian/vault` | 保留 | Obsidian 笔记、共享数据目录 |
| worktree | `--workspace worktree` | 保留 | 编码任务（git 自动管理） |

对于编码任务，worktree 模式尤其强大——每个 Worker 在自己的 git worktree 中工作，互不干扰，完成后 PR 自动提交。

---

## 六、结语

Hermes Agent 的 Kanban Swarm 代表了多 Agent 协作从**原型验证走向工程化**的一个重要方向。它不是试图做一个"通用 Agent 框架"，而是选择了一个具体的、工程化的切入点——持久化的 SQLite 任务队列 + 独立 OS 进程的 Worker + 自动化的 Dispatcher。

这个设计哲学的代价是**不支持跨主机**（SQLite 是单机数据库），但换来的收益是**工程可靠性的质变**：持久化状态、崩溃恢复、审计追溯、人工介入，这些都是生产系统不可或缺的能力，而许多更花哨的 Agent 框架在这些方面几乎是空白。

如果你正在寻找一个"真正能在生产环境中跑"的多 Agent 协作方案，Hermes Kanban Swarm 值得一试。

---

*本文撰写过程中使用的工具链：Hermes Agent Kanban Swarm 自动拆解 → Researcher 调研官方文档和社区文章 → Writer 撰写 → Reviewer 审核 → Publisher 发布至 GitHub Pages。*

## 参考链接

- [Hermes Agent 官方仓库](https://github.com/NousResearch/hermes-agent)
- [Kanban 文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- [Hermes Agent v0.16.0 Release Notes](https://github.com/NousResearch/hermes-agent/releases/tag/v2026.6.5)
- [Hermes Agent v0.15.0 Release Notes](https://github.com/NousResearch/hermes-agent/releases/tag/v2026.5.28)
