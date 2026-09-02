---
title: 基于 Gin 与 Nuxt 的跨境电商独立站
tags: [go, gin, gorm, vue3, nuxt4, nuxt-ui, mysql, redis, paypal, ssr, 跨境电商]
created: 2026-09-02
updated: 2026-09-02
aliases: [跨境电商独立站, zhuyi-store, 毕业设计电商项目]
summary: 独立设计并实现的跨境电商独立站毕业设计，Go+Gin+GORM 后端、Vue3+AntDV 管理后台、Nuxt4+Nuxt UI SSR 前台商城，打通选品、下单、PayPal 跨境支付与退款全链路
type: resume-project
---

# 概述

**基于 Gin 与 Nuxt 的跨境电商独立站**（2025.08 - 2026.05 · 个人独立设计与开发，毕业设计）— 前后端分离的跨境电商系统，打通「商品上架 → SSR 前台展示 → 加购下单 → PayPal 跨境支付 → 订单履约与退款」全链路，支持游客免注册购物。技术栈：Go 1.24 · Gin · GORM · MySQL · Redis · Vue 3 · Ant Design Vue · Nuxt 4 · Nuxt UI 4 · Docker

# 核心贡献

- 独立设计 Handler-Service-Repository 三层架构与工厂模式依赖注入，实现管理端 Redis Session、前台 JWT + 设备指纹的双认证体系
- 设计游客购物车 + 设备指纹 + 查询码方案，支撑未注册用户完成加购、支付、查单全流程；对接 PayPal 完成创建订单、捕获支付、累计退款金额校验的完整支付链路
- 独立排查并修复多对多关联表主键/外键混用导致的标签查询失败、UI 组件库升级后 v-model 契约不匹配导致的分页失效等生产级 bug
- 主导前台 UI 技术栈从 Naive UI 全量迁移至 Nuxt UI 4 + Tailwind CSS 4，并系统性清理 AI 辅助生成但业务未启用的冗余 repository/handler 代码

# 核心成果

- 后端交付 **165** 个 Go 源文件、**107** 处 RESTful 接口、**47** 个数据模型，覆盖 **30+** 张业务表
- 管理后台交付 **98** 个页面级组件，覆盖商品、订单、CMS、RBAC 权限等 **10+** 业务模块
- 前台 SSR 商城交付 **10** 个核心页面，实现容器化部署下浏览器与 SSR 双 BaseURL 隔离
- 定位并修复标签查询、分页失效等 **2** 处具体生产级 bug（均有对应 diff 可追溯），完成 **1** 次前台 UI 框架整体迁移

# 相关链接

- [[resume-广东真格软件有限公司-初级前端工程师（实习）]]
