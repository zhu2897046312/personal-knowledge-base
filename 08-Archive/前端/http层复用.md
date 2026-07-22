# HTTP 层（Axios 统一封装模板）

> 适用：React / Vue / Nuxt + TanStack Query 项目
> 目标：统一处理 Token、401 刷新、请求重放、回跳地址、并发刷新等问题。

---

# 一、职责

HTTP 层只负责：

* 创建统一 axios 实例
* 自动携带 Access Token
* 自动设置 Header
* 处理 FormData
* 处理 401 Token 过期
* 自动刷新 Token
* 失败后清理登录态
* 保存登录前页面地址
* 重放原请求（Retry Original Request）

不负责：

* 缓存
* 请求去重
* loading/error 状态
* invalidateQueries
* 服务端状态管理

这些由 TanStack Query 负责。

---

# 二、推荐目录结构

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

---

# 三、必须替换的字段（新项目只改这里）

```ts
// 登录接口
const LOGIN_URL = "/api/login";

// 刷新 token 接口
const REFRESH_URL = "/api/user/refresh";

// access token 字段
const ACCESS_TOKEN_FIELD = "access_token";

// expires 字段（秒）
const EXPIRES_FIELD = "expires_in";
```

后端字段不同，只改这里即可。

---

# 四、核心流程

## 请求阶段

```text
请求
  ↓
读取本地 token
  ↓
Authorization: Bearer xxx
  ↓
发送请求
```

---

## 401 自动刷新流程

```text
接口请求
  ↓
401
  ↓
调用 Refresh API
  ↓
拿到新 Token
  ↓
更新本地 Token
  ↓
重新发送原请求
```

页面无感知。

---

# 五、Single Flight（必须保留）

多个接口同时返回 401 时：

错误做法：

```text
A → refresh
B → refresh
C → refresh
```

会发送多次刷新请求。

正确做法：

```text
A → refresh（唯一一次）
B → 等待同一个 Promise
C → 等待同一个 Promise
```

实现方式：

```ts
let refreshPromise: Promise<string> | null = null;
```

整个项目始终只有一个刷新请求。

---

# 六、为什么要排除认证接口

登录和刷新接口本身不能再次触发刷新：

```text
/login
/refresh
```

否则会出现：

```text
refresh
  ↓
401
  ↓
refresh
  ↓
401
  ↓
无限循环
```

因此需要：

```ts
isAuthEndpoint(url)
```

进行排除。

---

# 七、FormData 处理规则

普通 JSON：

```http
Content-Type: application/json
```

上传文件：

```ts
const form = new FormData();
```

不要手动设置 `Content-Type`，浏览器会自动生成 boundary。

因此：

```ts
if (config.data instanceof FormData) {
  delete config.headers["Content-Type"];
}
```

---

# 八、401 最终失败处理

当：

* refresh 接口失败
* refresh 返回无 token
* refresh 已失效

执行：

```text
clearAccessToken()
saveRedirectPath(currentPath)
reject(error)
```

然后由 QueryProvider 的全局 401 逻辑统一跳转登录页。

这样 HTTP 层与路由层解耦。

---

# 九、与 TanStack Query 的配合

```text
页面
  ↓
useQuery / useMutation
  ↓
api/user.ts
  ↓
http.ts
  ↓
后端
```

职责划分：

| 层        | 职责       |
| -------- | -------- |
| http.ts  | HTTP 通信  |
| api/*.ts | 接口定义     |
| Query    | 读数据、缓存   |
| Mutation | 写数据、刷新缓存 |
| 页面       | 业务逻辑     |

---

# 十、最小可复用模板

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

---

# 十一、复用检查清单

新项目只检查：

* [ ] 登录接口路径
* [ ] 刷新接口路径
* [ ] access_token 字段名
* [ ] expires_in 字段名
* [ ] Token 存储方式
* [ ] 登录页路由
* [ ] baseURL

其余逻辑通常可以直接复用。

---

# 十二、记忆口诀

> **HTTP 层负责：发请求、带 Token、刷 Token、重放请求。**
> **TanStack Query 负责：缓存、状态、重试、刷新、共享数据。**
> **页面只负责业务逻辑。**

# TY示例
```js
import axios from "axios";
import { saveRedirectPath, getCurrentFullPath } from "./redirectUtils";
import { getAccessToken, setAccessToken, clearAccessToken } from "./authToken";

// 认证接口：命中这些路径的请求不触发「401 自动刷新」，避免登录/刷新自身失败时递归刷新。
const AUTH_ENDPOINTS = ["/api-org/v1/login", "/api-org/v1/user/refresh"];
const isAuthEndpoint = (url = "") => AUTH_ENDPOINTS.some((path) => url.includes(path));

export const isRemoteAccess = () => {
    const hostname = window.location.hostname;

    // 本地地址列表
    const localAddresses = ["localhost", "127.0.0.1", "::1", "0.0.0.0"];

    // 检查是否为本地地址
    const isLocal = localAddresses.includes(hostname);

    // 检查是否为局域网地址
    const isLAN = /^(10\.|172\.(1[6-9]|2[0-9]|3[01])\.|192\.168\.)/.test(hostname);

    // 如果不是本地地址也不是局域网地址，则认为是远程访问
    return !isLocal && !isLAN;
};

export const createHttpClient = (baseURL = import.meta.env.PUBLIC_API_MAIN_TARGET) => {
    const instance = axios.create({
        baseURL,
        timeout: 500000, // 30 seconds
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
        (error) => {
            return Promise.reject(error);
        },
    );

    // 刷新 token 的单飞（single-flight）Promise：并发 401 只发起一次 /api-org/v1/user/refresh，
    // 其余请求复用同一次刷新结果，避免刷新风暴。
    let refreshPromise = null;
    const refreshAccessToken = () => {
        if (!refreshPromise) {
            refreshPromise = instance
                .post("/api-org/v1/user/refresh")
                .then((res) => {
                    const token = res.data?.data?.access_token;
                    if (!token) throw new Error("refresh token: 响应缺少 access_token");
                    // expires_in 单位为秒，交由 authToken 统一换算
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

            // 401 且非认证接口本身：先尝试用 /api-org/v1/user/refresh 刷新 token 再重放原请求（每个请求只重试一次）。
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
                    // 刷新失败：丢弃 token 并记录回跳路径，reject 原始 401。
                    // 跳登录由 QueryProvider 的 queryCache/mutationCache 全局 onError（收到 401 即 navigate("/login")）兜底，
                    // 覆盖所有经 react-query 的请求；此处不自行导航，避免与全局逻辑重复。
                    clearAccessToken();
                    saveRedirectPath(getCurrentFullPath());
                    return Promise.reject(error);
                }
            }

            // 其余 401（刷新接口失败 / 无 token）：记录当前 URI 供登录后回跳
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