---
title: 数据上报助手 · Meta Ads Manager UTM 数据看板
tags: [chrome-extension, manifest-v3, wxt, vue3, typescript, performance, refactor]
created: 2026-09-02
updated: 2026-09-02
aliases: [数据上报助手, FB UTM 插件, Ads Manager UTM 插件, utm-plugin]
summary: 面向跨境电商投放团队的 Chrome MV3 插件，在 Meta Ads Manager 原生表格里叠加真实订单数据，独立完成从油猴脚本到 MV3 正式交付、再到 WXT+Vue3+TS 架构重构的全生命周期
type: resume-project
---

# 概述

**数据上报助手 · Meta Ads Manager UTM 数据看板**（2026.05 - 至今 · 个人独立设计与开发）— 面向跨境电商投放团队的 Chrome MV3 插件，在 Meta Ads Manager 原生表格中叠加真实订单数据（订单数/金额/自算 ROAS），解决与平台"成效"数据的归因口径差异问题。技术栈：Chrome Extension Manifest V3 · WXT · Vue 3 · TypeScript · Service Worker

# 核心贡献

- 逆向解析 Meta Ads Manager 虚拟滚动表格 DOM 结构，实现无官方接口下的数据精准叠加，并开发隐身脚本拦截平台反插件检测弹窗
- 设计批量订单接口与并行请求架构，将逐行 GET 重构为 `⌈N/80⌉` 次并行分片 POST，并实现占位先显、增量注入的无感刷新策略
- 构建 5 级时区降级解析链路（`MAIN World` 网络探针 hook `fetch`/`XHR` + DOM 抓取 + 按账户缓存 + 静默弹窗兜底），实现跨时区账户日期自动对齐
- 主导将约 4700 行单体 `content.js` 重构为 WXT + Vue 3 + TypeScript 模块化工程（9 个职责域、40+ 模块），全程新旧代码树并行、业务零感知

# 核心成果

- 订单接口请求量从 **N 次/页** 降至 **⌈N/80⌉ 次/页**，大账户加载等待显著缩短
- 表格自动刷新间隔从 **约 3 秒/轮** 优化至 **约 1.5 秒/轮**，滚动回显延迟从约 0.9 秒降为即时显示
- 存量代码从 **约 4700 行单文件** 重构为 **9 目录、40+ 模块**（约 6600 行 TS/Vue），维护成本显著下降
- 独立产出并维护 **50 个迭代里程碑记录**，保障团队协作与交接效率

# 相关链接

- [[resume-天呓网络科技有限公司-前端工程师]]
