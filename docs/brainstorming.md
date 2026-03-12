# medical-server 部署到 Dokploy 头脑风暴（精简版）

## 目标

把 medical-server（Raidar Server）部署到**自托管版 Dokploy**，产出一份可直接照着实施的方案。范围见 [issue157.md](./issue157.md)。

---

## 一、核心结论

### 1. 唯一难点：MongoDB Replica Set（rs0）

- 应用依赖 **MongoDB 副本集**（事务支持），本地 dev 用单节点 rs0。
- **Dokploy 内置 MongoDB 服务**不支持直接开 rs0，或开了也容易因 hostname 不匹配连不上。
- **做法**：用**自定义 Mongo 服务**（非内置 Database），可参考 Checkmate 的 Mongo 配置。
- **必须满足**：
  - 连接字符串带 `?replicaSet=rs0`
  - 服务名固定为 `mongodb`，这样 `rs.initiate()` 里的 hostname 才对。

### 2. RabbitMQ 与 Redis：用内置即可

- 直接用 Dokploy 的 **Database 服务**（RabbitMQ / Redis）。
- 默认 `guest/guest`、`redis://redis:6379` 即可，走内部网络，不暴露公网。
- 与项目现有 Docker 示例（外部连接）兼容。

### 3. 整体部署 vs 分开部署

- **推荐：整体部署**。Dokploy 支持 `docker-compose.yml`，一次性导入 app + mongo + rabbit + redis。
- 不推荐拆成多个“应用”再用 Advanced → Custom Command 拼 Mongo，维护成本高。

### 4. 代码来源：GitHub 优先

- 从 **GitHub 仓库**拉代码在 Dokploy 里构建，便于迭代；若已有稳定镜像也可用 Docker Hub，按需选择。

---

## 二、应用与依赖一览

| 组件        | 建议方式           | 备注 |
|-------------|--------------------|------|
| Raidar Server | Compose / 单应用   | JDK 21，需连 Mongo/Rabbit/Redis |
| MongoDB     | 自定义服务（Compose 内） | 必须 rs0，服务名 `mongodb` |
| RabbitMQ    | Dokploy 内置 Database | 默认 guest/guest |
| Redis       | Dokploy 内置 Database | redis://redis:6379 |

还需考虑：环境变量、内存限制、数据导入（mongorestore）、Swagger UI 测试等。

---

## 三、文档与记录规范

### 建议目录结构（deploy-spike 内）

```
deploy-spike/
├── README.md
├── brainstorming.md          # 本文档
├── operating-guide.md       # 浏览器一步步操作 Dokploy
├── journal/                  # 时间线、每次尝试
│   ├── timeline-and-attempts.md
│   └── 2026-03-10-session-1.md
├── findings/                 # 结论与方案
│   ├── repo-runtime-summary.md
│   ├── recommended-setup.md
│   └── blockers-and-risks.md
├── configs/                  # 配置草案
│   ├── env-template.md
│   └── docker-compose-notes.md
└── evidence/                 # 截图、错误日志片段
    ├── screenshots/
    └── error-snippets.md
```

### 单条记录格式

每条记录建议包含：**时间 | 目标 | 操作 | 结果 | 证据 | 结论 | 下一步**。  
并注明**为什么**选某方案、为什么排除另一方案；结论标注来源：**Confirmed**（已验证）/ **Inferred**（代码/文档推断）/ **Unknown**（未确认）。

### 敏感信息

- 文档中只写占位符，如 `DB_PASSWORD=<在 Dokploy 中设置>`，不写真实密钥。
- 截图前打码，日志中敏感值替换为占位符。.gitignore 禁止提交真实密钥。

---

## 四、前置与后续事项

### 非 Dokploy 内完成的前置

- VPS 规格、防火墙端口、DNS 指向、反向代理/TLS 由谁接管、备份/监控策略等，在文档中标注为「VPS/infra 前置」或「需应用团队补充」。

### 可运行 vs 可上线

- **最小可运行**：Dokploy 自托管 + 一个 app + Mongo（rs0）+ Rabbit + Redis + 环境变量 + 健康检查。
- **生产加固**：备份、监控告警、密钥轮换、多环境隔离、灰度/回滚、资源限制与扩缩容——作为后续任务列在 blockers-and-risks 中。

---

## 五、下一步文档

- **operating-guide.md**：按浏览器操作顺序写，包括安装 Dokploy（SSH）、在 Dokploy 里创建项目/应用、填配置、Deploy、Mongo rs0 配置、数据导入（在 Dokploy 终端用 mongosh/mongorestore）等，随探索更新。
- **journal/timeline-and-attempts.md**：记录尝试过的路径、错误方案及放弃原因、最终推荐方案。
