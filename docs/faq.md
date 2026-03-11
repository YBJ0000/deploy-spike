# 常见问题（FAQ）

本文档汇总自托管 Dokploy 部署 medical-server 时常见的问题与简要说明，详细步骤见 [operating-guide.md](../operating-guide.md)。

---

## 账号与数据存在哪里？销毁 VM 会怎样？

自托管版 Dokploy 的**账号、密码、项目、部署配置等**都保存在**你安装 Dokploy 的那台机器上**（安装脚本会在该机上部署 PostgreSQL 和 Redis，数据就存在那里），**不会**同步到 Dokploy 官方的任何数据库或云端。

- **本机 Multipass VM**：账号和数据都在 VM 内的 PostgreSQL 里；只有你这台 Mac 能访问该实例。
- **销毁该 VM 后**：该实例上的所有账号和数据都会丢失；之前创建的管理员账号也就“不存在了”（因为整台机器没了）。若需要保留，应在删除 VM 前做好数据库或卷的备份。

---

## 本机 VM：外部能访问吗？合上盖子会怎样？

### 外部/公网能否访问本机 VM 里部署的后端？

本机 Multipass VM 的 IP 是**私网地址**（如 `192.168.64.x`），只在你这台 Mac（以及通常同一局域网）内可访问，**互联网上的其他设备不能直接访问**该 IP。

若要让「外部网站/请求」访问你在 VM 里部署的 Java 后端，需要至少其一：

- 在路由器上做**端口映射**（把公网 IP 的某端口转到你 Mac，再在 Mac 上转发到 VM），并保证 Mac 有固定内网 IP；或
- 使用**内网穿透/隧道**（如 ngrok、Cloudflare Tunnel）把本机端口暴露到公网；或
- 把应用部署到**有公网 IP 的 Linux VPS**，由 VPS 对外提供服务。

因此：**本机 VM 适合做本地验证、联调；要做成“外部随时可访问”的服务，通常需 VPS 或隧道。**

### 合上 MacBook 盖子、电脑休眠之后呢？

Mac 休眠后，Multipass VM 也会随之挂起或停止，**VM 里的服务（包括 Dokploy、你部署的后端）会不可用**，外部无法访问。

若需要 **7×24 持久化**（随时可被外部访问、不因合盖/关机中断），要么：

- **保持 Mac 常开、不合盖休眠**（且做好端口映射或隧道），要么
- **租用一台 Linux VPS**，在 VPS 上安装 Dokploy 并部署应用，由 VPS 一直运行。

**当前阶段**：本机 VM 主要用于**探索可行方案、验证部署流程**；确认流程可行后，若需对外稳定提供服务，再迁到 VPS 或使用隧道即可。

---

## 没有远程 Linux VPS 时怎么办？Mac 能否获得同等效果？必须订阅 Dokploy 吗？

可以不用 VPS，也**不需要**订阅 Dokploy Cloud。在 Mac 上用 **Multipass** 起一台 Linux 虚拟机，在 VM 内执行同样的 Dokploy 安装脚本，效果等同于「有一台 Linux 服务器」；issue #157 要求 self-hosted、not paid hosted，因此**不要**订阅 Hobby/Startup 等计划。

详细区别与建议下一步见 [findings/self-hosted-vs-cloud.md](../findings/self-hosted-vs-cloud.md)。

---

## Raidar 镜像拉取失败（ghoshorn/raidar 不可用）时怎么办？

若 Deploy 报错：`pull access denied for ghoshorn/raidar` 或 `repository does not exist`，说明当前无法使用该镜像（私有且无权限，或该仓库已不可用）。需**换用其他可拉取的 Raidar 镜像**，例如在本地从 medical-server 源码构建并推送到你自己的 Docker Hub，再把 compose 里 raidar 的 `image:` 改为该镜像。

完整方案与逐步命令见 [findings/raidar-image-source.md](../findings/raidar-image-source.md) 与 [docs/build-and-push-raidar-image.md](./build-and-push-raidar-image.md)。

---

## Select repository 里没有 medical-server（组织私有库）时怎么办？

Dokploy 的 Provider 若选 **GitHub**，则「Select repository」只会列出**当前已连接 GitHub 账户有权限**的仓库。若 medical-server 在公司组织下且为 private，个人账号连接后不会在列表里看到该 repo。

**重要**：Compose 里只用到了公开镜像（mongo、rabbitmq、redis、raidar 镜像），**不必**从 medical-server 仓库取 compose；只要有一份 YAML 即可，来源可以是任意你有权限的 repo（例如 **deploy-spike**）。把 `configs/docker-compose-medical-server.yml` 放到该 repo，在 Dokploy 的 Compose Path 中填写该路径即可。

更多方案（Raw、组织授权、Fork）见 [operating-guide.md](../operating-guide.md) 第三步「Select repository 里没有 medical-server 时怎么办」。

---

## Raidar 容器启动失败（APPLICATION FAILED TO START / expected single matching bean）怎么办？

通常是应用启动阶段的 **Spring 依赖注入冲突**（同类型 Bean 存在多个），需要通过修改 medical-server 代码并重新构建/推送镜像解决。同一份日志里也常伴随 MongoDB rs0 未初始化的报错，需先完成 operating-guide 中的 rs0 初始化步骤。

排查步骤与常见错误（GridFsTemplate、ConversionService 等）见 [findings/raidar-startup-failures.md](../findings/raidar-startup-failures.md)。

---

## 构建/推送 Raidar 镜像时常见错误（EXPORTING 失败、平台不匹配、pull access denied）

| 现象 | 处理 |
|------|------|
| 方案 A 在 **EXPORTING** 反复报错 `content digest ... not found` | 改用 **方案 B（Dockerfile 构建）**，见 [docs/build-and-push-raidar-image.md](./build-and-push-raidar-image.md) 文档开头。 |
| `Image platform mismatch ... linux/amd64 ... but got linux/arm64` | 用 `docker pull --platform linux/amd64 ...` 预拉，或删掉本地 run/builder 镜像后重新构建。 |
| `pull access denied` / `denied: requested access to the resource is denied` | 先执行 `docker login`，再执行推送。 |
| Dokploy 报 `no matching manifest for linux/arm64` | 推送 multi-arch 镜像（amd64+arm64）或至少推送 arm64，见 build-and-push-raidar-image 中的「为 arm64 / 多架构推送镜像」。 |

完整表格与说明见 [docs/build-and-push-raidar-image.md](./build-and-push-raidar-image.md) 末尾「常见错误」。
