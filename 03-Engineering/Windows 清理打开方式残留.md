---
title: Windows 清理打开方式残留
tags: [windows, registry]
created: 2026-07-22
updated: 2026-07-22
aliases: [打开方式残留, OpenWith]
summary: 卸载后「打开方式」仍显示旧应用，需清理注册表残留
type: pitfall
---

# 问题

右键文件 → **打开方式 → 选择其他应用** 时，仍显示已卸载的软件（如 Qt 启动器、HBuilderX）。

# 原因

卸载程序通常不会清理注册表中的打开方式关联。Qt / HBuilderX / JetBrains 是高频残留源。

# 解决方案

## 推荐：手动清理注册表

1. `Win + R` → 输入 `regedit`
2. 清理以下三个位置：

**文件类型关联（最常见）**

```text
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\FileExts
```

删除对应后缀（如 `.js`、`.html`）下 `OpenWithList` / `OpenWithProgids` 里的残留应用。

**系统级打开方式来源**

```text
HKEY_CLASSES_ROOT\Applications
```

删除相关项，例如 `Qt*.exe`、`HBuilderX.exe`。

**用户级打开方式来源**

```text
HKEY_CURRENT_USER\Software\Classes\Applications
```

同样删除相关 exe 项。

3. 任务管理器 → **Windows 资源管理器 → 重新启动**

## 可选：不改注册表

用 **ShellExView** 禁用残留「打开方式」（安全，不删除）。

删除 `Applications` 下对应 exe 项通常即可彻底消失。

# 验证

再次右键同类型文件 → 打开方式 → 选择其他应用，确认已卸载应用不再出现。

# 相关链接

- [[Windows 注册表工具选型]]
- [[Windows 开机启动软件]]
