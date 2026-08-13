---
title: 从 API 调用师到 AI 应用工程师
tags: [career, ai-engineering, rag, agent]
created: 2026-08-13
updated: 2026-08-13
aliases: [AI 应用开发进阶路线图, API 调用师]
summary: 简单拼 Prompt 调大模型 API 只是 AI 应用开发的门槛；真正的壁垒在于用工程化手段解决大模型的不确定性、高延迟、幻觉和边界局限，体现在检索工程、Agent 工作流、系统性能、可观测性四个维度
type: learning
---

# 目标

回应"写 RAG 不就是调 API 吗，干脆叫 API 调用师算了"这种幻灭感——梳理清楚简单 API 拼接和真正的 AI 应用工程化之间的差距具体在哪，以及从"调包侠"进阶的可落地路径。

# 知识点

## 核心壁垒迁移：AI 应用工程要解决的四类问题

### 1. 检索与数据工程（RAG 的下半场）

- 基础：把 Markdown 扔进数据库做简单 Cosine 向量检索
- 进阶：混合检索架构（BM25 稀疏检索 + Dense 向量检索，RRF 融合重排）、复杂表格/PDF/代码块的自适应父子块切分、Qdrant/Milvus 的 HNSW 索引调优

### 2. 工作流与状态机设计（Agent & Workflow）

- 基础：一个 Prompt 从头问到尾
- 进阶：用 LangGraph/Eino 等图结构把业务拆成确定性状态机，支持条件分支、循环修正、Human-in-the-loop；让 LLM 安全调用内部数据库/RPC/第三方 API，处理调用失败的容错和自纠错逻辑（详见 [[Eino 与 LangGraph：Chain vs Graph 选型]]）

### 3. 性能、成本与系统架构

- 基础：等 LLM 生成完一次性返回，用户等待
- 进阶：SSE/WebSocket 流式输出 + 异步队列解耦长任务；语义缓存（如 GPTCache）对高相似 Query 直接命中不走 LLM；模型分级路由（简单意图用轻量模型，复杂逻辑用推理大模型）

### 4. 可评估性与可观测性（Evals & Observability）

- 基础：靠人工点赞/点踩凭感觉判断
- 进阶：自动化评估流水线（Hit Rate、MRR、Faithfulness、Answer Relevance，如 Ragas/TruLens）；链路追踪（LangFuse、OpenTelemetry）定位改写/召回/组装/生成各环节的耗时与 Token 开销瓶颈

## 转型路线图

```text
1. 前后端工程能力 (TypeScript/Vue/Go/Node)
   打字机流式交互、Docker/K8s 容器化、中间件生态
                    │
                    ▼
2. 深入 RAG 与向量工程
   混合检索、Rerank、文档解析与智能切分
                    │
                    ▼
3. Agent 架构与复杂工作流
   状态机控制、Tool Calling、Self-Correction
                    │
                    ▼
4. 工程化防护与可观测性
   Prompt 注入防御、语义缓存、链路追踪、自动化 Evals
```

## 积累经验的具体路径（无需大公司预算）

把每个"大概念"拆成自己可掌控的最小实验（PoC）：

- **一次混合检索实验**：在向量检索之外接入轻量全文检索（Bleve 或 MySQL/SQLite Full-Text Search），对比专有名词/代码符号查询时单向量检索 vs 混合检索的召回率差异，手写一次 RRF 融合排序
- **一个极简状态机实验**：不依赖重型框架，用 Go `switch-case` 写"先判断意图（查笔记/写代码/闲聊）→ 分支处理"，亲手处理一次意图识别错误的容错回退
- **一套简单 Evals**：准备 20 个常问测试问题和标准答案，每次调整 Chunk 长度/System Prompt/Embedding 模型后，用 LLM-as-a-Judge 打分对比效果变化

经验的本质是对"边界条件"和 Bad Case 的掌控力，例如：

| 场景 | 初学者会遇到的问题 | 有经验的思考维度 |
| --- | --- | --- |
| Chunk 切分 | 按 500 字死板切分，代码块/表格被切断 | 结构化解析保持代码块/标题完整性，引入父子块 |
| 检索召回 | 用户搜缩写/错别字查不到 | 检索前做同义词扩展，或用 BM25 兜底 |
| 提示词注入 | 用户诱导模型泄露 System Prompt | XML 结构化隔离输入 + 后端敏感输出正则/语义拦截 |

# 总结

> Web 开发刚兴起时也有人说"不就是拿 HTTP 调数据库 API 吗"，最终区分普通程序员和资深架构师的是高并发、高可用、分布式与安全设计能力。AI 应用开发同理：调 API 解决的是"从 0 到 1 跑通 Demo"，工程化解决的是"从 1 到 100 做到稳定、准确、低延迟、低成本、高并发生产落地"。描述项目时应该讲清楚具体的工程设计取舍和量化结果（如"引入轻量 Query 改写流水线，混合检索+动态切片把回答准确率提升 35%，首字延迟控制在 800ms 以内"），而不是停留在"用 Go 和 Nuxt 搭了个 RAG 系统"这种调包侠式描述。

# 相关链接

- [[RAG 进阶：Chunk 拆分与 Prompt 组合策略]]
- [[多轮 RAG 架构：Query 改写与成本优化]]
- [[Eino 与 LangGraph：Chain vs Graph 选型]]
