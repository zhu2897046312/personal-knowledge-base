---
title: Godot Label 撑大布局
tags: [godot, ui, layout]
created: 2026-07-22
updated: 2026-07-22
aliases: [Godot Label 高度, font-size 撑开]
summary: Label/ProgressBar 默认字体最小高度会把父 Container 撑大
type: pitfall
---

# 问题

`header` 下布局高度失控。结构示意：

```text
HBoxContainer          70px
  VBoxContainer        70px
    HBoxContainer      16px
      Label            默认最小约 23px（font-size=16px）→ 把 HBoxContainer 撑到约 23px
    BoxContainer       16px
      ProgressBar      默认最小约 23px（font-size=16px）→ 同样撑大父容器
    BoxContainer       16px
    BoxContainer       16px
```

期望父级保持约 16px 高度，实际被带字体的控件顶开。

# 原因

带字体的控件（Label、ProgressBar 等）有默认最小高度（约 23px），大于父容器设定高度时会向上撑开父 `Container`。

# 解决方案

把带 font 相关内容的字体大小从默认 16px **下调到合适大小**，使其最小高度不超过父容器设定，就不会继续撑大父元素。

# 验证

调整 font-size 后，在编辑器中确认父 `HBoxContainer` / `VBoxContainer` 高度回到预期（如 16px / 70px），不再被 Label / ProgressBar 顶开。

# 相关链接

- [[Godot]]
