---
title: 个人博客全栈站点「番薯头个人博客」— Nuxt 4 + Go + Medusa 电商
tags: [nuxt4, vue3, typescript, go, gin, gorm, postgresql, redis, nuxt-ui, tailwindcss4, i18n, ssr, medusa, headless-commerce, paypal, rag, monorepo, vercel, docker]
created: 2026-09-02
updated: 2026-09-02
aliases: [番薯头个人博客, personal-blog, sweetphotohead]
summary: 独立设计、开发并持续运营的个人博客 + 轻量电商全栈站点：pnpm Monorepo 拆分 Nuxt 4 SSR 前台与 Go（Gin+GORM）后端，完成 MySQL→PostgreSQL 破坏性迁移、接入 Medusa v2 headless 电商并用 Nitro 同源代理绕开裸 IP+HTTP 后端的混合内容拦截、购物车抽屉与立即购买、品牌色系统从 450+ 处硬编码收拢为 NuxtUI 主题色阶、第三方 PayPal 插件源码级核实支付会话契约、接入独立 RAG 知识库问答服务并做安全代理（限流/超时余量/密钥不下发浏览器）等真实工程细节
type: blog-project
---

# 概述

**番薯头个人博客**（sweetphotohead.com，2026.05 - 至今 · 个人独立设计、开发与持续运营）

一套前后端分离的全栈个人站点：Nuxt 4 SSR 前台承载文章阅读、会员投稿与商品浏览/加购体验，Go（Gin + GORM）后端提供内容、鉴权、支付与文件存储 API；近期在博客基础上接入 Medusa v2 headless 电商，打通商品详情、规格选择与购物车前置能力。

# 系统架构

前台统一部署在 Vercel（Nitro `vercel` preset，仅生产环境按 `VERCEL` 环境变量切换），按路径同源代理到三个各自独立部署、互不感知的后端服务，浏览器不直连任何一个，避免裸 IP/跨域问题：

- **Go 运营后台**（本仓库 `apps/backend`）：内容、鉴权、支付、文件存储 API，数据落 PostgreSQL。
- **Medusa v2 电商后端**（独立仓库）：商品、购物车、订单、PayPal 支付，数据落 PostgreSQL；生产环境通过 `redisUrl` 配置项隐式启用 Redis 驱动的 event bus / workflow engine / cache（Medusa v2 框架按配置项自动切换，不需要在 `modules` 里显式声明 `*-redis` 系列模块），Docker Compose 编排 postgres/redis/medusa 三个容器并配好健康检查依赖顺序。
- **pkb RAG 问答服务**（独立仓库，[[blog-个人知识库 RAG 问答服务「pkb」— Go 与 CloudWeGo Eino 框架]]）：向量检索用 Qdrant，embedding 用阿里云百炼 DashScope `text-embedding-v3`，生成用 DeepSeek `deepseek-chat`，索引的是宿主机上一个独立维护的 Obsidian 笔记库目录，容器内只读挂载，和本站代码仓库是完全不同的目录。

Go 后端对 pkb 做安全代理转发（详见下方"接入独立 RAG 服务"一节），前端对 Medusa 走 Nitro 同源代理直接转发，两条代理路径职责不同：一个是"转发前二次校验/限流"，一个是"纯粹绕开混合内容拦截"。

# 技术栈

- **前端**：Nuxt 4 · Vue 3 · TypeScript · Nuxt UI 4 · Tailwind CSS 4 · `@nuxtjs/i18n`（zh-CN/en）· `@nuxtjs/sitemap` · `@nuxt/image` · `@tanstack/vue-query`
- **后端**：Go 1.24 · Gin · GORM · PostgreSQL（原 MySQL）· JWT · Viper · goldmark（服务端 Markdown 渲染）· Resend 邮件 · 支付宝 / 微信支付 / PayPal SDK
- **电商**：Medusa v2（headless commerce，`@medusajs/js-sdk` 直连 Store API）· 第三方支付插件 `@alphabite/medusa-paypal` · PostgreSQL · Redis（生产环境 event bus/workflow engine/cache）
- **工程与部署**：pnpm Monorepo（无 Turborepo）· ESLint + Husky + lint-staged · Vercel（前台）+ Docker（Go API + PostgreSQL）· Nitro 同源代理

# 核心工作与技术亮点

1. **pnpm Monorepo 架构与 Nitro 同源代理**
   - `apps/frontend`（Nuxt 4）与 `apps/backend`（Go）独立构建部署，根目录纯 pnpm workspace；Nitro `routeRules` 把 `/api/**`、`/oss/**` 同源代理到 Go，避免浏览器跨域，管理端/会员端关闭 SSR 降低复杂度，首页与联系页走预渲染 + 5 分钟 SWR，文章详情强制关闭缓存避免返回不完整 HTML。

2. **MySQL → PostgreSQL 的破坏性迁移**
   - 驱动从 `gorm.io/driver/mysql` 换成 `postgres`，修复 12 个仓储文件里约 60 处原始 SQL 大小写标识符与 `LIKE`→`ILIKE` 的方言差异；编写一次性数据迁移 CLI（`cmd/migrate-mysql-to-postgres`）完成生产数据搬迁；`go.mod` 固定 `go 1.24.0` 以匹配生产 `golang:1.24-alpine` 镜像。本地起真实 Postgres 容器验证了建表、管理员登录、分类/文章/评论 CRUD、大小写不敏感搜索、支付订单查询等全链路。

3. **接入 Medusa v2 headless 电商，绕开裸 IP + HTTP 后端的混合内容拦截**
   - Medusa 后端只有裸 IP + HTTP，浏览器在 HTTPS 页面下直连（购物车写操作、商品图片）会被混合内容策略静默拦截；新增 `/medusa-api/**` 的 Nitro 同源代理，浏览器端 SDK 改用 `window.location.origin` 拼出合法的同源 `baseUrl`（SDK 不支持传裸路径前缀），服务端 SSR 分支保持直连真实地址，并把这条排查路径整理进独立 Runbook 供复用。

4. **购物车抽屉与立即购买**
   - 拆分 `useCartState`（纯状态 + Cookie 持久化 + 派生 itemCount/subtotal）与 `useCart`（写操作，统一 pending/error、失败不自动重试）两层 composable；全局购物车抽屉严格复用 NuxtUI 组件（`UDrawer`/`UChip`/`UInputNumber`）而非手写弹层，Header 图标与商品详情页的加购/立即购买共用同一份禁用态判断，但各自独立展示 loading 文案，避免共享状态下两个按钮的反馈互相串扰。

5. **第三方 PayPal 插件源码级核实，避免凭文档臆测返工**
   - 电商结账要接的是社区插件 `@alphabite/medusa-paypal` 而非 Medusa 官方插件；直接读其 GitHub 源码确认支付会话在 `initiatePayment` 阶段就已在服务端建好 PayPal 订单（前端无需重复建单），并发现"捕获失败会静默创建新订单返回 `PENDING`"这一特殊分支，提前把前端重试逻辑写进实施计划，避免了按官方通用文档实现后上线才发现契约不符。

6. **品牌色系统治理：从 450+ 处硬编码收拢为 NuxtUI 主题色阶**
   - 排查出全站约 450 处 `bg-[#hex]` 任意值硬编码分布在 40 个文件；发现 4 个已在使用的品牌红色值色相高度一致，以它们为锚点反推出符合 Tailwind 50–950 规范的完整色阶写入 `app.config.ts` 的 `ui.colors.primary`；改动后全站约 110 处未显式传 `color` 的 NuxtUI 组件自动统一为品牌色，替换用脚本批量处理后逐一核对目标 hex 不再以字面量形式残留。

7. **接入独立 RAG 服务，做安全代理而非直连**
   - 知识库问答能力由独立部署的 [[blog-个人知识库 RAG 问答服务「pkb」— Go 与 CloudWeGo Eino 框架]]（Go + CloudWeGo Eino + Qdrant + DeepSeek）提供；博客 Go 后端不是简单转发，而是承担了一层安全代理：校验请求体大小（16 KiB）、单 JSON object、空问题与 2000 字符上限，服务端读取共享密钥后注入 `X-Api-Key` 调用 PKB（浏览器拿不到这个密钥），95 秒上游超时（比 PKB 自身 90 秒超时多留 5 秒余量，保证 PKB 先返回干净的超时错误码），并做按 IP 的固定窗口限流；前台用 Nuxt UI 的悬浮聊天窗口承载交互，`zh-CN`/`en` 双语言文案与来源路径展示同步做了本地化。

# 量化成果

- 项目周期约 **3.5 个月**（2026.05 至今），**115** 次提交，独立完成从 0 到 1 的全部设计与开发；
- 后端：**69** 个 Go 源文件，**23** 个 service、**13** 个 repository、**6** 个 handler，**56** 处 REST 路由注册；
- 前端：**53** 个 Vue 文件、**65** 个 TS 文件、**36** 个按业务域拆分的 composable；
- 完成 **1** 次数据库大版本迁移（MySQL→PostgreSQL，约 60 处方言差异逐处修复）、定位并修复 **2** 处混合内容/CORS 类生产问题（购物车写操作、商品图片）；
- 品牌色治理覆盖 **40** 个文件、约 **450** 处硬编码，联动统一约 **110** 处默认色组件。

# 项目收获

- **一次真实的数据库大版本迁移**：从驱动替换到方言差异逐处核对，再到编写迁移工具验证数据完整性，比"看文档"更扎实地理解了 GORM 抽象层之下 MySQL/PostgreSQL 在大小写、模糊匹配上的真实差异。
- **不臆测第三方依赖契约的习惯**：无论是 Medusa Store API 的字段展开规则（`fields` 参数需要真实调接口核实才知道哪些关联字段默认不返回），还是社区 PayPal 插件的支付会话流程，都坚持先读源码或用真实后端验证，再落笔写实施方案，明显减少了返工风险。
- **组件库优先、克制自造轮子**：在项目自身约定（NuxtUI 已覆盖的交互一律不重复实现）的约束下，把购物车抽屉、商品卡片等交互都收敛到 NuxtUI 基础组件之上，同时保留了对组件库默认行为（如响应式断点覆盖、`v-model` 契约）保持怀疑、动手验证的习惯。
- **治理性工作同样是工程能力**：品牌色收拢、`components`/`composables`/`utils` 按业务域强制二级目录、Git 提交规范与文件行数阈值等治理性文档，让"和 AI 协作开发"这件事本身变得可检查、可复现，而不只是功能堆叠。

# 相关链接

- [[blog-个人知识库 RAG 问答服务「pkb」— Go 与 CloudWeGo Eino 框架]]
