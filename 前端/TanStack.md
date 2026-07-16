# TanStack Query（React Query）

> **定位：服务端状态（Server State）管理库。**
>
> **一句话理解：**
> 不负责发送 HTTP 请求（axios/http.ts 负责），负责管理请求后的所有事情（缓存、状态、刷新、重试等）。

---

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

职责划分：

- **http.ts（axios）**
  - 发起 HTTP 请求
  - BaseURL
  - Token
  - 请求/响应拦截器
  - Timeout
  - Header 等

- **TanStack Query**
  - 管理请求状态
  - 管理缓存
  - 管理数据同步
  - 管理刷新策略

---

# Query（读数据，GET）

负责所有查询接口（GET）。

## 功能

- ✅ 请求缓存（Cache）
- ✅ 请求去重（多个组件同时请求同一接口，仅发送一次请求）
- ✅ 多组件共享数据（共享同一份缓存）
- ✅ 自动维护请求状态（`data`、`error`、`loading`、`pending`）
- ✅ 请求失败自动重试（Retry）
- ✅ 后台自动刷新（Refetch）
  - 例如：Chrome → 微信 → Chrome，可自动重新获取最新数据（可配置关闭）
- ✅ Cache Invalidate（缓存失效后自动重新获取）
- ✅ 分页 / 无限滚动（`useInfiniteQuery`，自动维护 `page` / `cursor` / `nextPage`）

---

# Mutation（写数据，POST / PUT / DELETE / PATCH）

负责所有修改接口。

## 生命周期

```text
onMutate
    ↓
发送请求
    ↓
成功？──────────────┐
 ↓                 ↓
onSuccess      onError
      \         /
       onSettled
```

## 常用回调

- `onMutate`
  - 请求发送前
  - 乐观更新（Optimistic Update）

- `onSuccess`
  - 请求成功
  - 通常执行 `invalidateQueries()`

- `onError`
  - 请求失败
  - 回滚乐观更新、错误提示

- `onSettled`
  - 无论成功或失败都会执行
  - 类似 `finally`

---

# Cache Invalidate（缓存失效）

**一句话理解：**

> 告诉 TanStack Query："这份缓存已经不可信了，请重新获取最新数据。"

例如：

```text
订单列表
    ↓
GET /orders
    ↓
缓存（orders）
```

执行删除订单：

```text
DELETE /orders/1
```

成功后：

```ts
queryClient.invalidateQueries({
  queryKey: ["orders"],
});
```

TanStack Query 会自动：

```text
orders 缓存
    ↓
标记为过期（Invalid）
    ↓
自动重新 GET /orders
    ↓
页面更新最新数据
```

> **不是刷新整个页面，而是仅刷新指定 Query。**

---

# 与 Nuxt `useFetch` 对比

| Nuxt `useFetch` | TanStack Query |
|-----------------|----------------|
| 获取数据 | ✅ |
| 自动维护 `data/error/loading` | ✅ |
| 请求缓存 | ⭐⭐⭐⭐⭐ |
| 请求去重 | ⭐⭐⭐⭐⭐ |
| 多组件共享缓存 | ⭐⭐⭐⭐⭐ |
| 自动重试 | ⭐⭐⭐⭐⭐ |
| Cache Invalidate | ⭐⭐⭐⭐⭐ |
| Mutation 生命周期 | ⭐⭐⭐⭐⭐ |

可以理解为：

> **TanStack Query ≈ `useFetch` 的企业增强版（专注于服务端状态管理）。**

---

# 核心总结（必记）

TanStack Query 不负责 **"怎么发请求"**，负责 **"请求完成后的所有事情"**。

## Query（读）

- 获取数据（GET）
- 缓存
- 请求去重
- 多组件共享数据
- 自动维护 `loading / error / data`
- 自动重试
- 后台刷新
- Cache Invalidate（缓存失效后重新获取）

## Mutation（写）

- POST / PUT / DELETE / PATCH
- Loading 状态
- `onMutate`
- `onSuccess`
- `onError`
- `onSettled`
- 乐观更新（Optimistic Update）
- 修改成功后通知 Query 刷新缓存（`invalidateQueries`）

---

# 最终记忆口诀 ⭐⭐⭐⭐⭐

> **Query 管读，Mutation 管写。**
>
> **Axios 管发送请求，TanStack Query 管服务端状态。**