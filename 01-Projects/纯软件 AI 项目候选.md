---
title: 纯软件 AI 项目候选
tags: [idea, rag, cli, agent]
created: 2026-08-17
updated: 2026-08-17
aliases: [不涉及硬件的 AI 项目]
summary: 智能硬件改造涉及红外/GPIO/接线门槛较高，整理三个零硬件门槛、开发反馈更快的纯软件 AI 项目方向作为备选
type: learning
---

# 目标

[[智能硬件 Tool Calling 改造构想]] 涉及红外学习、GPIO 接线、220V 强电等硬件门槛。作为备选，记录三个不需要买硬件、纯软件/算法即可玩起来的 AI 项目方向，开发循环更快、调试更直观、也更容易在 GitHub 上展示。

# 知识点

## 项目一：个人专属知识库 + 网页/文档智能助手（RAG 系统）

把收藏的技术文章、PDF、代码库、聊天记录塞给它，打造"比自己还懂自己"的私有智脑。核心功能：网页一键剪藏与总结（Chrome 插件）、语义搜索（不需要精确关键字匹配，搜"硬件控制"也能把"MQTT、ESP32"相关文章找出来）、定位式问答。技术栈：Go（Eino/LangChain-Go）+ 向量库（PGVector/Milvus/Qdrant）+ Vue 3 前端 + DeepSeek API + 免费 Embedding 模型（SiliconFlow 或本地 Ollama 跑 bge-m3）。与已有的 [[个人知识库 RAG 流程]] 是同一方向的延伸。

## 项目二：AI 自动化工作流 / 个人数字秘书

让 AI 在软件之间批处理搬运和分析数据。核心功能：每天定时抓取关注的开源项目更新/资讯 RSS，总结成 Markdown 简报推送到钉钉/飞书/Telegram；Git Commit 时本地 Hook 触发，让 AI 检查代码 Bug、命名规范并自动补充注释。技术栈：Go 服务负责定时任务（Cron）、爬虫、Git API 对接；用 Eino 的 Graph 编排"抓取 → 过滤 → LLM 摘要 → 格式化输出"流程。

## 项目三：纯端侧本地命令行 AI 工具（CLI Agent）

在终端里用自然语言直接干活，不用打开浏览器。核心功能：自然语言转 Shell 指令（如 `ai "帮我找到当前目录下所有大于 100M 的 log 文件并压缩"` 生成对应命令并询问是否执行）；本地代码库问答（扫描当前项目 AST 树回答"这个函数在哪里被调用"）。技术栈：Go 写单文件 CLI 工具，结合 Ollama 本地跑 Qwen2.5-Coder，完全脱离外网 API。

# 总结

> 三个方向里，"个人 RAG 知识库"和"CLI 命令行工具"最适合先动手：代码写完 `go run` 就能看效果，不需要等硬件串口或断线重连，2~3 天就能做出能展示的成品。等这两个练熟了，再回头搭配 [[智能硬件 Tool Calling 改造构想]] 里的红外/MQTT 硬件层。

# 相关链接

- [[智能硬件 Tool Calling 改造构想]]
- [[个人知识库 RAG 流程]]
