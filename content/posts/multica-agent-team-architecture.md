---
title: "Multica 是什么：把 AI Agent 变成团队成员"
slug: multica-agent-team-architecture
date: 2026-09-04T12:00:00+08:00
description: "从 Agent、Runtime 和 Run 三个概念出发，理解 Multica 如何让人和 AI 编程智能体协同工作。"
categories:
  - AI Agent
  - 架构
tags:
  - Multica
  - Agent
  - 多智能体
  - 自托管
  - 软件工程
ShowToc: true
TocOpen: false
draft: false
---

最近在研究 Multica 时，我发现一个很容易混淆的问题：它到底是一个 AI 编程工具，还是一个项目管理系统？

更准确的答案是：

> **Multica 是一个让人和 AI Agent 一起工作的团队协作与任务调度平台。**

它不提供 Codex、Claude 或 Gemini 模型，也不替你训练智能体。它做的是把这些已经存在的 Agent 组织起来，让它们像团队成员一样接收任务、汇报进度、提出阻塞、提交结果，再由人类完成审核和交付。

## 一、先把三个概念分清楚

Multica 最重要的设计，不是“支持 26 个 Agent CLI”，而是把 Agent、Runtime 和 Run 分成了三个独立概念。

```text
Agent    = 谁来做、按什么规则做
Runtime  = 在哪台电脑、用哪款工具做
Run      = 这一次具体执行发生了什么
```

### Agent：虚拟团队成员

Agent 是一个可以重复使用的身份和配置，例如：

```text
名称：后端工程师
职责：负责 API、数据库和后端测试
模型：Codex
Skills：项目 API 规范、数据库规范
权限：允许修改工作区代码，但敏感操作需要审批
```

以后可以把很多任务都交给这个 Agent。它不是一个一直运行的进程，只有在收到任务、评论 @提及、直接对话或自动化触发时，才会产生一次执行。

### Runtime：Agent 的工位

Runtime 是真正执行工作的电脑和工具组合，例如：

```text
Runtime：张三的开发电脑
工具：Codex CLI
操作系统：Windows
```

也可以是：

```text
Runtime：企业 Linux 执行节点
工具：Claude Code
```

一个 Runtime 可以承载多个 Agent，一个 Agent 也可以长期绑定某个 Runtime。

### Run：一次工作记录

当任务分配给 Agent 后，Multica 会创建一次 Run。Run 会记录：

- 什么时候开始；
- 由哪个 Agent 执行；
- 使用哪个 Runtime；
- 执行了哪些命令；
- 修改了哪些文件；
- 消耗了多少 Token；
- 是否成功、失败或需要重试。

因此，一个任务可以反复执行多次，而不会覆盖历史：

```text
任务：增加订单导出功能
  ├── Run 1：Codex 初次实现
  ├── Run 2：根据 Review 意见修复
  └── Run 3：补充测试并处理边界情况
```

## 二、Multica 的整体架构

Multica 采用“服务端控制，执行节点干活”的架构：

```text
┌────────────────────────────────────┐
│ Multica Server                     │
│ Web/API + PostgreSQL + WebSocket   │
│ 任务、评论、权限、Agent、Run       │
└─────────────────┬──────────────────┘
                  │
         WebSocket / HTTPS
                  │
┌─────────────────▼──────────────────┐
│ Agent Runtime                      │
│ multica daemon                     │
│ Git + Worktree + 项目依赖           │
│ Codex / Claude / Gemini CLI         │
└────────────────────────────────────┘
```

服务端负责：

- 工作区和成员；
- 项目和任务；
- Agent 配置和权限；
- Run 队列和状态；
- 评论、通知和审计；
- 自动化、Webhook 和团队协作。

执行节点负责：

- 连接服务端；
- 发现本机安装的 Agent CLI；
- 领取待执行的 Run；
- 准备本地工作目录；
- 启动 Agent 进程；
- 回传日志、消息、用量和最终结果。

## 三、一次任务是怎样跑起来的？

以“给订单服务增加 CSV 导出”为例：

```text
人类创建任务
        ↓
选择负责人：后端工程师 Agent
        ↓
服务端创建 queued Run
        ↓
绑定的 Runtime 收到唤醒通知
        ↓
Daemon 领取 Run
        ↓
创建隔离工作目录或 Git Worktree
        ↓
启动 Codex/Claude 等 CLI
        ↓
Agent 修改代码并运行测试
        ↓
Daemon 回传执行日志和结果
        ↓
人类 Review Diff 和测试
        ↓
创建 PR 或要求 Agent 继续修改
```

这里有一个很重要的边界：**服务端不会直接启动开发者电脑上的 Agent CLI。** 服务端只是创建任务和发出调度信号，真正的 Agent 进程由 Runtime 所在电脑上的 Daemon 启动。

## 四、为什么不把 Agent 直接放进 Server？

把 Agent 放进云端服务器看起来很集中，但代价很高。

### 1. 源码和凭据边界

Agent 需要访问代码、Git 凭据、模型账号和项目环境变量。如果全部放到公共服务端，服务端一旦被攻破，影响范围会非常大。

让 Agent 在执行节点运行，可以把源码和凭据控制在：

- 开发者电脑；
- 企业内网服务器；
- 专用云主机；
- 隔离的容器或虚拟机。

### 2. 项目环境不同

不同项目可能需要完全不同的环境：

```text
项目 A：Node.js + PostgreSQL + Redis
项目 B：Python + CUDA + 私有依赖
项目 C：Rust + Docker + 内网服务
```

如果统一在云端执行，就需要解决镜像、依赖、网络、GPU、缓存和隔离问题。Multica 把这些环境责任交给 Runtime，更灵活。

### 3. 云端执行成本

Agent 会持续消耗 CPU、内存、磁盘、网络和模型额度。平台还要为每个任务维护：

- 独立工作目录；
- 进程隔离；
- 资源配额；
- 超时和重试；
- 崩溃恢复；
- 垃圾回收。

所以，Agent 放在独立 Runtime 上，既降低 Server 的复杂度，也让企业能够自己控制成本。

## 五、4 个人、4 台电脑时应该怎么部署？

假设团队有 4 个人，每人一台电脑。

### 不建议：把所有人的电脑都当共享服务器

如果任务随意分配到某个人的办公电脑，会带来几个问题：

- Agent 会占用这台电脑的 CPU、内存和磁盘；
- 会消耗这台电脑上的模型账号额度；
- 电脑关机后任务无法执行；
- 运行时可能接触到不应该访问的文件和网络；
- Agent 的构建任务可能影响人类正常开发。

### 推荐：个人 Runtime + 专用执行节点

更合理的架构是：

```text
Multica Server
  ├── Web/API
  ├── PostgreSQL
  └── Redis（可选）

张三电脑：个人 Agent Runtime
李四电脑：个人 Agent Runtime
王五电脑：个人 Agent Runtime
赵六电脑：个人 Agent Runtime

企业服务器：共享 Agent Runtime（推荐）
  ├── multica daemon
  ├── Codex/Claude/Gemini
  ├── Git 仓库缓存
  └── 容器或 VM 隔离
```

任务可以按类型分配：

| 任务 | 执行位置 |
|---|---|
| 张三临时修一个小 Bug | 张三的 Runtime |
| 李四与 Agent 交互式开发页面 | 李四的 Runtime |
| 夜间跑全量测试 | 企业执行节点 |
| 每天生成项目报告 | 企业执行节点 |
| 需要 Docker/GPU 的任务 | 专用云主机 |
| 需要访问内网数据库的任务 | 企业内网节点 |

### 张三到底做什么？

张三不是“被占用的服务器管理员”，而是一个正常的工程师。若他的电脑作为个人 Runtime：

- Agent 在独立工作目录中运行；
- 张三仍然可以使用自己的 IDE 和原始工作目录；
- 张三可以开发其他任务；
- 张三可以查看 Agent 的结果并进行 Review。

但如果任务很重，还是会影响机器性能。因此，长时间任务、自动化任务和高并发任务应放到专用执行节点，不要全部放到个人办公电脑。

另外，Runtime owner、Agent owner 和 Reviewer 不一定是同一个人：

```text
张三：提供执行 Runtime
李四：创建任务
王五：负责代码 Review
```

这三种角色可以分开。

## 六、个人使用时怎么部署？

个人用户通常有两种选择。

### 选择一：Multica Cloud + 本地 Daemon

```text
Multica Cloud
      ↓
本地 multica daemon
      ↓
本地 Agent CLI
```

适合想快速体验的人：服务端协作数据由 Cloud 管理，代码和 Agent 仍在本机执行。

### 选择二：本机 Docker Compose + 本地 Daemon

```bash
git clone https://github.com/multica-ai/multica.git
cd multica
make selfhost
multica setup self-host
```

这时 Web、API 和 PostgreSQL 在本机 Docker 中运行，Daemon 和 Agent CLI 在本机后台运行。

如果只是单人开发，Multica 的完整团队能力可能显得偏重；但当你需要多个长期 Agent、自动化任务、运行记录和失败重试时，它的价值会明显增加。

## 七、Multica 的多智能体不是“让所有 Agent 一起聊天”

Multica 的 Squad 机制也很有特点。

把任务分给一个 Squad 后，不是所有成员同时启动，而是：

```text
父任务
  ↓
Squad leader Agent
  ↓
根据任务内容 @某个成员
  ↓
成员 Agent 分别执行
  ↓
leader 根据结果继续派活或上报
```

例如：

```text
产品交付小队
  ├── leader：负责拆解和路由
  ├── 前端 Agent
  ├── 后端 Agent
  └── 测试 Agent
```

这更像一个“AI 项目负责人 + 多个执行成员”，而不是多个 Agent 无目的地互相对话。

## 八、自动化能做什么？

Autopilot 可以按时间或 Webhook 触发 Agent：

- 每天生成项目进展；
- 定期检查依赖更新；
- CI 失败后自动分析；
- 定时检查安全公告；
- 从企业聊天工具触发任务；
- 生成周报或客户交付报告。

建议：

- 需要人类查看和跟进的工作，使用“创建任务模式”；
- 真正后台化、无需讨论的工作，使用“仅运行模式”；
- 自动化必须有清晰的权限、超时和失败处理规则。

## 九、人类的作用真的会减少吗？

会减少，但减少的是重复协调，不是责任。

Multica 自动处理：

- 任务排队；
- Runtime 通知；
- Agent 启动；
- 进度回传；
- 运行记录；
- 失败重试；
- 状态同步。

人类仍然负责：

- 定义目标；
- 设定验收标准；
- 配置 Agent 边界；
- 处理架构取舍；
- 审查代码和测试；
- 决定是否合并和上线。

因此，人的角色从“亲自操作每一个终端”转变成：

```text
工程师 + 任务负责人 + Reviewer + Agent 管理者
```

尤其不能把下面两件事混为一谈：

```text
Run completed = 一次 Agent 执行结束
Issue done    = 人类确认任务已经完成
```

## 十、采用 Multica 前要注意什么？

### 1. Agent CLI 需要自己安装和登录

Multica 不捆绑模型。执行电脑必须提前安装并登录 Codex、Claude Code、Gemini 等工具。

### 2. Daemon 权限很重要

Daemon 以所属操作系统用户的权限运行。建议使用专用账号、隔离目录、容器或虚拟机，不要直接用拥有大量生产权限的个人账号。

### 3. WebSocket 是关键链路

远程部署时必须正确代理：

- 用户实时 WebSocket；
- Daemon WebSocket；
- `/health` 和 `/readyz` 健康检查。

否则任务可能一直排队，或页面没有实时更新。

### 4. `custom_env` 不等于本地秘密

智能体的自定义环境变量会存储在服务端，并在运行时发送给 Daemon。真正敏感的长期凭据应放进企业的 Secret 管理系统或执行节点本地安全存储。

### 5. 许可证要提前确认

仓库使用 Multica License，在 Apache License 2.0 基础上增加了托管服务、商业嵌入、品牌和归属条件。单一组织内部使用与向第三方提供托管服务不是一回事，企业正式部署前应让法务审查完整 `LICENSE`。

## 结论

Multica 的核心不是“又一个 AI 聊天工具”，而是把 AI 编程 Agent 纳入一套团队工作系统：

```text
任务
  → Agent
  → Runtime
  → Daemon
  → 本地执行
  → Run 记录
  → 人类 Review
  → PR 与交付
```

个人使用时，它可以是一个多 Agent 本地工作台；团队使用时，它可以是一个统一的 Agent 调度和协作平台；企业自建时，它可以把任务控制中心放在内网，把 Agent 执行节点放在开发机、专用服务器或云主机。

最重要的判断标准不是“Agent 是否完全自动化”，而是：

> **哪些工作可以交给 Agent，哪些决策必须由人承担，以及这两者之间的边界是否被系统清楚地记录下来。**

## 参考资料

- [Multica GitHub 仓库](https://github.com/multica-ai/multica)
- [Multica 如何工作](https://multica.ai/docs/how-multica-works)
- [项目架构](https://multica.ai/docs/developers/architecture)
- [智能体](https://multica.ai/docs/agents)
- [守护进程与运行时](https://multica.ai/docs/daemon-runtimes)
- [运行](https://multica.ai/docs/tasks)
- [自托管快速上手](https://multica.ai/docs/self-host-quickstart)
- [Multica CLI 与守护进程指南](https://github.com/multica-ai/multica/blob/main/CLI_AND_DAEMON.md)
- [Multica License](https://github.com/multica-ai/multica/blob/main/LICENSE)
