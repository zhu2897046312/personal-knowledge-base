---
title: React 组件编排：分组 Props 与 memo 子树
tags: [frontend, react, typescript, component-design]
created: 2026-08-24
updated: 2026-08-24
aliases: [分组 Props, useLatestCallback]
summary: 编排组件 props 爆炸时，按关注点分组、useLatestCallback 钉回调、memo 子树按组浅比较的一整套写法
type: cheatsheet
---

# 概述

容器/编排组件（负责取数、状态编排、拼 props 往下传）一旦 props 超过一二十个，扁平列表会
难以维护、容易漏传，也没法靠一次浅比较让展示子树的 `memo` 真正生效——因为父级每次 render
都会传新的箭头函数，默认的浅比较会认为「props 变了」。这篇速查记录一套按关注点分组、用
`useLatestCallback` 钉住回调身份、展示子树按组做浅比较的写法，来自 Tianyi ERP 项目
`mailintel` 模块 `MailDetail` 组件的落地模式，后来在同项目的资产管理表格组件上复用过一次，
效果稳定，值得当通用模式记下来。

# 要点

- 编排组件的 props 按业务关注点分组（比如「外壳装配 / 核心数据 / 行为操作」，或按领域拆成
  更贴切的名字），每组是一个独立的具名类型，组合成整体 `type XxxProps = { groupA, groupB, ... }`。
  分组要按真实的关注点切，不是凑数量。
- 编排层（`index.tsx`）只做三件事：接收分组 props → 用 `useLatestCallback` 钉住会往下传的
  `onXxx` 回调身份 → 把钉住的回调写回对应分组，整体交给 memo 的展示子树。
- 展示子树用 `memo(Component, areXxxPropsEqual)`，`areXxxPropsEqual` 对每个分组分别做浅比较
  （`Object.is` 兜底 + 逐 key 比较）。不能用默认 memo：分组对象每次 render 都是新引用，默认
  浅比较会一直判定为「变了」。
- `useLatestCallback` 是让分组浅比较真正生效的关键：用 `useRef` 存最新回调，
  `useCallback(() => ref.current(...), [])` 返回一个身份永远不变的函数。父级传入新的箭头
  函数不会导致这个函数标识变化——没有这一步，「分组」只是好看，memo 照样每次都被判定为变了。
- 组合多个子组件时，按子组件自己的 Props 类型组装出对应形状再 `{...childProps}` 传入；不要
  把父级整个分组或整包 props 甩给形状不同的子组件。
- 什么时候值得这么做：编排组件的 props 明显能按关注点切开（不是硬凑分组），且下游子树的
  重绘成本不低（复杂表单、富文本、iframe、大表格等）。简单展示组件、props 不多、或下游重绘
  很轻的场景，扁平 props 更简单，不必套这套模式——判断标准是「有没有真实的重绘成本和分组
  语义」，不是「props 数量超过某个阈值就一定要分组」。

# 用法

目录结构（以一个叫 `Widget` 的编排组件为例）：

```text
Widget/
  index.tsx              # 编排层：钉回调 → 写回分组 → 渲染 memo 视图
  Widget.types.ts        # 分组 Props 类型
  useWidgetViewProps.ts  # useLatestCallback 钉回调、写回分组
  Widget.helpers.ts      # areWidgetViewPropsEqual（按组浅比较）+ 其它纯函数
  WidgetView.tsx         # memo 展示组件，实际渲染内容
```

```ts
// Widget.types.ts
export type WidgetChromeProps = {
  title: string;
  onClose: () => void;
};

export type WidgetDataProps = {
  items: readonly Item[];
  isLoading: boolean;
};

export type WidgetActionsProps = {
  onSelect: (item: Item) => void;
  onDelete?: (item: Item) => void;
};

export type WidgetProps = {
  chrome: WidgetChromeProps;
  data: WidgetDataProps;
  actions: WidgetActionsProps;
};
```

```ts
// useWidgetViewProps.ts
export function useWidgetViewProps({ chrome, data, actions }: WidgetProps): WidgetProps {
  const onClose = useLatestCallback(chrome.onClose);
  const onSelect = useLatestCallback(actions.onSelect);
  const onDelete = useLatestCallback(actions.onDelete);

  return {
    chrome: { ...chrome, onClose },
    data,
    actions: { ...actions, onSelect, onDelete },
  };
}
```

```ts
// Widget.helpers.ts
function areShallowEqual<T extends object>(left: T, right: T): boolean {
  if (Object.is(left, right)) return true;
  const keys = Object.keys(left) as Array<keyof T>;
  if (keys.length !== Object.keys(right).length) return false;
  return keys.every((key) => left[key] === right[key]);
}

// 组对象每次都是新引用；按组浅比较，效果等同于展平后的默认 memo。
export function areWidgetViewPropsEqual(prev: WidgetProps, next: WidgetProps): boolean {
  return (
    areShallowEqual(prev.chrome, next.chrome) &&
    areShallowEqual(prev.data, next.data) &&
    areShallowEqual(prev.actions, next.actions)
  );
}
```

```tsx
// WidgetView.tsx
export const WidgetView = memo(function WidgetView({ chrome, data, actions }: WidgetProps) {
  // ...实际渲染
}, areWidgetViewPropsEqual);

// index.tsx
export function Widget(props: WidgetProps) {
  const viewProps = useWidgetViewProps(props);
  return <WidgetView {...viewProps} />;
}
```

`useLatestCallback` 本身（通用 hook，跟业务无关，可以直接抄）：

```ts
export function useLatestCallback<TArgs extends readonly unknown[], TResult>(
  callback: (...args: TArgs) => TResult,
): (...args: TArgs) => TResult {
  const ref = useRef(callback);
  ref.current = callback;
  // 空依赖是故意的：只钉身份，不跟着 callback 更新；调用时读 ref.current 保证不过期
  return useCallback((...args: TArgs) => ref.current(...args), []);
}
```

# 相关链接

- [[TypeScript 注释与 JSDoc 规范]]
- [[Git 提交规范（个人习惯）]]
