---
title: Windows 开机启动软件
tags: [windows, startup]
created: 2026-07-22
updated: 2026-07-22
aliases: [开机启动, shell:startup]
summary: 通过当前用户 Startup 文件夹设置开机启动软件
type: cheatsheet
---

# 概述

把软件快捷方式放入当前用户的 Startup 文件夹，即可随 Windows 登录自动启动。

# 要点

- 作用于**当前用户**，不影响其他账户
- 删除快捷方式即可取消启动，不影响软件本体
- 最常用入口：`shell:startup`

# 用法

1. `Win + R`
2. 输入：

```text
shell:startup
```

3. 将软件快捷方式复制到该文件夹

# 相关链接

- [[Windows 清理打开方式残留]]
- [[Windows 关闭快速访问自动添加]]
