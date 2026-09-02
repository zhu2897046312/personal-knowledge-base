---
title: INCAM 企业官网与电商站 · Nuxt 4 + Shopify Storefront API
tags: [nuxt4, vue3, typescript, shopify, graphql, i18n, seo, performance]
created: 2026-09-02
updated: 2026-09-02
aliases: [INCAM, INCAM Systems 官网, incam-nuxt-ui]
summary: 基于 Nuxt 4 + Shopify Storefront API 的 CMS 驱动型企业电商官网，独立负责从架构设计到上线交付的全流程，涵盖无后端数据层稳定性、CMS 驱动三级导航、多语言 SEO 与图片性能优化
type: blog-project
---

# 概述

**INCAM Systems 企业官网 / 电商站**（2025.12 - 2026.03 · 个人独立负责前端架构与核心功能开发，已上线运行）

一套面向 B2B/B2C 场景的企业级电商官网：商品、页面、博客等内容全部来自 **Shopify 后台（CMS）**，前端不自建业务后端，通过 **Storefront API（GraphQL）** 拉取数据，交付给非技术运营人员可自主维护、且对 SEO 与首屏性能友好的官网。

# 技术栈

**框架**：Nuxt 4（SSR/SSG）、Vue 3（Composition API）、TypeScript
**数据层**：Shopify Storefront API（GraphQL）、`@shopify/storefront-api-client`
**UI/样式**：Nuxt UI、Tailwind CSS 4、Font Awesome
**国际化**：`@nuxtjs/i18n`（en / zh-CN，`prefix_except_default` 策略）
**其他**：`@emailjs/browser`（前端发信）、Resend（服务端定时/批量发信）、飞书 Open API（表格数据联动）、`@nuxt/image`

# 核心工作与技术亮点

1. **无自建后端下的 Shopify 数据层稳定性**
   - 封装单例 API 客户端 `shopify.server.ts`：相同 `query + variables` 的请求复用同一个 Promise（请求去重），并发请求数硬限制在 2（请求队列），对网络错误 / 5xx / 429 做指数退避 + 随机抖动重试，避免瞬时并发触发 Shopify 限流。
   - 页面层用 `useAsyncData` 做服务端取数与缓存，布局层统一用 `usePagesShared`、`useBlogsShared` 等共享 key，避免同一份导航/博客列表被多个组件重复请求。

2. **CMS 驱动的三级导航（无需改代码即可调整层级）**
   - 利用 Shopify Page 的 Metafield（`parent_page` 父页面引用、`nav_order` 排序、`nav_label` 自定义标签）在后台维护导航结构。
   - 前端一次遍历建 `Map`、二次遍历挂父子关系，将扁平页面列表转为树结构并按 `nav_order` 排序，供 Mega Menu 与普通下拉菜单统一消费，运营改层级无需前端发版。

3. **中英文 i18n 与多语言 SEO 对齐**
   - `@nuxtjs/i18n` 采用 `prefix_except_default`（英文无前缀，中文 `/zh-CN/...`），配合浏览器语言检测与 Cookie 持久化。
   - Shopify 侧用 Translate & Adapt 维护多语言内容，GraphQL 查询统一带 `@inContext(language: $language)`，保证前端路由语言与 CMS 内容语言一致，`canonical`/`og:locale` 随语言联动生成绝对 URL。

4. **Contact 表单：前端直接发信，配置全部下放到 CMS**
   - 联系表单用 EmailJS 在浏览器端直接完成发信，无需为单一表单场景自建邮件服务；`serviceId`/`templateId`/`publicKey`/收件人等配置存在 Shopify Metafield，运营可自行调整。
   - 定时、批量发信场景（如飞书表格记录批量通知）改走 Nuxt server API + Resend，服务端发送更可靠，并支持 `/admin/feishu-records` 页面可视化管理与单条/批量发送。

5. **图片与静态资源性能优化（WebP 响应式改造）**
   - Shopify CMS 富文本以 `v-html` 注入 HTML，图片尺寸不可控。通过正则拦截注入 HTML 中的 `<img>` 与内联 `background-image`，自动追加 `?width=&format=webp` 并生成响应式 `srcset`（375/750/1080/1440/1920 五档），背景图则转写为 CSS 变量 + `image-set()` 按 DPR 自动选 1x/2x 图，全程不破坏 Tailwind 的 `w-full`/`aspect-*` 响应式布局。
   - 首屏图追加 `fetchpriority="high"`，非首屏统一 `loading="lazy"`；视频用 `preload="metadata"` 降低首屏带宽占用。

6. **构建产物与关键请求延迟治理**
   - 排查生产构建中 `@nuxt/ui` 按需引入与 Vite 默认分包策略冲突导致的请求碎片化问题（大量 1-5KB 微文件 + 600KB+ 单一入口并存），通过 `vite:extendConfig` Hook 自定义 `manualChunks` 合并第三方依赖与 UI 框架产物。
   - 将 i18n 语言包与 favicon 从独立异步 Fetch（约 250ms 延迟）改为内联/Base64，减少首屏关键请求数。

# 量化成果

- CMS 图片资源体积从 **约 5MB 降至 3.3MB**，生产部署（Vercel）下传输体积降至 **约 1.2MB**，页面完成加载时间从 **约 4s 降至 2s**；
- SEO 完成度评估从约 **60% 提升至 85%**：全站页面（首页、产品详情、文章、动态页、标签页等）均补齐 `title`/`description`/Open Graph/Twitter Card/canonical；
- Shopify 请求并发控制在 **最多 2 个**，配合去重与指数退避重试，显著降低高并发场景下的限流失败率；
- 项目累计 **180+ 次提交**，独立完成需求分析、架构设计、开发、SEO/性能优化到上线的全流程交付。

# 项目收获

在"前端无自建后端、内容与商品全部来自第三方 CMS"的约束下，建立了一套可复用的分层思路：页面只声明要什么数据，Composables 封装与 Shopify 的交互细节，API 客户端统一承担去重/并发/重试等稳定性责任。同时在真实生产环境里完整走过一轮"发现问题（资源碎片化、图片过大、关键请求延迟）→定位成因（构建策略、CDN 参数、CMS 富文本不可控）→设计并落地方案→用线上指标验证效果"的性能优化闭环，积累了 SSR/SSG 选型判断、多语言 SEO 落地与前端性能排查的可迁移经验。

# 相关链接

- [[blog-智为深视（深圳）科技有限公司-前端工程师]]
