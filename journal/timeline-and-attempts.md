# 时间线与尝试记录

按「时间 | 目标 | 操作 | 结果 | 证据 | 结论 | 下一步」记录每次操作；并标注结论类型：**Confirmed**（已验证）/ **Inferred**（推断）/ **Unknown**（未确认）。

---

## 记录模板（每条可复制使用）

```markdown
### YYYY-MM-DD HH:MM — 简短标题

- **目标**：
- **操作**：
- **结果**：（成功 / 失败 / 部分成功）
- **证据**：（截图路径、错误日志片段、链接）
- **结论**：Confirmed / Inferred / Unknown
- **下一步**：
```

---

## 前置条件检查结果（2026-03-10）

| 检查项       | 结果说明 |
|--------------|----------|
| 当前环境     | macOS (Darwin)，非 Linux VPS |
| Docker       | 已安装 v28.0.1；沙箱内无法访问 daemon，正常终端可用 |
| 端口 3000/80/443 | 本机未被监听，可用于将来在 VPS 上安装 |
| 官方安装脚本 | 仅支持 Linux、须 root；在 Mac 上执行会报错退出 |
| 结论         | 安装须在 Linux VPS 上执行；Mac 仅能做文档与配置准备 |

---

## 已尝试过的路径与错误方案

（随实际操作在此补充：哪些方案试过、报什么错、为什么放弃。）

| 日期       | 方案简述           | 结果   | 放弃/保留原因 |
|------------|--------------------|--------|----------------|
| 2026-03-10 | 在 Mac 本机运行官方安装脚本 `curl ... \| sh` | 失败   | 脚本要求 root，且明确拒绝 Darwin，仅支持 Linux |
| 2026-03-10 | 在 Dokploy Cloud 界面直接 Create project | 失败   | 托管版需订阅才有 Server；与 issue #157「not a paid hosted solution」不符，应改用自托管 |
| 2026-03-10 | 在 Cursor 环境自动执行 `brew install --cask multipass` | 未完成 | 安装需 sudo 密码，无交互环境无法输入；已记录于 [docs/mac-multipass-dokploy.md](../docs/mac-multipass-dokploy.md)，需在本机终端手动执行第一步 |
| 2026-03-10 | 在 Multipass VM 内执行安装脚本安装 Dokploy | 成功 | 输出 “Congratulations, Dokploy is installed!”；进入自托管面板继续后续步骤 |
| 2026-03-10 | Compose 部署时从 GitHub Select repository 选 medical-server | 不可见 | medical-server 为公司组织私有库，个人 GitHub 连接后看不到；改用个人有权限的 repo（如 deploy-spike）作 compose 来源，或 Raw/组织授权/Fork，见 operating-guide |
| 2026-03-10 | Deploy Compose 拉取 ghoshorn/raidar:server-latest | 失败   | pull access denied；该镜像私有/不可用，需自建并推送或改用 Fork/公司构建，见 findings/raidar-image-source.md |
| 2026-03-10 | 本地执行 bootBuildImage（方案 3）步骤 2 | 失败   | 构建到 EXPORTING 阶段报错：`content digest sha256:...: not found`；Docker 使用 containerd 存储时与 buildpacks 导出不兼容。处理：预拉 run 镜像、或在 Docker Desktop 关闭 containerd、或清理 pack 缓存后重试，见 docs/build-and-push-raidar-image.md |
| 2026-03-10 | 按文档清理缓存后再次 bootBuildImage | 失败   | 清理后 `docker pull` 在 Mac arm64 上拉得 run/builder 为 **linux/arm64**，构建请求 **linux/amd64**，报错：`Image platform mismatch ... Requested platform 'linux/amd64' but got 'linux/arm64'`。处理：build.gradle 增加 `imagePlatform = "linux/amd64"`；预拉时使用 `docker pull --platform linux/amd64 ...`，见 docs/build-and-push-raidar-image.md |
| 2026-03-10 | 预拉 amd64 镜像后 / 删镜像后由构建拉取，再执行 bootBuildImage | 失败   | 两种方式均在 **EXPORTING** 再次报错 `content digest sha256:5d23b2b93067195845d0fd3fa7ce2e88...: not found`，与 Docker Desktop containerd 存储兼容问题一致。 |
| 2026-03-10 | 引入方案 B：Dockerfile 构建 | 已落实 | 在 medical-server/app 新增 Dockerfile + .dockerignore，用 `docker build --platform linux/amd64` 构建镜像，不经过 Buildpacks 导出；docs/build-and-push-raidar-image.md 将方案 B 置于文首并推荐在 EXPORTING 反复失败时使用；README 补充 Dockerfile 说明。 |
| 2026-03-10 | 方案 B Dockerfile 构建 | 成功   | 在本机执行 `docker build --platform linux/amd64 -t yangbingjia1206/raidar:server-latest .` 构建成功。 |
| 2026-03-10 | 恢复 build.gradle 至 main 后再次方案 B 构建 | 成功   | 将 medical-server/app/build.gradle 恢复为与 main (edd8be23) 一致并提交；再次用 Dockerfile 构建验证通过，确认无需在 build.gradle 中保留 bootBuildImage 专用配置。 |
| 2026-03-10 | 方案 3 步骤 4–5：推送镜像并修改 Compose 重新部署 | 成功   | `docker push yangbingjia1206/raidar:server-latest` 成功；compose 中 raidar 的 image 已改为该镜像，在 Dokploy 中完成重新 Deploy。 |
| 2026-03-11 | Dokploy Deploy 报 no matching manifest (arm64) | 失败   | Dokploy（Multipass VM）为 `linux/arm64`，拉取 `yangbingjia1206/raidar:server-latest` 报 `no matching manifest for linux/arm64/v8`，说明该 tag 只有 amd64。需用 `docker buildx build --platform linux/amd64,linux/arm64 --push` 推送 multi-arch 或至少推送 arm64。 |
| 2026-03-11 | Dokploy Compose Deployed ✅ | 成功 | Dokploy 日志显示镜像均 Pulled、网络/卷创建完成、容器（mongodb/redis/rabbitmq/raidar）均 Started，最终输出 `Docker Compose Deployed: ✅`。下一步：初始化 MongoDB rs0 并验收 Swagger。 |
| 2026-03-11 | Raidar 容器启动失败（仍是 GridFsTemplate 注入冲突） | 失败 | 下载 raidar logs（`raidar-20260311_075328.log.txt`）显示仍报：`expected single matching bean but found 2: masterTenantGridFsTemplate, subTenantGridFsTemplate`。结论：Dokploy 仍在运行旧镜像；需用包含修复 commit 的源码重新 buildx build 并 --push（建议换 tag 或确保强制拉取）。 |
| 2026-03-11 | Raidar 容器启动失败（ConversionService 注入冲突） | 失败 | 下载 raidar logs（`raidar-20260311_081135.log.txt`）显示：`ThymeleafService required a single bean, but 2 were found: templateFormattingConversionService,mvcConversionService`；同时 Mongo 仍提示 `REPLICA_SET_GHOST`（rs0 未初始化）。结论：镜像已更新但应用又遇到新的 Spring 启动阻塞；需在 `ThymeleafService` 注入点加 `@Qualifier("templateFormattingConversionService")` 并重新构建推送；rs0 初始化仍是后续必做步骤。 |
| （示例）   | 用内置 MongoDB     | 失败   | 无法开 rs0     |

---

## 最终推荐方案摘要

（在探索结束后填写：采用哪种部署方式、关键配置、文档索引。）

- 推荐方式：自建 Raidar 镜像时优先使用 **方案 B（Dockerfile 构建）**，已在恢复 build.gradle 至 main 后验证通过；方案 A（bootBuildImage）在 Docker Desktop + containerd 下易在 EXPORTING 失败，可作备选。
- 关键配置/文档：见 [docs/build-and-push-raidar-image.md](../docs/build-and-push-raidar-image.md)（含方案 A/B）、[configs/docker-compose-medical-server.yml](../configs/docker-compose-medical-server.yml)、medical-server/app/Dockerfile。medical-server 的 `app/build.gradle` 已恢复为与 main 一致，无需 bootBuildImage 专用配置。
- 已知 blocker / 后续事项：部署到 Dokploy 后仍需完成 MongoDB rs0 初始化、可选数据导入与验证。
