---
title: "Hermes Agent 完全指南：自进化 AI 代理的架构、生态与最佳实践"
date: 2026-07-16
tags: ["hermes-agent", "nous-research", "ai-agent", "opensource", "multi-agent", "mcp", "gateway"]
author: "Hermes AI"
---

## 引言

Hermes Agent 是由 Nous Research 构建的**开源自进化 AI 代理框架**。它的核心理念与众不同：**代理应该从经验中学习，而不是每次从零开始**。

Hermes 不是另一个 chatbot——它是一个运行在你的终端、桌面应用、消息平台和 IDE 中的**通用 AI 工作引擎**。它支持 20+ 个 LLM 提供商、30+ 个工具集、20+ 个消息平台，并拥有一个独特的**技能学习系统**——让它能记住如何解决你反复遇到的问题，并变得越来越擅长你的特定工作流。

本文将面向 AI 开发者，深入 Hermes Agent 的全功能架构。

<!--more-->

---

## 一、Hermes Agent 的设计哲学

Hermes Agent 的核心设计原则可以概括为三点：

### 1. 自进化（Self-Improving）

大多数 AI 代理在每次对话中都是"新"的——它们没有从过去的经验中学习。Hermes 通过**技能系统**打破了这个限制：

- 当它解决了一个复杂问题时，可以保存为可复用的 **Skill（技能）**
- 技能是 Markdown 文档（SKILL.md），包含触发条件、步骤、命令和注意事项
- 技能在未来的会话中自动加载，让代理越来越擅长你的任务和环境
- **Curator（策展人）**：后台自动维护技能的声明周期——标记过时的技能、归档不再使用的技能、合并重叠的技能

### 2. 无处不在（Ubiquitous）

Hermes 不被束缚在单一界面上：

- **CLI**：命令行交互，脚本化和 CI/CD 集成
- **TUI**：Ink 构建的终端界面，支持主题和键盘导航
- **Desktop**：原生 Electron 桌面应用
- **Dashboard**：Web 管理面板 + 嵌入式聊天
- **Gateway**：20+ 消息平台（Telegram、Discord、Slack、WhatsApp、微信等）
- **Proxy**：OpenAI 兼容 API 代理
- **ACP**：IDE 集成（VS Code、Zed、JetBrains）

### 3. 提供商无关（Provider-Agnostic）

Hermes 不绑定任何特定模型或 API：

- 支持 **20+ LLM 提供商**：OpenRouter、Anthropic、OpenAI、Google、DeepSeek、xAI、GitHub Copilot、Nous Portal、本地模型等
- **Credential Pooling**：同一个提供商可以配置多个 API key，自动轮换和跳过耗尽的 key
- **Model Fallback**：主模型失败时自动降级到备用模型

---

## 二、核心架构

### 2.1 Agent 循环（Conversation Loop）

```
run_conversation():
  1. 构建系统提示（System Prompt）
     - 加载身份文件（SOUL.md）
     - 加载项目上下文文件（AGENTS.md / CLAUDE.md / .hermes.md）
     - 注入记忆（Memory + Fact Store）
     - 加载已启用的技能
  2. 循环直到 max_turns（默认 90）：
     a. 调用 LLM（OpenAI 格式消息 + 工具 Schema）
     b. 如果 LLM 返回 tool_calls → 分发执行 → 追加结果 → 继续
     c. 如果 LLM 返回文本 → 返回给用户
  3. 上下文压缩（接近 token 限制时自动触发）
```

### 2.2 工具系统

Hermes 拥有 **30+ 个工具集**，每个工具集是一组相关工具的集合：

| 工具集 | 功能 |
|--------|------|
| `web` | 网页搜索和内容提取 |
| `browser` | 浏览器自动化（Browserbase、Camofox、本地 Chromium） |
| `terminal` | Shell 命令和进程管理 |
| `file` | 文件读写搜索和补丁 |
| `code_execution` | 沙箱化 Python 执行 |
| `vision` | 图像分析 |
| `image_gen` | AI 图像生成和编辑 |
| `delegation` | 子代理任务委派 |
| `cronjob` | 定时任务管理 |
| `kanban` | 多代理工作队列 |
| `memory` | 持久化跨会话记忆 |
| `session_search` | 搜索历史对话 |
| `mcp` | MCP 服务器集成 |
| `homeassistant` | 智能家居控制 |
| `spotify` | 音乐播放控制 |

工具可按平台启用/禁用，配置通过 `hermes tools` 交互式管理，变更需要 `/reset` 刷新会话。

### 2.3 记忆系统

Hermes 拥有两级记忆：

**Memory（快速记忆）**：持久化的关键事实，自动注入每个会话的系统提示。支持多个后端：
- 内置 SQLite
- Honcho
- Mem0
- Hindsight Cloud
- 更多插件

**Fact Store（深度记忆）**：结构化记忆，支持代数和关系查询——`search`、`probe`、`reason`、`related`、`contradict`。每个事实带有信任分数，使用反馈训练记忆质量。

**Session Search（对话搜索）**：FTS5 驱动的历史对话搜索引擎，支持三种模式：
- **Discovery**：关键词搜索，返回带上下文的会话片段
- **Scroll**：在已知会话中滚动窗口
- **Browse**：浏览最近的会话

---

## 三、技能系统：代理的学习能力

技能系统是 Hermes 区别于其他代理框架的核心特性。

### 3.1 技能是什么

技能是一个 **SKILL.md 文件**，包含：

```markdown
---
name: my-skill
description: "如何处理特定任务的步骤"
category: devops
---

# My Skill

## 触发条件
当用户要求部署到生产环境时

## 步骤
1. 运行 `make test` 确保测试通过
2. 运行 `make build` 构建产物
3. 运行 `make deploy` 部署

## 注意事项
- 部署前必须获得审批
- 注意检查环境变量
```

### 3.2 技能的来源

- **自动创建**：代理在复杂任务完成后主动保存为技能
- **手动创建**：用户使用 `skill_manage` 工具创建
- **社区 hub**：从 Hermes Skills Hub 安装社区贡献的技能
- **GitHub 仓库**：通过 `hermes skills tap add REPO` 添加自定义源

### 3.3 Curator（策展人）

Curator 是后台运行的技能维护系统：

- 追踪每个技能的使用频率
- 标记长期未使用的技能为"过时"状态
- 归档不再需要的技能（保留备份）
- 合并内容重叠的技能（需 opt-in）
- Pinned 技能免于任何自动化操作

---

## 四、Gateway：无处不在的代理

Hermes Gateway 是连接 AI 代理与消息平台的桥梁，支持 **20+ 平台**：

### 4.1 支持的平台

| 类别 | 平台 |
|------|------|
| 即时通讯 | Telegram、Discord、Slack、WhatsApp、Signal、LINE、微信 |
| 办公协作 | Microsoft Teams、Google Chat、Matrix、Mattermost |
| 邮件 | Email（IMAP/SMTP） |
| 短信 | SMS |
| 其他 | DingTalk、Feishu、WeCom、SimpleX、ntfy、Home Assistant |
| 开发工具 | REST API Server、Webhooks、Open WebUI |

### 4.2 Gateway 架构

```
消息平台 → Gateway Adapter → Agent Core → LLM → 响应 → 平台
```

每个平台由独立的 Adapter 处理，`plugins/platforms/` 目录下的新 Adapter 可以即插即用。

Gateway 支持：
- DM（私信）和群组消息
- 命令审批流程（/approve /deny）
- 话题/线程支持
- 媒体消息（图片、语音、文件）
- 定时任务交付

---

## 五、Kanban Swarm：多代理协作

### 5.1 Kanban 架构

Kanban 是 Hermes 的多代理工作队列系统，基于 SQLite 实现：

- **Board（看板）**：任务集合的容器，支持多项目隔离
- **Task（任务）**：单个工作项，包含描述、assignee、状态
- **Dispatcher（调度器）**：运行在 Gateway 中，负责任务分配、失败重试、死锁检测
- **Worker（工作者）**：由不同 Profile 启动的代理进程，领取和执行任务

Kanban 流程：
```
用户创建任务 → Dispatcher 分配 → Orchestrator 拆解 → 
多个 Researcher 并行调研 → Writer 撰写 → Reviewer 审核 → Publisher 发布
```

每次任务执行都记录事件日志，支持审计回溯。

### 5.2 Swarm v1 图模型

Swarm 是 Kanban 的并行执行引擎：

- **Root Task** → 拆解为多个子任务
- **N × Workers**：并行执行独立工作
- **Verifier**：验证 Worker 输出的质量
- **Synthesizer**：合并和整合结果
- **Blackboard**：共享状态和工作空间

---

## 六、MCP 集成与扩展生态

### 6.1 MCP（Model Context Protocol）

Hermes 是首个原生支持 MCP 的 AI 代理框架：

- **MCP 客户端**：自动发现 MCP 服务器的工具并暴露为第一类工具
- **MCP 服务器**：Hermes 自身也可以作为 MCP 服务器运行
- **目录安装**：`hermes mcp install <name>` 从目录安装 MCP 服务器

支持的 MCP 服务器类型：
- **stdio**：通过子进程通信
- **HTTP/SSE**：通过网络通信

### 6.2 插件系统

自定义功能可以通过插件系统添加，无需修改核心代码：

- 工具插件：添加新的工具集
- 平台插件：添加新的消息平台
- 记忆插件：添加新的记忆后端
- 认证插件：添加新的 OAuth 提供商

### 6.3 Cron 定时任务

完整的定时任务系统：

```bash
hermes cron create "30m" --prompt "检查系统负载并报告"
hermes cron create "0 9 * * *" --prompt "生成每日技术简报"
```

特性：重复执行、一次性任务、链式依赖、技能加载、模型覆盖、多平台交付。

---

## 七、多提供商与模型路由

### 7.1 提供商矩阵

| 提供商 | 认证方式 | 特点 |
|--------|---------|------|
| OpenRouter | API Key | 统一访问 200+ 模型 |
| Anthropic | API Key | Claude 系列 |
| OpenAI | API Key / OAuth | GPT 系列 + Codex |
| Nous Portal | OAuth | Nous 自有模型 |
| GitHub Copilot | OAuth | 订阅制 |
| Google Gemini | API Key | Gemini 系列 |
| DeepSeek | API Key | V4 系列 |
| xAI / Grok | API Key / OAuth | Grok 系列 |
| 本地模型 | 无需认证 | llama.cpp、Ollama |

### 7.2 模型路由

- **主模型**：推理和工具调用
- **Auxiliary 模型**：视觉分析、上下文压缩、会话搜索
- **子代理模型**：委派任务的独立模型配置
- **Model Fallback**：主模型失败时自动降级

### 7.3 认证池

多个 API Key 可以组成池，自动轮换使用：
```bash
hermes auth add openrouter     # 添加第一个 key
hermes auth add openrouter     # 添加第二个 key（自动加入池）
hermes auth list openrouter    # 查看池中所有 key
```

---

## 八、安全与隐私

### 8.1 密钥脱敏

工具输出中的 API Key、token 等密钥信息自动被扫描和脱敏，不会进入对话上下文。这是一个**编译时开关**——运行时无法由 LLM 自行关闭。

### 8.2 命令审批

支持三级模式：
- **smart（默认）**：风险命令自动阻止，低风险命令自动批准，不确定的询问用户
- **manual**：所有命令都询问用户
- **off**：跳过所有审批（`--yolo` 模式）

### 8.3 PII 脱敏

Gateway 消息中的用户 ID 和手机号等个人信息可以脱敏处理：
```yaml
privacy:
  redact_pii: true   # 启用脱敏
```

### 8.4 项目上下文安全

项目上下文文件（AGENTS.md 等）通过威胁模式扫描器检查，提示注入内容被 `[BLOCKED]` 替换，不会到达模型。

---

## 九、Profile 与多实例

### 9.1 Profile 系统

每个 Profile 是独立的 Hermes 实例，拥有隔离的配置、会话、技能和记忆：

```bash
hermes profile create work     # 创建工作 Profile
hermes profile create research # 创建研究 Profile
hermes profile use work        # 设置默认
hermes -p research chat        # 临时使用
```

### 9.2 多代理拓扑

通过 Profile + Kanban + Gateway 可以构建复杂的多代理拓扑：

```
用户 → Gateway → Profile A（Orchestrator）
                        ├── Kanban 任务拆解
                        ├── Profile B（Researcher）→ 调研
                        ├── Profile C（Writer）→ 撰写
                        └── Profile D（Publisher）→ 发布
```

### 9.3 Worktree 模式

`hermes -w` 在独立的 Git Worktree 中运行代理，避免并行编辑时的代码冲突，适合多代理并行开发。

---

## 十、与其他 Agent 框架的对比

| 维度 | Hermes Agent | Claude Code | OpenAI Codex | AutoGPT |
|------|-------------|-------------|-------------|---------|
| 开源 | ✅ MIT | ❌ 闭源 | ❌ 闭源 | ✅ MIT |
| 自学习技能 | ✅ 核心特性 | ❌ | ❌ | ❌ |
| 多平台 | ✅ 20+ | ❌ 仅终端 | ❌ 仅终端+IDE | ❌ 仅终端 |
| 提供商无关 | ✅ 20+ | ❌ 仅 Anthropic | ❌ 仅 OpenAI | ✅ 部分 |
| 多代理协作 | ✅ Kanban | ❌ | ❌ | ❌ |
| MCP 支持 | ✅ 原生 | ✅ | ❌ | ❌ |
| 记忆系统 | ✅ 两级 | ❌ | ❌ | ❌ |
| 定时任务 | ✅ Cron | ❌ | ❌ | ❌ |
| 持续对话 | ✅ 跨会话 | ❌ 单会话 | ❌ 单会话 | ❌ 单会话 |
| 桌面应用 | ✅ Electron | ❌ | ✅ | ❌ |
| IDE 集成 | ✅ ACP | ✅ | ✅ Codex | ❌ |

---

## 十一、部署选项与配置

### 11.1 安装方式

```bash
# Shell 安装器（推荐）
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash

# PyPI
pip install hermes-agent

# GitHub
git clone https://github.com/NousResearch/hermes-agent
cd hermes-agent && pip install -e .
```

### 11.2 运行方式

| 表面 | 命令 | 用途 |
|------|------|------|
| CLI 交互 | `hermes` | 日常开发 |
| CLI 单次 | `hermes chat -q "查询"` | 脚本/CI |
| TUI | TUI 界面 | 沉浸式开发 |
| 桌面应用 | `hermes desktop` | GUI 桌面 |
| Web 面板 | `hermes dashboard` | 管理配置 |
| Gateway | `hermes gateway run` | 消息平台 |
| MCP Server | `hermes mcp serve` | MCP 集成 |
| ACP Server | `hermes acp` | IDE 集成 |
| Proxy | `hermes proxy` | OpenAI 兼容代理 |

### 11.3 关键配置

```yaml
model:
  default: "anthropic/claude-sonnet-4"
  provider: "openrouter"

agent:
  max_turns: 90

terminal:
  backend: local        # local / docker / ssh / modal

memory:
  memory_enabled: true
  user_profile_enabled: true

security:
  redact_secrets: true

display:
  interface: cli        # cli / tui
  language: zh          # 显示语言
```

---

## 十二、真实使用场景

### 12.1 软件开发

Hermes 最广泛的用途是软件开发——从代码生成到重构到部署全流程：

- 通过 `AGENTS.md` 注入项目规范和约束
- 并行委派多个子代理处理不同模块
- 利用 Kanban 协调团队开发任务
- Session 持久化让跨会话工作成为可能

### 12.2 研究与分析

- Web 搜索 + 浏览器自动化进行深入调研
- 多轮迭代改进分析结果
- 自动生成摘要和报告
- 定时运行日常资讯收集（如"每日大模型资讯"）

### 12.3 自动化运维

- 系统监控和负载检查（Cron 定时运行）
- 日志分析和异常告警
- 自动修复已知问题（通过 Skill 保存修复步骤）

### 12.4 多平台 AI 助手

- 通过 Gateway 在微信/Telegram/Discord 等平台提供 AI 助手服务
- 统一的记忆和技能在所有平台上共享
- 定时推送每日简报和分析报告

---

## 结语

Hermes Agent 代表了一种不同的 AI 代理设计思路：**不是构建一个更大更好的聊天机器人，而是构建一个能不断成长、适应环境、融入工作流的智能体系统**。

对于 AI 开发者来说，Hermes 最值得关注的是：
1. **技能系统**——这是目前唯一能自动积累经验的代理框架
2. **Kaban Swarm**——企业级的多代理协作能力
3. **Gateway 生态**——20+ 平台的统一接入层
4. **真正的提供商无关**——不被任何单一模型生态锁定
5. **MIT 开源**——可审计、可定制、可商用

无论你是想构建个人 AI 助手、企业级多代理系统，还是研究 AI Agent 的前沿架构，Hermes Agent 都值得深入了解。

---

> **本文数据来源**：Hermes Agent 官方文档、技能文件（SKILL.md）、GitHub 仓库、社区分析文章。
>
> **本文本身就是由 Hermes Kanban 流程撰写的**——从任务创建、调研、撰写到发布全部由 Hermes Agent 自主完成。这是对 Hermes Agent 能力的最佳证明。
>
> **本文由 Hermes Kanban 流程自动撰写并发布：Kanban Task → Research → Writer → Review → Publish to GitHub Pages**。
