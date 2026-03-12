# Dokploy 实际操作流程（浏览器操作指南）

本文档针对 **自托管版 Dokploy**（符合 issue #157：self-hosted, not a paid hosted solution）。按**浏览器里用鼠标、键盘操作 Dokploy** 的顺序编写，随实际探索会持续更新。

> **重要**：若在界面上看到「No servers available, Please subscribe to a plan」，说明当前打开的是 **Dokploy Cloud（付费托管版）**，不是自托管实例。请改为访问**你自己安装的 Dokploy**（`http://<你的 Linux 服务器或 VM 的 IP>:3000`）。详见 [findings/self-hosted-vs-cloud.md](./findings/self-hosted-vs-cloud.md)。

---

## 当前进度（已验证）

- **自托管 Dokploy 已安装成功**（在 Mac 的 Multipass VM `dokploy-vm` 内）。
- **正确访问方式（本机 Multipass）**：使用 `multipass list` 看到的 VM 私网 IP（`192.168.64.x`）访问 `http://<VM私网IP>:3000`。  
  安装脚本打印的 `http://<公网/出口IP>:3000` 在本机场景可能不可用（未做端口映射）。
- **已完成**：2.1 初始化（创建管理员账号）、2.2 登录并创建项目（项目名 `medical-server`）；**第三步** Compose 整体部署（Mongo + RabbitMQ + Redis + Raidar）；方案 3（自建镜像 `yangbingjia1206/raidar:server-latest`）并完成 Deploy（日志出现 `Docker Compose Deployed: ✅`，容器均 Started）；**第四步** MongoDB 副本集 rs0 初始化（通过本机 `mongosh "mongodb://<VM私网IP>:27017"` 执行 `rs.initiate()` + `rs.status().ok` 返回 1）。
- **你当前下一步**：修复并验证 `medical-server`（Raidar）的启动阻塞与业务功能：按下方 6.1 的“注入冲突”排错与 [findings/raidar-startup-failures.md](./findings/raidar-startup-failures.md) 用新 tag 重新 buildx+push+Deploy，重启后检查日志不再有 `REPLICA_SET_GHOST` / `MongoTimeoutException`，再按 **第六步** 逐项验收（Swagger/依赖服务/日志）。

（更详细的 VM/IP 说明与排错见 [docs/mac-multipass-dokploy.md](./docs/mac-multipass-dokploy.md)。）

**常见问题**（账号与数据存在哪里、本机 VM 能否被外网访问、合上盖子会怎样、镜像/仓库失败怎么办等）见 **[docs/faq.md](./docs/faq.md)**。

---

## 前置条件（在动手点 Dokploy 之前）

- [ ] **已有一个 Linux 环境**：Dokploy 官方安装脚本**仅支持 Linux**（如 Ubuntu、Debian、CentOS、Fedora 等），**不支持 macOS**。可选两种方式获得 Linux：
  - **方式 A**：租用一台 Linux VPS，SSH 登录后在该机上安装；
  - **方式 B**：在 Mac 上用 **Multipass** 跑 Linux 虚拟机，在 VM 内安装。**逐步命令见** [docs/mac-multipass-dokploy.md](./docs/mac-multipass-dokploy.md)。
- [ ] 该 Linux 满足：**至少 4GB 内存、40GB 磁盘**（部署 medical-server 全栈时 2GB 易不足，见 [docs/mac-multipass-dokploy.md](./docs/mac-multipass-dokploy.md)）；端口 **80、443、3000** 未被占用，防火墙已放行。
- [ ] 该 Linux 已安装 Docker（或由安装脚本自动安装）。
- [ ] 如需域名访问，DNS 已指向该机；TLS/反向代理按你方约定配置。

**本地 Mac 可做检查**：`docker --version` 确认 Docker 已装（用于本地构建等）；实际安装须在**上述 Linux 环境**内完成。

---

## 第一步：安装 Dokploy（须在 Linux 上执行）

安装**必须在 Linux 环境**内进行（可以是远程 VPS，或 Mac 上的 Linux 虚拟机），不能在本机 macOS 上直接运行官方脚本。

1. 获得该 Linux 的 shell：
   - **若用 VPS**：`ssh 用户名@服务器IP或域名`
   - **若用 Mac 上的 Multipass**：按 [docs/mac-multipass-dokploy.md](./docs/mac-multipass-dokploy.md) 创建 VM（推荐 `multipass launch 22.04 --name dokploy-vm --memory 4G --disk 40G --cpus 2`），再用 `multipass shell dokploy-vm` 进入。
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
2. **第一步：Create Compose 表单**（当前界面没有 YAML，只填元数据）  
   - **Name**：填本 Compose 栈的名称，例如 `medical-server-stack`（不要用默认的 Frontend）。  
   - **App Name**：一般为项目或应用前缀，若界面已带出 `medicalserver-` 之类可保留或改成 `medical-server`，保持与项目一致即可。  
   - **Compose Type**：选 **Docker Compose**。  
   - **Description**：可选，如「Mongo + RabbitMQ + Redis + Raidar 整体栈」。  
   填完后点击 **Create**。
3. **第二步：在 Settings 里连接 GitHub 账户并授权 GitHub App**。先在 Dokploy 顶部/侧边栏进入 **Settings → Git Providers / Github**，连接你的 GitHub 账号并按提示安装/授权 Dokploy 的 GitHub App（选择允许访问包含 compose 文件的仓库）。  
4. **第三步：回到项目 General 界面，选择仓库与 compose 路径**。回到该项目的 **General → Deploy Settings**，在 Provider 选择 **GitHub**，选中刚授权可见的仓库和分支；此时 **repo 中已有的 `.yml` / compose 文件会直接被使用**，在 **Compose Path** 中填写（或下拉选择）该文件路径即可，例如 `configs/docker-compose-medical-server.yml` ，里面会记录Docker Hub的镜像地址。
5. **第四步：设置环境变量**。在 **General → Environment** 中设置环境变量，例如 `RAIDAR_TAG=server-20260311-2` ，确保镜像 tag 正确。
6. **第五步：点击 Deploy**。确保填写了正确的仓库和 compose 路径，然后点击 **Deploy** 按钮，等待部署完成。

**Deploy 时拉取的是什么、镜像拉取失败、Select repository 没有 medical-server、Compose 不支持多服务等**说明见 **[docs/faq.md](./docs/faq.md)** 中对应小节。

---

## 第四步：初始化 MongoDB 副本集（rs0）

Compose 部署完成后，Mongo 已以 `--replSet rs0` 启动，但尚未执行 `rs.initiate()`，应用会报错直到副本集初始化完成。

1. 在**你本机终端**使用 `mongosh` 直接连接 Dokploy VM 上的 Mongo（Compose 已将 27017 暴露出来）。连接字符串示例：
   ```bash
   mongosh "mongodb://<Dokploy VM 私网 IP>:27017"
   rs.initiate()
   ```
   - `<Dokploy VM 私网 IP>` 一般形如 `192.168.64.x`，可从：
     - `multipass list` 输出中的 IPv4，或
     - 访问 Dokploy 面板时浏览器地址栏里的 IP（例如 `http://192.168.64.4:3000`，则 Mongo 为 `192.168.64.4:27017`）。
2. `rs.initiate()` 返回成功后，可以用下述命令快速自检：
   ```bash
   mongosh --eval "rs.status().ok"
   ```
   若输出为 `1`，说明 rs0 已正常初始化。
3. 完成初始化后，回到 Dokploy 中**重启 `raidar` 服务**（若未自动重连），使应用连上已初始化的 rs0。

---

## 第五步：数据导入（MongoDB 初始数据，可选）

完成第四步 rs0 初始化后，如需导入测试/初始数据（见 [medical-server/app/README.md](../medical-server/app/README.md) 的 db 目录说明），推荐直接在**本机终端**使用 `mongorestore` 连接 Dokploy VM 上的 Mongo：

1. **确保可以从本机连上 Dokploy VM 的 Mongo**  
   在本机终端执行（将 `<VM私网IP>` 替换为你的 Dokploy VM IP，例如 `192.168.64.4`）：
   ```bash
   mongosh "mongodb://<VM私网IP>:27017"
   ```
   看到 `rs0 [direct: primary]` 等提示说明连接正常，可在该会话中用 `show dbs` 简单确认。
2. **在本机解压测试数据归档（若尚未解压）**  
   切换到 `medical-server/db` 目录（以你的仓库路径为准）：
   ```bash
   cd /Users/<your-user>/WorkSpace/medical-server/db
   tar -xzvf db_backup.tgz
   ```
   解压后会得到 `raidar-master/` 与 `tenant_mp_test_tenant/` 两个目录。  
   > 注意：**这里目录名本身就是 `raidar-master`、`tenant_mp_test_tenant`，不要再人为加 `db_backup/` 前缀**。
3. **从本机将数据导入 Dokploy VM 上的 Mongo**  
   仍在 `medical-server/db` 目录下执行（将 `<VM私网IP>` 替换为实际 IP）：
   ```bash
   mongorestore --host <VM私网IP> --port 27017 -d raidar-master ./raidar-master
   mongorestore --host <VM私网IP> --port 27017 -d tenant_mp_test_tenant ./tenant_mp_test_tenant
   ```
   - 若看到关于 `--db` / `--collection` 的 deprecated 提示，可暂时忽略；只要没有报路径不存在或连接失败即可。
4. **在 mongosh 中验证导入结果**  
   回到第 1 步已连上的 `mongosh` 会话中，依次执行：
   ```javascript
   show dbs
   use raidar-master
   db.getCollectionNames()
   use tenant_mp_test_tenant
   db.getCollectionNames()
   ```
   - 若两个库下都能看到大量业务集合（而不只是 `shedLock`），说明导入成功。

---

## 第六步：验证

按下列顺序验收（**建议逐条勾选**）。注意：**在 rs0 初始化前**，Raidar 可能“容器在跑但业务报错”，因此务必先完成第四步。

### 6.1 Dokploy 界面快速验收（无需进容器）

1. 在 Dokploy 界面，找到我们创建的 Project 的 Service 的 **Deployments** 页面，查看最新的一条是否为 `Done` 状态，点击 View 查看详情。
2. 在当前 Service 的 **Logs** 页面，观察是否存在明显的连接失败（常见关键字：`replicaSet` / `rs0` / `MongoTimeoutException` / `RabbitMQ` / `Redis`）。
3. 在 Dokploy 界面侧边栏找到 **Docker** 容器列表，查看是否所有容器都处于 `Running` 状态。

若出现 `invalid reference format` 且日志中尝试拉取 `yangbingjia1206/raidar:`（tag 为空），通常是因为 Dokploy 环境中未设置 `RAIDAR_TAG`；请在 Dokploy 的 Environment Settings 中增加 `RAIDAR_TAG=<与你本地 .env 相同的值>`，再重新 Deploy。


### 6.2 初始化 rs0 后的数据库验收（在 mongodb 容器终端）

在 `mongodb` 容器终端执行：

```bash
mongosh --eval "rs.status().ok"
```

- 若输出 `1`：说明 rs0 已就绪。
- 若报 “not yet initialized”：按第四步执行 `rs.initiate()`，然后再执行一次上述命令确认。

完成后，**重启 `raidar` 容器**（或在 Dokploy 对 `raidar` 执行 Restart），确保它重新连接到已初始化的副本集。

### 6.3 RabbitMQ / Redis 验收（可选但建议）

- **RabbitMQ 管理台**：在浏览器打开 `http://<VM私网IP>:15672`，用 `guest/guest` 登录（本 compose 默认）。
- **Redis PONG**：在 `redis` 容器终端执行 `redis-cli ping`，应返回 `PONG`。

### 6.4 Raidar API / Swagger 验收

在浏览器打开：

- `http://<VM私网IP>:8080/swagger-ui/index.html`

能打开并能正常加载接口文档，即为通过的强信号。  
在**完成数据导入后**，可在 Swagger 中选用一个无需认证的公开 GET 接口做功能验收，例如：

- `GET /api/spa/page-config`：若执行后返回 HTTP 200，响应体类似：
  ```json
  {
    "authenticated": false,
    "tenantInfo": {
      "name": "Raidar Software",
      "tenantId": null,
      "code": null,
      "domain": "localhost:8080",
      "primaryLogo": { "fileId": "..." },
      "compactLogo": { "fileId": "..." }
    }
  }
  ```
  说明在「Mongo 副本集 rs0 + 测试数据导入 + RabbitMQ + Redis」均就绪的前提下，Raidar 的公开 API 已能正常读到配置数据。

若 Swagger 无法打开或上述接口请求失败：

- 先看 `raidar` 容器 Logs 是否仍在报 Mongo rs0 未就绪或连接失败；
- 再确认 Dokploy/compose 是否对外暴露了 8080（compose 中为 `8080:8080`）。

---

## 后续可补充

- **域名与 TLS**：在 Dokploy 或前置反向代理中绑定域名、配置 HTTPS。
- **健康检查**：在应用配置中增加健康检查路径与端口。
- **备份、监控、密钥轮换**：见 `findings/blockers-and-risks.md` 与头脑风暴中的「生产加固」部分。

### 查看服务器资源使用情况（Monitoring）

在 Dokploy 侧边栏进入 **Monitoring**，可查看当前服务器（即运行 Dokploy 的 Linux/VM）的 **CPU、内存（Memory）、磁盘（Disk）、Block I/O、Network I/O** 等使用情况。若内存或磁盘接近上限，可考虑为 VM 扩容或迁至更大规格的机器后再按本文档重装 Dokploy 与部署。

---

*文档会随你在 Dokploy 中的实际点击与结果持续更新；若某步与当前 Dokploy 界面不一致，请在该步骤下注明版本或截图，并更新本文档。*
