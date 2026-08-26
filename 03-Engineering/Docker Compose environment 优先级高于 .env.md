---
title: Docker Compose environment 优先级高于 .env
tags: [docker, compose, env]
created: 2026-08-26
updated: 2026-08-26
aliases: [env_file 不生效, docker-compose environment 覆盖 .env]
summary: docker-compose.yml 里 environment 块写死的值优先级永远高于 env_file/.env，改 .env 不生效不是取不到默认值，而是被 environment 里的硬编码值直接覆盖
type: pitfall
---

# 问题

`docker-compose.yml` 配合 `.env` 使用时，改了 `.env` 里某个变量（如 `DATABASE_URL`、`STORE_CORS`）的值，容器里读到的还是旧值，改了跟没改一样，容器行为完全没变化。

# 原因

`docker-compose.yml` 里有两套完全不同的变量替换机制，非常容易搞混：

1. **`image: ${MEDUSA_IMAGE:-medusa-server-medusa}` 这种写法**：是 Compose 解析 YAML 文件本身时做的变量替换，从 `.env`（或 shell 环境变量）里取值，取不到才用冒号后面的默认值兜底。
2. **`environment:` 块**：是塞进容器进程里的环境变量。规则是——**同一个变量名，`environment:` 里写死的值优先级永远高于 `env_file: .env` 读进来的值**。不是"取不到默认值再用 env_file"，而是"只要 `environment:` 里写了，`env_file` 里再怎么改都没用"。

如果 `environment:` 块里把 `DATABASE_URL`、`REDIS_URL`、`JWT_SECRET` 等变量直接写死成具体值（不是 `${VAR:-default}` 这种带默认值的写法，就是纯写死），`.env` 对这几个变量的修改会被静默吃掉，且不会有任何报错提示。

# 解决方案

1. **排查**：变量改了不生效时，先搜 `docker-compose.yml` 的 `environment:` 块，看这个变量名是不是在里面被硬编码了。
2. **修复**：需要该变量真正由 `.env` 驱动时，把它从 `environment:` 块里删掉，只留 `env_file: .env`；或者在 `environment:` 里也改成 `${VAR}`/`${VAR:-default}` 引用形式，而不是写死具体值。
3. **区分两套机制不要搞混**：`${VAR:-default}` 这种写法出现在 `image:` 等字段里，是 YAML 解析期的变量替换，跟 `environment:` 块的运行时优先级规则是两回事，改一个不会影响另一个。

# 验证

```bash
docker compose config
```

用这条命令看 Compose 实际解析出的最终配置，确认目标变量的值来自预期的来源（`.env` 还是 `environment:` 硬编码），而不是直接改完 `.env` 就假设生效，容器重启后再用 `docker exec <container> env | grep VAR_NAME` 确认容器内实际值。

# 相关链接

- [[Docker Compose 容器名冲突与 no-cache 构建]]
- [[Docker MySQL 远程连接]]
