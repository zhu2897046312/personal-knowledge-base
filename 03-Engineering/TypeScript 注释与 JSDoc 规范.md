---
title: TypeScript 注释与 JSDoc 规范
tags: [typescript, jsdoc, comments, code-style]
created: 2026-08-24
updated: 2026-08-24
aliases: [JSDoc 规范, 注释规范]
summary: 注释解释 WHY 不解释 WHAT；哪些代码必须写完整 JSDoc、JSDoc 该写哪些字段、行内注释怎么放
type: cheatsheet
---

# 概述

注释和 JSDoc 的核心原则：**解释代码本身表达不出来的信息**（业务目的、设计原因、边界条件、
副作用、兼容约束、失败语义），不是逐行翻译代码在做什么。TypeScript 已经声明了参数和返回值
类型，JSDoc 里不需要再重复 `{string}` 这类类型标记。这套规范是从 Tianyi ERP 项目的个人编码
补充里搬出来的通用部分（那个项目自己特有的三目/enum 禁令没搬，那些是那个仓库的特殊约束，
不通用）。

# 要点

- 注释与 JSDoc 正文统一用中文；类型名、参数名、API、协议名和没有通用译法的技术术语保留英文。
- 注释必须随代码同步更新：改了函数的行为/参数/返回值/异常条件，必须同时检查并更新对应
  JSDoc。过期注释比没有注释更危险。
- 禁止：逐行翻译代码、描述显而易见的语法、保留注释掉的旧代码、内容与实际行为不一致的过期
  注释。
- TODO/FIXME 必须带责任人或可追踪信息：`// TODO(负责人或 issue): 待处理事项及原因`；没有
  归属和上下文的临时标记不提交。
- **必须写完整 JSDoc** 的代码：
  - 对外导出的函数、类、接口、类型、常量。
  - 自定义 Hook、公共 utility、跨模块调用的方法、涉及网络请求的封装函数。
  - 包含业务规则、权限判断、数据转换、缓存、重试或副作用的函数。
  - 参数/返回值/异常语义仅靠类型签名表达不清楚的函数。
- **按复杂度决定要不要写** 的代码：私有复杂函数至少说明作用、关键参数、返回结果和副作用；
  简单局部函数、事件转发函数、字段 getter，以及名称和类型已经说清楚语义的代码，不强制写。
  不为了「注释覆盖率」写「获取数据」「返回结果」这类没有信息量的注释。
- JSDoc 内容按实际需要包含：
  - 第一行：一句话说明业务作用（动宾结构），不复述函数名。
  - 补充段落：关键算法、前置条件、副作用、缓存策略、权限要求、调用时机——没必要就省略。
  - `@param`：每个参数单独一行，说明业务含义、单位、格式、有效范围、特殊值；不重复参数名
    或类型。
  - `@returns`：说明返回值的业务含义、成功/失败状态、空值语义。即使返回类型已经显式声明，
    公共函数也要写明返回值代表什么。
  - `@throws`：只在函数会主动抛错或透传不可恢复错误时写，说明错误类型和触发条件。
  - `@template`：泛型参数的职责或约束不直观时才用。
  - `@example`：调用方式、输入格式、边界行为不直观时给最小可运行示例。
  - `@deprecated`：弃用 API 必须写明替代方案和移除计划。
- 行内注释放在被解释代码的上一行，缩进跟代码一致；除很短的单位/协议说明，不写在语句末尾。
  说明「为什么这么写」而不是「这行代码做了什么」；复杂分支应该先提取成命名清晰的函数或变量，
  不能靠长注释掩盖糟糕结构。
- 魔法数字提取为具名常量，用注释说明来源、单位或业务依据。
- 临时兼容代码必须写明兼容对象、移除条件和关联 issue，避免永久遗留。

# 用法

标准函数示例（业务函数，带完整 JSDoc）：

```ts
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
  // ...
}
```

泛型函数示例：

```ts
/**
 * 根据唯一键将列表转换为索引，便于后续以常数时间查找元素。
 *
 * 当多个元素生成相同键时，后出现的元素会覆盖先前元素。
 *
 * @template T - 列表元素类型
 * @param items - 需要建立索引的元素列表
 * @param resolveKey - 从元素中生成唯一字符串键的函数
 * @returns 以 resolveKey 返回值为键、原始元素为值的只读映射
 */
export function indexBy<T>(
  items: readonly T[],
  resolveKey: (item: T) => string,
): ReadonlyMap<string, T> {
  return new Map(items.map((item) => [resolveKey(item), item]));
}
```

React Hook 示例（说明调用时机和副作用，而不是逐行翻译实现）：

```ts
/**
 * 管理订单列表的分页查询状态，并在查询条件变化时重新加载第一页。
 *
 * Hook 会触发请求并暴露响应式状态，只能在 React 组件或上层 Hook 中调用。
 *
 * @param initialQuery - 初始筛选条件；后续修改返回的 query 不会回写该对象
 * @returns 查询条件、订单数据、加载状态以及主动刷新方法
 */
export function useOrderList(initialQuery: OrderQuery) {
  // ...
}
```

行内注释示例（说明「为什么」，不说明「做了什么」）：

```ts
// 后端列表接口 pageSize 上限为 100，超出会被截断或返回 400
const MAX_LIST_PAGE_SIZE = 100;
```

反例（禁止这样写——只是复述变量名/函数名，没有信息量）：

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

# 相关链接

- [[Git 提交规范（个人习惯）]]
- [[React 组件编排：分组 Props 与 memo 子树]]
