# deploy-spike

本仓库用于研究和记录如何将 **medical-server**（Raidar Server）部署到自托管版 [Dokploy](https://dokploy.com/)，并产出一份可直接实施的方案。任务范围见 [docs/issue157.md](./docs/issue157.md)。

若要在 MacBook 上安装 Dokploy 并部署 medical-server，请直接看 [operating-guide.md](./operating-guide.md)。

## 目录结构

| 目录/文件 | 说明 |
|-----------|------|
| **operating-guide.md** | 操作指南精简版：从准备环境到部署与验收的步骤摘要 |
| **docs/** | 任务说明、操作文档、FAQ、构建与 Multipass 等说明 |
| **journal/** | 时间线、会话记录、分支代码变更整理等 |
| **findings/** | 自托管 vs Cloud、镜像来源、启动失败排查等结论 |
| **configs/** | Docker Compose 等配置草案 |
| **downloaded-logs/** | 从 Dokploy 下载的 raidar 容器日志（排错用） |

## 文档索引

| 文档 | 说明 |
|------|------|
| [docs/issue157.md](./docs/issue157.md) | 任务目标、范围与完成标准 |
| [docs/brainstorming.md](./docs/brainstorming.md) | 头脑风暴精简版：核心结论、部署方式、文档规范 |
| [operating-guide.md](./operating-guide.md) | 操作指南**精简版**：浏览器与终端操作顺序 |
| [docs/original-operating-guide.md](./docs/original-operating-guide.md) | 操作指南完整版（含前置条件、FAQ 引用与排错细节） |
| [docs/faq.md](./docs/faq.md) | 常见问题（账号与数据、外网访问、合盖休眠、镜像/仓库/启动失败等） |
| [docs/mac-multipass-dokploy.md](./docs/mac-multipass-dokploy.md) | Mac + Multipass 安装 Dokploy 逐步命令清单（含排错记录） |
| [docs/build-and-push-raidar-image-simple.md](./docs/build-and-push-raidar-image-simple.md) | 本地构建并推送 Raidar 镜像（精简版） |
| [docs/build-and-push-raidar-image.md](./docs/build-and-push-raidar-image.md) | 构建镜像完整版与排错（方案 A/B、常见错误表） |
| [journal/timeline-and-attempts.md](./journal/timeline-and-attempts.md) | 时间线、尝试过的方案与错误记录 |
| [journal/2026-03-10-session-1.md](./journal/2026-03-10-session-1.md) | 按会话记录的具体操作与结论 |
| [journal/medical-server-spike-dokploy-deployment-branch-changes.md](./journal/medical-server-spike-dokploy-deployment-branch-changes.md) | medical-server 分支 spike/dokploy-deployment 相对 main 的代码变更整理 |
| [findings/self-hosted-vs-cloud.md](./findings/self-hosted-vs-cloud.md) | 自托管 vs Cloud、与 issue #157 符合性、Mac 用 Multipass 获得 Linux |
| [findings/raidar-image-source.md](./findings/raidar-image-source.md) | Raidar 镜像拉取失败时的方案（自建推送 / Fork / 公司 GitHub） |
| [findings/raidar-startup-failures.md](./findings/raidar-startup-failures.md) | Raidar 容器启动失败排查（GridFsTemplate / ConversionService 等） |
| [configs/docker-compose-medical-server.yml](./configs/docker-compose-medical-server.yml) | medical-server 栈的 Compose 草案（Mongo rs0 + RabbitMQ + Redis + Raidar） |
