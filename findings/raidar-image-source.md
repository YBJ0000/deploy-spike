# Raidar 镜像来源与拉取失败时的方案

## 问题现象

Deploy 时日志报错：

```text
Image ghoshorn/raidar:server-latest Pulling
Image ghoshorn/raidar:server-latest Error pull access denied for ghoshorn/raidar, repository does not exist or may require 'docker login': denied
```

说明 Dokploy 所在机器无法拉取 `ghoshorn/raidar:server-latest`（私有或该仓库/账号已不可用）。若原维护者 ghoshorn 已不参与项目，就需要**换一个可用的镜像来源**。

---

## 三种方向对比

| 方向 | 做什么 | 能否不依赖公司管理员 | 说明 |
|------|--------|----------------------|------|
| **1. 公司 GitHub 连 Dokploy** | 组织管理员在 GitHub 安装 Dokploy 的 GitHub App，授权组织/仓库。 | 否，需管理员 | 这样 Dokploy 能**克隆并构建**公司仓库里的 medical-server，但**不解决**「从 Docker Hub 拉 ghoshorn/raidar」——除非公司同时把镜像推到某仓库并在 Dokploy 配 Registry。若改为在 Dokploy 里用 **Application** 从该 repo 构建镜像，则不再依赖 ghoshorn 的镜像。 |
| **2. Fork 到个人 + 在 Dokploy 构建** | 开放 Fork 后，把 medical-server fork 到个人账号；在 Dokploy 用 **Application** 从该 fork 构建（Dockerfile / Gradle），生成镜像并部署。 | 需管理员开放 Fork，之后可自助 | Compose 里仍可只负责 Mongo/Rabbit/Redis；Raidar 用 Application 从 fork 构建并运行。或：在 fork 里用 GitHub Actions 构建镜像并推到 Docker Hub/GHCR，compose 再拉该镜像。 |
| **3. 本地构建镜像并推到自己的 Docker Hub** | 在本地（或 CI）从 medical-server 源码执行 `./gradlew bootBuildImage`，打镜像后推到**你自己的** Docker Hub（或 GHCR），compose 改为使用该镜像。 | **可以，完全自助** | 不需要公司 GitHub 管理员、不需要 Fork。Docker Hub 与 GitHub 是两套账号；你只要有 medical-server 的**源码**（已能克隆）即可构建，推到你自己的 Docker Hub 不需要公司授权。 |

**结论**：若希望**尽快打通部署、少依赖他人**，优先用 **方案 3**：本地构建 → 推到自己 Docker Hub（或 GHCR）→ 修改 compose 中的 `image:` → 在 Dokploy 配置该 Registry（若设为私有）。

---

## 方案 3 的具体步骤（推荐）

1. **在本地（或已有 CI）构建 Raidar 镜像**  
   在 medical-server 的 `app/` 目录下执行（见 [medical-server/app/README.md](../../medical-server/app/README.md)）。  
   **注意**：若本地 Docker 使用 credsStore 等，构建可能报 `'username' must not be null`，需先设置环境变量再构建：
   ```bash
   export DOCKER_HUB_USERNAME=<你的DockerHub用户名>
   cd /path/to/medical-server/app
   ./gradlew bootBuildImage --imageName=<你的DockerHub用户名>/raidar:server-latest
   ```
   若用 GitHub Container Registry，可改为 `ghcr.io/<你的GitHub用户名>/raidar:server-latest`。  
   **逐步命令清单**见 [docs/build-and-push-raidar-image.md](../docs/build-and-push-raidar-image.md)。

2. **推送到镜像仓库**  
   ```bash
   docker login   # 登录你的 Docker Hub 或 ghcr.io
   docker push <你的DockerHub用户名>/raidar:server-latest
   ```

3. **修改 compose 中的镜像名**  
   将 `configs/docker-compose-medical-server.yml` 里 raidar 的 `image:` 改为你刚推送的镜像，例如：
   `image: <你的DockerHub用户名>/raidar:server-latest`。  
   若使用私有仓库，需在 Dokploy 的 **Settings → Registries** 中配置该 Docker Hub（或 GHCR）账号。

4. **重新 Deploy**  
   将修改后的 compose 推送到 deploy-spike（或 Raw），在 Dokploy 中重新 Deploy。

---

## 方案 2（Fork + 构建）简述

- 管理员开放 Fork 后，将 medical-server fork 到个人 GitHub。
- **方式 A**：在 Dokploy 中为该栈只部署 Mongo/Rabbit/Redis（Compose），Raidar 单独用 **Application** 从 fork 的 repo 构建并部署（需 Dokploy 支持从 GitHub 构建）。
- **方式 B**：在 fork 中配置 GitHub Actions，在 push 时构建镜像并推到 Docker Hub/GHCR，compose 中 raidar 的 `image:` 指向该新镜像。

---

## 方案 1（公司 GitHub + Dokploy）简述

- 由组织管理员在 GitHub 安装 Dokploy 的 GitHub App 并授权相应仓库。
- 若公司希望**由 Dokploy 从官方 medical-server 仓库构建**：在 Dokploy 中用 Application 从该 repo 构建，不再使用 ghoshorn/raidar 镜像；或由公司 CI 构建并推送到公司镜像仓库，compose 引用该仓库中的镜像并在 Dokploy 配置 Registry。
