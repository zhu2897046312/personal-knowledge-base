---
title: 阿里云低配 docker pull OOM 宕机
tags: [docker, linux, aliyun, oom, swap]
created: 2026-07-29
updated: 2026-08-25
aliases: [docker pull 卡死, OOM Killer, 2G 服务器 Swap]
summary: 2核 2G 无 Swap 时 docker pull 解压打满内存/IO，触发 OOM 或系统挂起
type: pitfall
---

# 问题

阿里云低配服务器（2核 2G / 轻量应用服务器、突发性能实例）上执行 `docker pull` 后：

- SSH 直接断开
- 系统卡死 / 无响应
- 需在控制台「强制重启」才能恢复

实测已按下方「救急三招」严格执行，之后拉镜像不再直接拉崩。

# 原因

本质是内存/IO 被打满，触发 OOM Killer 或系统挂起，而不是「网络不好」本身。

1. **Docker 解压镜像极耗 CPU/内存**  
   `docker pull` 不只是下载：层（Layers）下载后会并行解压并写盘。2 核机器 CPU 易飙到 100%，解压缓冲区把内存挤爆。

2. **默认没有 Swap**  
   阿里云初始镜像常不开 Swap。2G 扣掉系统与 Docker daemon 后可用内存往往不足 1G。大镜像（如 embedding 相关、上 G 镜像）解压时内核可能直接 OOM，SSH 乃至整机「挂起」。

3. **云盘 IOPS 打满**  
   高效云盘 / ESSD Entry 等 IOPS 上限低，Docker 大量碎片写入时磁盘 I/O 到 100%，表现为整机像宕机一样无响应。

# 解决方案

重启并重新登录后，按顺序做完这三步（已执行）。

## 1. 开启 4G Swap（最关键）

物理内存爆了会落到磁盘，最多变慢，一般不会直接卡死。

```bash
# 创建 4GB Swap 文件
sudo fallocate -l 4G /swapfile

# 权限
sudo chmod 600 /swapfile

# 格式化并启用
sudo mkswap /swapfile
sudo swapon /swapfile

# 开机自动挂载
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

确认：

```bash
free -m
```

`Swap` 一行总量约 `4096` 即成功。

## 2. 限制 Docker 并发下载/解压

默认会同时处理多层，对 2 核是灭顶之灾。编辑或创建 `/etc/docker/daemon.json`：

```json
{
  "max-concurrent-downloads": 1,
  "max-concurrent-uploads": 1
}
```

重启 Docker：

```bash
sudo systemctl restart docker
```

## 3. 调整 Swappiness（必须做，否则 Swap 建了也不生效）

关键点：光靠第 1 步 `swapon` 建好 Swap 分区/文件**不代表 OOM 时内核真的会用它**。`vm.swappiness` 默认值偏低，内核会尽量死扛物理内存、迟迟不换页到 Swap，等真正判定内存耗尽时往往已经来不及、直接触发 OOM Killer 或整机卡死。必须调高 `vm.swappiness`，让内核更早、更愿意把闲置页换出到 Swap，Swap 才能真正在扛内存压力时发挥作用：

```bash
# 临时生效
sudo sysctl vm.swappiness=80

# 永久生效（修改 /etc/sysctl.conf）
echo "vm.swappiness=80" | sudo tee -a /etc/sysctl.conf
```

## 终极建议

- 2C2G 上**先配好 Swap**，再拉大镜像。
- 镜像特别大时：在本机或更高配机器 `docker pull` → `docker save -o image.tar <image>` → 传到服务器 → `docker load -i image.tar`，减轻服务器端高负载解压。

# 验证

```bash
free -m          # Swap 约 4096
# 再执行 docker pull
```

预期：拉取/解压时可能变慢，但不应再直接断 SSH 或整机宕机。

# 相关链接

- [[Docker]]
- [[Docker Compose 容器名冲突与 no-cache 构建]]
- [[Docker MySQL 远程连接]]
