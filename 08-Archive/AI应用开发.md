# 检索
1. 混合检索架构（Hybrid Search）：同时搭建 BM25 稀疏检索（关键词匹配）与 Dense Vector 向量检索，引入 RRF（Reciprocal Rank Fusion）算法做融合重排。
2. 多模态与复杂结构解析：处理带有复杂表格、PDF 跨页图表、代码块的数据源，设计自适应的父子块（Parent-Child Chunking）策略。
3. 向量数据库高性能调优：在大规模数据量下优化 Qdrant / Milvus 的 HNSW 索引参数、内存占用与召回率。

# Prompt组合设计
1. 标准 Prompt 结构模板，使用显式的结构化标签
2. 复杂图结构（Graph-based Agent）：将复杂业务拆解为确定性的状态机（如 LangGraph 或自定义状态机），控制 Agent 在条件分支、循环修正、人工干预（Human-in-the-loop）之间平滑切换。
3. Tool Call 与系统集成：如何让 LLM 安全且精准地调用企业内部的数据库、RPC 接口或第三方 OpenAPI，并处理工具调用失败时的容错和自我纠错逻辑（Self-Correction Loop）。
4. Prompt 注入防御

# 性能、成本与系统架构（System Engineering）
1. 流式响应与极致首字延迟（TTFT Optimization）：如何利用 Server-Sent Events (SSE) / WebSocket 实现打字机流式输出，配合异步任务队列（如 Redis / RabbitMQ）解耦长时间任务。{标准的SSE流式输出}
2. 多级缓存策略：引入语义缓存（Semantic Cache，如 GPTCache），对高度相似的 Query 直接走向量缓存返回，不消耗 LLM Token。
3. 型分级与路由机制（Model Routing）：基于用户问题的复杂程度，用规则或小模型路由到不同模型（简单意图用轻量小模型，复杂逻辑用推理大模型），在控制 Token 成本的同时保证用户体验。

# 可评估性与可观测性（Evals & Observability）
1. 自动化评估流水线（Evals）：建立包含 Hit Rate、MRR（平均倒数排名）、Faithfulness（忠实度）、Answer Relevance 的自动化评估套件（如 Ragas / TruLens），在提交代码或更新 Prompt 时自动跑分。
2. 链路追踪（LLM Tracing）：集成 LangFuse、OpenTelemetry 等工具，监控单次请求在节点改写、向量召回、Prompt 组装、大模型输出各个环节的耗时与 Token 开销，准确定位性能瓶颈。


# pkb项目做了
1. 按 Markdown 1~3 级标题（#/##/###）切成 section，每个 section 再按 maxChunkSize=1200 字符做二次拆分；没有重叠（overlap）设计，是"标题优先、定长兜底"的朴素策略。