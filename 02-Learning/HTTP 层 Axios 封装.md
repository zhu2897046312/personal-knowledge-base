---
title: HTTP 层 Axios 封装
tags: [frontend, axios, http, auth]
created: 2026-07-22
updated: 2026-07-22
aliases: [http.ts, Token 刷新, Single Flight]
summary: Axios 统一封装：带 Token、401 单飞刷新、重放原请求，与 TanStack Query 分工
type: learning
---

# 目标

在 React / Vue / Nuxt + TanStack Query 项目中复用 HTTP 层：统一 Token、401 刷新、请求重放、回跳地址、并发刷新。

# 知识点

## 职责

HTTP 层只负责：统一 axios 实例、带 Access Token、Header、FormData、401 刷新、失败清登录态、保存登录前回跳、重放原请求。

不负责：缓存、去重、loading/error、invalidateQueries、服务端状态（归 TanStack Query）。

## 推荐目录

```text
src/
├── api/
│   └── user.ts
├── utils/
│   ├── http.ts
│   ├── authToken.ts
│   └── redirectUtils.ts
└── providers/
    └── QueryProvider.tsx
```

## 新项目需替换的字段

```ts
const LOGIN_URL = "/api/login";
const REFRESH_URL = "/api/user/refresh";
const ACCESS_TOKEN_FIELD = "access_token";
const EXPIRES_FIELD = "expires_in";
```

## 核心流程

请求：读本地 token → `Authorization: Bearer xxx` → 发送。

401：调 Refresh → 更新 Token → 重发原请求（页面无感知）。

## Single Flight（必须）

并发 401 时只发起一次 refresh，其余等待同一 Promise：

```ts
let refreshPromise: Promise<string> | null = null;
```

## 排除认证接口

`/login`、`/refresh` 不能再触发刷新，否则无限循环。用 `isAuthEndpoint(url)` 排除。

## FormData

上传时不要手动设 `Content-Type`，让浏览器带 boundary：

```ts
if (config.data instanceof FormData) {
  delete config.headers["Content-Type"];
}
```

## 401 最终失败

refresh 失败 / 无 token / 已失效 → `clearAccessToken()` + `saveRedirectPath` + reject；跳登录由 QueryProvider 全局 401 处理，HTTP 层不直接导航。

## 与 TanStack Query 配合

| 层 | 职责 |
|----|------|
| http.ts | HTTP 通信 |
| api/*.ts | 接口定义 |
| Query | 读数据、缓存 |
| Mutation | 写数据、刷新缓存 |
| 页面 | 业务逻辑 |

## 复用检查清单

- [ ] 登录 / 刷新接口路径
- [ ] access_token / expires_in 字段名
- [ ] Token 存储方式
- [ ] 登录页路由
- [ ] baseURL

# 示例

## 最小可复用模板

```ts
import axios from "axios";

const instance = axios.create({
  baseURL: import.meta.env.PUBLIC_API_MAIN_TARGET,
  timeout: 30000,
});

instance.interceptors.request.use((config) => {
  const token = getAccessToken();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

let refreshPromise = null;

const refreshAccessToken = async () => {
  if (!refreshPromise) {
    refreshPromise = instance
      .post(REFRESH_URL)
      .then((res) => {
        const token = res.data.data[ACCESS_TOKEN_FIELD];
        setAccessToken(token, res.data.data[EXPIRES_FIELD]);
        return token;
      })
      .finally(() => {
        refreshPromise = null;
      });
  }
  return refreshPromise;
};

instance.interceptors.response.use(
  (res) => res,
  async (error) => {
    const original = error.config;
    if (
      error.response?.status === 401 &&
      original &&
      !original._retried &&
      !isAuthEndpoint(original.url)
    ) {
      original._retried = true;
      try {
        const token = await refreshAccessToken();
        original.headers.Authorization = `Bearer ${token}`;
        return instance(original);
      } catch {
        clearAccessToken();
      }
    }
    return Promise.reject(error);
  },
);

export default instance;
```

## TY 示例（完整 createHttpClient）

```js
import axios from "axios";
import { saveRedirectPath, getCurrentFullPath } from "./redirectUtils";
import { getAccessToken, setAccessToken, clearAccessToken } from "./authToken";

// 认证接口：命中这些路径的请求不触发「401 自动刷新」，避免登录/刷新自身失败时递归刷新。
const AUTH_ENDPOINTS = ["/api-org/v1/login", "/api-org/v1/user/refresh"];
const isAuthEndpoint = (url = "") => AUTH_ENDPOINTS.some((path) => url.includes(path));

export const isRemoteAccess = () => {
    const hostname = window.location.hostname;
    const localAddresses = ["localhost", "127.0.0.1", "::1", "0.0.0.0"];
    const isLocal = localAddresses.includes(hostname);
    const isLAN = /^(10\.|172\.(1[6-9]|2[0-9]|3[01])\.|192\.168\.)/.test(hostname);
    return !isLocal && !isLAN;
};

export const createHttpClient = (baseURL = import.meta.env.PUBLIC_API_MAIN_TARGET) => {
    const instance = axios.create({
        baseURL,
        timeout: 500000,
        headers: {
            "X-Requested-With": "XMLHttpRequest",
            Accept: "application/json",
        },
    });

    instance.interceptors.request.use(
        async (config) => {
            const token = getAccessToken();
            const isFormDataPayload =
                typeof FormData !== "undefined" && config.data instanceof FormData;

            if (isFormDataPayload) {
                if (typeof config.headers?.delete === "function") {
                    config.headers.delete("Content-Type");
                } else if (config.headers) {
                    delete config.headers["Content-Type"];
                    delete config.headers["content-type"];
                }
            } else if (config.headers) {
                const hasExplicitContentType =
                    typeof config.headers.get === "function"
                        ? Boolean(config.headers.get("Content-Type"))
                        : Boolean(config.headers["Content-Type"] ?? config.headers["content-type"]);
                if (!hasExplicitContentType) {
                    if (typeof config.headers.set === "function") {
                        config.headers.set("Content-Type", "application/json");
                    } else {
                        config.headers["Content-Type"] = "application/json";
                    }
                }
            }

            if (token) {
                config.headers.Authorization = `Bearer ${token}`;
            }
            return config;
        },
        (error) => Promise.reject(error),
    );

    let refreshPromise = null;
    const refreshAccessToken = () => {
        if (!refreshPromise) {
            refreshPromise = instance
                .post("/api-org/v1/user/refresh")
                .then((res) => {
                    const token = res.data?.data?.access_token;
                    if (!token) throw new Error("refresh token: 响应缺少 access_token");
                    setAccessToken(token, res.data?.data?.expires_in);
                    return token;
                })
                .finally(() => {
                    refreshPromise = null;
                });
        }
        return refreshPromise;
    };

    instance.interceptors.response.use(
        (response) => response,
        async (error) => {
            const original = error.config;
            const status = error.response?.status;

            if (
                status === 401 &&
                original &&
                !original._retried &&
                !isAuthEndpoint(original.url) &&
                getAccessToken()
            ) {
                original._retried = true;
                try {
                    const newToken = await refreshAccessToken();
                    original.headers = original.headers || {};
                    original.headers.Authorization = `Bearer ${newToken}`;
                    return instance(original);
                } catch {
                    clearAccessToken();
                    saveRedirectPath(getCurrentFullPath());
                    return Promise.reject(error);
                }
            }

            if (status === 401) {
                saveRedirectPath(getCurrentFullPath());
            }
            return Promise.reject(error);
        },
    );

    return instance;
};

const http = createHttpClient();
export default http;
```

# 总结

> HTTP 层：发请求、带 Token、刷 Token、重放请求。  
> TanStack Query：缓存、状态、重试、刷新、共享数据。  
> 页面只负责业务逻辑。

# 相关链接

- [[TanStack Query]]
- [[前端性能优化实践]]
- [[前端底座与性能优化速查]]
