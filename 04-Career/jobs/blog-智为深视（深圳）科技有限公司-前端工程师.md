---
title: 智为深视（深圳）科技有限公司 · 前端工程师
tags: [job, frontend, nuxt4, shopify]
created: 2026-09-02
updated: 2026-09-02
aliases: [智为深视, INCAM 前端]
summary: 独立负责 INCAM Systems 企业官网/电商站从架构设计到上线的完整交付，基于 Nuxt 4 + Shopify Storefront API 构建 CMS 驱动、无自建业务后端的电商官网
type: blog-job
---

# 概述

**智为深视（深圳）科技有限公司 · 前端工程师**（2025.12 - 2026.03，约，请核实结束月）

独立负责 INCAM Systems 企业官网/电商站从架构设计到上线的完整交付。

# 背景

源笔记（`D:\workspace\blog`）中这段任职经历本身未单独撰写背景段落，本文内容整理自关联项目 [[blog-INCAM 企业官网与电商站]] 与已有简历文档的交叉记录；任职区间标注为约数，对外使用前需本人核实准确的入职/离职月份。

# 主要工作内容与成果

独立负责 INCAM Systems 企业官网/电商站（详见 [[blog-INCAM 企业官网与电商站]]）从架构设计到上线的完整交付：

- 基于 Nuxt 4 + Shopify Storefront API（GraphQL）构建 CMS 驱动、无自建业务后端的企业电商官网
- 封装 Shopify GraphQL 客户端，实现请求去重（相同 query 复用同一个 Promise）、并发限流（2）与指数退避重试，提升无后端架构下的数据稳定性
- 基于 Shopify Page Metafield 落地 CMS 驱动的三级导航（扁平页面转树结构 + 排序），运营可自主调整层级，无需前端发版
- 落地中英文 i18n 与多语言 SEO 对齐，`canonical`/`og:locale` 随语言联动生成绝对 URL
- 设计 CMS 富文本图片/背景图的响应式 WebP 改造方案，并治理生产构建的资源碎片化与关键请求延迟问题

# 量化成果

- CMS 图片资源从 5MB 降至 3.3MB，生产部署传输体积降至约 1.2MB，页面完成加载时间从 4s 降至 2s
- SEO 完成度评估从约 60% 提升至 85%
- 项目累计 180+ 次提交，独立完成需求分析、架构设计、开发、SEO/性能优化到上线的全流程交付

# 技术与能力提升

建立了"前端无自建后端、内容与商品全部来自第三方 CMS"约束下的可复用分层思路，积累了 SSR/SSG 选型判断、多语言 SEO 落地与前端性能排查的可迁移经验（详见 [[blog-INCAM 企业官网与电商站]]）。

# 相关链接

- [[blog-INCAM 企业官网与电商站]]
