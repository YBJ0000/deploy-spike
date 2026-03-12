# 本地构建并推送 Raidar 镜像（精简版）

从 `medical-server/app` 用 **Dockerfile** 构建镜像并推送到 Docker Hub，供 Dokploy 拉取。Docker Hub 用户名请按需替换（示例：`yangbingjia1206`）。

详细方案对比、踩坑与排错见 [build-and-push-raidar-image.md](./build-and-push-raidar-image.md)，以及 [journal/](../journal/)、[faq.md](./faq.md)。

---

## 前置条件

- **Docker Desktop（或 Docker 引擎）已启动**，终端可执行 `docker info`
- 已克隆 **medical-server** 仓库，能进入 `medical-server/app` 目录

---

## 一、加载环境变量（RAIDAR_TAG）

**无 deploy-spike 仓库**：在终端直接执行，例如：

```bash
export RAIDAR_TAG=server-20260311-2
```

**有 deploy-spike 仓库**：可用 `.env` 统一管理。在 `deploy-spike/.env` 中设置：

```env
RAIDAR_TAG=server-20260311-2
```

在终端加载（每次新开终端执行一次）：

```bash
cd /path/to/deploy-spike
set -a && source .env && set +a
```

之后本地构建与 Dokploy 环境变量都应对齐到同一 tag。

---

## 二、登录 Docker Hub

推送前先登录：

```bash
docker login
```

按提示输入 Docker Hub 用户名和密码（或 Access Token）。

---

## 三、在 medical-server/app 下构建并推送（多架构）

在 **medical-server/app** 目录下执行，使用 **Dockerfile**、**多架构（amd64 + arm64）**、**带 tag**，并直接 `--push`：

```bash
cd /path/to/medical-server/app
docker buildx create --use --name raidar-multiarch 2>/dev/null || docker buildx use raidar-multiarch
docker buildx build --platform linux/amd64,linux/arm64 -t yangbingjia1206/raidar:${RAIDAR_TAG} --push .
```

- 首次构建会拉取基础镜像并执行 Gradle 编译，耗时可能数分钟。
- 请将 `yangbingjia1206` 替换为你自己的 Docker Hub 用户名。

---

## 四、推送后自检

1. **Docker Hub**：打开 [Docker Hub](https://hub.docker.com) → 你的 **Repository** → **Tags**，确认刚推送的 tag（如 `server-20260311-2`）存在且为最新。
2. **Dokploy**：进入对应 Service 的 **Environment**，确认 **RAIDAR_TAG** 的值与 Docker Hub 上显示的 tag **完全一致**，然后执行 **Deploy**（若需拉取新镜像）。

---

*详细说明（单架构构建、Dokploy 仍用旧镜像时的处理等）见 [build-and-push-raidar-image.md](./build-and-push-raidar-image.md)。*
