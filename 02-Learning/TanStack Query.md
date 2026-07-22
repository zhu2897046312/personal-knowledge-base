---
title: TanStack Query
tags: [frontend, react, tanstack, query]
created: 2026-07-22
updated: 2026-07-22
aliases: [React Query, 服务端状态]
summary: TanStack Query 管服务端状态（缓存、重试、失效），不负责发 HTTP
type: learning
---

# 目标

理解 TanStack Query 与 axios/http 层的职责划分，掌握 Query、Mutation 与 Cache Invalidate。

# 知识点

**定位：** 服务端状态（Server State）管理库。

**一句话：** 不负责发送 HTTP（axios/http.ts 负责），负责请求后的缓存、状态、刷新、重试等。

## 整体架构

```text
React 页面
      │
      ▼
TanStack Query（管理服务端状态）
      │
      ▼
http.ts（axios，请求封装）
      │
      ▼
后端 API
```

- **http.ts：** BaseURL、Token、拦截器、Timeout、Header
- **TanStack Query：** 请求状态、缓存、同步、刷新策略

## Query（读，GET）

- 请求缓存、去重、多组件共享缓存
- 自动维护 `data` / `error` / `loading` / `pending`
- 失败重试、后台 Refetch、Cache Invalidate
- 分页 / 无限滚动（`useInfiniteQuery`）

## Mutation（写）

生命周期：`onMutate` → 发请求 → `onSuccess` / `onError` → `onSettled`

- `onMutate`：乐观更新
- `onSuccess`：常配合 `invalidateQueries()`
- `onError`：回滚、提示
- `onSettled`：类似 `finally`

## Cache Invalidate

告诉 Query：「这份缓存不可信，请重新拉取」。例如删除订单后：

```ts
queryClient.invalidateQueries({
  queryKey: ["orders"],
});
```

只刷新指定 Query，不是整页刷新。

## 与 Nuxt `useFetch` 对比

TanStack Query 在缓存、去重、共享、重试、Invalidate、Mutation 生命周期上更完整，可理解为 `useFetch` 的企业增强版（专注服务端状态）。

# 示例

删除后失效订单列表缓存：

```ts
queryClient.invalidateQueries({
  queryKey: ["orders"],
});
```

# 总结

> Query 管读，Mutation 管写。  
> Axios 管发送请求，TanStack Query 管服务端状态。

# 相关链接

- [[HTTP 层 Axios 封装]]
- [[前端性能优化实践]]
- [[前端底座与性能优化速查]]
