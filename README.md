# deploy-spike

本仓库用于研究和记录如何将 **medical-server**（Raidar Server）部署到自托管版 [Dokploy](https://dokploy.com/)，并产出一份可直接实施的方案。任务范围见 [issue157.md](./issue157.md)。

## 文档索引

| 文档 | 说明 |
|------|------|
| [brainstorming.md](./brainstorming.md) | 头脑风暴精简版：核心结论、部署方式、文档规范 |
| [operating-guide.md](./operating-guide.md) | 浏览器操作 Dokploy 的逐步指南（随探索更新） |
| [journal/timeline-and-attempts.md](./journal/timeline-and-attempts.md) | 时间线、尝试过的方案与错误记录 |
| [journal/2026-03-10-session-1.md](./journal/2026-03-10-session-1.md) | 按会话记录的具体操作与结论 |
| [findings/self-hosted-vs-cloud.md](./findings/self-hosted-vs-cloud.md) | 自托管 vs Cloud、与 issue #157 符合性、Mac 用 Multipass 获得 Linux |
| [docs/mac-multipass-dokploy.md](./docs/mac-multipass-dokploy.md) | Mac + Multipass 安装 Dokploy 逐步命令清单（含排错记录） |
| [configs/docker-compose-medical-server.yml](./configs/docker-compose-medical-server.yml) | medical-server 栈的 Compose 草案（Mongo rs0 + RabbitMQ + Redis + Raidar） |

后续将补充：`findings/` 内推荐方案与风险、`configs/` 内环境变量说明、`evidence/`（截图与错误片段）。