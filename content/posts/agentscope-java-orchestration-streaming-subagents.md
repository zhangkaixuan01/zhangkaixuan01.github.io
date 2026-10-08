---
title: "AgentScope Java 的编排边界：从 create_agent、LangGraph 到流式输出与子 Agent"
slug: agentscope-java-orchestration-streaming-subagents
date: 2026-10-08T11:52:00+08:00
description: "结合 AgentScope Java 2.x 源码，梳理 SDK 与 Service Workflow 的编排边界，以及流式输出、结构化结果、Harness 同步委派和流程级状态分别由谁负责。"
categories:
  - AI Agent
  - 架构
tags:
  - AgentScope
  - LangChain
  - LangGraph
  - 多智能体
  - 流式输出
  - 软件工程
ShowToc: true
TocOpen: false
draft: false
---

研究 AgentScope Java 时，我最初想弄清一个具体问题：它有没有类似 LangGraph 的编排 API，能在代码里定义节点、条件边、并行分支和循环？

顺着这个问题往下看，还需要回答：多个 Agent 的流式输出怎样合并？Harness 已有的子 Agent 委派，能承担多少编排工作？

结合文档与源码，我的判断是：**AgentScope Java SDK 提供 Agent 执行与委派能力，跨 Agent 的业务流程则需要应用选择合适的编排方式。** 仓库中的 Service 平台另有 Workflow 引擎，可以承担一部分流程管理，但它的用法和约束与 LangGraph 不同。

下面先厘清 SDK 与 Service 的边界，再看单 Agent 输出、Harness 委派，以及应用仍需补齐的流程管理。分析基于 `2.0.4-SNAPSHOT`、提交 [`e05d45e0`](https://github.com/agentscope-ai/agentscope-java/tree/e05d45e08a0e980ce75f214f4941c3bffe67cefe)，对应 2026 年 10 月 6 日的代码。AgentScope 的来源链接均固定到该提交。本文未运行完整集成测试，示例用于说明接口与设计。

## 一、先把三个层次分开

讨论“有没有编排”之前，需要先明确编排的对象。

| 层次 | 主要问题 | 典型过程 |
|---|---|---|
| Agent 内部执行 | 模型何时调用工具，何时继续推理或结束 | 模型 → 工具 → 模型 → 结果 |
| 子 Agent 委派 | 一个 Agent 怎样把任务交给其他 Agent | 主 Agent → 专家子 Agent → 返回结果 |
| 业务流程编排 | 多个步骤怎样按依赖、条件和状态执行 | 检索 → 并行分析 → 汇总 → 审批 → 发布 |

Agent 内部可以循环，子 Agent 之间也可以并行；业务流程还需要额外定义依赖、完成条件和恢复规则。

例如，主 Agent 根据提示词决定“先调查，再找专家，最后总结”，与程序定义“只有 A、B 都成功，C 才能启动”，是两种不同的控制方式。前者依赖模型在运行中作决定，后者把依赖关系写进可执行规则。

两种方式可以组合使用。选择时需要明确：哪些步骤允许模型灵活决定，哪些约束必须由程序保证，以及失败后怎样继续。

## 二、AgentScope Java 与 create_agent，大致对应在哪一层

从使用层次看，可以作下面的近似对照：

| 能力或层次 | 可以怎样理解 |
|---|---|
| `ReActAgent` | 提供模型、工具调用、执行循环、中间件和状态管理，职责接近 LangChain 的 `create_agent` |
| `HarnessAgent` | 在基础 Agent 上增加工作区、上下文管理、子 Agent 等执行设施 |
| 应用层控制流 | 组织多个 Agent 或普通业务步骤，决定分支、并行与汇合 |
| Service Workflow | 平台化定义并执行多步骤任务，具有独立的服务与运行模型 |

这里的“接近”是职责层次相近，不是底层实现相同。LangChain 的 `create_agent` 本身构建在 LangGraph 上，使用者可以直接使用 Agent 入口，不必手写节点和边。[LangChain Agent 实现说明](https://github.com/langchain-ai/langchain/blob/master/openwiki/agent-factory.md)

AgentScope Java 的 `ReActAgent` 也已经实现了模型与工具循环。“应用负责编排”指的是更高层的业务流程；应用既可以自己组织简单控制流，也可以借助独立编排引擎。

## 三、当前 SDK 的编排接口，与 StateGraph 有什么差别

LangGraph 的图 API 以状态、节点和边组织执行，支持条件路由、循环和动态分发，并可结合 checkpointer 保存执行检查点。[LangGraph Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)、[持久化说明](https://docs.langchain.com/oss/python/langgraph/persistence)

在本文检查的 AgentScope Java SDK 中，我没有发现与 `StateGraph` 对等的公共接口，即“定义状态 → 添加节点与边 → 编译为图 → 执行”这类统一入口。

查阅旧教程时尤其要注意版本。2.0 迁移指南明确列出了已删除的包 `io.agentscope.core.pipeline.*`，其中包括 `Pipeline`、`Pipelines`、`SequentialPipeline`、`FanoutPipeline` 和 `MsgHub`。文档给出的替代方向是 middleware、子 Agent 和事件流。[2.0 迁移指南][migration]

这些机制可以参与编排，但职责并不相同：middleware 介入 Agent 生命周期，子 Agent 承接委派，事件流传递执行过程。它们不会自动组成一套业务图运行时。

### 简单流程可以通过 Java 与 Reactor 组合

Agent 的 `call()` 返回 `Mono<Msg>`，应用可以据此组合顺序执行、条件判断和并行调用。[CallableAgent 接口][callable]

例如，两个 Agent 已初始化时，可以这样表达顺序关系：

```java
Mono<Msg> result = researcher.call("收集这个项目的相关资料")
    .flatMap(researchResult -> writer.call(researchResult));
```

真实业务中，还要设计输入映射：上游消息能否直接交给下游，哪些字段需要提取，是否需要补充写作要求。并发调用时也要划分好会话与运行标识，避免无意共享状态。

Reactor 能组织异步执行，应用仍需要决定持久化哪些业务状态、怎样恢复失败步骤，以及是否允许重做已经产生外部副作用的操作。

### Agent 状态存储不等于图级检查点

`AgentStateStore` 提供按用户、会话和 key 保存、加载状态的接口，可以支撑会话恢复。[状态存储接口][state-store]

但“恢复一段 Agent 对话”与“恢复整个工作流”不是同一个问题。后者还要知道哪些节点已经完成、条件走了哪条分支、并行任务是否汇合、哪些外部动作已经执行。

因此，接上状态存储，并不会让应用自行组合的任意 Reactor 流程自动具备图级恢复能力。这部分需要执行系统明确建模。

## 四、Service 层有 Workflow，但使用方式和限制不同

SDK 之外，`agentscope-service/aistio` 中已有 Workflow 定义、校验、执行引擎和 REST API，核心编排代码位于 Go 服务中。这是需要部署的平台能力。

它采用 `nodes`、`edges` 描述流程，支持以下节点：

| 节点 | 用途 |
|---|---|
| `agent` / `team` | 执行 Agent 或团队任务 |
| `condition` | 条件控制，配合 CEL 表达式选择路径 |
| `join` | 汇合分支，支持 `all`、`any`、`quorum` |
| `approval` | 等待人工审批 |
| `timer` / `signal` | 等待时间条件或具名外部信号 |
| `subrun` | 调用指定版本的子流程 |

定义中还有输入映射、超时、重试和失败策略，执行引擎也有对应处理。[定义与校验][workflow-spec]、[执行引擎][workflow-engine]

从应用侧，可以通过这些实际注册的 API 管理流程：

```text
POST /api/v1/orchestration-definitions
POST /api/v1/orchestration-definitions/:definitionId/validate
POST /api/v1/orchestration-definitions/:definitionId/publish
POST /api/v1/orchestration-definitions/:definitionId/runs

POST /api/v1/orchestration-runs/:runId/pause
POST /api/v1/orchestration-runs/:runId/resume
POST /api/v1/orchestration-runs/:runId/cancel
POST /api/v1/orchestration-runs/:runId/rerun
POST /api/v1/orchestration-runs/:runId/signals/:name
```

这些接口需要相应服务、配置和授权；Java 应用可以通过 HTTP 调用它们。[路由定义][workflow-routes]

### 三个边界决定它是否适合你的业务

**第一，声明式流程必须是无环图。** 校验代码会拒绝环路，返回 `definition graph must be acyclic`。因此，“生成 → 审查 → 不通过就回到生成”不能直接画成一条回边。节点内部的 Agent 可以循环推理，但那属于节点内部逻辑。

**第二，平台节点有明确的类型。** 它主要组织注册的 Agent、Team、审批与子流程，不是把任意 Java 函数直接注册为节点。LangGraph 的函数式节点抽象与此不同。

**第三，发布与执行有独立生命周期。** 草稿发布后形成不可变 revision，Run 绑定具体版本；暂停阻止新节点调度，已启动的步骤仍可能返回。Rerun 创建新的 Run，也不意味着自动补偿上一次发生的外部操作。

截至本文所用提交，Workflow 文档仍标注为预览、正式版本尚未发布。仓库已有实现和测试，生产适用性仍需通过部署与故障场景验证。[Workflow 使用说明][workflow-doc]

## 五、流式与结构化输出，不需要从头实现

编排边界厘清后，再看输出。单个 Agent 的流式事件与结构化结果，框架已经提供支持；应用主要负责业务数据与展示约定。

这里需要拆成三层：Agent 产生什么结果，整个流程怎样组织这些结果，以及前端怎样展示。

| 输出需求 | 框架已有能力 | 应用需要补充的内容 |
|---|---|---|
| 单 Agent 流式 | `streamEvents()` 返回 `Flux<AgentEvent>` | 选择展示哪些事件 |
| 结构化结果 | 接收 Java 类型或 JSON Schema，提供结果提取接口 | 定义业务结构，校验字段含义 |
| Harness 同步子调用 | 转发子事件并标记来源 | 区分父子内容、决定呈现方式 |
| 自行编排多个 Agent | 每个 Agent 都能输出事件 | 统一流程标识、节点状态与最终结果 |
| 前端协议适配 | AG-UI 扩展与 Spring Boot starter | 接入配置和业务 UI |

### 单 Agent 的流式入口

新代码应使用 `streamEvents()`。旧的 `stream()` 系列在 2.0 中已被标记为弃用。细粒度事件包括文本增量、工具调用、工具结果、Agent 生命周期和确认请求等。[流式实现][react-stream]、[事件类型][agent-event]

```java
Flux<AgentEvent> events = agent.streamEvents("分析这个项目");
```

这是应用内的响应式事件流。宿主或适配层负责将它暴露为前端接口，模型增量与 Agent 生命周期事件已经由框架生成。

流中还有 `AgentResultEvent`，携带本次执行的最终 `Msg`。需要同时消费过程与结果时，可以围绕同一次事件流处理；在包含子 Agent 的流里，还要辨别这个结果属于父还是子。最终结果与结束事件的行为可在 `ReActAgent` 的统一执行路径中核对。[执行与结果事件][react-result]

### 结构化结果入口

下面是接口使用示意，`agent` 已完成初始化：

```java
public record Report(String summary, List<String> risks) {}

Mono<Report> report = agent.call("分析这个项目", Report.class)
    .map(msg -> msg.getStructuredData(Report.class));
```

框架支持 Java 类型和 JSON Schema，并根据模型能力选择相应输出路径。[结构化输出说明][structured]

业务仍需处理失败并校验字段含义。格式符合 schema，不等于风险判断准确；能返回完整对象，也不代表每一段流式增量都是有效 JSON。结构化结果的中间态怎样展示，需要另外设计。

上面两个片段演示不同调用方式。一次业务执行应复用同一次调用的过程与结果，避免为获取两种输出而重复启动任务；直接订阅同一条未共享的执行流多次，也需要核对是否会触发重复执行。

## 六、Harness 的同步子 Agent 委派，有哪些形态

对于由主 Agent 动态分派任务的场景，Harness 已经封装了委派机制。按使用方式，可以归纳为下面三种形态。它们是本文对使用模式的分类，可以相互组合，并非三个互斥的类型枚举。

| 形态 | 过程 | 适合场景 |
|---|---|---|
| 单次同步委派 | 创建子 Agent，等待本次调用返回，再继续 | 调研、审查、提取信息 |
| 并行同步委派 | 同一轮发起多个子调用，等待本批工具调用返回 | 独立来源调查、多角度分析 |
| 多轮同步协作 | 保留实例，继续向它发送消息并等待本轮返回 | 追问、补充要求、修改结果 |

### 单次委派：agent_spawn

主 Agent 通过 `agent_spawn` 创建子 Agent，也可以同时提交任务。`timeout_seconds > 0` 表示同步等待，默认 30 秒，最大 600 秒。等待有时间上限，超时后的处理会在下一节展开。

这里的同步，是主 Agent 等待工具结果后再继续推理。等待期间仍可输出事件，接口也使用 Reactor 的异步返回方式，因此不必将它实现为阻塞请求线程的调用。[委派工具实现][spawn-tool]

### 并行委派：同一轮的多个同步调用

当模型在同一轮发起多个同步 `agent_spawn` 或 `agent_send`，且 Toolkit 启用工具并行时，它们可以并发执行。父 Agent 等待这批工具调用返回后，再进入下一轮推理。返回内容可能是成功结果、错误，也可能是超时转后台的状态，不能直接当成“所有子任务已经成功”。

```text
           ┌→ 子 Agent A ─┐
主 Agent ──┤              ├→ 取得本批工具结果 → 继续推理
           └→ 子 Agent B ─┘
```

所以，同步与并行并不冲突：前者描述父 Agent 是否等待，后者描述多个子调用是否同时推进。默认 Toolkit 支持并行，也可以配置为串行。[中间件使用说明][subagent-middleware]

但这里有两个条件：模型确实生成了这一批调用，子任务也适合并行。框架支持并发执行，并不保证模型每次都按照业务预期拆解任务；多个子 Agent 写同一份文件时，还要处理资源冲突。

### 多轮协作：agent_send

`agent_spawn` 返回 `agent_key`，后续可通过 `agent_send` 向已有实例追加消息，也可以用创建时设置的 `label` 寻址。

默认重新 `spawn` 会创建新的实例和会话。配置 `persistSession(true)` 后，可以在同一父会话、相同子 Agent 和标签等条件下复用上下文。实例复用与跨进程恢复仍是两个问题，后者还依赖状态存储等配置。[子 Agent 会话与调用说明][subagent-doc]

## 七、子 Agent 的配置与等待策略，要分别看

协作形态确定后，还要分别选择执行位置、声明入口、工作区和等待策略。这些配置共同影响隔离程度、响应时间与恢复方式。

| 维度 | 选项 | 决定什么 |
|---|---|---|
| 执行位置 | 本地、远程 | 在当前运行环境执行，还是通过 Agent Protocol 调用远端服务 |
| 声明入口 | 内置、工作区文件、编程式声明 | 从哪里获得子 Agent 定义 |
| 工作区 | `ISOLATED`、`SHARED` | 是否使用独立工作区 |
| 等待方式 | 普通同步、强制同步、后台 | 父 Agent 怎样等待，超时以后如何处理 |

内置 `general-purpose` 用于通用委派；项目专用角色可以写入 `workspace/subagents/<id>.md`，也可以通过 `SubagentDeclaration` 编程式声明。编程式声明可从工作区、内联正文或远程 URL 获得定义，对应的 `workspace(...)`、`inlineAgentsBody(...)`、`url(...)` 不能同时配置。[子 Agent 声明说明][subagent-doc]

共享工作区也不等于共享整段对话。工作区控制文件与资源的使用方式，会话上下文仍有独立的管理规则；默认通用子 Agent 使用共享工作区，仍可用于隔离一项任务的推理上下文。

### 普通同步超时，默认会转后台

这条行为对业务影响很大：

| 等待方式 | 设置 | 等待结束时的行为 |
|---|---|---|
| 普通同步 | `timeout_seconds > 0` | 完成则返回结果；等待超时默认转后台，返回 `timeout_promoted` 与 `task_id` |
| 后台 | `timeout_seconds = 0` | 立即返回任务句柄，子任务继续运行 |
| 强制同步 | 应用在 `RuntimeContext` 启用 `CTX_FORCE_SYNC` | 超时返回 `timeout`、发起中断，不升级成后台任务 |

如果下一步必须使用子任务结果，就不能把“同步工具已经返回”直接当成“子任务已经成功完成”。普通同步返回的可能只是转后台的状态，应用或主 Agent 需要识别它。

应用可通过 `AgentSpawnTool.CTX_FORCE_SYNC` 覆盖模型的后台选择，还可以设置 `CTX_FORCE_SYNC_TIMEOUT_SECONDS` 指定等待时间。这些设置同时适用于 `agent_spawn` 和 `agent_send`。[强制同步实现说明][force-sync]

中断不等于撤销：已经写入的数据或发出的请求，仍需通过工具的幂等机制与业务补偿处理。

### 后台结果怎样收集

后台任务完成后，框架会在父 Agent 下一轮推理前注入结果提醒。它不等于向一个已经结束的 HTTP 请求自动补发完整业务结果，也不能假设父 Agent 会立即开始新一轮推理。

需要主动管理时，可以使用 `task_output` 查看结果、`task_cancel` 取消任务，以及 `wait_async_results` 等待一组结果。

其中，`wait_async_results(task_ids=...)` 等待指定集合，`wait_all=true` 等待调用开始时当前会话未完成任务的快照；等待期间新建的任务不会自动加入这个集合。不指定集合的默认模式只等待任意一条 inbox 消息。集合内任务进入终态后，仍需分别检查成功与失败。[后台任务与等待说明][subagent-doc]

## 八、Harness 怎样把子事件带回父流

父 Agent 使用 `streamEvents()`，并同步调用本地子 Agent 时，子事件会进入父事件流。事件带有 `source` 路径，用于区分来源；父 Agent 自己的事件通常没有子来源标记。远程子事件还可带有 `taskId`、`parentSessionId` 等关联信息。[子 Agent 流式说明][subagent-stream]

流程大致如下：

```text
父 Agent 开始
  → 发起 agent_spawn
  → 子 Agent 开始、文本增量、工具事件、结束（带 source）
  → 子调用结果返回给父 Agent
  → 父 Agent 继续处理
父 Agent 结束
```

框架由此提供了父子事件的统一通道。消费这个通道时，需要注意四个边界。

**第一，source 用于标识来源，不是全局唯一的业务执行编号。** 同一子角色可能被重复调用，业务关联还要结合任务、调用或应用自己的节点与尝试标识。

**第二，后台任务不能直接套用同步转发的展示方式。** 本地后台任务主要通过终态通知让父 Agent 收集结果；远程流式还取决于父调用方式和 `remoteStreaming` 等配置。不能把“支持远程流式”泛化成所有后台任务都持续汇入原来的父流。

**第三，子 Agent 失败未必让父流报错。** 子调用内部错误可以被转换为工具结果交给父 Agent，而不是直接传播成父 `Flux` 的 `onError`。因此，流正常结束不等于所有子任务成功，业务应检查工具结果与最终状态。

**第四，事件转发不决定前端该显示什么。** 中间文本、工具状态和用户最终答案需要区分，尤其不要把不同子 Agent 的文本直接拼成一段回答。

### AG-UI 可以复用哪些工作

`agentscope-extensions-agui` 可以把 AgentScope 事件转换为 AG-UI 事件，也有 Spring Boot starter 提供接入支持。

在本文所用版本中，带 `source` 的子事件默认映射到 `subagent.*` 命名空间下的自定义事件，例如 `subagent.lifecycle`、`subagent.text`、`subagent.tool_call`。这样可以避免子 Agent 的结束事件被误当成父流程结束，或者子文本混入父答案。[AG-UI 适配说明][agui]

前端需要为这些自定义事件安排相应的展示方式。多模态内容则应逐项核对适配范围，该版本的转换器并不支持所有消息类型无损往返。

## 九、应用自己编排时，还要统一流程状态与输出

Harness 覆盖了以父子关系组织任务的场景。如果应用绕过这套委派机制，直接调用多个独立 Agent，就需要自己定义它们共同属于哪次业务执行。以这个流程为例：

```text
              ┌→ 技术分析 A ─┐
资料检索 ─────┤              ├→ 汇总 → 人工审批
              └→ 业务分析 B ─┘
```

A、B、汇总 Agent 都会产生事件。把它们合并成一个 `Flux`，只解决了事件传输问题，还没有回答：

- 同时到达的文本属于哪个节点、哪次尝试？
- 一个 Agent 结束，是该节点完成，还是整个流程完成？
- A 成功、B 失败时，是重试、部分交付，还是终止？
- 哪些内容是过程信息，哪个对象才是最终业务结果？
- 用户断开连接以后，任务是否继续？重连从哪里恢复？

自行编排时，我倾向于在 Agent 事件之外增加应用自己的流程事件层。下面只是建议字段，不是 AgentScope 的内置 schema：

```text
workflowRunId    整次业务执行
nodeId           当前步骤
attempt          本步骤的执行尝试
eventType        节点开始、文本增量、节点失败、流程完成等
payload          原始事件或业务数据
```

这层封装可以保留 Agent 原始事件，同时区分节点与流程的生命周期。若需要断线续传，还要设计事件序号、存储和重放，SSE 传输本身不能替代这些工作。

LangGraph 的流式机制提供了图状态更新、模型输出和子图来源等抽象，能够承接一部分工作；前端展示、业务结果 schema 与对外接口仍然需要应用定义。[LangGraph 流式说明](https://github.com/langchain-ai/docs/blob/main/src/oss/langgraph/streaming.mdx)

因此，评估自建编排的成本时，除了把 Agent 连起来，还要计算状态、事件、取消与恢复的整合工作。

## 十、怎样选择编排方式，估算应用工作量

前面的能力可以归纳为以下选型依据。关键是流程需要哪些约束，以及团队愿意维护哪一层。

| 业务需求 | 可以优先考虑的方式 | 需要承担的主要工作 |
|---|---|---|
| 一个 Agent 完成任务，支持流式或 JSON 输出 | `ReActAgent` 或 `HarnessAgent` | 业务输入、结果校验和前端展示 |
| 主 Agent 动态找专家、并行调查、继续追问 | Harness 子 Agent 委派 | 角色描述、超时策略、结果判断和父子 UI |
| 少量稳定步骤，依赖关系简单 | Java / Reactor 组合 | 输入输出映射、错误处理和流程状态 |
| 多分支、循环、检查点恢复与统一事件语义 | 独立图运行时，AgentScope 作为执行节点 | 两层之间的消息、状态、取消与事件适配 |
| 平台化审批、等待信号、版本化多 Agent 流程 | 评估 Service Workflow | 服务部署、权限、预览版本风险与无环限制 |

使用独立编排层，不一定意味着放弃 AgentScope。可以让 AgentScope 负责节点内部的 Agent 行为，外层负责业务依赖和运行状态。不过，这是一种架构组合，需要自行适配，不能默认存在开箱即用的跨框架接口。

无论采用哪种方式，我都会检查三个阶段的责任：开始时由谁确定输入和执行身份；运行中由谁决定下一步、记录状态和管理副作用；结束时由谁判断任务完成，并向调用方交付最终结果。

如果这些责任都明确，简单组合也能有效工作。如果分支、重试、审批和恢复越来越多，应用就可能逐步写出自己的工作流引擎。此时应该重新评估编排层，而不是继续把所有流程约束堆进提示词。

## 十一、我对这套边界的理解

AgentScope Java 的 Agent 执行能力与 LangChain `create_agent` 处在相近层次；Harness 进一步提供了子 Agent、工作区和事件转发等设施。它们已经承担了很多执行细节，单 Agent 的流式与结构化输出也不需要应用从头实现。

当任务变成跨 Agent 的业务流程，问题就转向另一层：怎样定义依赖、合并事件、判断完成、等待人工，以及从失败处恢复。Harness 委派能覆盖其中一类动态协作；Service Workflow 提供另一种平台化方式；更一般的图运行时则适合明确控制节点和状态流转的需求。

**选型最终要明确：哪些决定交给模型，哪些规则固化为流程，哪一层负责完整任务的生命周期。** 分工清楚后，流式输出、结果汇合和失败恢复各自需要多少工作，也就更容易判断。

[migration]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/docs/v2/zh/docs/change-log.md
[callable]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/agentscope-core/src/main/java/io/agentscope/core/agent/CallableAgent.java
[state-store]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/agentscope-core/src/main/java/io/agentscope/core/state/AgentStateStore.java
[workflow-spec]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/agentscope-service/aistio/internal/orchestration/spec.go
[workflow-engine]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/agentscope-service/aistio/internal/orchestration/engine.go
[workflow-routes]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/agentscope-service/aistio/internal/httpapi/server.go
[workflow-doc]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/docs/v2/zh/service/workflows.md
[react-stream]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/agentscope-core/src/main/java/io/agentscope/core/ReActAgent.java
[react-result]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/agentscope-core/src/main/java/io/agentscope/core/ReActAgent.java#L1048
[agent-event]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/agentscope-core/src/main/java/io/agentscope/core/event/AgentEvent.java
[structured]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/docs/v2/zh/docs/building-blocks/agent.md#结构化输出
[spawn-tool]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/agentscope-harness/src/main/java/io/agentscope/harness/agent/tool/AgentSpawnTool.java
[force-sync]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/agentscope-harness/src/main/java/io/agentscope/harness/agent/tool/AgentSpawnTool.java#L159
[subagent-middleware]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/agentscope-harness/src/main/java/io/agentscope/harness/agent/middleware/SubagentsMiddleware.java
[subagent-doc]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/docs/v2/zh/docs/harness/subagent.md
[subagent-stream]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/docs/v2/zh/docs/harness/subagent.md#子-agent-流式
[agui]: https://github.com/agentscope-ai/agentscope-java/blob/e05d45e08a0e980ce75f214f4941c3bffe67cefe/docs/v2/zh/integration/protocol/agui.md
