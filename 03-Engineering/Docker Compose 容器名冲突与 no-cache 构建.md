---
title: Docker Compose 容器名冲突与 no-cache 构建
tags: [docker, compose]
created: 2026-07-22
updated: 2026-07-22
aliases: [Docker 容器冲突, no-cache]
summary: 不同目录 compose 导致容器名冲突，以及强制无缓存重建
type: pitfall
---

# 问题

报错：`Conflict. The container name is already in use`。在不同目录下执行 `docker compose` 时，无法自动覆盖已有同名容器。

# 原因

Docker Compose 默认用当前文件夹名作为 `COMPOSE_PROJECT_NAME`。目录路径不一致会被当成不同项目，无法平滑覆写同名容器。

# 解决方案

## 推荐：显式指定项目名

```bash
docker compose -p personal-blog up -d
```

无论在哪个子目录执行，都能覆写同一项目下的旧容器。

## 其他预防手段

- 在 `docker-compose.yml` 同级 `.env` 中锁定：`COMPOSE_PROJECT_NAME=personal-blog`
- 尽量避免在 compose 里硬编码 `container_name`，交给 Compose 自动管理

## 构建与部署常用指令

```bash
# 强制重新构建镜像（忽略旧缓存）
docker compose build --no-cache

# 后台启动/升级（配合 -p 覆写旧容器）
docker compose -p personal-blog up -d
```

国内服务器构建时，建议在 `Dockerfile` 中配置国内镜像源（如 `GOPROXY=https://goproxy.cn,direct`，以及 Alpine/Ubuntu 软件源），避免依赖下载超时。

# 验证

```bash
docker compose -p personal-blog ps
docker compose -p personal-blog up -d
```

确认不再出现容器名冲突，且新镜像已替换旧容器。

# 相关链接

- [[Docker MySQL 远程连接]]
- [[Docker]]
- [[Docker Compose environment 优先级高于 .env]]
