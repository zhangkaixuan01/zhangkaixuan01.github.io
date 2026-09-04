---
title: "Vibe Kanban 架构拆解：云端协作，Agent 本地执行"
slug: vibe-kanban-architecture-deployment
date: 2026-09-04T11:00:00+08:00
description: "从源码理解 Vibe Kanban 如何编排 AI 编程 Agent，以及个人使用和中小企业自建 Remote 的正确姿势。"
categories:
  - 架构
  - AI Agent
tags:
  - Vibe Kanban
  - Agent
  - 多智能体
  - 自托管
  - Git Worktree
ShowToc: true
TocOpen: false
draft: false
---

最近研究 Vibe Kanban，最容易产生的误解是：它是不是一个“把 Agent 部署到云端”的产品？

答案是否定的。

**Vibe Kanban 不是大模型，也不是某一个 AI Agent。它更像连接“人、Agent、代码仓库和交付流程”的控制平面。** Codex、Claude Code、Gemini CLI、Cursor Agent 等负责读代码、改代码和跑测试；Vibe Kanban 负责分配任务、创建隔离环境、管理会话、展示日志、处理审批、审查差异并推动代码交付。

本文基于 Vibe Kanban 源码仓库 `main` 分支和官方文档，重点回答四个问题：

1. 它的 Local 与 Remote/Cloud 架构分别是什么？
2. Remote/Cloud 到底如何操作 Agent？
3. 个人和中小企业应该怎么部署？
4. 公司停止商业运营后，这个项目还有没有价值？

## 先说结论：Remote 是控制面，Host 才是执行面

Vibe Kanban 可以拆成四层：

| 层次 | 负责什么 | 典型组件 |
|---|---|---|
| 模型层 | 推理和代码生成 | OpenAI、Anthropic、Google 等模型 |
| Agent 层 | 读代码、改文件、执行命令 | Codex、Claude Code、Gemini CLI、Cursor Agent |
| 编排层 | 任务、会话、权限、日志、审批 | Vibe Kanban Workspace、Session、Executor |
| 交付层 | 隔离、测试、评审和合并 | Git Worktree、Diff、Preview、PR、CI |

Remote/Cloud 主要负责组织、项目、Issue、评论、成员和实时同步；Agent 通常运行在开发者电脑或企业内部的执行 Host 上。Remote Server 容器本身不会自动安装或启动 Codex、Claude Code。

这是一种很实用的分工：

```text
Remote Server：团队协作和统一状态
        │
        ├── PostgreSQL：业务数据
        ├── ElectricSQL：实时同步
        └── Relay（可选）：远程连接

执行 Host：真正运行代码和 Agent
        ├── Vibe Kanban 本地服务
        ├── Git 仓库与 Worktree
        ├── Agent CLI 与登录凭据
        └── 项目运行环境
```

## 一、源码架构：两套后端、一个共享工作流

### 1. Local/Desktop：本地优先的执行环境

本地形态由 Rust 和 React 组成：

- `crates/server`：Axum HTTP API、SSE/WebSocket、终端、Preview 和静态前端入口。
- `crates/local-deployment`：本地部署实现，组合数据库、Git、进程、文件、审批、Relay 和 PTY。
- `crates/executors`：适配不同 Agent CLI，负责 Profile、Variant、命令构造、日志解析和审批。
- `crates/workspace-manager`、`crates/worktree-manager`：管理 Workspace、多仓库和 Git Worktree。
- `crates/services`：Container、Event、Filesystem、Repo、Auth、通知等领域服务。
- `packages/local-web`、`packages/web-core`：本地 Web UI 和共享前端业务逻辑。

本地数据默认放在 SQLite 中。创建 Workspace 时，系统会为任务创建独立的 Git Worktree 和工作分支，Agent 的命令在这个目录中运行，主分支不会被直接修改。

本地服务启动时还会做一些恢复工作：迁移数据库、清理孤儿进程、回填 Git 提交信息、清理过期 Workspace，并预热 Agent 配置缓存。这些细节决定了它更像一个执行运行时，而不是一个简单的看板前端。

### 2. Remote/Cloud：云端协作和实时同步

`crates/remote` 是独立的 Rust 服务，前端是 `packages/remote-web`。它的职责主要是：

- 用户登录、JWT 和 GitHub/Google OAuth；
- Organization、Project、Issue、评论和成员权限；
- Workspace 与 Pull Request 的云端元数据；
- PostgreSQL 持久化；
- 通过 ElectricSQL 将数据实时同步给客户端；
- 可选的附件、邮件、R2、GitHub App 和 Relay 能力。

Remote 的一个重要设计是“REST 写入，Electric Shape 读同步”：

1. 创建、更新和删除通过 REST API 写入 PostgreSQL。
2. 服务返回数据库事务 ID（`txid`）。
3. ElectricSQL 从 PostgreSQL 逻辑复制读取变化，并以 Shape 流的形式提供给客户端。
4. 前端等到对应 `txid` 出现在同步流中，再结束 optimistic state，避免界面短暂显示旧数据。

ElectricSQL 只应该在内网使用，客户端请求要经过 Remote 的授权代理。Shape 的表名和查询范围由服务端定义，不能让客户端任意读取数据库。

## 二、Remote/Cloud 是如何操作 Agent 的？

关键答案是：**Remote 不直接操作 Agent 进程，而是通过执行 Host 间接完成。**

在典型的混合部署中，执行链路如下：

```text
用户在 Cloud 创建 Issue 或 Workspace
        ↓
Remote 保存任务、成员、评论和状态
        ↓
本地 Vibe Kanban 客户端连接 Remote
        ↓
本地 server 创建 Workspace 和 Git Worktree
        ↓
ContainerService 根据 ExecutorConfig 启动 Agent CLI
        ↓
Agent 在 Worktree 中修改代码、跑命令和测试
        ↓
本地 UI 展示日志、审批、Diff 和 Preview
        ↓
Workspace 状态、统计、PR 和 Issue 关联同步回 Remote
```

从源码职责看，真正启动 Agent 的调用链在本地服务中：

```text
server route
  → local-deployment
  → services::ContainerService
  → executors
  → Agent CLI 子进程
```

Remote 侧保存的通常是 Workspace 的云端记录和 `local_workspace_id` 关联，而不是替用户上传一份可执行的代码副本。Agent 使用的是 Host 上的：

- 本地 Git 仓库和 SSH key；
- Agent CLI 登录状态或 API Key；
- Node、Python、Rust、Docker 等项目依赖；
- 本地环境变量和凭据。

### Relay/Tunnel 做什么？

当执行 Host 在内网，用户又希望从浏览器或另一台电脑访问它时，可以启用 Relay/Tunnel：

1. Host 向 Relay 建立出站 WebSocket 控制连接。
2. Remote/Relay 记录 Host 的机器 ID、在线状态和可访问关系。
3. 用户访问 `/api/host/{host_id}/...` 时，请求通过 Relay（或可用时 WebRTC）转发。
4. Relay 把请求送回 Host 的本地 Axum 服务。
5. Host 再执行终端、Preview、Workspace API 或 Agent 控制操作。

Relay 是传输通道，不是推理引擎；Agent 始终运行在 Host 上。

## 三、个人使用：直接用 Local 版

如果只是个人开发，最简单的方式是：

```bash
npx vibe-kanban
```

它会启动本地 Rust 服务和 Web UI，并管理本机的 SQLite、Git Worktree、终端和 Agent CLI。你需要提前安装并登录至少一个 Agent，例如 Codex 或 Claude Code。

个人工作流可以概括为：

1. 导入一个 Git 仓库，配置 `setup script`、`dev server script` 和 `cleanup script`。
2. 创建 Workspace，选择目标分支并描述任务。
3. 选择 Agent 和 Variant，例如 Plan、Approval 或不同模型。
4. Agent 在独立 Worktree 中执行。
5. 在 Logs、Changes、Preview 和 Terminal 中观察结果。
6. 通过对话或行级评论让 Agent 继续修改。
7. 测试通过后创建 PR、Rebase 或合并。

这里最重要的认识是：Vibe Kanban 管理的是 Agent 的工作过程，而不是替你提供 Agent。Agent 没装好、没有登录、API 配额不足时，Workspace 仍可能创建成功，但执行会失败。

## 四、中小企业自建：Remote + 本地 Agent Host

对中小企业来说，比较理想的方案不是把 Agent 全部塞进 Remote Server，而是：

```text
Remote Server
  ├── 组织、项目、Issue、评论、权限
  ├── PostgreSQL + ElectricSQL
  └── Caddy/Nginx + HTTPS

每位开发者或每台执行节点
  ├── npx vibe-kanban
  ├── 本地 Git 仓库
  ├── 一个或多个 Agent CLI
  └── 项目依赖和运行环境
```

本地客户端通过环境变量连接企业 Remote：

```bash
export VK_SHARED_API_BASE=https://kanban.example.com
npx vibe-kanban
```

PowerShell：

```powershell
$env:VK_SHARED_API_BASE = "https://kanban.example.com"
npx vibe-kanban
```

这套架构的优点很明确：

- 源码、Git 凭据和 Agent 凭据可以留在企业控制的机器上；
- 团队共享统一的 Issue、Workspace、评论和 PR 状态；
- 每个任务有独立 Worktree，多个 Agent 可以并行；
- 不同开发者可以使用不同 Agent，但 Review 和交付流程保持一致；
- Remote 只承担协作和同步，计算压力相对可控。

### 适合自建的团队

自建 Remote 比较适合以下条件：

- 研发团队大约 5～50 人；
- 同时使用多个 Agent 或并行处理多个任务；
- 对源代码和模型凭据有较高隐私要求；
- 已经有 Docker、Linux、Git、CI 和基本数据库运维能力；
- 愿意自己处理备份、升级、监控和安全响应。

如果只有一个开发者，Local 版通常更简单。如果企业需要完整的 SSO/SCIM、合规审计、SLA 和厂商支持，也不能把当前社区版本直接当作成熟企业 SaaS。

### 自建的成本不能忽略

自建并不等于“免费 SaaS”。企业需要自行承担：

- PostgreSQL、ElectricSQL、Relay、OAuth 和 HTTPS 运维；
- 数据库备份、恢复演练和版本升级；
- Agent CLI 快速升级带来的兼容性问题；
- 每个 Host 的 Git key、模型登录、环境变量和并发额度；
- Agent 执行任意命令带来的网络和权限风险；
- 社区版本的漏洞修复、依赖更新和发布工作。

因此，自建是否划算，取决于企业已经拥有的基础设施和运维能力。它节省的是 SaaS 许可和数据托管成本，增加的是企业自己的维护责任。

## 五、为什么这么好的 Remote 仍然停止商业运营？

2026 年 4 月 10 日，Louis Knight-Webb 发布公告，宣布关闭 Bloop，并表示 Vibe Kanban 将继续作为开源、社区维护项目。公告给出的直接原因是：每天有大量工程师使用产品，但绝大多数用户是免费用户，团队没有找到满意的商业模式。[官方公告](https://www.vibekanban.com/blog/shutdown)

公告同时说明，远程服务只会在过渡期内继续运行，之后看板问题、评论、项目和组织等 Cloud 能力会被移除；本地 Workspace 则继续可用。这也是今天讨论 Remote 自建价值时必须考虑的生命周期背景。

这不是“技术不行”，而是产品价值和商业模型没有对齐。

### 1. Vibe Kanban 不掌握主要收费点

用户通常已经为 OpenAI、Anthropic、Google、GitHub 或其他代码平台付费。Vibe Kanban 位于这些服务之上的编排层，很难像模型厂商那样按调用量获得收入，也很难让用户长期为另一个云端控制台支付高额费用。

### 2. Remote 的运维成本比界面看起来高

一个看板背后其实是一套多租户 SaaS：数据库、实时同步、认证、邀请邮件、附件、远程连接、备份、可用性、安全和客户支持都需要持续投入。免费用户占比高时，这些固定成本很难由订阅收入覆盖。

### 3. Agent 厂商开始自己做编排

Codex Desktop、Claude Code 等产品正在快速增加会话管理、并行执行和工作区能力。Louis 在社区讨论中也表示，不希望社区路线图变成不断追赶那些拥有更多资源的实验室产品。[社区维护讨论](https://github.com/BloopAI/vibe-kanban/discussions/3424)

所以，Remote 架构仍然可能很好，但对一家资源有限的创业公司来说，继续提供官方托管服务意味着：既要维护复杂基础设施，又要和模型厂商的原生 Agent 产品竞争，还要面对较低的付费意愿。

## 六、社区维护路线图：目前不要把愿望当承诺

截至本文写作日，官方还没有发布一份带版本号、里程碑和时间表的正式社区版路线图。2026 年 6 月 2 日的 GitHub Discussion 更像一次公开征集：讨论项目当前解决什么问题、路线图应该加入什么、谁愿意投入维护。[Discussion #3424](https://github.com/BloopAI/vibe-kanban/discussions/3424)

从源码现状和社区讨论看，比较合理的维护方向是：

- 以 Local-first 为主，保留多 Agent、Worktree、Diff、Preview、MCP 和跨平台能力；
- 优先修复 Agent 卡住、队列、审批、日志和长 Diff 性能问题；
- 改善多仓库、多 Session、上下文 Brief 和子 Agent 可视化；
- 增加容器/VM 隔离和更完整的运行配置；
- 对接 Linear、Jira、GitHub Issues 等外部任务系统，减少与成熟项目管理产品的正面竞争；
- 建立社区发布、依赖升级、Agent CLI 兼容性和安全响应机制。

这些是社区讨论方向，不是原团队的交付承诺。选择它时，应该关注实际提交、Issue 响应和发布节奏，而不是只看原产品时期的宣传页面。

## 七、如何判断是否值得采用？

可以用下面的决策表快速判断：

| 场景 | 建议 |
|---|---|
| 个人开发、单机使用 | 直接使用 Local 版 |
| 个人希望多 Agent 并行 | Local 版 + 多个 Workspace/Session |
| 5～50 人研发团队 | 评估 Remote 自建 + 本地 Agent Host |
| 代码隐私要求高 | Remote 只存协作元数据，Agent 在内网 Host 执行 |
| 没有运维人员 | 不建议直接自建 Remote |
| 强 SLA、SSO、合规审计要求 | 选择仍提供商业支持的产品，或准备二次开发 |
| 关键生产系统 | 先试点、固定版本、备份演练，再逐步扩大 |

如果决定自建，建议先用 2～5 名研发人员和 1～2 个非关键项目试运行 2～4 周，验证：

1. OAuth、组织和项目权限；
2. 本地 Host 连接和 Agent 执行；
3. Git Worktree、Setup/Dev/Cleanup Script；
4. 日志、Preview、Diff、评论和 PR；
5. PostgreSQL 备份恢复；
6. ElectricSQL 同步延迟和网络中断恢复；
7. Agent CLI 升级后的兼容性。

## 结语

Vibe Kanban 最值得保留的，不只是一个 Kanban 页面，而是一套围绕 AI 编程 Agent 的工程化闭环：

```text
任务拆分
  → Agent 编排
  → Worktree 隔离
  → 实时执行
  → 人工审批
  → Diff/Preview Review
  → PR 与合并
```

Remote/Cloud 的关闭说明了一个现实：**好的产品架构不一定自动产生可持续的商业模式。** 但这不意味着架构失去价值。对个人来说，Local 版仍然是一个很好的多 Agent 工作台；对有运维能力的中小企业，Remote + 本地 Agent Host 仍然可以成为一套有隐私边界、可定制、可控成本的内部 Agent 协作平台。

真正需要评估的，不是“它曾经是不是一个商业云产品”，而是：你的团队是否愿意接手它的控制面、执行面和长期维护责任。

## 参考资料

- [Vibe Kanban GitHub 仓库](https://github.com/BloopAI/vibe-kanban)
- [官方 Docker Compose 自托管文档](https://www.vibekanban.com/docs/self-hosting/deploy-docker)
- [官方本地开发文档](https://www.vibekanban.com/docs/self-hosting/local-development)
- [官方停服公告：Goodbye bloop](https://www.vibekanban.com/blog/shutdown)
- [社区维护版本讨论 #3424](https://github.com/BloopAI/vibe-kanban/discussions/3424)
