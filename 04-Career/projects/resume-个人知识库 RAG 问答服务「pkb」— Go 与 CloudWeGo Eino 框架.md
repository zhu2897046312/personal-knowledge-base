---
title: 个人知识库 RAG 问答服务「pkb」— Go + CloudWeGo Eino + Qdrant + DeepSeek
tags: [go, rag, cloudwego-eino, qdrant, deepseek, dashscope, gin, sqlite, docker]
created: 2026-09-02
updated: 2026-09-02
aliases: [pkb, PKB, personal-knowledge-base-rag, pkb-rag]
summary: 独立设计并持续迭代的 Go 原生 RAG 问答服务，23 天从 0 到 1，索引 Obsidian 知识库并结合向量检索与 DeepSeek 生成带引用答案，完成向 CloudWeGo Eino 官方组件的框架迁移、多轮对话与 prompt 注入防御，修复过生产环境内存故障
type: resume-project
---

# 概述

**个人知识库 RAG 问答服务「pkb」**（2026.07 - 2026.08 · 个人独立设计与开发）— 索引外部 Obsidian 知识库、用向量检索匹配相关笔记片段、结合大模型生成带引用来源答案的 Go 原生 RAG 服务，提供 CLI 与被博客后端安全代理调用的 HTTP API。技术栈：Go · CloudWeGo Eino · Gin · Qdrant · DeepSeek · DashScope · SQLite · Docker

# 核心贡献

- 独立完成 RAG 全链路设计：Obsidian 笔记解析、按标题分块、向量检索、prompt 组装与生成
- 主导框架级重构，将自建 HTTP 客户端替换为 CloudWeGo Eino 官方组件（embedding/检索/对话模型），删除冗余自定义实现
- 设计并实现多轮对话能力，新增 SQLite 会话历史存储与 session_id 请求契约
- 用 XML 标签隔离与转义防御 prompt 注入，保障检索内容与用户输入不被模型误判为指令
- 排查并修复生产环境 2GB 内存共享导致模型加载卡死整机的故障，编写含依赖顺序启动、内存预算检查与失败自动回滚的生产部署脚本

# 核心成果

- 独立完成 **18** 次提交、**23 天**从 0 到 1，覆盖 **23** 个 Go 源文件、**10** 个职责单一模块
- **8** 个测试文件、**28** 个测试函数覆盖全部核心模块（含 HTTP 鉴权、边界、超时与健康检查）
- 完成 **1** 次框架级迁移与 **1** 次 embedding 服务商切换（本地 Ollama → 远程 DashScope），交付 **251 + 83** 行生产部署脚本，修复 **1** 次生产内存故障

# 相关链接

- [[resume-个人博客全栈站点「番薯头个人博客」— Nuxt 4 前台与 Go 运营后台]]
