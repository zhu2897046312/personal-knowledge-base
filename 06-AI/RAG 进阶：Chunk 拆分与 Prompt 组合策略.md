---
title: RAG 进阶：Chunk 拆分与 Prompt 组合策略
tags: [rag, chunking, prompt-engineering]
created: 2026-08-13
updated: 2026-08-13
aliases: [Chunk 拆分策略, Prompt 组合策略]
summary: Chunk 拆分决定检索召回的准确度，Prompt 组合决定大模型利用检索内容生成回答的质量与抗幻觉能力
type: cheatsheet
---

# 概述

在 RAG（检索增强生成）系统中：

- **Chunk 拆分策略**决定检索召回的准确度（Precision/Recall）
- **Prompt 组合策略**决定大模型利用检索内容生成回答的质量与抗幻觉能力

文档切分核心在于**保持切分块内语义的完整性**，同时控制在向量模型的最佳表达长度内，并非"越小/越大越好"。

# 要点

- 固定长度切分简单但容易截断语义，仅适合无结构自由文本
- 递归字符切分（按 `["\n\n", "\n", "。", " ", ""]` 层级分隔符）是目前通用文本切分的标配
- 结构/语义感知切分（按标题层级、代码 AST、语义相似度断点）更适合 Obsidian 笔记、技术文档、源码
- 父子块策略：**用小块检索，用大块生成**，解决"小块精准但缺上文，大块完整但检索稀释"的矛盾
- Prompt 组合要解决**上下文混淆、Lost in the Middle（中间信息丢失）、模型幻觉**三个问题
- 抗 Lost in the Middle：把最高相关性的 Chunk 放在 Prompt **最前**和**最后**，中等分数放中间

# 用法

## 一、Chunk 拆分策略

### 1. 基础物理切分

- **固定长度切分（Fixed-size Chunking）**：设定固定 Token/字符数（如 512 Tokens），配置 10%~20% 重叠度（Overlap）。优点是简单、开销小；缺点是容易截断句子/段落。
- **递归字符切分（Recursive Character Chunking）**：按层级分隔符列表递归拆分，优先在段落/句子边界断开，尽量填满目标 Chunk 尺寸（如 LangChain `RecursiveCharacterTextSplitter`）。

### 2. 结构/语义感知切分（推荐）

- **基于文档结构切分**：按 Markdown/HTML 标题层级递归拆分，把标题路径（如 `架构设计 > 数据库 > 索引`）作为元数据注入 Chunk 头部；代码按类/函数/接口作用域切分。
- **基于语义相关性切分（Semantic Chunking）**：按句拆分后用 Embedding 计算相邻句子余弦相似度，相似度突变时划分断点。边界完全由语义驱动，但耗费更多 Embedding 计算资源。

### 3. 检索块与上下文块分离（高级架构）

- **父子块策略（Parent-Child / Small-to-Big）**：切出较大的 Parent Chunk（如 1000 Tokens），再切成多个 Child Chunk（如 128 Tokens）。Child 向量化存库，匹配到 Child 后把关联 Parent 送给大模型生成。
- **摘要/假设问题切分**：对每个 Chunk 用 LLM 生成 3~5 个潜在问题或一句摘要，把"问题/摘要"向量化存储，匹配后返回原 Chunk 内容（HyDE / Inverse HyDE 思路）。

## 二、Prompt 组合策略

### 1. 标准 Prompt 结构（XML 标签隔离）

用显式结构化标签（`<system_instructions>`、`<retrieved_context>`、`<user_query>`）隔离检索内容与指令，能防止 Prompt 注入并提高模型解析准确度：

```markdown
<system_instructions>
你是一个严谨的技术知识库助手。请严格基于后文提供的 <retrieved_context> 中的信息回答用户的询问。
规则如下：
1. 答案必须严格来自给定的检索上下文，不得包含上下文未提及的假设。
2. 如果给定的上下文不足以回答问题，请直接回答："根据已有资料，无法回答该问题。"，严禁编造答案。
3. 回答时在引用观点后方标注数据源 ID（格式：[文档名/ID]）。
</system_instructions>

<retrieved_context>
  <doc id="doc_1" title="部署指南.md">
    Qdrant 服务默认占用 6333 端口...
  </doc>
</retrieved_context>

<user_query>
PKB 服务默认的超时时间是多少？
</user_query>
```

### 2. 上下文排列优化

- **抗 "Lost in the Middle" 重排序**：大模型对 Prompt 头部和尾部信息关注度最高。用 Rerank 打分后，最高分放最前和最后，中等分放中间（如 `[1st, 3rd, 5th, 4th, 2nd]`）。
- **元数据显式注入**：在 Chunk 头部拼接文件路径、修改时间等，如 `[来源: docs/api.md | 更新时间: 2026-03-10]`。

### 3. 多 Chunk 融合范式（Chunk 数量过多时）

| 范式 | 工作原理 | 适用场景 | 优缺点 |
| --- | --- | --- | --- |
| **Stuffing（直接拼接）** | 把所有检索到的 Chunk 按排序全部塞入同一个 Prompt | 大多数常规对话/长上下文模型 | 优点：单次调用最快；缺点：受限于 Token 上限 |
| **Map-Reduce** | 逐个 Chunk 独立生成中间答案（Map），最后汇总交给 LLM 输出最终回答（Reduce） | 全量文档总结、跨多篇文档对比 | 优点：突破上下文限制；缺点：多次调用，延迟与成本高 |
| **Refine（链式迭代）** | 第一个 Chunk 生成初始答案，再传入第二个 Chunk 修改，循环往复 | 需要逐层补充精炼的递进式问答 | 优点：回答较完整；缺点：无法并发，串行延迟高 |
| **Rerank + Cutoff** | 向量检索后加 Cross-Encoder Rerank，只保留 Top 3~5 个精简 Chunk | 高准确率要求的生产级 RAG（推荐） | 大幅降低噪声与 Token 消耗，提高准确度 |

# 相关链接

- [[个人知识库 RAG 流程]]
- [[多轮 RAG 架构：Query 改写与成本优化]]
