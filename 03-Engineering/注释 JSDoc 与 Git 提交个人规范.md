---
title: 注释 JSDoc 与 Git 提交个人规范
tags: [jsdoc, git, coding-standards, typescript]
created: 2026-08-26
updated: 2026-08-26
aliases: [JSDoc 规范, Git 提交规范, TODO FIXME 格式]
summary: 个人在项目仓库里实际遵守的注释/JSDoc 撰写标准和提交前自查流程，与具体项目结构、语言特性无关
type: cheatsheet
---

# 概述

与项目结构、语言特性无关，纯粹是个人在仓库里实际遵守的开发习惯：怎么写注释/JSDoc、提交前做什么检查。与 [[WXT 项目文件组织规范]]、[[TypeScript 表达式复杂度与常量枚举规范]] 同属一套个人工程规范，但这份文档本身按约定不进项目仓库历史（`.gitignore` 忽略），只作为本机参考，故单独归档进知识库。

# 要点

- 注释与 JSDoc 正文统一用中文；类型名、参数名、API、协议名等无通用译法的术语保留英文
- 注释必须解释代码无法直接表达的信息（业务目的、设计原因、边界条件、副作用、兼容约束、失败语义），禁止逐行翻译代码或描述显而易见的语法
- 对外导出的函数/类/接口/类型/常量，以及带副作用的公共 utility 必须写完整 JSDoc；TypeScript 已声明的类型不在 JSDoc 里重复标注
- TODO/FIXME 必须包含责任人或 issue 与原因：`// TODO(责任人或 issue): 待处理事项及原因`
- Git 提交格式固定为 `<type>(<scope>可选): <中文说明>`，一次提交只做一类改动
- 提交前必须先对 `git diff` 走一遍自查、再跑一次编译构建，有问题先停下修，不带病提交
- 暂存显式 `git add <path>`，禁止 `git add -A`/`git add .`，避免误带本机个人文件

# 用法

## 注释语言与基本原则

- 注释必须随代码同步更新：修改函数行为、参数、返回值或异常条件时必须同时检查并更新对应 JSDoc
- 禁止保留注释掉的旧代码，或使用与实际行为不一致的过期注释

## 分级注释要求

以下代码必须使用完整 JSDoc：

- 对外导出的函数、类、接口、类型和常量
- 公共 utility、跨模块调用的方法，以及带副作用（网络请求、storage 读写、日志）的函数
- 包含业务规则、权限判断、数据转换、缓存、重试或副作用的函数
- 参数、返回值或异常语义仅靠 TypeScript 类型无法充分表达的函数

以下代码按复杂度决定是否使用 JSDoc：

- 私有复杂函数至少说明函数作用、关键参数、返回结果和可能的副作用
- 简单局部函数、事件转发函数、字段 getter，以及名称和类型已完整表达语义的代码，不强制加 JSDoc
- 不得为了"注释覆盖率"编写无信息量注释，例如"获取数据""返回结果""设置名称"

## JSDoc 内容要求

- 第一行：用一句中文说明函数的业务作用，动宾结构，不复述函数名
- 补充段落：说明关键算法、前置条件、副作用、缓存策略、权限要求或调用时机，没必要时省略
- `@param`：每个参数单独一行，说明业务含义、单位、格式、有效范围和特殊值，不得只重复参数名或类型
- `@returns`：说明返回值的业务含义、成功/失败状态及空值语义，即使返回类型已显式声明也要写
- `@throws`：函数会主动抛错或透传不可恢复错误时说明错误类型和触发条件；不抛异常则不加
- `@template`：泛型参数的职责或约束不直观时使用
- `@example`：调用方式、输入格式或边界行为不直观时提供最小可运行示例
- `@deprecated`：弃用 API 必须说明替代方案和计划移除的版本或条件

标准函数示例：

```ts
type OrderQuery = {
  status?: string;
  page: number;
  pageSize: number;
};

type OrderPage = {
  items: Order[];
  total: number;
};

/**
 * 按筛选条件查询订单列表，并返回指定页的数据。
 *
 * 调用方负责 loading 与错误提示；本函数只负责参数校验与响应结构收窄。
 *
 * @param query - 查询条件；page 从 1 开始，pageSize 必须为 1 到 100 的整数
 * @returns 当前页订单和满足条件的订单总数；无匹配时 items 为空数组
 * @throws {RangeError} page 或 pageSize 超出允许范围时抛出
 * @throws {Error} 请求失败或接口响应结构无效时抛出，并保留原始错误作为 cause
 *
 * @example
 * ```ts
 * const result = await fetchOrders({ status: 'pending', page: 1, pageSize: 20 })
 * ```
 */
export async function fetchOrders(query: OrderQuery): Promise<OrderPage> {
  if (query.page < 1) throw new RangeError('page 必须从 1 开始');
  if (query.pageSize < 1 || query.pageSize > 100) {
    throw new RangeError('pageSize 必须介于 1 和 100 之间');
  }

  const response: unknown = await fetch('/api/orders', {
    method: 'POST',
    body: JSON.stringify(query),
  }).then((res) => res.json());

  if (!isOrderPage(response)) {
    throw new TypeError('订单列表接口返回结构无效');
  }

  return response;
}
```

带副作用的异步函数（无需向上抛错的场景）示例：

```ts
/**
 * 请求一次报表接口，并把核心参数、原始 Response 打印到控制台。
 *
 * 请求失败或返回内容不是合法 JSON 时，只打印错误，不向上抛出——调用方（拦截回调、定时轮询）
 * 不需要针对单次失败做额外处理。
 *
 * @param url - 完整的报表请求 URL
 * @param label - 打印日志时使用的标签，用于区分是拦截触发还是定时轮询、以及具体账户
 * @returns 请求完成（无论成功或失败）后解析
 */
export async function fetchAndLogReport(url: string, label: string): Promise<void> {
  try {
    const res = await fetch(url, { credentials: 'include' });
    const json = await res.json();
    console.log(`[${label}] Response 原始数据:`, json);
  } catch (err) {
    console.error(`[${label}] 获取 Response 失败:`, err);
  }
}
```

## 行内注释约束

- 行内注释放在被解释代码的上一行，与代码保持相同缩进；除极短的单位/协议说明外不在语句末尾追加
- 说明"为什么采用该实现"而不是"代码做了什么"；复杂分支应优先提取为命名清晰的函数或变量，不能用长注释掩盖糟糕结构
- 魔法数字应提取为具名常量，并通过注释说明来源、单位或业务依据
- 临时兼容代码必须写明兼容对象、移除条件和关联 issue，避免永久遗留

```ts
// 后端列表接口 pageSize 上限为 100，超出会被截断或返回 400
const MAX_LIST_PAGE_SIZE = 100;
```

## 禁止的注释示例

```ts
// 定义用户名
const userName = user.name;

/**
 * 获取订单。
 * @param id - id
 * @returns 订单
 */
async function getOrder(id: string): Promise<Order> {
  return fetchOrderById(id);
}
```

以上注释只是复述变量名、函数名和类型，没有说明业务用途、参数格式、空值语义或失败条件，应删除或补充有效信息。

## Git 提交规范

- 提交格式：`<type>(<scope>可选): <中文说明>`；`type` 只用 `feat`/`fix`/`docs`/`style`/`refactor`/`perf`/`test`/`chore`/`build`/`ci`/`revert`。一次提交只做一类改动，功能变更和纯结构重构分开提交，不把不同 `type` 的改动混进同一次提交。
- 提交前必须先对本次 `git diff` 走一遍相关规范文档自查，再跑一次编译/构建命令（如 `pnpm run compile && pnpm run build`）；有问题先停下修，不带病提交。
- 暂存时用显式 `git add <path>`，禁止 `git add -A`/`git add .`，避免把本机个人文件（如 `CLAUDE.md`、个人规范文档、`.claude/launch.json` 等）误带进提交。
- 本机个人补充文档（不进仓库历史的规范文档）永远不提交、不推送，对应项目 `.gitignore` 里的忽略规则。
- 不使用 `--no-verify`/`--no-gpg-sign` 跳过校验；不对主分支强制推送。
- 未经用户明确要求，不自动创建 Commit、推送分支或创建 MR。
- 合并/解决冲突后，完成合并提交（`git commit --no-edit`）前同样要跑一遍编译/构建，确认冲突解决后的整棵树仍能编译、构建通过。

# 相关链接

- [[WXT 项目文件组织规范]]
- [[TypeScript 表达式复杂度与常量枚举规范]]
