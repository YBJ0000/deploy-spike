# 本地构建并推送 Raidar 镜像到 Docker Hub（方案 3）

用于在本地从 `medical-server` 构建 Docker 镜像并推送到你自己的 Docker Hub，供 Dokploy 拉取。  
Docker Hub 用户名示例：`yangbingjia1206`，请按需替换。

**两种构建方式**：**方案 A** 使用 Spring Boot 的 `bootBuildImage`（Cloud Native Buildpacks）；**方案 B** 使用 `medical-server/app` 下的 **Dockerfile**（`docker build`）。  
若方案 A 在 **EXPORTING** 阶段反复报错 `content digest ... not found`（Docker Desktop + containerd 的已知问题），请**改用方案 B**，无需关闭 containerd 即可成功构建并推送。  
**方案 B 已在实际环境中验证**：在将 `medical-server/app/build.gradle` 恢复为与 main 一致（无 bootBuildImage 专用配置）后，使用 Dockerfile 构建并推送镜像成功。

---

## 前置条件

- **Docker Desktop（或 Docker 引擎）已启动**，终端可执行 `docker info`
- 已克隆 **medical-server** 仓库，能进入 `medical-server/app` 目录  
  （方案 A 还需本机 **Java 21**；方案 B 由镜像内 JDK 构建，本机可不装 Java）

---

# 方案 B：Dockerfile 构建（推荐，已验证）

不经过 Buildpacks，直接 `docker build`，可规避「content digest not found」等导出问题。在 Mac + Docker Desktop、`build.gradle` 与 main 一致的前提下已成功构建并推送。

### 步骤 B1：在 medical-server/app 下构建镜像

```bash
cd /path/to/medical-server/app
docker build --platform linux/amd64 -t yangbingjia1206/raidar:server-latest .
```

- 首次构建会拉取 `eclipse-temurin` 镜像并执行 Gradle 编译，耗时可能数分钟。
- 若需带测试构建，可先在本机执行 `./gradlew test` 通过后，再使用当前 Dockerfile（默认 `-x test` 以加快镜像构建）。

### 步骤 B2：登录并推送

```bash
docker login
docker push yangbingjia1206/raidar:server-latest
```

### 步骤 B3：修改 Compose 并重新部署

与下方「步骤 5」相同：将 compose 中 raidar 的 `image:` 改为 `yangbingjia1206/raidar:server-latest`，在 Dokploy 中重新 Deploy。

---

# 方案 A：bootBuildImage（Buildpacks）构建

若你希望继续使用 Spring Boot 官方 Buildpacks 构建，可按以下步骤；若在 **EXPORTING** 阶段反复失败，请改用上方**方案 B**。

## 步骤 1：设置 Docker Hub 用户名（必做）

Spring Boot 的 `bootBuildImage` 在部分环境下会因读不到 Docker 配置里的 username 而报错 `'username' must not be null`，因此需要显式设置环境变量：

```bash
export DOCKER_HUB_USERNAME=yangbingjia1206
```

（若使用其他 Docker Hub 账号，改为你的用户名。）

---

## 步骤 2：在 medical-server/app 下构建镜像

**建议先做（可减少导出失败与平台冲突）**：在构建前按 **linux/amd64** 预拉镜像（镜像用于部署到常见 VPS/Dokploy，且与 build 请求的平台一致；在 Mac arm64 上若不加 `--platform` 会拉成 arm64 导致平台不匹配）：

```bash
docker pull --platform linux/amd64 paketobuildpacks/run-jammy-base:latest
docker pull --platform linux/amd64 paketobuildpacks/builder-jammy-base:latest
```

然后执行构建：

```bash
cd /path/to/medical-server/app
./gradlew bootBuildImage --imageName=yangbingjia1206/raidar:server-latest
```

- 首次构建会拉取 Paketo 构建镜像，耗时可能较长。
- 若报错 **Connection to the Docker daemon ... refused**：请先启动 Docker Desktop（或本机 Docker 服务）。
- 若仍报 **'username' must not be null**：确认已执行步骤 1 的 `export DOCKER_HUB_USERNAME=...`。
- 若在 **EXPORTING** 阶段报错 **content digest ... not found**：见下方「步骤 2 导出失败（content digest not found）」。
- 若报错 **Image platform mismatch ... Requested platform 'linux/amd64' but got 'linux/arm64'**：见下方「步骤 2 平台不匹配（arm64 vs amd64）」。

### 步骤 2 导出失败：`content digest ... not found`

若构建在 **EXPORTING** 阶段报错类似：

```text
ERROR: failed to export: saving image: failed to fetch base layers: ... unable to create manifests file: NotFound: content digest sha256:...: not found
```

这是 Cloud Native Buildpacks 把镜像写入本地 Docker 时，与 **Docker 使用 containerd 存储** 的已知兼容问题（常见于 Docker Desktop 默认设置、Mac arm64）。

**按顺序尝试：**

1. **预拉 run 镜像后再构建**（有时可避免层缺失；在 Mac 上请加 `--platform linux/amd64`，见下方「平台不匹配」）：
   ```bash
   docker pull --platform linux/amd64 paketobuildpacks/run-jammy-base:latest
   docker pull --platform linux/amd64 paketobuildpacks/builder-jammy-base:latest
   ```
   然后重新执行步骤 2 的 `./gradlew bootBuildImage ...`。

2. **关闭 Docker Desktop 的 containerd 存储**（多数情况下可彻底解决）：
   - 打开 **Docker Desktop** → **Settings**（设置）→ **General**（或 **Docker Engine**）
   - 找到并**取消勾选**「Use the containerd image store」/「Use containerd for pulling and storing images」等与 containerd 相关的选项
   - 应用并重启 Docker，再重新执行步骤 2

3. **清理后重试**：若仍失败，可清理未使用镜像与 buildpacks 缓存后再构建。**注意**：清理后预拉时在 Mac（arm64）上必须加 `--platform linux/amd64`，否则会拉成 arm64，构建时报「平台不匹配」：
   ```bash
   docker image prune -a -f
   docker volume ls | grep pack
   # 若有 pack-cache-*、pack-layers-* 等，可删除：docker volume rm <卷名>
   docker pull --platform linux/amd64 paketobuildpacks/run-jammy-base:latest
   docker pull --platform linux/amd64 paketobuildpacks/builder-jammy-base:latest
   ```
   然后再次执行 `./gradlew bootBuildImage ...`。

4. **改用方案 B（Dockerfile 构建）**：若上述 1–3 仍无法解决（尤其在 Docker Desktop 使用 containerd 存储时），请直接使用本文档开头的 **方案 B**：在 `medical-server/app` 下执行 `docker build --platform linux/amd64 -t <用户名>/raidar:server-latest .`，再 `docker push`。Dockerfile 不经过 Buildpacks 导出，可规避该问题。

### 步骤 2 平台不匹配：`Requested platform 'linux/amd64' but got 'linux/arm64'`

若报错类似：

```text
Image platform mismatch detected. The configured platform 'linux/amd64' is not supported by the image '...run-jammy-base:latest'. Requested platform 'linux/amd64' but got 'linux/arm64'
```

**原因**：在 Mac（Apple Silicon/arm64）上执行 `docker pull` 时未指定平台，Docker 会拉取 **arm64** 版本；而构建任务需要 **linux/amd64**（用于部署到常见 VPS/Dokploy），因此出现不匹配。

**处理**（任选其一）：

1. **按 amd64 重新拉取后再构建**（推荐）：
   ```bash
   docker pull --platform linux/amd64 paketobuildpacks/run-jammy-base:latest
   docker pull --platform linux/amd64 paketobuildpacks/builder-jammy-base:latest
   ```
   然后重新执行步骤 2 的 `./gradlew bootBuildImage ...`。  
   （`medical-server/app/build.gradle` 中已设置 `imagePlatform = "linux/amd64"`，构建会使用上述 amd64 镜像。）

2. **删掉本地 arm64 镜像，让构建自己拉 amd64**：
   ```bash
   docker rmi paketobuildpacks/run-jammy-base:latest paketobuildpacks/builder-jammy-base:latest 2>/dev/null || true
   ```
   然后直接执行 `./gradlew bootBuildImage ...`，插件会按 `imagePlatform` 拉取 amd64 镜像。

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
| 方案 A 步骤 2 在 **EXPORTING** 反复报错 `content digest ... not found` | **推荐**：改用 **方案 B（Dockerfile 构建）**，见文档开头。或尝试关闭 Docker containerd 存储、预拉 amd64 镜像、清理 pack 缓存后重试。 |
| `'username' must not be null` | 执行 `export DOCKER_HUB_USERNAME=yangbingjia1206` 后再执行方案 A 步骤 2。 |
| `Connection to the Docker daemon ... refused` | 启动 Docker Desktop 或本机 Docker 服务。 |
| 方案 A 步骤 2 报错 `Image platform mismatch ... linux/amd64 ... but got linux/arm64` | 见上文「步骤 2 平台不匹配」：用 `docker pull --platform linux/amd64 ...` 预拉，或删掉本地 run/builder 镜像后重新构建。 |
| `pull access denied` / `denied: requested access to the resource is denied` | 先执行 `docker login`，再执行推送。 |
| Gradle 构建失败（编译/测试） | 确认 JDK 21、网络正常，参考 `medical-server/app/README.md`。方案 B 在镜像内构建，可不装本机 Java。 |

---

更多背景与方案对比见 [findings/raidar-image-source.md](../findings/raidar-image-source.md)。
