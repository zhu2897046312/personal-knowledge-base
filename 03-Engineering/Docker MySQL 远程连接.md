---
title: Docker MySQL 远程连接
tags: [docker, mysql]
created: 2026-07-22
updated: 2026-07-22
aliases: [MySQL Docker Remote]
summary: 解决容器 MySQL 无法远程访问的问题
type: pitfall
---

# 问题

宿主机或其他机器无法连接 Docker 容器内的 MySQL，常见表现：

- 连接超时 / 无法连通
- `Access denied for user ...`
- 能连上但立刻断开

按你的 `docker-compose.yml` / 云厂商安全组微调下列步骤。

# 原因

常见是多层问题叠加，而不是单一配置错误：

1. **未映射端口**：容器内 `3306` 未映射到宿主机
2. **只监听本机**：MySQL `bind-address` 限制为 `127.0.0.1`
3. **用户 host 过窄**：账号仅为 `'user'@'localhost'`，远程 host 被拒
4. **防火墙 / 安全组未放行**：宿主机或云厂商未开放 `3306`
5. **连错地址**：误用短暂的容器 IP，而应用应连宿主机 IP / 映射端口

# 解决方案

## 1. 确认端口映射

```yaml
services:
  mysql:
    image: mysql:8
    ports:
      - "3306:3306"
    # 其余环境变量、volume 按项目配置
```

重启：

```bash
docker compose up -d
```

## 2. 确认可远程登录的用户

官方镜像常通过环境变量控制 root 可登录 host；或手动授权：

```sql
CREATE USER 'app'@'%' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON your_db.* TO 'app'@'%';
FLUSH PRIVILEGES;
```

仅本机调试可用 `'app'@'localhost'`；远程必须允许对应 host（常用 `%`，生产宜收紧为具体网段）。

## 3. 检查 bind-address（若改过配置）

确保不仅监听 `127.0.0.1`。自定义 `my.cnf` 时注意与镜像默认行为一致，改完重建/重启容器。

## 4. 放行网络

- 本机防火墙允许入站 `3306`（若需局域网访问）
- 云服务器安全组入站放行来源 IP 的 `3306`
- 生产环境优先 VPN / SSH 隧道，避免公网裸奔 MySQL

## 5. 连接地址怎么选

| 场景 | 建议地址 |
|------|----------|
| 同一宿主机上的客户端 | `127.0.0.1` + 映射端口 |
| 局域网其他机器 | 宿主机局域网 IP + 映射端口 |
| 另一容器访问（同 compose 网络） | 服务名（如 `mysql`）+ 容器内 `3306`，不必走宿主机映射 |

不要依赖会变化的容器 IP。

# 验证

```bash
# 容器内存活
docker compose exec mysql mysqladmin ping -h 127.0.0.1 -uroot -p

# 宿主机经端口映射
mysql -h 127.0.0.1 -P 3306 -u app -p

# 远端：替换为宿主机 IP
mysql -h <host-ip> -P 3306 -u app -p
```

也可用：

```bash
nc -vz <host-ip> 3306
# 或
telnet <host-ip> 3306
```

确认端口通且账号可登录后，再查业务连接串。

# 相关链接

- [[Docker]]
- [[MySQL]]
- [[Docker Compose 容器名冲突与 no-cache 构建]]
- [[MySQL 表名大小写与 GORM]]
