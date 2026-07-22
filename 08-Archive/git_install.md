# Git 配置 SSH Key（GitHub）

## 1. 检查 Git 是否安装

查看 Git 是否已安装：

```bash
git --version
```

示例：

```text
git version 2.51.0.windows.1
```

---

## 2. 配置 Git 用户信息

查看当前配置：

```bash
git config --global user.name
git config --global user.email
```

如果未配置，执行：

```bash
git config --global user.name "你的GitHub用户名"
git config --global user.email "你的GitHub邮箱"
```

例如：

```bash
git config --global user.name "zhu2897046312"
git config --global user.email "example@gmail.com"
```

查看所有全局配置：

```bash
git config --global --list
```

---

## 3. 生成 SSH Key（Ed25519）

### 查看是否已有 SSH Key

Git Bash：

```bash
ls ~/.ssh
```

Windows CMD：

```cmd
dir %USERPROFILE%\.ssh
```

如果目录中已经存在：

```text
id_ed25519
id_ed25519.pub
```

则无需重新生成。

---

### 生成新的 SSH Key

执行：

```bash
ssh-keygen -t ed25519 -C "你的GitHub邮箱"
```

例如：

```bash
ssh-keygen -t ed25519 -C "example@gmail.com"
```

随后会出现提示：

```text
Enter file in which to save the key
```

直接按 **Enter**，使用默认路径：

```text
C:\Users\你的用户名\.ssh\id_ed25519
```

继续提示：

```text
Enter passphrase (empty for no passphrase)
```

可选择：

- 直接按 Enter（方便日常开发）
- 设置密码（更安全）

最终生成：

```text
C:\Users\你的用户名\.ssh\
├── id_ed25519        # 私钥（不要泄露）
└── id_ed25519.pub    # 公钥（上传到 GitHub）
```

---

## 4. 查看公钥内容

Git Bash：

```bash
cat ~/.ssh/id_ed25519.pub
```

Windows CMD：

```cmd
type %USERPROFILE%\.ssh\id_ed25519.pub
```

PowerShell：

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

复制输出的整行内容，例如：

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... example@gmail.com
```

---

## 5. 添加公钥到 GitHub

登录 GitHub：

**Settings → SSH and GPG keys → New SSH key**

填写：

- **Title**：当前电脑名称（例如：Windows-PC）
- **Key type**：Authentication Key（默认）
- **Key**：粘贴刚刚复制的公钥

点击 **Add SSH key** 保存。

---

## 6. 测试 SSH 连接

执行：

```bash
ssh -T git@github.com
```

第一次连接会提示：

```text
The authenticity of host 'github.com' can't be established.
Are you sure you want to continue connecting (yes/no)?
```

输入：

```text
yes
```

成功后会显示：

```text
Hi <GitHub用户名>! You've successfully authenticated, but GitHub does not provide shell access.
```

说明 SSH 配置成功。

---

## 7. 使用 SSH 克隆仓库

推荐使用 SSH 地址，而不是 HTTPS。

HTTPS：

```text
https://github.com/用户名/仓库.git
```

SSH：

```text
git@github.com:用户名/仓库.git
```

示例：

```bash
git clone git@github.com:zhu2897046312/personal-knowledge-base.git
```

---

# 常用命令

## 查看 Git 配置

```bash
git config --global --list
```

---

## 查看远程仓库地址

```bash
git remote -v
```

---

## 修改远程仓库为 SSH

```bash
git remote set-url origin git@github.com:用户名/仓库.git
```

---

## 查看已生成的 SSH Key

```bash
ls ~/.ssh
```

Windows：

```cmd
dir %USERPROFILE%\.ssh
```

---

## 查看 SSH 公钥

```bash
cat ~/.ssh/id_ed25519.pub
```

Windows：

```cmd
type %USERPROFILE%\.ssh\id_ed25519.pub
```