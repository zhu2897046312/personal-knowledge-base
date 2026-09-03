---
title: Snapchat 广告投放后端服务「svr_ads」
tags: [go, backend, resume, snapchat, asynq]
created: 2026-09-03
updated: 2026-09-03
aliases: [svr_ads, ad-tools-api]
summary: 独立设计并实现广告投放后端服务「svr_ads」的 Snapchat 渠道模块，覆盖 SDK 封装、异步发布引擎、OAuth 授权与资产同步
type: resume-project
---

# 概述

**Snapchat 广告投放后端服务「svr_ads」**（2026.08 - 至今 · 核心成员，独立负责后端从 0 到 1 开发）— 自研多渠道广告投放后端服务中独立承接的 Snapchat 渠道模块，覆盖第三方 API SDK 封装、异步发布引擎、OAuth 授权与资产同步。技术栈：Go · Gin · GORM · PostgreSQL · Redis · asynq

# 核心贡献

- 独立设计并实现 Snapchat 渠道 SDK，覆盖 Marketing/Business/Ad Library 三大 API 组，统一封装限流降级、语义化错误分类、响应信封归一化与 OAuth 分布式并发刷新
- 基于 asynq 设计六阶段异步发布引擎（campaign→ad_squad→ad），通过幂等落库与 Redis 阶段推进锁解决消息重投导致的数据错乱，并实现僵死任务双阈值自愈扫描
- 落地跨账户对象复制、发布模板复用同一套校验、生活方式受众多国交集树等定向能力
- 建立 85 个单元测试（覆盖率约 45%）与脱库脱网 mock 体系，保障发布链路可回归

# 核心成果

- 3 周内产出 **72 个**提交，净变更约 **+53,093 / -3,980** 行代码，新增 **9 个**数据库迁移
- SDK 覆盖 Marketing API **11 个**业务子服务 + Business/Ad Library 两个独立 API 组
- 188 个 Go 文件中测试文件占比约 **45%**

# 相关链接

- [[resume-天呓网络科技有限公司-前端工程师]]
- [[resume-天呓ERP-广告资产与运营协同管理平台]]
