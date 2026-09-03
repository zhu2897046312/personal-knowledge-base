---
title: Snapchat 广告投放后端服务「svr_ads」
tags: [go, gin, gorm, postgresql, redis, asynq, oauth2, backend, snapchat, third-party-integration]
created: 2026-09-03
updated: 2026-09-03
aliases: [svr_ads, ad-tools-api, Snapchat 渠道后端]
summary: 独立设计并实现自研广告投放后端服务「svr_ads」的 Snapchat 渠道模块，覆盖 SDK 封装、基于 asynq 的分阶段异步发布引擎、OAuth 授权与资产同步，是该渠道前后端全栈交付的后端部分
type: blog-project
---

# 概述

**Snapchat 广告投放后端服务「svr_ads」**（2026.08 - 至今 · 核心成员，独立负责 Snapchat 渠道后端从 0 到 1 设计与开发）

自研多渠道广告投放后端服务 svr_ads（Go + Gin + GORM + PostgreSQL + Redis + asynq）中，独立承接 Snapchat 渠道的完整后端建设：对接 Snapchat Marketing API / Business API / Ad Library API 三大 API 组的 SDK 封装、campaign→ad_squad→ad 三层结构的异步发布引擎、OAuth 授权与 token 刷新、资产同步与定向能力，与自己同步开发的前端发布模块（详见 [[blog-天呓ERP-广告资产与运营协同管理平台]]）共同构成该渠道的端到端全栈交付。

# 技术栈

**语言/框架**：Go 1.26 · Gin · GORM · PostgreSQL
**基础设施**：Redis（分布式锁 / 令牌桶限流 / singleflight 缓存）· asynq（异步任务队列）
**第三方集成**：Snapchat Marketing API / Business API / Ad Library API · OAuth 2.0
**工程实践**：httptest / 自定义 HTTPDoer mock 脱库脱网测试 · 手写 SQL 迁移

# 核心工作与技术亮点

1. **SDK 层：三大 API 组封装与响应归一化**
   - 覆盖 Marketing API 下 11 个业务子服务（广告账户 / 广告系列 / 广告集 / 广告 / 创意 / 素材 / 像素 / 身份 / 组织 / 受众定向等）、Business API（Public Profile 与共享策略）、Ad Library（竞品广告库检索，独立于已授权账户体系，内置出口代理池应对 429 与网络错误）三大 API 组。
   - 识别并修复 Snapchat 响应信封的隐蔽不一致：`request_status` 大小写不统一（多数端点大写，`/targeting/geo/*` 返回小写）、HTTP 200 下子请求级失败（`sub_request_status=ERROR`）、少数端点无子请求包装等情况，统一在 `envelope.go` 内用 `isSuccessStatus`/`UnwrapList`/`UnwrapListLenient`/`UnwrapObject` 分场景处理，避免"解出 0 条但不报错"的隐蔽故障。
   - 按 SDK 规范定义 `ErrRateLimited`/`ErrPlatformAPI`/`ErrTransport`/`ErrRefreshRejected` 四类语义错误，上层统一 `errors.As` 判断，杜绝字符串匹配；素材上传（multipart）改为一次性渲染 `[]byte` 而非流式 `io.Reader`，专门支持失败重试重放请求体，并按扩展名设置真实 Content-Type，避免平台因类型不符返回无 debug_message 的 400。

2. **OAuth 授权与 token 刷新：分布式并发安全**
   - 一次授权同时覆盖 Marketing / Offline Conversions / Profile 三种 scope；授权状态用 HMAC-SHA256 对 `{user_id}.{app_id}.{nonce}` 签名防跨用户绑定，`hmac.Equal` 恒定时间比较防时序攻击。
   - 应对 Snapchat 刷新 token 时会轮换 `refresh_token` 的特性，设计进程内 `singleflight` + Redis `SETNX` 跨进程锁双层收敛，解决并发刷新互相作废旧凭证导致渠道掉线的问题；`invalid_grant` 时直接置为 EXPIRED 并停止自动重试，等待人工重新授权而非无意义空转重试。

3. **异步发布引擎：分阶段流水线与幂等保证**
   - 基于 asynq 设计 Gateway（预检 + 凭证快照）→ Setup（账户级 Redis 锁）→ Media（素材下载/上传，未就绪自重入队）→ CampaignSegment → AdSegment → Finalizer 六阶段异步发布流水线，campaign/ad_squad/ad 三层对象先落库为 pending 草稿再逐条创建，`AttachSnapID` 用 `WHERE snap_id=''` 保证 asynq at-least-once 重投不会重复计数。
   - 用 Redis 实现 `ClaimStageTransition`（阶段推进一次性认领）与按 stage 分键的分段计数，解决"最后一个分段被重投导致阶段被推进两次、广告静默消失"的问题；僵死任务清理区分"排队中"（阈值为"发布中"的 8 倍）与"发布中"两种卡死场景，分别采用不释放锁 / `ForceRelease` 两种策略，避免误伤同账户下其他正在跑的任务。

4. **模板与跨账户复制、生活方式受众多国交集树**
   - 发布、模板、跨层级/跨账户复制三条路径共用同一个 `SpecBuilder`/`PublishSpec` 校验，避免配置口径分叉；模板放宽"开始时间需晚于当前"等发布态专属校验，跨账户复制清空 `snap_id` 并重置状态为 pending。
   - 生活方式受众（SCLS）下拉树用 `errgroup` 并发拉取多国数据后取交集（节点须在所有选中国家均存在才展示），并做父链循环检测，结果按 `singleflight` + 内存缓存复用，避免每次打开定向面板都重新拉取全量树。

5. **资产同步与限流降级设计**
   - 资产同步以"应用 + 授权人"为最小单位而非应用级，单人授权失效不影响全局同步；同步锁基于 Redis `SETNX` + compare-and-delete 脚本释放，防止连点触发重复全量同步（Redis 不可用时同步仍可用，仅退化为无互斥）。
   - 限流基于 Redis + Lua 脚本实现两级令牌桶（App 级与 token 级独立限速），时间戳取自 Redis `TIME` 避免多实例时钟偏差；Redis 故障时降级为进程内限速器而非直接放行，保证任何情况下都不会打穿平台限流。

6. **队列层：错误分类驱动的重试策略**
   - `Classify` 函数把发布失败统一收敛为三种处置：限流/网络/5xx 类重试当前分段、未授权/无权限/父资源不存在等直接判定任务失败、参数错误或素材死链等仅跳过单条不影响其余素材；`StageRunner` 接口把流水线阶段与 asynq handler 解耦，支撑脱库脱网单测覆盖分段派发逻辑。

# 量化成果

- 约 3 周内（2026-08-13 ~ 2026-09-02）产出 **72 个**非合并提交，净变更约 **+53,093 / -3,980** 行代码；
- 覆盖 **188 个** Go 文件，其中测试文件 **85 个**（占比约 45%），脱库脱网 mock（自定义 `HTTPDoer`）驱动 SDK 层限流/重试/多段上传等用例；
- 新增 **9 个**数据库迁移文件，涵盖 Snapchat 专属表结构与并发场景下的唯一约束修复；
- SDK 封装 Marketing API 下 **11 个**业务子服务 + Business API + Ad Library **两个**独立 API 组。

# 项目收获

从 0 到 1 独立完成一个第三方广告开放平台的完整后端集成：从 SDK 封装的防御性解析、语义化错误分类，到基于消息队列的分阶段异步发布引擎，再到分布式环境下的幂等、分布式锁与 singleflight 并发收敛，系统性积累了对接不稳定第三方 API 与设计可靠异步任务编排系统的工程判断力；与自己同步交付的前端发布模块形成完整闭环，也强化了跨端接口契约设计与联调排障的全栈视角。

# 相关链接

- [[blog-天呓网络科技有限公司-前端工程师]]
- [[blog-天呓ERP-广告资产与运营协同管理平台]]
