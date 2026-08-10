---
title: Gmail 手动发送渲染好的 HTML 邮件
tags: [gmail, email, html]
created: 2026-08-10
updated: 2026-08-10
aliases: [Gmail HTML 邮件, 手动发送 HTML 邮件]
summary: 不写代码调用 Resend 等 API，借助浏览器 + Gmail 界面手动发送渲染好的 HTML 邮件
type: cheatsheet
---

# 概述

如果不想写代码调用 Resend 之类的邮件 API，只想借用 Gmail 界面手动把渲染好的 HTML 内容发出去，可以用浏览器复制粘贴的方式。

# 要点

- Gmail 撰写框粘贴的是渲染后的排版/样式（富文本），不是原始 HTML 代码
- 核心思路：先在浏览器里把 HTML 渲染成页面，再把渲染结果复制进 Gmail

# 用法

方法一：在浏览器打开 HTML 页面，直接复制粘贴

1. 将 HTML 代码保存为本地文件（如 `offer.html`）
2. 用 Chrome 或 Edge 双击打开该文件
3. 页面打开后按 `Ctrl + A`（Mac: `Cmd + A`）全选内容
4. 按 `Ctrl + C`（Mac: `Cmd + C`）复制
5. 打开 Gmail，点击「撰写」，在邮件正文区域按 `Ctrl + V`（Mac: `Cmd + V`）粘贴
   - 粘贴进 Gmail 的是渲染好的排版卡片和样式，而非原始代码

# 相关链接

- [[]]
