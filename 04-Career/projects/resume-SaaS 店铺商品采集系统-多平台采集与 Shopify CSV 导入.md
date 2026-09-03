---
title: SaaS 店铺商品采集系统 · 多平台采集与 Shopify CSV 导入服务
tags: [python, fastapi, web-scraping, anti-bot, resume]
created: 2026-09-03
updated: 2026-09-03
aliases: [scraper-tools, SaaS 采集系统, 店铺商品采集系统]
summary: 主导设计多平台商品采集后端服务，覆盖 12 个独立站/电商平台专属适配器与通用兜底解析，攻坚 1688/TikTok Shop 等无官方接口平台，落地反爬、代理池评分与字段治理
type: resume-project
---

# 概述

**SaaS 店铺商品采集系统**（2026.07 - 至今 · 核心开发者）— 面向跨境电商运营团队的多平台商品采集后端服务（FastAPI + PostgreSQL），覆盖 12 个独立站/电商平台专属适配器与通用兜底解析，产出可直接导入 Shopify 的商品 CSV。技术栈：Python · FastAPI · curl_cffi · Playwright · PostgreSQL · Docker

# 核心贡献

- 主导多平台自动识别与通用兜底解析架构设计，覆盖 Shopify/店匠/Shopline/Etsy/Amazon/1688/TikTok Shop 等 12 个专属适配器平台，其余平台走兜底解析链不因"不在列表"拒绝采集
- 攻坚无官方接口 / 强反爬平台：1688 从 HTML 抓取切换为官方开放平台接口，TikTok Shop 用 Node.js + Playwright 渲染真实 Chrome 页面绕过前端空壳限制
- 设计代理池评分机制与数据库持久化方案，解决服务重启后代理健康状态清零问题；落地 curl_cffi TLS 指纹模拟、按域名限流退避等反爬能力
- 梳理类目 / 库存 / 销量等歧义字段的多源优先级取值策略，建立"歧义字段宁可留空或固定值"的数据治理原则，避免误导入 Shopify 标准分类

# 核心成果

- 约 1 个月完成 **86 次**提交，落地 **12 个**平台专属适配器 + 通用兜底覆盖任意其余平台
- 1688 采集从 HTML 抓取升级为官方接口对接，规避滑块验证拦截
- 代理池引入评分与持久化机制，消除服务重启后健康状态清零问题
- **38 个**测试文件覆盖核心采集与任务链路

# 相关链接

- [[resume-天呓网络科技有限公司-前端工程师]]
