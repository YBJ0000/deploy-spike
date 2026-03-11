# Dokploy 实际操作流程（浏览器操作指南）

本文档针对 **自托管版 Dokploy**（符合 issue #157：self-hosted, not a paid hosted solution）。按**浏览器里用鼠标、键盘操作 Dokploy** 的顺序编写，随实际探索会持续更新。

> **重要**：若在界面上看到「No servers available, Please subscribe to a plan」，说明当前打开的是 **Dokploy Cloud（付费托管版）**，不是自托管实例。请改为访问**你自己安装的 Dokploy**（`http://<你的 Linux 服务器或 VM 的 IP>:3000`）。详见 [findings/self-hosted-vs-cloud.md](./findings/self-hosted-vs-cloud.md)。

---

## 当前进度（已验证）

- **自托管 Dokploy 已安装成功**（在 Mac 的 Multipass VM `dokploy-vm` 内）。
- **正确访问方式（本机 Multipass）**：使用 `multipass list` 看到的 VM 私网 IP（`192.168.64.x`）访问 `http://<VM私网IP>:3000`。  
  安装脚本打印的 `http://<公网/出口IP>:3000` 在本机场景可能不可用（未做端口映射）。
- **已完成**：2.1 初始化（创建管理员账号）、2.2 登录并创建项目（项目名 `medical-server`，当前 0 services）。
- **你当前下一步**：按 **第三步** 在该项目里用 **Compose** 整体部署（Mongo + RabbitMQ + Redis + Raidar），再初始化 Mongo 副本集并可选导入数据。

（更详细的 VM/IP 说明与排错见 [docs/mac-multipass-dokploy.md](./docs/mac-multipass-dokploy.md)。）

### 账号与数据存在哪里？销毁 VM 会怎样？

自托管版 Dokploy 的**账号、密码、项目、部署配置等**都保存在**你安装 Dokploy 的那台机器上**（安装脚本会在该机上部署 PostgreSQL 和 Redis，数据就存在那里），**不会**同步到 Dokploy 官方的任何数据库或云端。

- **本机 Multipass VM**：账号和数据都在 VM 内的 PostgreSQL 里；只有你这台 Mac 能访问该实例。
- **销毁该 VM 后**：该实例上的所有账号和数据都会丢失；之前创建的管理员账号也就“不存在了”（因为整台机器没了）。若需要保留，应在删除 VM 前做好数据库或卷的备份。

### 本机 VM：外部能访问吗？合上盖子会怎样？

- **外部/公网能否访问本机 VM 里部署的后端？**  
  - 本机 Multipass VM 的 IP 是**私网地址**（如 `192.168.64.x`），只在你这台 Mac（以及通常同一局域网）内可访问，**互联网上的其他设备不能直接访问**该 IP。  
  - 若要让「外部网站/请求」访问你在 VM 里部署的 Java 后端，需要至少其一：  
    - 在路由器上做**端口映射**（把公网 IP 的某端口转到你 Mac，再在 Mac 上转发到 VM），并保证 Mac 有固定内网 IP；或  
    - 使用**内网穿透/隧道**（如 ngrok、Cloudflare Tunnel）把本机端口暴露到公网；或  
    - 把应用部署到**有公网 IP 的 Linux VPS**，由 VPS 对外提供服务。  
  - 因此：**本机 VM 适合做本地验证、联调；要做成“外部随时可访问”的服务，通常需 VPS 或隧道。**

- **合上 MacBook 盖子、电脑休眠之后呢？**  
  - Mac 休眠后，Multipass VM 也会随之挂起或停止，**VM 里的服务（包括 Dokploy、你部署的后端）会不可用**，外部无法访问。  
  - 若需要** 7×24 持久化**（随时可被外部访问、不因合盖/关机中断），要么：  
    - **保持 Mac 常开、不合盖休眠**（且做好端口映射或隧道），要么  
    - **租用一台 Linux VPS**，在 VPS 上安装 Dokploy 并部署应用，由 VPS 一直运行。

- **当前阶段**：本机 VM 主要用于**探索可行方案、验证部署流程**；确认流程可行后，若需对外稳定提供服务，再迁到 VPS 或使用隧道即可。

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
   - **若用 Mac 上的 Multipass**：按 [docs/mac-multipass-dokploy.md](./docs/mac-multipass-dokploy.md) 创建 VM（如 `multipass launch 22.04 --name dokploy-vm --memory 2G --disk 30G`），再用 `multipass shell dokploy-vm` 进入。
2. 在该 Linux 上执行官方一键安装（需 root，通常加 `sudo`）：  
   ```bash
   curl -sSL https://dokploy.com/install.sh | sudo sh
   ```  
   脚本会自动检测最新稳定版并安装 Dokploy + PostgreSQL + Redis；若需指定版本可先设置 `export DOKPLOY_VERSION=v0.26.6` 再执行上述命令。详见 [Dokploy 官方安装文档](https://docs.dokploy.com/docs/core/installation)。
3. 安装完成后在浏览器访问：`http://<该 Linux 的 IP>:3000`（VPS 用公网 IP；**本机 Multipass VM 用 `multipass list` 显示的 `192.168.64.x`**），进入注册页创建管理员账号。

安装完成后，Dokploy 自带 Web 终端：**Settings → Servers → 对应服务器 → Enter Terminal** 可进该机 shell；容器/应用也有终端或 Run Command。日常操作不必每次都 SSH，但**首次安装**必须在该 Linux 上执行上述脚本。

*若在 Mac 上直接执行上述 `curl ... | sh`，脚本会先要求 root，在 root 下会检测到 Darwin 并报错退出：「This script must be run on Linux」。*

---

## 第二步：登录并创建项目

### 2.1 初始化：创建管理员账号（首次访问）

1. 在浏览器打开**你自托管实例**的地址：`http://<你的 Linux IP>:3000`。  
   - **本机 Multipass**：优先用 `http://192.168.64.x:3000`（以 `multipass list` 为准）。  
   - **不要**使用 app.dokploy.com 等托管版地址，否则会看到「No servers available, Please subscribe to a plan」且不符合 issue #157。
2. 若自动跳转到注册页，按页面 **Setup the server / Register** 表单填写信息并提交（如 `http://<IP>:3000/register`）。

### 2.2 登录并创建项目

1. 用刚创建的管理员账号登录。
2. 在左侧或顶部导航找到 **「项目」/「Project」**，点击 **「新建项目」/「Create Project」**。
3. 填写项目名称（例如 `medical-server` 或 `raidar`），保存。

---

## 第三步：用 Compose 整体部署（推荐）

与 [brainstorming.md](./brainstorming.md) 一致：**推荐整体部署**，在一个 `docker-compose` 里同时定义 MongoDB（rs0）、RabbitMQ、Redis 和 Raidar Server，避免在 Dokploy 里逐个找服务类型。

### Dokploy 里 Create service 的选项说明

点击 **Create service**（或 Add service）后，常见有：**Application**、**Database**、**Compose**、**Template**、**AI Assistant** 等。

- **Database**：内置数据库模板，一般有 **Redis** 等，**多数版本没有 RabbitMQ**，无法单独点选 RabbitMQ。
- **Compose**：用 **docker-compose 的 YAML** 自己定义多个服务，可同时包含 Mongo、RabbitMQ、Redis、应用，**推荐用此方式** 一次性部署 medical-server 所需全部组件。

因此：不要依赖「在 Database 里先加 RabbitMQ 再加 Redis」，应直接选 **Compose**，在一个 compose 里写 Mongo + RabbitMQ + Redis + Raidar。

### 操作步骤

1. 进入已创建的项目（如 `medical-server`），点击 **Create service**，选择 **Compose**。
2. 填写 Compose 名称（例如 `medical-server-stack`），在 YAML 编辑区粘贴或编写 `docker-compose` 内容。
3. **Compose 内容要点**（可直接参考 [configs/docker-compose-medical-server.yml](./configs/docker-compose-medical-server.yml)）：
   - **mongodb**：镜像 `mongo:8`，命令带 `--replSet rs0`，**服务名必须为 `mongodb`**（rs0 的 hostname 依赖此名）；挂载卷持久化。
   - **rabbitmq**：镜像 `rabbitmq:3-management`，环境变量 `RABBITMQ_DEFAULT_USER/PASS=guest`；应用里用 `RABBITMQ_HOST=rabbitmq`。
   - **redis**：镜像 `redis:7-alpine`；应用里用 `SPRING_DATA_REDIS_HOST=redis`。
   - **raidar**：镜像 `ghoshorn/raidar:server-latest`（与 [medical-server/app/README.md](../medical-server/app/README.md) 一致），环境变量：
     - `MONGODB_HOST=mongodb`
     - `RABBITMQ_HOST=rabbitmq`、`RABBITMQ_PORT=5672`、`RABBITMQ_USERNAME=guest`、`RABBITMQ_PASSWORD=guest`
     - `SPRING_DATA_REDIS_HOST=redis`
   - 应用已配置 `replica-set-name: rs0`、`database: raidar-master`（见 medical-server `application.yml`），只需保证 Mongo 服务名为 `mongodb` 并在部署后执行一次 `rs.initiate()`。
4. 保存并点击 **Deploy**，等待镜像拉取与容器启动。

### 若 Dokploy 的 Compose 不支持多服务或格式有差异

可在 journal 中记录实际界面与限制；备选方案为：用 **Application** 单独部署 Raidar，用 **Database** 只加 Redis，**RabbitMQ 则需用 Compose 单独建一个栈**（只含 rabbitmq 服务）或改用其他方式。仍建议优先用 Compose 整体部署。

---

## 第四步：初始化 MongoDB 副本集（rs0）

Compose 部署完成后，Mongo 已以 `--replSet rs0` 启动，但尚未执行 `rs.initiate()`，应用会报错直到副本集初始化完成。

1. 在 Dokploy 中打开 **mongodb 容器**的**终端**（该 Compose 栈下的 mongodb 服务 → Terminal / Execute 等）。
2. 在容器内执行：
   ```bash
   mongosh
   rs.initiate()
   ```
3. 退出 `mongosh` 后，重启 **raidar** 服务（若未自动重连），使应用连上已初始化的 rs0。

---

## 第五步：数据导入（MongoDB 初始数据，可选）

完成第四步 rs0 初始化后，如需导入测试/初始数据（见 [medical-server/app/README.md](../medical-server/app/README.md) 的 db 目录说明）：

1. 在 Dokploy 中打开 **mongodb 容器**的**终端**（或 SSH 到服务器后 `docker exec` 进该容器）。
2. 将 `db_backup.tgz`（或你方提供的备份）上传到服务器或挂载进容器。
3. 在容器内执行（路径按实际调整）：
   ```bash
   mongorestore -d raidar-master ./db_backup/raidar-master
   mongorestore -d tenant_mp_test_tenant ./db_backup/tenant_mp_test_tenant
   ```

*（若备份在宿主机，可用 `docker cp` 拷入容器后再执行 mongorestore。）*

---

## 第六步：验证

1. 在 Dokploy 中查看 **raidar**（Raidar Server）服务状态为 **Running**，日志无报错。
2. 若已配置端口或域名，在浏览器访问：  
   `http://<IP或域名>:8080/swagger-ui/index.html`  
   确认 Swagger UI 可打开并调通关键接口（与 [medical-server/app/README.md](../medical-server/app/README.md) 一致）。
3. 确认 Mongo 已执行 `rs.initiate()`，应用环境变量中 `MONGODB_HOST=mongodb`、`RABBITMQ_HOST=rabbitmq`、`SPRING_DATA_REDIS_HOST=redis`。

---

## 后续可补充

- **域名与 TLS**：在 Dokploy 或前置反向代理中绑定域名、配置 HTTPS。
- **健康检查**：在应用配置中增加健康检查路径与端口。
- **备份、监控、密钥轮换**：见 `findings/blockers-and-risks.md` 与头脑风暴中的「生产加固」部分。

---

*文档会随你在 Dokploy 中的实际点击与结果持续更新；若某步与当前 Dokploy 界面不一致，请在该步骤下注明版本或截图，并更新本文档。*
