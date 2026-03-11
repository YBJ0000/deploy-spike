# 本地构建并推送 Raidar 镜像到 Docker Hub（方案 3）

用于在本地从 `medical-server` 构建 Docker 镜像并推送到你自己的 Docker Hub，供 Dokploy 拉取。  
Docker Hub 用户名示例：`yangbingjia1206`，请按需替换。

---

## 前置条件

- **Docker Desktop（或 Docker 引擎）已启动**，终端可执行 `docker info`
- **Java 21**（与 medical-server 一致）
- 已克隆 **medical-server** 仓库，能进入 `medical-server/app` 目录

---

## 步骤 1：设置 Docker Hub 用户名（必做）

Spring Boot 的 `bootBuildImage` 在部分环境下会因读不到 Docker 配置里的 username 而报错 `'username' must not be null`，因此需要显式设置环境变量：

```bash
export DOCKER_HUB_USERNAME=yangbingjia1206
```

（若使用其他 Docker Hub 账号，改为你的用户名。）

---

## 步骤 2：在 medical-server/app 下构建镜像

```bash
cd /path/to/medical-server/app
./gradlew bootBuildImage --imageName=yangbingjia1206/raidar:server-latest
```

- 首次构建会拉取 Paketo 构建镜像，耗时可能较长。
- 若报错 **Connection to the Docker daemon ... refused**：请先启动 Docker Desktop（或本机 Docker 服务）。
- 若仍报 **'username' must not be null**：确认已执行步骤 1 的 `export DOCKER_HUB_USERNAME=...`。

---

## 步骤 3：登录 Docker Hub（推送前）

若尚未登录，在终端执行：

```bash
docker login
```

按提示输入 Docker Hub 用户名和密码（或 Access Token）。成功后即可执行推送。

---

## 步骤 4：推送镜像到 Docker Hub

```bash
docker push yangbingjia1206/raidar:server-latest
```

若将镜像设为私有，需在 Dokploy 的 **Settings → Registries** 中配置该 Docker Hub 账号，否则拉取会失败。

---

## 步骤 5：修改 Compose 并重新部署

1. 将 `deploy-spike/configs/docker-compose-medical-server.yml` 中 **raidar** 的 `image:` 改为：
   ```yaml
   image: yangbingjia1206/raidar:server-latest
   ```
2. 将该 compose 的变更推送到 Dokploy 使用的仓库（或 Raw），在 Dokploy 中对该 Compose 服务点击 **Deploy** 重新部署。

---

## 常见错误

| 现象 | 处理 |
|------|------|
| `'username' must not be null` | 执行 `export DOCKER_HUB_USERNAME=yangbingjia1206` 后再执行步骤 2。 |
| `Connection to the Docker daemon ... refused` | 启动 Docker Desktop 或本机 Docker 服务。 |
| `pull access denied` / `denied: requested access to the resource is denied` | 先执行 `docker login`，再执行步骤 4。 |
| Gradle 构建失败（编译/测试） | 确认 JDK 21、网络正常，参考 `medical-server/app/README.md`。 |

---

更多背景与方案对比见 [findings/raidar-image-source.md](../findings/raidar-image-source.md)。
