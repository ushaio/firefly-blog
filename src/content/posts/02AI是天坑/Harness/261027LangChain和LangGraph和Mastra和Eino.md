---
title: 261027LangChain和LangGraph
published: 2026-10-27
description: 261027LangChain和LangGraph
tags:
  - AI
  - Agent
  - Harness
  - LangChain
  - LangGraph
  - Mastra
category: Agent & Harness
draft: false
---
这俩名字挺像的，有啥区别呢？

![](assets/Pasted%20image%2020261027210735.png)

## 一句话区分

- **LangChain**：把「Prompt $\to$ 模型 $\to$ 解析器 $\to$ 工具」串成**有向无环的流水线**（LCEL 的 `|`），解决“组件怎么组装、模型怎么换”。
- **LangGraph**：把 Agent 建成**有环的状态机**（State / Nodes / Edges），解决“流程怎么循环、状态怎么持久、人怎么插手”。
- **Mastra**：TypeScript 生态里的「LangGraph + LangSmith + 脚手架」三合一，解决“在 Node/前端工程里怎么开箱即用地跑起带状态的 Agent”。

## LangChain vs LangGraph：不是兄弟，是分层

| 维度 | LangChain | LangGraph |
| :--- | :--- | :--- |
| 抽象 | Chain（链）、Runnable 协议 | Graph（图）、共享 State |
| 拓扑 | 有向**无环**，线性/并行分支 | 允许 **环**（ReAct 循环天然表达） |
| 控制流 | 隐式，写死在管道里 | 显式，`add_conditional_edges` 决定下一步 |
| 状态 | 靠上下文在组件间传递，无全局状态 | 显式 `TypedDict` 状态，随节点增量更新 |
| 持久化 | 基本没有（需自己接） | **Checkpointer**：断点续跑、时间旅行、故障恢复 |
| 人机协同 | 难（黑盒 AgentExecutor） | 原生 `interrupt`，随时暂停/改状态/恢复 |
| 记忆 | 靠 Memory 类拼装 | 短期 = thread 级 state；长期 = store |
| 适合 | RAG 管道、ETL 式处理、批量/流式调用 | 长任务、多步自主 Agent、需人工审核的生产系统 |

关系要点：

1. LangGraph 是**同一团队**的产品，但定位是「编排层升级」而非替代 LangChain 的模型/工具抽象——它复用 `langchain-core` 的 `BaseChatModel`、消息类型和 `@tool`，也可以完全脱离 LangChain 单独使用。
2. 两者是**上下游**：LangChain 管“单步怎么调”，LangGraph 管“多步怎么走、走到哪、能不能回头”。
3. 老 `AgentExecutor` 是黑盒，LangGraph 就是把那个黑盒拆开、摊平成你能看见的图——这也是它成为当前生产级 Python Agent 首选的原因。

## Mastra 的位置

Mastra 是 2024 年底开源（Gatsby 团队）的 **TypeScript-first Agent 框架**，理念上几乎是“把 LangGraph 的有状态编排 + LangSmith 的观测/评测 + Next.js 式的开发体验”打成一个包。

- **Agent 是一等公民**：`new Agent({ name, instructions, model, tools, memory })`，内置多轮记忆与 thread 概念。
- **Workflow 用 `.step().then().branch().parallel()` 串**，比手搓状态图省事；底层也是 DAG/图，但默认帮你把持久化、重试、挂起做了。
- **配套齐全**：本地 Studio（可视化调试/对话）、evals、scorers、RAG（chunk/embed/vector store）、MCP 客户端、部署到 Vercel/Cloudflare。
- **代价**：生态和 Python 侧集成远不如 LangChain，模型/向量库/工具的“连接器数量”不在一个量级；跨语言团队维护成本高。

## 选型速记

| 场景 | 选 |
| :--- | :--- |
| Python 技术栈、复杂有环 Agent、要断点续跑/人工审核 | **LangGraph**（+ LangSmith） |
| 只是线性管道、RAG 组装、想用现成连接器 | **LangChain（LCEL）** |
| Node/TS 服务、全栈 TypeScript、想要开箱即用的 Agent + Studio | **Mastra** |
| 单轮调用、工具循环不复杂 | 直接官方 SDK，别上框架 |

> 三者共同的演进方向：**从“链”走向“图 + 持久执行”**。区别只在于语言生态和电池是否自带。

# LangChain和LangGraph和Mastra和Eino

| 层         | 负责什么                 | LangChain 系                                    | Mastra                            | Eino                                |
| :-------- | :------------------- | :--------------------------------------------- | :-------------------------------- | :---------------------------------- |
| 组件 / 模型层  | 统一模型、工具、检索器等接口       | LangChain（`langchain-core` + 各种 integrations）  | 底层借用 Vercel AI SDK                | `components` + `eino-ext`           |
| 编排运行时层    | 图、状态、checkpoint、中断恢复 | LangGraph                                      | Workflow（step / branch / suspend） | `compose`（Chain / Graph / Workflow） |
| Agent 抽象层 | 现成的 ReAct、多 Agent    | LangChain 1.0 的 `create_agent`（跑在 LangGraph 上） | `Agent` 类                         | `flow` / ADK（ReAct、多 Agent）         |
| 观测 / 调试层  | 链路追踪、评测、可视化          | LangSmith（单独的产品）                               | 内置 Studio、evals                   | Eino Dev 插件、CozeLoop                |
|           |                      |                                                |                                   |                                     |
