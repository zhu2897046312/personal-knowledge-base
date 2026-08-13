---
title: 个人知识库 RAG 流程
tags: [rag, qdrant, embedding, deepseek]
created: 2026-07-22
updated: 2026-07-22
aliases: [RAG, 检索增强生成]
summary: 建库用 Embedding 模型把 md 文档转向量存入 Qdrant，问答时用同一 Embedding 模型编码问题去检索，再交给 deepseek chat 生成流式回答
type: learning
---

# 目标

梳理个人知识库 RAG（检索增强生成）系统的两条流程——离线建库、在线问答，并厘清其中容易混淆的一点：**向量化用的是 Embedding 模型，不是 deepseek 这类 Chat 模型**。

# 知识点

**整体分两个阶段：** 建库（离线，处理文档）与问答（在线，处理用户请求）。

## 建库流程（离线索引）

```text
个人知识库 md 文档
      │
      ▼
Go 后端服务：拆 chunk
      │
      ▼
Embedding 模型：文本 → 向量
      │
      ▼
写入 Qdrant（向量 + payload 元数据，如原文、来源路径）
```

## 问答流程（在线检索）

```text
用户提问
      │
      ▼
Embedding 模型：问题 → 向量（必须与建库时用的是同一个模型）
      │
      ▼
Qdrant：向量检索，取回 topK 相关片段
      │
      ▼
拼接 Prompt（system + 检索片段 + 用户问题）
      │
      ▼
deepseek chat 模型：流式输出给前端
```

## Embedding 模型 vs Chat 模型

两条流程里各出现一次"模型"，但不是同一类：

- **Embedding（向量）模型**：只负责把文本映射成向量，不做对话生成。建库和问答两个阶段都要用它，且**必须是同一个模型**——因为检索靠的是向量之间的相似度，如果文档和问题用不同模型编码，向量空间不一致，相似度计算就没有意义。
- **Chat 模型（deepseek chat）**：负责理解拼好的 prompt 并生成自然语言回答，只在问答阶段的最后一步出现，不参与向量化。

# 示例

见上方两张流程图。

# 总结

> 向量化用的是 Embedding 模型，不是 deepseek 这种 Chat 模型；建库和问答两个阶段的 Embedding 模型必须保持一致，否则检索到的相似度没有意义。

# 相关链接

- [[RAG 进阶：Chunk 拆分与 Prompt 组合策略]]
- [[多轮 RAG 架构：Query 改写与成本优化]]
