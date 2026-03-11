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

### Deploy 时拉取的是什么？需要本地构建或推送到 Docker Hub 吗？

当前这份 compose（[configs/docker-compose-medical-server.yml](./configs/docker-compose-medical-server.yml)）没有 `build:`，只会按 `image:` 拉取镜像：

- Dokploy 点击 **Deploy** 后，会先从你选的 GitHub 仓库（如 deploy-spike）拉取 **compose 文件本身**（即该 YAML）。
- 然后根据 YAML 里的 **`image:`** 在运行 Dokploy 的机器上执行 `docker pull`：拉取 `mongo:8`、`rabbitmq:3-management`、`redis:7-alpine`、`yangbingjia1206/raidar:server-latest` 等镜像，再启动容器。

因此：deploy-spike 不需要存 medical-server 的代码；你只需要确保 `raidar.image` 指向一个 Dokploy 机器可拉取的镜像即可。  
- **若镜像是 public**：不需要在 Dokploy 配 Docker Registry，直接 Deploy 即可拉取。  
- **若镜像是 private**：需要在 Dokploy 的 **Settings → Docker Registry** 中添加 Docker Hub（`docker.io` + username + password/token），否则会 `pull access denied`。  
 - **若 Dokploy 服务器是 arm64（常见于 Mac 的 Multipass VM）**：你推到 Docker Hub 的镜像也必须包含 **linux/arm64**，否则会报 `no matching manifest for linux/arm64/v8`。推荐推送 multi-arch（amd64+arm64）同一 tag，见 [docs/build-and-push-raidar-image.md](./docs/build-and-push-raidar-image.md) 的「为 arm64 / 多架构推送镜像」。
本次已采用 **方案 3**：本地构建并推送 `yangbingjia1206/raidar:server-latest`，compose 已切换到该镜像；构建/推送步骤见 [docs/build-and-push-raidar-image.md](./docs/build-and-push-raidar-image.md)。

### Raidar 镜像拉取失败（ghoshorn/raidar 不可用）时怎么办？

若 Deploy 报错：`pull access denied for ghoshorn/raidar, repository does not exist or may require 'docker login'`，说明当前无法使用该镜像（私有且无权限，或该仓库已不可用）。此时需**换用其他可拉取的 Raidar 镜像**。详见 [findings/raidar-image-source.md](./findings/raidar-image-source.md)，这里仅摘要：

- **方案 3（推荐，可完全自助）**：在本地从 medical-server 源码执行 `./gradlew bootBuildImage --imageName=<你的DockerHub用户名>/raidar:server-latest`，再 `docker push` 到你自己的 Docker Hub；把 compose 里 raidar 的 `image:` 改为该镜像；若设为私有，在 Dokploy 的 Registries 中配置你的 Docker Hub。**不需要**公司 GitHub 管理员或 Fork——Docker Hub 与 GitHub 是两套账号，你只要有源码即可构建并推到你自己的 Hub。**逐步命令**见 [docs/build-and-push-raidar-image.md](./docs/build-and-push-raidar-image.md)。
- **方案 2**：若组织开放 Fork，可把 medical-server fork 到个人账号，再在 Dokploy 用 Application 从 fork 构建，或由 fork 的 CI 构建并推送镜像，compose 引用该镜像。
- **方案 1**：由公司组织管理员在 GitHub 安装 Dokploy 的 GitHub App，在 Dokploy 从官方 repo 构建或使用公司推送的镜像并配置 Registry。

修改 compose 后需将变更推送到 deploy-spike（或更新 Raw），再在 Dokploy 重新 Deploy。

### Select repository 里没有 medical-server（组织私有库）时怎么办？

Dokploy 的 Provider 若选 **GitHub**，则「Select repository」只会列出**当前已连接 GitHub 账户有权限**的仓库。若 medical-server 在公司 **GitHub 组织（Organization）** 下且为 **private**，个人账号连接后**不会**在列表里看到该 repo。

**重要**：Compose 文件里只用到了**公开镜像**（mongo、rabbitmq、redis、ghoshorn/raidar:server-latest），**不必**从 medical-server 仓库取 compose；只要有一份 YAML 即可，来源可以是任意你有权限的 repo、或 Raw 粘贴（若 Dokploy 支持）。

可选做法（任选其一即可）：

| 方案 | 做法 | 说明 |
|------|------|------|
| **A. 用你已有权限的仓库** | 把 `configs/docker-compose-medical-server.yml` 放到你**个人 GitHub** 下某个 repo（例如 **deploy-spike**，若已 push 到 GitHub）。在 Dokploy 的 Provider 选 GitHub，Repository 选该 repo，Branch 选对应分支，**Compose Path** 填 `configs/docker-compose-medical-server.yml`（或把文件放到仓库根目录后填 `./docker-compose.yml`）。 | 无需 org 授权或 fork，立即可用。 |
| **B. 用 Raw 来源** | Provider 选 **Raw**（若有该选项），按界面说明粘贴或填写 compose 内容/URL。 | 不依赖 GitHub 仓库，需确认当前 Dokploy 版本是否支持 Raw 及格式。 |
| **C. 组织安装 GitHub App** | 请公司 **GitHub 组织管理员** 在 GitHub 中安装/授权当前 Dokploy 实例使用的 **GitHub App**，并勾选可访问的仓库（含 medical-server）。安装后，若 Dokploy 支持多账户/多安装，在「Github Account」中选该组织，即可在 Repository 里看到 medical-server。 | 需要组织侧权限与管理员操作。 |
| **D. Fork 到个人账号** | 若组织允许 Fork：在 GitHub 上把 medical-server fork 到你个人账号，在 fork 里添加或保留 `docker-compose.yml`（可从 deploy-spike 的 `configs/docker-compose-medical-server.yml` 复制），在 Dokploy 里选该 fork 为 Repository，Compose Path 指向该文件。 | Fork 后你个人账号可见该 repo，Dokploy 即可列出；需管理员先开放 fork。 |

**建议优先尝试 A**：若 deploy-spike 已在你的 GitHub 上，直接选该 repo 并设置 Compose Path 为 `configs/docker-compose-medical-server.yml`，保存后即可在该 Compose 的配置页点 **Deploy** 进行部署（无需动 medical-server 或组织权限）。部署时 Dokploy 会从 deploy-spike 拉取**该 YAML 文件**，再按 YAML 中的 `image:` 从 Docker Hub 等拉取镜像并启动容器；见下方「Deploy 时拉取的是什么？」。

### 若 Dokploy 的 Compose 不支持多服务或格式有差异

可在 journal 中记录实际界面与限制；备选方案为：用 **Application** 单独部署 Raidar，用 **Database** 只加 Redis，**RabbitMQ 则需用 Compose 单独建一个栈**（只含 rabbitmq 服务）或改用其他方式。仍建议优先用 Compose 整体部署。

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
