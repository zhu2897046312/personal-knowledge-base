---
title: Git 常用命令
tags: [git]
created: 2026-07-22
updated: 2026-07-28
aliases: [git commands]
summary: 日常 Git 暂存、提交、分支与状态查看速查
type: cheatsheet
---

# 概述

常用 Git 命令速查：暂存、提交、推送、分支与状态。

# 要点

- `git branch -d` 是安全删除：未合并到当前分支时会报错
- 推送前可用 `git status` 确认变更

# 用法

```bash
# 添加所有到暂存区
git add .

# 提交暂存区
git commit -m "init"

# 推送到远程分支
git push origin master

# 新建分支并切换
git checkout -b "new_branch"

# 删除已合并的分支（安全删除）
git branch -d feat/img

# 查看修改/新增了哪些文件
git status
```

# 清理已合并分支

先同步远端引用，再确认哪些分支已经合并到 `master`：

```bash
git fetch --prune

# 查看已合并的本地分支
git branch --merged master

# 查看已合并的远端分支
git branch -r --merged origin/master
```

删除确认不再需要的分支：

```bash
# 安全删除已合并的本地分支
git branch -d feat/example

# 删除远端分支
git push origin --delete feat/example

# 可以一次删除多个远端分支
git push origin --delete feat/example-a feat/example-b
```

删除后再次检查：

```bash
git fetch --prune
git branch -r --merged origin/master
```

注意：

- 不要删除 `master`、`origin/master`、`origin/HEAD` 或当前所在分支。
- `dev` 等长期分支即使已经合并，也应先确认团队是否仍在使用。
- 本地优先使用 `git branch -d`；`git branch -D` 会强制删除未合并分支，使用前必须确认提交无需保留。
- `git push origin --delete` 会修改共享远端状态，执行前逐个核对分支名。

# 相关链接

- [[Git 配置 SSH Key]]
