---
title: Git 提交规范（个人习惯）
tags: [git, commit, code-review]
created: 2026-08-24
updated: 2026-08-24
aliases: [commit 规范, 提交习惯]
summary: 提交信息格式、暂存方式、提交前自查与几条不允许触碰的红线
type: cheatsheet
---

# 概述

日常写代码时对「Git 提交」这一步的个人习惯补充，跟 [[Git 常用命令]] 侧重命令操作不同，这篇
记的是「什么时候能提交、怎么组织提交信息、提交前要过一遍什么」。原始版本是给 Tianyi ERP
项目写的（引用了那个项目专属的 `pnpm check`、`docs/standards/code-review.md` 等），这里只留
通用部分，具体项目里遇到对应的检查命令/自查清单直接替换进去用。

# 要点

- 提交信息格式：`<type>(<scope>可选): <说明>`；`type` 只用
  `feat`/`fix`/`docs`/`style`/`refactor`/`perf`/`test`/`chore`/`build`/`ci`/`revert`。
- 首行简洁，一次提交只表达一类目的；不要把不相关的改动混进一个 commit。
- 暂存时用显式 `git add <path>`，不用 `git add -A`/`git add .`——避免把本机个人文件（本地
  笔记、IDE 配置、临时脚本）误带进提交。
- 本机专属的个人补充文档/配置（比如自己写的编码规范笔记、本地 launch 配置）永远不提交、
  不推送，只留在本机。
- 不用 `--no-verify`/`--no-gpg-sign` 跳过校验和签名；hook 挡下来的问题去修，不去绕过。
- 不对主分支/联调分支强制推送；确实需要强推前先跟协作者确认。
- 未经明确要求，不自动创建 commit、推送分支或开 MR/PR——这些是会影响共享状态的动作，
  应该由用户决定时机。

# 用法

提交信息示例：

```text
feat(order-list): 支持按状态批量导出订单
fix(auth): 修复 token 刷新失败后未清理登录态的问题
refactor(ip-management): 按文件职责把纯工具函数归位到 helpers/constants
```

# 提交前自查

帮忙提交前，先对本次 diff 走一遍自查，再跑一次项目的检查命令（lint / typecheck / build /
test，视项目而定）；自查和检查都过了才提交，中间发现问题就停下来列出问题等确认，不带病提交。
自查至少覆盖：

- 改动范围是不是跟当前任务一致，有没有夹带无关重构。
- 有没有凭空猜测接口字段、错误码、权限规则等本该跟人确认的契约。
- `git status`/`git diff` 里有没有不该出现的文件（尤其是看起来无害但内容可能带密钥的文件，
  文件名不可信，要打开看内容）。
- 合并主分支或解决完冲突后，完成合并提交前，同样要跑一遍检查——确认冲突解决后的整棵树
  还能正常编译、通过检查。

# 相关链接

- [[Git 常用命令]]
- [[TypeScript 注释与 JSDoc 规范]]
- [[React 组件编排：分组 Props 与 memo 子树]]
