---
title: TypeScript 表达式复杂度与常量枚举规范
tags: [typescript, vue, coding-standards]
created: 2026-08-26
updated: 2026-08-26
aliases: [禁止嵌套三目, enum 替代方案, as const 常量对象]
summary: 不依赖具体项目结构的通用 TS/Vue 规范——禁止嵌套三目、模板片段超 3 行必须提取、禁用 enum 改用 as const 常量对象派生联合类型
type: cheatsheet
---

# 概述

写 TypeScript（含 Vue `<script setup>`/`<template>`）时的通用规范，核心是"表达式复杂度不下沉到模板/内联里"和"业务枚举值统一走常量对象"，与具体项目的目录结构无关，来自 [[WXT 项目文件组织规范]] 同一套个人工程规范。

# 要点

- 三目运算符只允许单层"二选一"，禁止嵌套；`??` 和 `?.` 不算三目，不受此限制
- Vue 模板里任意独立渲染片段（`v-for` 循环体、`v-if` 分支）展开超过 3 行标签就必须提取
- 事件绑定只允许单次函数调用转发，出现参数转换/分支/多语句必须提成命名 `handleXxx` 函数
- 表达一组业务固定取值禁止用 TS `enum`，改用 `as const` 常量对象 + 派生联合类型
- 业务代码禁止直接写字面量做比较/赋值/传参，必须点常量对象取值
- 第三方库/协议自带的字面量（如 `method: 'GET'`）不受常量对象约束，只有"自己定义的业务语义"才需要包一层

# 用法

## 禁止嵌套三目运算符

嵌套三目一律改写为 `if`/`else` 或 `switch`；多分支判断提取为命名清晰的独立函数或提前返回的 `if` 链。

```ts
// 允许：单层三目
const label = status === 'active' ? '进行中' : '已结束';

// 禁止：嵌套三目
const label = status === 'active' ? '进行中' : status === 'paused' ? '已暂停' : '已结束';

// 改为：if/else 提前返回
function getStatusLabel(status: TaskStatus): string {
  if (status === 'active') return '进行中';
  if (status === 'paused') return '已暂停';
  return '已结束';
}
```

Vue 模板里"有条件渲染某个节点、否则不渲染"的场景用 `v-if`/`v-else`（或 `v-show`）表达，不要把三目塞进属性绑定或插值里。

## 模板片段超过 3 行需要提取

数行数时不含片段自身的开始/结束标签。

- 有复用价值、或内部有自己的状态/逻辑：提取成独立子组件（放 `components/`）
- 纯粹是一段取值/格式化逻辑内联在模板里：提取成 `<script setup>` 里的 `computed`

```vue
<!-- 禁止：v-for 循环体超过 3 行，直接写在父组件模板里 -->
<div v-for="account in accounts" :key="account.id">
  <input v-model="account.name" />
  <span>{{ account.status }}</span>
  <button @click="() => { account.enabled = !account.enabled; save(); }">切换</button>
</div>

<!-- 改为：提取成子组件，父组件模板只保留一行绑定 -->
<AccountRow
  v-for="account in accounts"
  :key="account.id"
  :account="account"
  @toggle="handleToggle"
/>
```

事件绑定同理，只做单次函数调用转发（如 `@click="save"`、`@click="() => remove(id)"`）允许内联；一旦出现分支或多条语句必须提取：

```vue
<!-- 禁止：事件绑定里塞多条语句/分支判断 -->
<button @click="() => { if (!id) return; remove(id); refresh(); }">删除</button>

<!-- 改为：提取命名 handler，模板里只转发调用 -->
<script setup lang="ts">
function handleRemove(id: string | undefined): void {
  if (!id) return;
  remove(id);
  refresh();
}
</script>

<button @click="() => handleRemove(id)">删除</button>
```

## 禁止 TypeScript enum，改用 as const 常量对象

命名：常量对象用 `UPPER_SNAKE_CASE`，派生类型用 `PascalCase`，两者不同名；常量对象和它派生的类型放在同一个文件（按值的性质放进 `constants/*.ts`）。

```ts
// 禁止：TypeScript enum
enum AssetExportScope {
  All = 'all',
  Selected = 'selected',
}

// 禁止：裸字面量硬编码比较，散落在多处
if (scope === 'selected') { /* ... */ }
setScope('all');

// 改为：as const 常量对象 + 派生联合类型，业务代码只点常量取值
export const ASSET_EXPORT_SCOPE = {
  ALL: 'all',
  SELECTED: 'selected',
} as const;

export type AssetExportScope = (typeof ASSET_EXPORT_SCOPE)[keyof typeof ASSET_EXPORT_SCOPE];

// 使用处
if (scope === ASSET_EXPORT_SCOPE.SELECTED) { /* ... */ }
setScope(ASSET_EXPORT_SCOPE.ALL);
```

理由：以后改某个取值的字面量内容或文案，只改常量对象这一处，不用在整个代码库搜字符串字面量逐处替换，也不会因为漏改一处产生运行时不一致却编译通过的 bug。已有代码发现用 `enum` 或裸字面量硬编码同一组业务取值时顺手按此改写，但改写范围只限于确实是"同一组业务枚举值"的字面量，不连带重构无关代码。

# 相关链接

- [[WXT 项目文件组织规范]]
- [[注释 JSDoc 与 Git 提交个人规范]]
