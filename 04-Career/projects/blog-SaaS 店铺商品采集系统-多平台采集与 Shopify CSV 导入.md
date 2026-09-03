---
title: SaaS 店铺商品采集系统 · 多平台采集与 Shopify CSV 导入服务
tags: [python, fastapi, curl-cffi, playwright, postgresql, web-scraping, anti-bot, proxy-pool, etl, docker, kubernetes]
created: 2026-09-03
updated: 2026-09-03
aliases: [scraper-tools, SaaS 采集系统, 店铺商品采集系统, 多平台商品采集]
summary: 面向跨境电商运营团队的多平台商品采集后端服务，覆盖 12 个独立站/电商平台专属适配器与通用兜底解析，产出可直接导入 Shopify 的商品 CSV，落地反爬、代理池评分与持久化、断点续采、字段口径治理等工程化能力
type: blog-project
---

# 概述

**SaaS 店铺商品采集系统**（2026.07 - 至今 · 核心开发者，主导后端服务架构与采集链路开发）

面向跨境电商运营团队的商品采集后端 REST API（FastAPI + PostgreSQL，无前端页面），覆盖 Shopify、店匠（Shoplazza）、Shopline、Xshoppy、Wshopon、Hotishop、LightInTheBox 等独立站建站平台，以及 Etsy、Amazon、1688、TikTok Shop 等电商平台；没有专属适配器的平台统一走通用兜底解析链，不会因"平台不在列表里"直接拒绝采集。采集结果统一产出可直接导入 Shopify 店铺的商品 CSV。

# 技术栈

**后端**：Python · FastAPI · uvicorn · asyncio 异步任务
**采集与反爬**：curl_cffi（Chrome TLS 指纹模拟）· BeautifulSoup4 + lxml · Node.js + Playwright（浏览器渲染子进程）
**存储**：PostgreSQL · psycopg3 连接池
**部署**：Docker / Docker Compose · GitLab CI → Kubernetes GitOps

# 核心工作与技术亮点

1. **多平台自动识别与全平台兜底覆盖架构**
   - `detect.quick_guess` 域名特征识别 + 低成本试探 Shopify `.json` 端点 + 页面特征识别三级判定链，命中已接入的 12 个专属适配器平台，字段解析最全（逐变体 SKU/价格/规格）；未命中平台一律进入通用兜底解析链（平台内嵌 JSON → Next.js App Router RSC flight 商品对象 → JSON-LD → 前端水合状态 → OG/meta 标签），平台标识落库为"通用规则"。
   - 目录/分类页采集支持自动翻页发现商品链接、按 URL 去重入队（`ON CONFLICT DO NOTHING`），支持暂停后按 `next_page` 断点续采；严格区分"发现页失败"与"分类确实为空"，避免第 1 页零商品时被误判为任务完成。

2. **无官方接口 / 强反爬平台的专项攻坚**
   - 1688：识别出 HTML 抓取反爬极严（正常浏览器访问都会被跳转到滑块验证码），改为对接阿里巴巴跨境分销官方开放平台接口（`product.search.queryProductDetail`）获取商品详情，不再依赖页面抓取；目录/分类页因官方接口权限未开放，暂时保留 HTML 抓取兜底。
   - TikTok Shop：定位到商品/店铺页服务端只返回前端渲染空壳，改用 Node.js + Playwright 子进程驱动真实 Chrome 渲染出完整 HTML 后再解析；进一步验证发现即便渲染成功，平台仍会按 IP 信誉判定为机器人跳转安全验证页，明确记录该限制需要真正干净的住宅代理才可能突破。
   - ShopPlus、Next.js RSC 独立站等无品牌特征、只能靠页面专属数据结构识别的平台，逐一用真实店铺验证定位其数据源（`window.ShopPlus`/`window.sinfo` 全局命名空间、RSC flight 流内嵌 JSON 等）完成专属解析。

3. **反爬与稳定性工程**
   - curl_cffi 模拟 Chrome TLS 指纹绕过基础指纹检测，按域名限流 + 指数退避重试，密码保护店铺自动解锁，人机验证页识别，补充手机端指纹请求兜底应对部分平台的移动端专属反爬策略。
   - 代理池从纯内存轮换升级为评分机制 + 数据库持久化（`tb_proxy_state`/`tb_proxy_usage`），解决服务重启后代理健康状态（失败计数/冷却）清零、劣质代理反复被选中的问题。

4. **字段口径治理**
   - 梳理 `product_type`（店铺类目原文）与 `product_category`（Shopify 标准分类）两套语义不同的类目字段，制定"类目原文取平台内嵌 JSON > JSON-LD > 页面面包屑，标准分类映射不上一律留空、不拿标题去猜"的取值优先级，避免误导入错误的 Shopify 标准类目、影响 Google Shopping 投放效果。
   - 销量字段按"真实销量字段 > 库存倒推 > 虚拟销量插件 > 评论数"优先级链路采集；确认源站公开端点普遍不暴露真实库存后，改为库存数量与库存跟踪字段统一固定值输出，不再采一个似是而非的近似值。

5. **任务系统与可观测性**
   - 基于 `asyncio.create_task` 的异步任务模型，服务重启后 `recover_stale_jobs` 自动把所有 `running` 任务转为 `paused`，避免僵死任务占位；补充任务失败重试与逐条采集耗时统计日志。
   - `/healthz` 与 K8s 存活/就绪探针共用，并单独屏蔽其高频访问日志；业务日志与 HTTP 访问日志统一按大小滚动（单文件 10MB、保留 5 份历史）。
   - 38 个测试文件覆盖各平台专属提取逻辑、代理池评分与持久化、任务生命周期等核心链路，遵循团队红绿测试与二分法定位失败的规范。

# 量化成果

- 约 1 个月（2026.07 - 2026.08）完成 **86 次**提交，落地 **12 个**平台专属适配器 + 覆盖任意其余平台的通用兜底解析；
- 1688 详情采集从 HTML 抓取切换为官方开放平台接口，规避滑块验证拦截；TikTok Shop 通过 Node.js + Playwright 渲染绕过前端空壳限制；
- 代理池引入评分机制与数据库持久化，解决服务重启后健康状态清零、劣质代理反复命中的问题；
- **38 个**测试文件覆盖平台提取、代理池、任务生命周期等核心模块。

# 项目收获

在多个平台完全无官方接口或接口受限的采集场景下，完整积累了从 HTML 逆向解析到官方接口对接、反爬体系设计、异步任务与断点续采架构的全链路工程经验；在类目、库存、销量等字段语义存在歧义时，建立了"信息不可靠时宁可留空或用固定值，也不给近似值"的数据治理原则，形成了可迁移到其他数据采集类项目的判断标准。

# 相关链接

- [[blog-天呓网络科技有限公司-前端工程师]]
