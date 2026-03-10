# Dokploy 实际操作流程（浏览器操作指南）

本文档针对 **自托管版 Dokploy**（符合 issue #157：self-hosted, not a paid hosted solution）。按**浏览器里用鼠标、键盘操作 Dokploy** 的顺序编写，随实际探索会持续更新。

> **重要**：若在界面上看到「No servers available, Please subscribe to a plan」，说明当前打开的是 **Dokploy Cloud（付费托管版）**，不是自托管实例。请改为访问**你自己安装的 Dokploy**（`http://<你的 Linux 服务器或 VM 的 IP>:3000`）。详见 [findings/self-hosted-vs-cloud.md](./findings/self-hosted-vs-cloud.md)。

---

## 前置条件（在动手点 Dokploy 之前）

- [ ] **已有一个 Linux 环境**：Dokploy 官方安装脚本**仅支持 Linux**（如 Ubuntu、Debian、CentOS、Fedora 等），**不支持 macOS**。可选两种方式获得 Linux：
  - **方式 A**：租用一台 Linux VPS，SSH 登录后在该机上安装；
  - **方式 B**：在 Mac 上用 **Multipass** 跑 Linux 虚拟机，在 VM 内安装。**逐步命令见** [docs/mac-multipass-dokploy.md](./docs/mac-multipass-dokploy.md)。
- [ ] 该 Linux 满足：至少 2GB 内存、30GB 磁盘；端口 **80、443、3000** 未被占用，防火墙已放行。
- [ ] 该 Linux 已安装 Docker（或由安装脚本自动安装）。
- [ ] 如需域名访问，DNS 已指向该机；TLS/反向代理按你方约定配置。

**本地 Mac 可做检查**：`docker --version` 确认 Docker 已装（用于本地构建等）；实际安装须在**上述 Linux 环境**内完成。

---

## 第一步：安装 Dokploy（须在 Linux 上执行）

安装**必须在 Linux 环境**内进行（可以是远程 VPS，或 Mac 上的 Linux 虚拟机），不能在本机 macOS 上直接运行官方脚本。

1. 获得该 Linux 的 shell：
   - **若用 VPS**：`ssh 用户名@服务器IP或域名`
   - **若用 Mac 上的 Multipass**：按 [docs/mac-multipass-dokploy.md](./docs/mac-multipass-dokploy.md) 创建 VM（如 `multipass launch 22.04 --name dokploy --memory 2G --disk 30G`），再用 `multipass shell dokploy` 进入。
2. 在该 Linux 上执行官方一键安装（需 root，通常加 `sudo`）：  
   ```bash
   curl -sSL https://dokploy.com/install.sh | sudo sh
   ```  
   脚本会自动检测最新稳定版并安装 Dokploy + PostgreSQL + Redis；若需指定版本可先设置 `export DOKPLOY_VERSION=v0.26.6` 再执行上述命令。详见 [Dokploy 官方安装文档](https://docs.dokploy.com/docs/core/installation)。
3. 安装完成后在浏览器访问：`http://<该 Linux 的 IP>:3000`（VPS 用公网 IP，Multipass VM 用 `multipass list` 显示的 IP），按提示创建管理员账号。

安装完成后，Dokploy 自带 Web 终端：**Settings → Servers → 对应服务器 → Enter Terminal** 可进该机 shell；容器/应用也有终端或 Run Command。日常操作不必每次都 SSH，但**首次安装**必须在该 Linux 上执行上述脚本。

*若在 Mac 上直接执行上述 `curl ... | sh`，脚本会先要求 root，在 root 下会检测到 Darwin 并报错退出：「This script must be run on Linux」。*

---

## 第二步：登录并创建项目

1. 在浏览器打开**你自托管实例**的地址（如 `http://<你的 Linux IP>:3000`）。**不要**使用 app.dokploy.com 等托管版地址，否则会看到「No servers available, Please subscribe to a plan」且不符合 issue #157。
2. 使用你在该实例上创建的管理员账号**登录**（自托管版首次访问会引导你创建管理员，无需「订阅计划」）。
3. 在左侧或顶部导航找到 **「项目」/「Project」**，点击 **「新建项目」/「Create Project」**。
4. 填写项目名称（例如 `medical-server` 或 `raidar`），保存。

---

## 第三步：添加依赖服务（RabbitMQ、Redis）

medical-server 依赖 RabbitMQ 和 Redis，建议先用 Dokploy 内置的 Database 服务：

1. 进入刚创建的项目，找到 **「添加服务」/「Add Service」** 或 **「Database」** 入口。
2. 添加 **RabbitMQ**：
   - 选择类型为 RabbitMQ，创建。
   - 默认账号一般为 `guest/guest`，如需修改可在创建时或环境变量中设置。
   - 记下**服务名**（如 `rabbitmq`），应用内用此名连接（如 `rabbitmq:5672`）。
3. 添加 **Redis**：
   - 选择类型为 Redis，创建。
   - 记下**服务名**（如 `redis`），应用内连接串一般为 `redis://redis:6379`。

*（若采用 docker-compose 一键部署整栈，则 Rabbit/Redis 可能在 compose 里由 Dokploy 以「Compose」方式统一创建，本步可能改为「在 Compose 中定义」，以实际 Dokploy 界面为准。）*

---

## 第四步：部署 MongoDB（副本集 rs0，难点）

MongoDB 必须跑成**副本集 rs0**，不能用 Dokploy 内置 MongoDB 的默认单机模式（或内置不支持 rs0）。

**做法之一：用 Docker Compose 在项目里定义自定义 Mongo 服务**

1. 在项目中选择 **「Compose」/「Docker Compose」** 方式创建应用。
2. 在 compose 文件中定义名为 `mongodb` 的服务（名称必须是 `mongodb`，以便 rs0 的 hostname 正确）。
3. 在 Mongo 镜像的启动命令或配置中开启副本集，例如在 `docker-compose.yml` 中为 mongodb 服务设置：
   - 命令或 entrypoint 包含 `--replSet rs0`（或等价配置）；
   - 或使用环境变量/配置文件使 Mongo 以 rs0 启动。
4. 首次启动后，需要**初始化副本集**：在 Dokploy 中打开该 Mongo 容器的**终端**（或通过 SSH 进容器），执行：
   ```bash
   mongosh
   rs.initiate()
   ```
5. 应用连接 Mongo 时，连接字符串必须包含 **`?replicaSet=rs0`**，例如：  
   `mongodb://mongodb:27017/your_db?replicaSet=rs0`

*（若你采用「单独一个 Mongo 应用 + Custom Command」方式，则需在 Advanced 里配置启动命令和 rs0，并在 journal 中记录为何选用/放弃该方案。）*

---

## 第五步：部署 Raidar Server（medical-server 应用）

### 方式 A：Docker Compose 与 Mongo/Rabbit/Redis 一起部署（推荐）

1. 在同一项目中新建 **「Compose 应用」**，上传或粘贴 `docker-compose.yml`。
2. `docker-compose.yml` 中需包含：
   - 服务名 `mongodb` 的 Mongo（见第四步）；
   - 若 Rabbit/Redis 未用内置服务，则在 compose 中一并定义；若用内置，则通过「同一项目内网络」或 Dokploy 提供的网络别名连接。
   - Raidar Server 服务：镜像可用 `ghoshorn/raidar:server-latest` 或从 GitHub 构建（见方式 B）。
3. 为 Raidar 服务设置**环境变量**，例如：
   - `MONGODB_URI` 或 `SPRING_DATA_MONGODB_URI`：`mongodb://mongodb:27017/raidar-master?replicaSet=rs0`
   - RabbitMQ：`SPRING_RABBITMQ_HOST=rabbitmq` 等（与第三步中服务名一致）
   - Redis：`SPRING_REDIS_HOST=redis` 或对应 Spring 配置
4. 在 Dokploy 界面中点击 **「Deploy」/「部署」**，等待构建/拉取并启动。

### 方式 B：从 GitHub 拉代码在 Dokploy 中构建

1. 在项目中新建 **「应用」/「Application」**，选择 **从 GitHub 部署**。
2. 关联 medical-server 仓库，选择分支（如 `main`）。
3. 构建方式选择 **Dockerfile** 或 **Gradle bootBuildImage**（若 Dokploy 支持）；构建上下文为 `app/` 目录（以仓库内实际 Dockerfile 或 build 说明为准）。
4. 填写**环境变量**（同方式 A）。
5. 设置**依赖/网络**：确保该应用与同一项目内的 `mongodb`、`rabbitmq`、`redis` 互通（同项目内通常自动同网段）。
6. 点击 **「Deploy」**。

---

## 第六步：数据导入（MongoDB 初始数据）

部署完 MongoDB 并完成 rs0 初始化后，如需导入测试/初始数据：

1. 在 Dokploy 中打开 **MongoDB 容器**的**终端**（或 SSH 到服务器后 `docker exec` 进该容器）。
2. 将 `db_backup.tgz`（或你方提供的备份）上传到服务器或挂载进容器。
3. 在容器内执行（路径按实际调整）：
   ```bash
   mongorestore -d raidar-master ./db_backup/raidar-master
   mongorestore -d tenant_mp_test_tenant ./db_backup/tenant_mp_test_tenant
   ```

*（若备份在宿主机，可用 `docker cp` 拷入容器后再执行 mongorestore。）*

---

## 第七步：验证

1. 在 Dokploy 中查看 Raidar Server 服务状态为 **Running**，日志无报错。
2. 若已配置端口或域名，在浏览器访问：  
   `http://<IP或域名>:8080/swagger-ui/index.html`  
   确认 Swagger UI 可打开并调通关键接口。
3. 检查环境变量、Mongo 连接串中是否包含 `replicaSet=rs0`，Rabbit/Redis 是否连到正确服务名。

---

## 后续可补充

- **域名与 TLS**：在 Dokploy 或前置反向代理中绑定域名、配置 HTTPS。
- **健康检查**：在应用配置中增加健康检查路径与端口。
- **备份、监控、密钥轮换**：见 `findings/blockers-and-risks.md` 与头脑风暴中的「生产加固」部分。

---

*文档会随你在 Dokploy 中的实际点击与结果持续更新；若某步与当前 Dokploy 界面不一致，请在该步骤下注明版本或截图，并更新本文档。*
