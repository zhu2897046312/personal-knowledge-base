---
title: 跨境电商 Shopify 主题「Shrine Theme Pro」
tags: [shopify, liquid, web-components, javascript, css, dawn, ecommerce-theme, i18n, rtl]
created: 2026-09-03
updated: 2026-09-03
aliases: [Shrine Theme Pro, shrine-theme-pro, shopify-theme]
summary: 天呓网络科技跨境电商业务线的 Shopify 主题项目，基于官方 Dawn 参考主题二次开发，新增购物车转化模块（倒计时锁单/阶梯进度条/条件赠品/购物车内追加销售）、捆绑套装、商品 3D/AR 展示、26 种语言与 RTL 布局等能力
type: blog-project
---

# 概述

**跨境电商 Shopify 主题「Shrine Theme Pro」**（2026.08 - 至今 · 天呓网络科技工作项目，核心成员）

面向跨境独立站/DTC 商家的 Shopify Online Store 2.0 主题，在官方 **Dawn** 参考主题基础上二次开发，重点强化购物车侧转化能力（倒计时锁单、阶梯满减/包邮进度条、条件赠品、购物车内追加销售等 App Block）与商品详情展示能力（3D/AR 模型查看、图片放大镜），并覆盖 26 种语言与 RTL 布局能力。

# 背景

天呓网络科技从事跨境电商相关业务（同期还有 [[blog-天呓ERP-广告资产与运营协同管理平台]] 等项目），本主题服务于公司独立站业务线，用于自建/交付跨境电商网店。

需要如实说明数据来源的局限：仓库内 `config`/`sections`/`blocks` 等目录文件均创建于 2026-08-10，但 git 仓库本身在 2026-09-03 才建库、以单次 `init` 提交的形式提交了当前快照，此前的开发过程未纳入版本控制。因此本文无法像 [[blog-天呓ERP-广告资产与运营协同管理平台]] 那样引用逐笔提交记录与净变更行数，以下内容基于对当前代码状态的通读整理，量化数据均为代码规模/功能覆盖口径，而非提交历史口径。此外 `config/settings_schema.json` 中的 `theme_name`/`theme_author` 字段仍是 `"Custom Theme"`/`"Theme Developer"` 占位符，尚未做品牌化命名。

# 主要工作内容与成果

1. **购物车侧转化模块矩阵（`blocks/cart_*`，均为可拖拽 App Block）**
   - `cart_countdown-timer`：可自定义时长/文案的倒计时锁单提示组件（`<countdown-timer>` 自定义元素）；
   - `cart_progress-bar`：单一满减/包邮进度条，按小计金额或数量触发；
   - `cart_checkpoints-bar`（508 行）：多阶梯奖励进度条，支持在同一进度条上设置多个里程碑礼品/折扣；
   - `cart_gift`（615 行）：条件赠品逻辑，支持「全局 / 指定商品 / 购物车含某商品 / 不含某商品 / 小计阈值 / 指定页面」六种触发规则；
   - `cart_product-upsells`（909 行）：购物车内追加销售模块，同样支持六种条件规则，并可按语言/国家定向；
   - `cart_discount-field`：折扣码输入框，带成功/失败提示与自动应用；
   - `cart_tnc-checkbox`：条款勾选框，可配置为「未勾选则禁用结账按钮」。

2. **营销与展示型 Section/Block**
   - `sections/bundle-deals.liquid`：捆绑套装促销区块，按阶梯动态计算折扣百分比；
   - `sections/pricing-table.liquid`：定价方案对比表；
   - `blocks/timeline*`（7 个文件）：可横向/纵向切换的时间轴内容块，用于品牌故事/使用步骤类展示；
   - `templates/page.advertorial-1/2/3.json`、`page.contact.json`、`page.faq.json`、`page.track-order.json`：面向独立站营销转化场景的落地页模板系列。

3. **商品详情与购物体验组件**
   - `assets/product-model.js`（继承 Dawn 的 `DeferredMedia`）+ `component-product-model.css`/`component-model-viewer-ui.css`：Shopify 官方 model-viewer/XR 封装，实现商品 3D/AR 查看；
   - `assets/magnify.js` + `assets/product-image-zoom.js`（464 行）：两套图片放大镜/缩放方案；
   - `assets/quick-add.js`（继承 `ModalDialog`）：快速加购弹窗；
   - `assets/main-search.js`（继承 `SearchForm`）+ `component-predictive-search.css`：搜索建议（预测搜索）；
   - `assets/facets.js`（506 行，含 `FacetFiltersForm`/`PriceRange`/`FacetRemove` 三个自定义元素）：分面筛选；
   - `assets/pickup-availability.js`：门店取货可用性查询；
   - `templates/gift_card.liquid`：独立礼品卡兑换页，内嵌二维码生成（`vendor/qrcode.js`）。

4. **国际化与 RTL 布局能力**
   - 覆盖 26 种语言的 locale 资源包（52 个 `locales/*.json` + `*.schema.json` 文件）；
   - `assets/rtl.css` 提供 RTL 排版样式，`layout/theme.liquid` 通过 `settings.enable_rtl` 开关与可自定义的 `settings.rtl_enabled_languages` 语言列表控制是否对特定语言启用 RTL 布局（阿拉伯语/希伯来语等具体翻译内容尚未随资源包提供，RTL 是已内建的布局能力而非开箱即用的完整语言包）。

5. **架构基础**
   - 延续 Dawn 的原生 Web Components 模式（`customElements.define`，无框架依赖），`cart-notification`/`media-gallery`/`facet-filters-form` 等命名与继承结构均遵循 Dawn 约定，便于后续跟随 Dawn 上游更新；
   - CSS 采用「全局基础层（`base.css`）+ 按需组件层（9 个 `component-*.css`，与对应 JS 模块一一对应）」的组件化组织方式。

# 量化成果

- 全仓 **446** 个文件（不含 `.git`），其中 `blocks` **98** 个、`sections` **91** 个、`snippets` **131** 个、`templates` **18** 个；
- 购物车侧自定义转化 App Block 达 **7** 个，其中赠品/追加销售模块的条件规则引擎最多支持 **6** 种触发方式；
- 覆盖 **26** 种语言（**52** 个 locale/schema 文件），并内建 RTL 布局切换能力；
- 模板层覆盖 **18** 类页面（含 3 个独立营销落地页模板、独立礼品卡兑换页、完整 `customers/*` 账户体系）。

# 技术与能力提升

在 Dawn 官方参考架构之上做二次开发，实践了如何在不破坏上游可维护性的前提下扩展自定义功能：新增模块沿用原生 Web Components 与继承关系（`DeferredMedia`/`ModalDialog`/`SearchForm`）而非另起框架，降低跟随 Dawn 后续更新的成本。购物车侧赠品/追加销售模块的「六种触发规则」设计，锻炼了把营销运营诉求（满赠、满减、定向投放）抽象为可配置规则引擎（而非写死条件分支）的产品化思维，这类经验与 [[blog-天呓ERP-广告资产与运营协同管理平台]] 中把重复弹窗抽取为公共组件的判断力是相通的。同时也补上了 Shopify Theme Architecture（OS 2.0 的 section/block/schema 驱动的主题自定义器集成）与多语言、RTL 布局设计的实战经验。

# 相关链接

- [[blog-天呓网络科技有限公司-前端工程师]]
