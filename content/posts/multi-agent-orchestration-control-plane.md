---
title: "多智能体性正在迁移：Agent 框架编排层的控制权外移"
slug: multi-agent-orchestration-control-plane
date: 2026-09-03T17:00:00+08:00
description: "在编码和工作执行类系统中，多智能体控制面如何从框架迁移到应用与平台。"
categories:
  - 架构
tags:
  - 多智能体
  - Agent
  - 编排
  - 架构
ShowToc: true
TocOpen: false
draft: false
---

> 核心论点：在编码和工作执行类系统中，多智能体框架的差异化正从“替用户编排角色”转向“提供可组合的执行原语”；而任务生命周期、身份、租约、审计与恢复等决定生产可用性的控制平面，正在迁移到应用和平台。本文把这种控制权迁移称为“编排层塌缩”。判断一个框架的路线图，看它把力气花在哪一层。

## 一、引子：一个文档与一行代码的矛盾

AgentScope 的部分介绍材料把它描述为角色型多智能体框架，并与 CrewAI 对照。但如果打开本文的案例系统 CrewScope（基于 AgentScope Java 2.0），你会在 import 里看到另一回事：

```java
import io.agentscope.core.ReActAgent;
import io.agentscope.harness.agent.HarnessAgent;
```

`HarnessAgent` 更准确地说是一个工程化执行层（harness）。AgentScope Java 2.0 的官方架构也把它描述为在 ReAct 核心之上叠加 workspace、state store、subagents 和安全能力的运行时。[AgentScope Harness Architecture](https://java.agentscope.io/v2/en/docs/harness/architecture.html) 它的近亲 LangChain `create_agent`，当前官方文档称其为基于 LangGraph 的 graph-based agent runtime：模型节点、工具节点和中间件共同组成一个可追踪的执行图。[LangChain Agents 文档](https://docs.langchain.com/oss/python/langchain/agents)

**模型、工具、循环、结构化输出、检查点。** 把两者的 API 并排放，几乎一一对应：

| 层次 | LangChain `create_agent` | AgentScope `HarnessAgent` |
|---|---|---|
| 推理执行 | tool loop + stop condition | `ReActAgent` |
| 工程管道 | `middleware=[...]` | Compaction / Telemetry Middleware |
| 结构化输出 | `response_format` | structured output |
| 状态与恢复 | LangGraph checkpointer（可选） | `AgentState` / `AgentStateStore` |
| 权限与工作区 | HITL / tool policy / graph interrupts | workspace、sandbox、tool allowlist |

所以，至少在 CrewScope 这个案例里，文档展示的是“角色团队”，生产调用路径首先依赖的是“单 agent 执行层”。这不是谁错了——框架同时长着这两层，只是展示厅和发动机不是同一个房间。需要强调的是：一次 import 不能证明一个系统没有启用 subagent 或 team 能力；下面的判断来自案例的实际调用路径和持久化设计。

这个矛盾引出本文的问题：多智能体层到底发生了什么？它该往哪里去？

## 二、先定义：本文说的“多智能体性”是什么

本文不把“是否创建了多个 Agent 对象”当作判据，而把多智能体性定义为一组可观察的系统属性：

1. **独立策略**：不同成员拥有不同的目标、提示词、工具或权限边界。
2. **独立上下文**：成员之间的上下文是否隔离，以及哪些信息可以跨边界传递。
3. **独立生命周期**：成员是否可以被单独调度、暂停、恢复、重试和追责。
4. **通信协议**：成员之间是直接对话，还是通过任务、事件、队列或数据库交换事实。
5. **责任归属**：谁拥有任务、预算、输出和失败后的恢复责任。

因此，本文后面的 A/B/C/D 不是互斥的技术“种属”，而是多智能体控制权主要落在哪一层的分类。一个系统可以同时包含 B、C、D；“塌缩”也不是多智能体消失，而是编排控制平面从框架默认值迁移到应用或平台。

## 三、去魅：多智能体层的机械本质

先剥皮。把主流框架的多智能体模式翻译成程序员语言，魔法消失：

```
SequentialPipeline        ≈ for 循环：for agent in agents: msg = agent(msg)
Agent Team 的 Task Board  ≈ 一个共享 dict + 消息路由器
Subagent 委托             ≈ 函数调用：result = spawn_agent(task)
角色(role)/背景(backstory) ≈ 提示词字符串
"agent 之间对话"           ≈ 把 A 的输出塞进 B 的输入
```

在本文讨论的许多调用模型里，agent instance 确实只是一次执行的载体；但这不是普遍规律。一个 stateless engine 仍然可以拥有持久的逻辑身份、独立策略和可恢复状态。更准确的说法是：**单智能体循环是执行原语，多智能体性取决于其外部的身份、上下文、生命周期和通信协议。**

那么角色到底有什么用？在编码和工作执行场景里，它至少有三个直接的技术价值，主要属于**上下文工程**，但不止是提示词包装：

1. **角色 = 上下文隔离器。** 研究员读了 20 万 token 的资料，总结员只需要结论。分成两个 agent，总结员根本看不到那 20 万 token。LangChain 的 subagent 文档明确把“clean context window”和避免 context bloat 作为这种模式的收益。[LangChain Subagents](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents) 多智能体很多时候只是“分窗口”的体面说法——和操作系统进程隔离是同一个动机。
2. **角色 = 工具分区器。** 一个 agent 挂 50 个工具，模型选工具的准确率暴跌。切成销售挂 CRM、客服挂工单，各自准确率都高。分角色不是"像团队"，是把决策空间切小。
3. **角色 = 提示词专注化。** "你是资深代码审查专家"优于"你是全能助手顺便帮我审代码"。窄提示词 + 窄工具 + 窄上下文 = 高质量输出。

早期多智能体叙事大量借用了社会学隐喻（“像人类团队一样协作”）。但角色也可能承载权限、责任、评估标准和资源预算。更准确的结论不是“角色只是营销”，而是：**在生产工作执行场景中，角色的主要工程价值首先体现为上下文、工具和责任边界。**

## 四、分类法：按“多智能体性主要归谁”划分

与其按“单/多智能体”给框架贴标签，不如按**多智能体控制权主要归谁**划分架构。CrewAI 已支持脱离 Crew 的 `Agent.kickoff()`；LangChain 也把 multi-agent 组织方式整理成 subagents、handoffs、router、skills 和 custom workflow 等模式。[CrewAI Agent 文档](https://github.com/crewaiInc/crewAI/blob/main/docs/en/concepts/agents.mdx)、[LangChain Multi-agent 文档](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents)

| 类型 | 控制权主要在哪里 | 代表 | 本文观察 |
|---|---|---|---|
| **A. 对话式** | 框架内（agent 互聊，框架路由消息） | CrewAI Crew/Process、AgentScope Pipeline/Agent Team、AutoGen GroupChat | 仍适合以对话为目标的场景 |
| **B. 委托式** | 应用代码（子 agent 是主 agent 的工具） | coding agents（如 Claude Code）、DeepAgents、AgentScope Subagent | 在工作执行场景中很常见 |
| **C. 图编排式** | 应用代码（用户显式声明拓扑，框架保证执行） | LangGraph（节点可以是 agent、函数或子图） | 适合复杂流程与可观测执行 |
| **D. 平台治理式** | 应用/平台控制平面（常见载体是数据库、队列或事件日志） | CrewScope、Devin 类产品 | 面向生产治理的补充层 |

这条轴提供了一个可检验的观察：从 A 到 D，**在本文考察的生产系统里**，框架默认编排逐渐减少，应用或平台对拓扑、生命周期和责任的决定权逐渐增加。但这不是线性演化定律，四类可以叠加；平台也可能在内部使用图、委托或群聊。

## 五、塌缩路径：证据链

下面不是一条已经被证明的行业进化定律，而是一个由公开文档和一个生产案例拼出的工作假设。证据支持的是“控制边界正在变得更可编程”，并不足以单独证明 A→B→C→D 的线性演化。

**AutoGen 的重构**提供了一个有趣的观察点。v0.4 继续维护 GroupChat，同时提供事件驱动的 Core API、RoundRobin/Selector/Swarm 以及 GraphFlow。更准确的结论是：它把“对话式团队”放进了更可组合的运行时，而不是已经放弃对话式编排。[AutoGen Group Chat](https://microsoft.github.io/autogen/0.4.9/user-guide/core-user-guide/design-patterns/group-chat.html)、[AutoGen Teams](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/tutorial/teams.html)

**LangChain 的文档结构**是第二个证据。它把 subagents、handoffs、router、skills 和 custom workflow 作为一等模式，说明框架正在把拓扑选择更多地交给应用开发者；但这不是把编排“降级”为教程，因为 `create_agent` 本身仍是正式的 graph-based runtime。

**CrewAI 的 API 表面**是第三个证据。`Agent.kickoff()` 确实允许脱离 Crew 独立执行，并支持 Pydantic 结构化输出；这说明 CrewAI 同时提供了单 agent 和团队两种入口，但“Crew 层只是提示词糖衣”仍然是本文的解释，而不是 API 事实。[CrewAI Agent 文档](https://github.com/crewaiInc/crewAI/blob/main/docs/en/concepts/agents.mdx)

**委托式的扩散**本身也值得注解。公开的 subagent 设计通常呈现出相似形状：主 agent 把子 agent 当作工具调用，子 agent 在隔离上下文中完成一次任务，再把结果交回主 agent。LangChain 文档明确把这种模式定义为 supervisor + tool，并指出它带来上下文隔离与并行调用；AgentScope 和 DeepAgents 也提供相近的能力。[LangChain Subagents](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents) 这更像一种工程收敛，而不是某个框架“赢了”。

## 六、经济学解释：框架更容易商品化哪些部分

为什么这个边界在工作执行场景里反复出现？

**框架更容易商品化的 = 可复用的部分。** 单循环质量、委托语义、图执行引擎、观测和重试机制，在不同应用之间有较高共性，因此适合沉淀为框架能力。这可能让 `create_agent`、`HarnessAgent`、`kickoff` 的基础形状趋于标准化，但不等于它们已经没有差异化空间。

**应用更可能保留的 = 业务控制面。** 拓扑往往是业务决策（销售流程和代码评审流程未必共用同一套状态机）；耐久和治理也取决于故障模型、合规要求与资源预算。LangGraph 的价值正在于让用户声明图结构，同时提供可复用的执行、持久化和观测原语；这不是“替你编排”与“完全不编排”的二选一。

一句话：**框架更擅长提供“快”的多智能体原语，生产系统还需要“稳”的控制面（崩溃后能恢复、同任务不被两个人抢、出了事能追责）。后者未必不能商品化，但通常必须与应用的业务事实和治理边界一起设计。**

## 七、案例研究：CrewScope，一个把多智能体外置到数据库的系统

CrewScope 是一个 32 万行、MVP 已发布的企业级 AI 工作执行平台（Java 17 / Spring Boot / PostgreSQL）。截至本文写作时，它的 agent 架构提供了一个观察样本，而不是整个行业的统计代表：

**单智能体层：尽量复用框架。** Coding Agent 直接使用 AgentScope 的 HarnessAgent，构建时通过工具权限配置收窄能力面。这一层优先复用框架，是为了减少自研执行循环、上下文管理和安全护栏的成本；“因为它是商品”是本文的经济学解释，不是系统需求本身。

**多智能体层：整体自建，落在 PostgreSQL。** 它没有使用任何框架的多智能体原语——没有 Pipeline、没有 Agent Team、没有 GroupChat。Agent 与 Agent 之间从不直接对话，协作完全由数据库中介：`TaskExecution` 是任务板，`Lease` + fencing token 保证同一任务不被两个 worker 抢占（旧 worker 复活会被栅栏令牌碾压），`StepExecution` 做委托，`RECOVERING` 状态 + 恢复代数做断点续跑，每步产出 `CommandEvidence`/`TestEvidence` 落库存证。

有趣的是，这套结构与 AgentScope 自带的 Agent Team 模式几乎同构——Lead、Worker、共享 Task Board——但每一个对应物都换成了耐久版本：内存任务板换成带租约的表，消息广播换成领域事件 + 事务性发件箱，"agent 身份"从提示词字符串换成数据库里的档案（每成员一条的 AgentProfile，带所有权、版本化配置、审计）。

它还顺带回答了一个反事实问题：**如果框架的多智能体层已经覆盖了系统的验收标准，这个系统为什么仍然自建这部分工程？** 当前验收口径是“100 次故障注入、自动恢复率 ≥99%、重复外部动作为零”；在这个故障模型和样本规模下，会话级内存态编排原语无法直接提供所需保证。要让这个结论可复核，还需要公开故障类型、分母、测试脚本和统计窗口。

## 八、边界与反例

论断要说清适用边界，否则就是口号。

**对话式并未死透。** 在创意发散、谈判模拟、社会行为研究这类**目标本身就是“对话”**的场景里，群聊是需求而非手段。需要被重新评估的，是把对话当作唯一执行机制、指望“聊着聊着活就干完”的场景。

**LangGraph 是一个重要的反例。** 一种解释是：它把多智能体拓扑变成用户显式声明的图，把框架的价值放在执行、持久化和观测保证上。但这只能证明“显式编排也可以被框架很好地支持”，不能单独证明内置编排不会商品化。

**批评有幸存者偏差的风险。** 本文的证据主要来自编码/工作执行类系统；在别的领域（游戏 NPC、仿真、教育陪练）对话式可能依然是最优解。分类法应该比"谁赢谁输"的结论更耐用。

## 九、一个可证伪的预测

立场论文的义务是给出可被推翻的判据。以下预测以 **2026-09-03** 为基线，观察窗口分别到 **2028-09-03** 和 **2030-09-03**：

1. **到 2028-09-03**，主流框架的多智能体文档将继续以“模式”和可组合原语组织；这不要求内置编排停止演进，只要求应用拥有更大的拓扑选择权。可观测指标包括官方文档目录、发布说明和默认 API 的变化。
2. CrewAI 若发布 v2 级重构，我预测新增重点会是委托式原语、显式流程和运行治理，而不只是继续堆叠 Crew 层的角色提示词。
3. **到 2030-09-03**，如果“多智能体编排”形成新的成功框架品类，我的论点需要下调；如果主要创新仍集中在应用图/DSL和平台控制面，则支持本文判断。
4. 生产级多智能体系统的差异化将**越来越多**来自 D 层：身份、责任、审计、恢复、预算；模型质量、工具集成与评估体系仍然是不可忽略的竞争维度。

若到 2030-09-03，主流框架重新以内置对话式编排为主要卖点，并且在生产系统中出现可验证的高采用率，本文关于“控制面外移”的判断就需要被推翻或至少显著收窄。

## 十、结语

回到开头的矛盾：文档说 AgentScope 是角色型多智能体框架，案例代码里用的是 HarnessAgent。两者都对——前者是框架的展示厅，后者是执行底座。而真正值得注意的，是在本文考察的工作执行系统里，**多智能体控制面正在从框架默认值迁出，穿过应用代码，最终沉淀在持久化的平台控制面。**

框架提供让单个模型和执行循环更可靠的能力；平台管的是让许多 agent 的劳动可以被组织。这两个不是竞争关系，而是同一条价值链的上游和下游——就像发动机厂和航空公司。判断你在做的事属于哪一层，比争论“哪个框架更好”重要得多：harness 正趋于标准化，编排更多是你的应用决策，治理可能成为你的产品护城河。

选择投入哪一层，就是选择了到 2030 年要和谁竞争。
