---
title: Git 常用命令
tags: [git]
created: 2026-07-22
updated: 2026-07-22
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

# 相关链接

- [[Git 配置 SSH Key]]
