---
title: INCAM 企业官网与电商站 · Nuxt 4 + Shopify Storefront API
tags: [nuxt4, vue3, typescript, shopify, graphql, i18n, seo, performance]
created: 2026-09-02
updated: 2026-09-02
aliases: [INCAM, INCAM Systems 官网, incam-nuxt-ui]
summary: 基于 Nuxt 4 + Shopify Storefront API 的 CMS 驱动型企业电商官网，独立负责从架构到上线的完整交付
type: resume-project
---

# 概述

**INCAM Systems 企业官网 / 电商站**（2025.12 - 2026.03 · 个人独立负责前端架构与核心功能开发，已上线运行）— 基于 Nuxt 4 + Shopify Storefront API 的企业级电商官网，内容与商品全部来自 Shopify CMS，无自建业务后端。技术栈：Nuxt 4、Vue 3、TypeScript、Tailwind CSS、@nuxtjs/i18n

# 核心贡献

- 封装 Shopify GraphQL 客户端，实现请求去重（相同 query 复用 Promise）、并发控制（限流 2）与指数退避重试，提升无后端架构下的数据稳定性
- 基于 Shopify Page Metafield 实现 CMS 驱动的三级导航（扁平数据转树 + 排序），运营可在后台自主调整层级，无需前端发版
- 落地中英文 i18n 与多语言 SEO 对齐（`@inContext` 按语言取 CMS 内容、canonical/og:locale 随语言联动），全站补齐动态 SEO Meta
- 设计 CMS 富文本图片/背景图的响应式 WebP 改造方案（正则改写 `<img>` 与 `background-image`，生成多档 `srcset` 与 `image-set`），并治理生产构建的资源碎片化与关键请求延迟问题

# 核心成果

- CMS 图片资源从 **5MB 降至 3.3MB**，生产部署传输体积降至 **约 1.2MB**，页面加载时间从 **4s 降至 2s**
- SEO 完成度从约 **60% 提升至 85%**，全站页面补齐 title/description/OG/Twitter Card/canonical
- 项目累计 **180+ 次提交**，独立完成从需求到上线的完整交付

# 相关链接

- [[resume-智为深视（深圳）科技有限公司-前端工程师]]
