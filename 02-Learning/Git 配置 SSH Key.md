---
title: Git 配置 SSH Key
tags: [git, ssh, github]
created: 2026-07-22
updated: 2026-07-22
aliases: [GitHub SSH, ssh-keygen]
summary: 在 Windows 上为 GitHub 配置 Ed25519 SSH Key 并验证连接
type: learning
---

# 目标

完成 Git 用户信息配置、生成 SSH Key、添加到 GitHub，并用 SSH 克隆仓库。

# 知识点

## 检查 Git 是否安装

```bash
git --version
```

示例：`git version 2.51.0.windows.1`

## 配置 Git 用户信息

```bash
git config --global user.name
git config --global user.email
```

未配置时：

```bash
git config --global user.name "你的GitHub用户名"
git config --global user.email "你的GitHub邮箱"
```

查看全部全局配置：

```bash
git config --global --list
```

## 生成 SSH Key（Ed25519）

先看是否已有：

```bash
ls ~/.ssh
```

Windows CMD：`dir %USERPROFILE%\.ssh`

若已有 `id_ed25519` / `id_ed25519.pub`，无需重生成。

生成：

```bash
ssh-keygen -t ed25519 -C "你的GitHub邮箱"
```

路径默认回车；passphrase 可选。生成：

```text
~/.ssh/id_ed25519      # 私钥，勿泄露
~/.ssh/id_ed25519.pub  # 公钥，上传 GitHub
```

## 查看公钥

```bash
cat ~/.ssh/id_ed25519.pub
```

## 添加到 GitHub

Settings → SSH and GPG keys → New SSH key：Title、Authentication Key、粘贴公钥。

## 测试连接

```bash
ssh -T git@github.com
```

首次输入 `yes`。成功示例：

```text
Hi <GitHub用户名>! You've successfully authenticated, but GitHub does not provide shell access.
```

## 用 SSH 克隆

推荐 SSH 地址：`git@github.com:用户名/仓库.git`

```bash
git clone git@github.com:zhu2897046312/personal-knowledge-base.git
```

# 示例

修改远程为 SSH：

```bash
git remote set-url origin git@github.com:用户名/仓库.git
git remote -v
```

# 总结

1. 配好 `user.name` / `user.email`
2. 生成并上传 `id_ed25519.pub`
3. `ssh -T git@github.com` 验证
4. 克隆与 `remote` 优先用 SSH

# 相关链接

- [[Git 常用命令]]
- [[GitHub]]
