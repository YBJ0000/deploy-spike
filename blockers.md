# Blockers 与解决方案速查

从 [journal/timeline-and-attempts.md](journal/timeline-and-attempts.md)、[journal/2026-03-10-session-1.md](journal/2026-03-10-session-1.md)、[journal/medical-server-spike-dokploy-deployment-branch-changes.md](journal/medical-server-spike-dokploy-deployment-branch-changes.md) 中提取的失败记录及对应解决/规避方式，便于快速查阅。

---

## 构建与镜像


| Blocker                                                                                               | 解决/规避                                                                                                                                                                                                      |
| ----------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **bootBuildImage** 在 EXPORTING 阶段报 `content digest ... not found`（Docker Desktop + containerd 兼容问题）   | 改用 **Dockerfile 构建**：在 medical-server/app 使用 `docker build --platform linux/amd64 -t <image> .`，不经过 Buildpacks。见 [docs/build-and-push-raidar-image-simple.md](docs/build-and-push-raidar-image-simple.md)。 |
| bootBuildImage 报 `Image platform mismatch ... Requested platform 'linux/amd64' but got 'linux/arm64'` | 预拉 `--platform linux/amd64` 或改 build.gradle；更稳妥是直接用 **Dockerfile 构建** 并指定 `--platform linux/amd64`。                                                                                                        |
| Dokploy（arm64 VM）拉取镜像报 `no matching manifest for linux/arm64`                                         | 使用 **buildx 多架构**：`docker buildx build --platform linux/amd64,linux/arm64 --push`，或至少推送 arm64 镜像。                                                                                                          |
| 拉取 `ghoshorn/raidar:server-latest` 报 **pull access denied**                                           | 镜像是私有/不可用；需**自建镜像并推送**到自有仓库（如 Docker Hub），Compose 中改用该镜像。见 [findings/raidar-image-source.md](findings/raidar-image-source.md)。                                                                             |


---

## 应用启动（Spring）


| Blocker                                                                                                                              | 解决/规避                                                                                                                                                                               |
| ------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Raidar 启动报 **expected single matching bean but found 2: masterTenantGridFsTemplate, subTenantGridFsTemplate**（FileService）           | 在 **FileService** 用显式构造器 + 参数上 `@Qualifier("masterTenantGridFsTemplate")` / `@Qualifier("subTenantGridFsTemplate")`，去掉 `@RequiredArgsConstructor`。已在 spike/dokploy-deployment 分支落实。 |
| Raidar 启动报 **required a single bean, but 2 were found: templateFormattingConversionService, mvcConversionService**（ThymeleafService） | 在 **ThymeleafService** 用显式构造器 + 参数上 `@Qualifier("templateFormattingConversionService")`，去掉 `@RequiredArgsConstructor`。已在 spike/dokploy-deployment 分支落实。                             |


详见 [journal/medical-server-spike-dokploy-deployment-branch-changes.md](journal/medical-server-spike-dokploy-deployment-branch-changes.md)、[findings/raidar-startup-failures.md](findings/raidar-startup-failures.md)。

---

## 环境与平台


| Blocker                                                                    | 解决/规避                                                                                                                             |
| -------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| 在 **Mac 本机**执行 Dokploy 官方安装脚本报错（须 root、仅支持 Linux）                          | 必须在 **Linux** 上安装：用 Multipass/UTM 起 Linux VM 或使用 VPS，在 VM 内执行 `curl -sSL https://dokploy.com/install.sh | sudo sh`。               |
| **Dokploy Cloud** 创建项目报 “No servers available, Please subscribe to a plan” | 与 issue #157「自托管、非付费托管」不符；改用 **自托管版**（在自己的 Linux 上安装 Dokploy），不订阅。                                                                |
| 自动化环境执行 `brew install --cask multipass` 失败（需 sudo 密码）                      | 在本机终端**手动**执行安装并输入密码；后续步骤再按 [docs/mac-multipass-dokploy.md](docs/mac-multipass-dokploy.md) 进行。                                    |
| Compose 从 GitHub **Select repository** 选 medical-server **不可见**            | 仓库为组织私有；改用个人有权限的 repo 作 compose 来源，或 Raw URL / 组织授权 / Fork。见 [operating-guide.md](operating-guide.md)、[docs/faq.md](docs/faq.md)。 |
| Multipass VM 在高负载下**崩溃/无法访问**（Mongo+RabbitMQ+Redis+Raidar 同时跑）             | 推断资源不足；**重建 VM** 并分配更高资源（建议 **4G RAM / 40G 磁盘**），见 [docs/mac-multipass-dokploy.md](docs/mac-multipass-dokploy.md)。                |


---

## Dokploy 与运维


| Blocker                                                                                             | 解决/规避                                                                                                               |
| --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| 用 Dokploy **Open Terminal** 想进 mongodb 容器执行 `mongosh` / `rs.initiate()`，终端里**没有 mongod、mongosh 命令** | Open Terminal 并非附着在 mongodb 容器；改用**宿主机**安装 mongosh，执行 `mongosh "mongodb://<VM 私网 IP>:27017"` 连接并执行 `rs.initiate()`。 |
| Dokploy 终端里执行 `redis-cli ping` 报 **command not found**                                              | 该终端不是 redis 容器；在**本机**执行 `redis-cli -h <VM IP> -p 6379 ping` 验收 Redis。                                              |
| **VM stop/start 后** Mongo 连不上（ECONNREFUSED），容器显示 Exited                                             | 容器不会随 VM 自动拉起；在 Dokploy 进入该 Compose 栈的 General 页，点击 **Reload**（或 Deploy）重新启动栈；数据在 Docker 卷中已持久化，无需重装或重新初始化。         |


---

## 小结

- **镜像构建**：bootBuildImage 在本地易在 EXPORTING 失败 → 用 **Dockerfile 构建** 解决。
- **应用启动**：多 Bean 注入歧义 → **FileService / ThymeleafService** 用 `@Qualifier` 显式指定。
- **Dokploy 终端**：不保证进入具体服务容器 → 对 Mongo/Redis 等用**宿主机客户端**连暴露端口。
- **VM 与稳定性**：资源不足易崩溃 → 使用 **4G/40G** VM；停启后需在 Dokploy 里 **Reload** 恢复栈。

