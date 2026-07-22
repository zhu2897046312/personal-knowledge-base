# docker compose build --no-cache重新构建镜像，--no-cache不使用缓存
# docker compose up -d 	启动新版本镜像

已帮你把 **Docker 容器冲突** 以及 **No-Cache 构建/部署规范** 整合并更新到了知识库中。你可以直接将以下内容补充或替换到你的 Markdown 知识库文件中：

---

## 🛠️ Docker & 部署踩坑速查

### 1. 容器名称冲突 (`Conflict. The container name is already in use`)

* **问题根源：** 在不同目录下运行 `docker compose` 时，Docker 会默认将当前文件夹名称作为 `COMPOSE_PROJECT_NAME`。目录路径不一致会导致 Compose 误认为是不同项目，从而无法自动覆盖同名容器。
* **最佳实践 (最佳解法 ⭐️)：** 统一使用 `-p` 参数显式指定项目名称，无论在哪个子目录下执行，都能平滑覆写旧容器。
```bash
docker compose -p personal-blog up -d

```


* **其他预防手段：**
* 在 `docker-compose.yml` 同级目录的 `.env` 中锁定：`COMPOSE_PROJECT_NAME=personal-blog`
* 尽可能避免在 `docker-compose.yml` 中强行硬编码 `container_name` 属性，交由 Compose 自动管理。



---

### 2. 构建与部署常用指令

```bash
# 1. 强制重新构建镜像（--no-cache 忽略旧缓存，确保拉取最新代码与配置）
docker compose build --no-cache

# 2. 后台启动/升级服务（自动覆写旧版本容器）
docker compose -p personal-blog up -d

```

> 💡 **提示：** 如果在国内服务器构建 Docker 镜像，建议在 `Dockerfile` 中置顶配置国内镜像源（如 `ENV GOPROXY=[https://goproxy.cn](https://goproxy.cn),direct` 及更换 Alpine/Ubuntu 软件源），避免依赖下载网络超时。