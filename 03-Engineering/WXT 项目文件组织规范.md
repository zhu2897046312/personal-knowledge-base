---
title: WXT 项目文件组织规范
tags: [wxt, vue, project-structure, coding-standards]
created: 2026-08-26
updated: 2026-08-26
aliases: [constants types utils components 归类, WXT 浏览器扩展目录结构]
summary: WXT 浏览器扩展项目按用途拆到 constants/types/utils/components 四个顶层目录，每个目录只放一类内容，用"是否被 ≥2 处复用"作为总判断标准
type: cheatsheet
---

# 概述

WXT 浏览器扩展项目按用途把代码拆到 `constants/`、`types/`、`utils/`、`components/` 四个顶层目录，每个文件只负责一类内容。核心判断标准：这段代码会不会被 ≥2 处复用，或者是不是一段可以脱离调用它的入口单独理解、单独测试的逻辑；只用一次、和入口强耦合的胶水代码不必为了"拆分"而拆分。

# 要点

- `constants/`：只放运行时用到的常量值，按领域拆文件，不建塞满所有常量的 `index.ts`
- 由常量对象派生的联合类型和常量对象放同一个文件，不拆到 `types/`（详见 [[TypeScript 表达式复杂度与常量枚举规范]]）
- `types/`：只放独立定义、不是从某个 `as const` 常量对象派生出来的类型/接口
- `utils/`：只放不含 Vue 响应式 API 的纯函数/带副作用函数，按"纯函数 / 带副作用"拆文件
- `components/`：只放会被 ≥2 处引用、或从入口模板里提取出的独立单元；只用一次的强耦合片段允许留在入口自己的 `.vue` 文件里
- 入口文件（`entrypoints/*`）只做编排，不在入口文件里定义常量/类型/可复用逻辑/可复用组件

# 用法

## 目录职责与归类规则

- **`constants/`**：`as const` 常量对象、固定字符串/数字、URL 匹配规则、正则等"值不会因为组件渲染或用户交互而变化"的东西。按领域拆文件（如 `constants/reporting.ts`、`constants/polling.ts`）。

- **`types/`**：领域实体的形状（如 `AdAccountId`）、函数入参/返回值的复合类型、跨多个文件共用的 Props 类型。归类标准：这个类型是不是从某个常量对象派生出来的；是的话归 `constants/`，不是的话归 `types/`。

- **`utils/`**：必须是"给定输入、做一件事"的独立逻辑，函数内部不出现 `ref`/`reactive`/`computed`/生命周期钩子。不发请求、不读写 storage、不打日志的纯函数放一类文件（如 `reportingUrl.ts` 只管 URL 拼接/解析）；会发网络请求、读写 storage/浏览器 API、打印日志的带副作用函数放另一类文件（如 `reportingFetch.ts`、`adAccounts.ts`），不混在同一文件。

- **`components/`**：只放可复用的 Vue 单文件组件。判断标准：会被 ≥2 个地方引用，或者是把某个入口模板里超过 3 行的片段提取出来的独立单元（呼应 [[TypeScript 表达式复杂度与常量枚举规范]] 的模板复杂度约束）。组件内部需要响应式状态、`onMounted` 等生命周期钩子时都写在这里——这是和 `utils/` 的核心区别：`utils/` 不允许出现 Vue 响应式 API，需要响应式状态的逻辑要么是组件本身，要么留在调用它的入口/组件里，不下沉到 `utils/`。

- **入口文件**（`entrypoints/*.ts`、`entrypoints/*/App.vue` 等）只做编排：注册事件监听、组装 `components/` 拼出页面、调用 `constants`/`types`/`utils` 里的定义。只属于这个入口自身生命周期的局部可变状态（比如后台脚本里的去重标志位、上一次抓到的模板 URL）允许留在入口文件，不必下沉。

## 目录结构示例

```
constants/
  reporting.ts   # 报表接口相关常量（URL 匹配规则、核心业务参数字段名）
  polling.ts     # 定时轮询相关常量（alarm 名称、轮询周期）
  storage.ts     # storage key、默认值
types/
  adAccount.ts   # 独立定义的领域类型（不是常量对象派生出来的）
utils/
  reportingUrl.ts    # 纯函数：URL 拼接/解析
  reportingFetch.ts  # 带副作用：发请求 + 打日志
  adAccounts.ts      # 带副作用：账户列表的 storage 读写
components/
  # 当前项目还没有需要复用/需要拆分的模板片段，先留空；出现符合上面标准的片段时再建
entrypoints/
  background.ts      # 只做编排：注册 webRequest/alarms 监听，调用上面这些定义
  popup/App.vue       # 账户列表编辑界面（目前逻辑简单，还没有需要拆出去的子组件）
```

# 相关链接

- [[TypeScript 表达式复杂度与常量枚举规范]]
- [[注释 JSDoc 与 Git 提交个人规范]]
