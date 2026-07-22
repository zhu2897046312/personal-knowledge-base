---
title: Windows Install Clean Up 解决重装失败
tags: [windows, msi, install]
created: 2026-07-22
updated: 2026-07-22
aliases: [Install Clean Up, MSI 残留]
summary: MSI 安装记录残留导致软件装不进或无法二次安装
type: pitfall
---

# 问题

- 软件第一次安装失败
- 已卸载 / 删目录，再次安装直接报错
- 常见于 UE5、开发工具、大型软件

# 原因

**Windows Installer（MSI）残留**：卸载不完整，安装记录还在，系统认为「已经装过 / 正在安装」。

一句话：装不进 = MSI 记录脏了。

# 解决方案

使用 **Windows Install Clean Up (2)**：

- 只清理 MSI 安装记录，不删真实文件
- 不会删除安装程序本体

步骤：

1. 打开 Windows Install Clean Up (2)
2. 找到问题软件（如 UE5 / Epic / Unreal Engine）
3. Remove / Delete 对应条目
4. 重启电脑
5. 重新安装

注意：

- 只删**确定装失败**的程序
- 不要乱删系统组件
- 属于最后兜底工具

# 验证

重启后重新运行安装程序，确认可正常完成安装。

# 相关链接

- [[Windows 注册表工具选型]]
