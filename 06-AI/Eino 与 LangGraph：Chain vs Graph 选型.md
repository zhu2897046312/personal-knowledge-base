---
title: Eino 与 LangGraph：Chain vs Graph 选型
tags: [eino, langgraph, agent, go, llm-framework]
created: 2026-08-13
updated: 2026-08-13
aliases: [Eino, LangGraph, Chain vs Graph]
summary: Eino（Go）和 LangGraph（Python）都只是 LLM 应用开发框架/SDK，不是开箱即用的成品；具体业务逻辑要自己写。个人知识库/博客问答用 Chain 就够，Graph 只在需要循环自纠错、动态工具路由、多 Agent 协作时才有必要
type: learning
---

# 目标

厘清 Eino / LangGraph 这类框架到底在解决什么问题、"学习 LangGraph"具体学的是什么，以及个人项目该选 Chain 还是 Graph，避免过度设计（Overengineering）或误判 Eino 的学习成本。

# 知识点

## 1. 框架 vs 成品：Eino/LangGraph ≈ Gin/Express，不是 Dify/AnythingLLM

- Eino、LangGraph 的本质**都是 LLM 应用开发框架（SDK/Lib）**，提供底层架构、编排能力与组件抽象，具体业务逻辑（Prompt 怎么写、数据怎么切、接口怎么调、状态怎么流转）必须自己写代码实现。
- 对比：Eino/LangGraph 好比 Web 开发里的 Gin/Express/Spring Boot；Dify/FastGPT/AnythingLLM 才是像 Wordpress 一样开箱即用、可视化拖拽部署的成品应用。
- 在 Eino 里"手动实现具体业务"包含四部分：自定义组件（Document Loader、Tools）、状态定义（State struct）、图结构与条件边（Graph & Conditional Edges）、暴露为标准后端 API（Gin/RPC + SSE 流式输出）。

## 2. "学习 LangGraph"学的是什么

传统 LangChain 是单向流水线（DAG）：`输入 -> 检索 -> 组装 Prompt -> 大模型 -> 输出`，遇到需要反复重试、循环自纠错、多 Agent 轮流对话的场景会很僵硬。LangGraph 引入有环图（Cyclic Graph），核心学习点：

1. **状态与节点设计**：State 是全局数据对象（记录进度、工具结果、Token 消耗），Node 是处理逻辑最小单元。
2. **条件循环与自我纠错**：用条件边（Conditional Edges）让图"打转"，如代码生成 → 运行测试 → 失败则带上报错信息回到生成节点重试。
3. **状态持久化与人工干预（Human-in-the-Loop）**：高风险操作（发邮件、退款）时图自动挂起，等人工审批后再恢复执行。
4. **多 Agent 路由与协作**：Supervisor Agent 判断任务该转交给哪个子 Agent，并控制消息传递。

## 3. Eino vs LangGraph 对照

| 维度 | LangGraph | ByteDance Eino |
| --- | --- | --- |
| 语言生态 | Python / TypeScript | Go |
| 核心思想 | 基于图和状态控制 Agent | 基于图和组件编排 RAG/Agent |
| 适用场景 | 算法验证、快速探索、LangChain 迁移 | 强类型、高并发、微服务架构、企业级生产落地 |

掌握了 Eino 的节点、编排、状态流转，就已经掌握了 LangGraph 核心思想的 80%，两者底层思维（状态机 + 图结构破解大模型不确定性）是通用的。

## 4. Eino 学习成本高的应对：抓主干、放分支

- **暂时抛弃 Graph，只用 Chain**：80% 的 RAG 场景是线性的（输入 → 改写 → 检索 → 组装 Prompt → LLM），先只看 Chain 抽象即可少看一大半文档。
- **不强行套用所有内置组件**：把 Eino 当作大模型调用与 Prompt 组装的控制器，向量查询/文本切分直接用最熟悉的 Go 原生代码（如 Qdrant Go SDK）包装成函数接入。
- **警惕"为了用框架而用框架"**：一个人开发或业务复杂度不高时，代码可读性和开发效率优先于框架规范性，可以退回原生 Go（goroutine/errgroup + if-else + 官方 SDK）。

## 5. Chain 能否支持多轮对话：可以，且大多数场景够用

多轮对话在执行层面仍是一条单向直线：每次新消息都是一次独立请求，把"历史聊天记录"作为参数传给 Chain 即可，不需要可循环的 Graph。

```text
[ 用户新提问 + 数据库中的历史对话 ]
               │
               ▼
   1. Query 改写节点 (LLM)
               │
               ▼
   2. 向量检索节点 (Retriever)
               │
               ▼
   3. Prompt 组装节点
               │
               ▼
   4. 大模型生成节点 (LLM) ──► 流式返回给前端
```

只有需要"反复自我检查代码并重新运行"或"检索不准时自动退回上一步换关键词重试"这类**内部循环重试**场景，才必须上 Graph。

## 6. 选型结论：个人博客/知识库问答 vs Agent 工具调用

- **个人博客/知识库问答**：核心逻辑是单向收敛（提问 → 检索 → 提炼 → 回答），结构定死、不需要试错决策。用 Graph 是过度设计；Chain（甚至原生 Go 函数）几行代码串完，加载快、好调试。唯一边缘例外是做"智能路由"（判断博客问题走 RAG、天气走 API、闲聊直接答），但这种简单分类用 `switch-case` 也足够。
- **Tool Call 与自我修正类场景（类似 Claude / Coding Agent）**：执行路径不确定且带回环，必须用 Graph，依赖三个核心能力：
  1. 有环循环（Cycles）：测试不通过带上报错信息回到上一步重试
  2. 动态 Tool Call（工具链路由）：执行完一个 Tool 后自主决定下一个调哪个 Tool
  3. 断点与人工干预：高危指令自动挂起等待人类确认

# 示例

Eino Graph 编排带自我纠错的伪代码：

```go
type RAGState struct {
    OriginalQuery  string
    RewrittenQuery string
    RetrievedDocs  []string
    FinalAnswer    string
}

g := graph.NewGraph[RAGState]()
g.AddNode("rewrite_query", rewriteQueryNode)
g.AddNode("retrieve_vector", vectorSearchNode)
g.AddNode("evaluate_result", evalNode)
g.AddNode("generate_answer", generateAnswerNode)

g.AddConditionalEdge("evaluate_result", func(ctx context.Context, state RAGState) string {
    if len(state.RetrievedDocs) == 0 {
        return "rewrite_query" // 召回为空，回退到改写节点重新跑
    }
    return "generate_answer"
})
```

# 总结

> Eino/LangGraph 是骨架，业务代码是骨架里填的肌肉。个人知识库/博客问答场景直接用 Chain 追求简单稳定、极致首字响应；只有做代码助手、工作流自动化、Agent 机器人这类需要循环修正和动态工具路由的场景，才值得上 Graph。

# 相关链接

- [[LLM Tool Calling 落地：CLI 与配置化 Agent]]
- [[个人知识库 RAG 流程]]
