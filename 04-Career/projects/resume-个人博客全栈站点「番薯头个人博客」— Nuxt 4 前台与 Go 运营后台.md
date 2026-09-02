---
title: 个人博客全栈站点「番薯头个人博客」— Nuxt 4 + Go + Medusa 电商
tags: [nuxt4, vue3, typescript, go, gin, gorm, postgresql, redis, nuxt-ui, medusa, rag, i18n, monorepo]
created: 2026-09-02
updated: 2026-09-02
aliases: [番薯头个人博客, personal-blog, sweetphotohead]
summary: 独立设计并持续运营的个人博客 + 轻量电商全栈站点，pnpm Monorepo 拆分 Nuxt 4 SSR 前台与 Go 后端，完成 MySQL→PostgreSQL 迁移与 Medusa v2 电商接入
type: resume-project
---

# 概述

**个人博客全栈站点「番薯头个人博客」**（2026.05 - 至今 · 个人独立设计与开发）— 前后端分离的个人内容站点 + 轻量电商：Nuxt 4 SSR 前台承载阅读与购物体验，Go 后端提供内容、鉴权、支付 API，近期接入 Medusa v2 打通商品详情与购物车。技术栈：Nuxt 4 · Vue 3 · TypeScript · Nuxt UI 4 · Go · Gin · GORM · PostgreSQL · Redis · Medusa · Docker · Vercel

# 核心贡献

- 独立设计 pnpm Monorepo 架构，Nitro 同源代理统一处理前后端跨域与静态资源转发，SSR / 预渲染 / SWR 按页面差异化配置
- 主导 MySQL → PostgreSQL 数据库迁移，修复约 60 处方言差异并交付一次性数据迁移工具，本地全链路验证后上线生产
- 接入 Medusa v2 headless 电商（独立部署、自带 PostgreSQL + Redis 的服务），设计 Nitro 代理方案解决裸 IP + HTTP 后端在 HTTPS 环境下的混合内容拦截问题，实现商品详情、规格选择、购物车抽屉与立即购买
- 通过源码级核实第三方 PayPal 支付插件的会话契约规避实施风险，并主导品牌色系统从 450+ 处硬编码治理为 NuxtUI 主题色阶
- 接入独立部署的 RAG 知识库问答服务，在 Go 后端设计安全代理层：密钥不下发浏览器、请求体/字数校验、按 IP 限流、跨服务超时余量控制

# 核心成果

- 独立完成 **115** 次提交、约 **3.5 个月**持续迭代的全栈项目，覆盖 **69** 个 Go 源文件（**56** 处 REST 接口）与 **53** 个 Vue 组件/页面
- 完成 **1** 次数据库大版本破坏性迁移，定位并修复 **2** 处生产级混合内容/CORS 问题
- 品牌色治理覆盖 **40** 个文件、**450+** 处硬编码，联动统一约 **110** 处默认色组件

# 相关链接

- [[resume-个人知识库 RAG 问答服务「pkb」— Go 与 CloudWeGo Eino 框架]]
