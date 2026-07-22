---
title: Windows Defender 扫描 CPU 限制
tags: [windows, defender, cpu]
created: 2026-07-22
updated: 2026-07-22
aliases: [Defender CPU, 扫描 CPU]
summary: 通过组策略限制 Defender 扫描期间的 CPU 占用
type: cheatsheet
---

# 概述

扫描导致 CPU 占用过高时，可用组策略限制 Defender 扫描期间的 CPU 使用上限。

# 要点

- 入口：`gpedit.msc`（组策略）
- 路径：管理模板 → Windows 组件 → Microsoft Defender 防病毒 → 扫描
- 策略名：指定扫描期间 CPU 使用率的最大百分比（以本地实际显示为准）
- 原笔记将上限设为 `0`（按本机组策略说明理解其含义后再改）

# 用法

```text
Win + R
  → gpedit.msc
  → 管理模板
  → Windows 组件
  → Microsoft Defender 防病毒
  → 扫描
  → 指定扫描期间 CPU...
  → 启用
  → 设置数值（如 0）
```

# 相关链接

- [[Windows 开机启动软件]]
