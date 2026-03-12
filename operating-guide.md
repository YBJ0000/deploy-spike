# Dokploy 部署 Spike 操作指南（精简版）

本文档为**精简版**操作流程，按**浏览器与终端操作顺序**编写，去除了冗余说明与已修正内容。自托管 Dokploy 的详细背景与排错见 [operating-guide.md](./operating-guide.md)、[docs/faq.md](./docs/faq.md)。

> **前置任务**：本地构建并推送 medical-server 镜像到 Docker Hub，供 Dokploy 拉取。详见 [docs/build-and-push-raidar-image-simple.md](./docs/build-and-push-raidar-image-simple.md)（完整版与排错见 [docs/build-and-push-raidar-image.md](./docs/build-and-push-raidar-image.md)）。

---

## 一、准备 Linux 环境并安装 Dokploy

Dokploy 官方安装脚本**仅支持 Linux**，可选两种方式：

- **方式 A**：租用一台 Linux VPS，SSH 登录后在该机上安装。
- **方式 B**：在 Mac 上用 **Multipass** 跑 Linux 虚拟机，在 VM 内安装。（下文以方式 B 为例。）

### 1.1 安装 Multipass（Mac）

```bash
brew install --cask multipass
multipass --version   # 出现版本号即安装成功
```

### 1.2 创建 Ubuntu 虚拟机

推荐 **4GB 内存、40GB 磁盘**（实测 2GB 在 Mongo + RabbitMQ + Redis + Raidar 同时运行时易卡死或崩溃）：

```bash
multipass launch 22.04 --name dokploy-vm --memory 4G --disk 40G --cpus 2
multipass list       # 确认 dokploy-vm 状态为 Running
```

### 1.3 进入 VM 并安装 Dokploy

```bash
multipass shell dokploy-vm
```

在 VM 内执行：

```bash
curl -sSL https://dokploy.com/install.sh | sudo sh
```

等待数分钟，直到出现 `Dokploy is installed`。

### 1.4 访问 Dokploy 控制台

在**本机终端**执行 `multipass list`，记下 **dokploy-vm** 的 IP，在浏览器访问：

`http://<VM IP>:3000`

---

## 二、在 Dokploy 中创建项目与 Compose 服务

### 2.1 创建管理员账号与项目

1. 浏览器成功打开 Dokploy 后，按页面提示**创建管理员账号**。
2. 进入 Dashboard，点击 **Create Project**。
3. 在创建好的 Project 里，点击 **Create service**，选择 **Compose** 类型。

### 2.2 填写 Compose 服务基本信息

- **Name**：`medical-server-stack`
- **App Name**：`medical-server`
- **Compose Type**：选 **Docker Compose**
- **Description**：可选，如 `Mongo + RabbitMQ + Redis + Raidar`

保存后进入该 Service 的 **Settings**。

### 2.3 连接 GitHub 并授权

在 Dokploy 侧边栏找到 **Git**，进入后选 **GitHub**（不要勾选 Organization），点击 **Create GitHub App**，按提示连接 GitHub 账号并授权 Dokploy 的 GitHub App（选择允许访问包含 compose 文件的仓库）。

### 2.4 配置仓库与 Compose 路径

回到 **Project → 该 Service → General**，填写：

- **Repository**：选 `deploy-spike`（需先在 GitHub 将 `https://github.com/YBJ0000/deploy-spike` fork 到自己账号；或自己在 GitHub 新建一个库，把 `deploy-spike/configs/docker-compose-medical-server.yml` 文件拷贝进去）
- **Branch**：`main`
- **Compose Path**：取决于配置文件在库中的位置，例如 `configs/docker-compose-medical-server.yml`（内含 Docker Hub 镜像地址，用于拉取 medical-server 镜像）
- **Trigger Type**：先选 **On Push**（当前以手动 Deploy 为主）
- **Watch Paths**：可不填

点击 **Save**。

### 2.5 设置环境变量并部署

1. 进入该 Service 的 **Environment**，设置例如：`RAIDAR_TAG=server-20260311-2`，确保与 Docker Hub 上的镜像 tag 一致。
2. 点击 **Deploy**，等待部署完成。
3. 若部署失败，在 Service 的 **Deployments** 或 **Logs** 页面排查原因。

---

## 三、初始化 MongoDB 副本集（rs0）

Compose 部署完成后，Mongo 已以 `--replSet rs0` 启动，但尚未执行 `rs.initiate()`，需在本机完成初始化。

在本机终端执行（将 `<VM IP>` 替换为 `multipass list` 中的 dokploy-vm IP）：

```bash
mongosh "mongodb://<VM IP>:27017"
```

若连接成功，会看到类似 `rs0 [direct: primary]` 的提示。在 mongosh 中依次执行：

```javascript
rs.initiate()
rs.status().ok   // 应返回 1 表示初始化成功
```

---

## 四、数据导入（可选）

在完成第三步、且能从本机连上 Dokploy VM 的 Mongo 后，可导入测试数据。

1. 进入 **medical-server/db** 目录。
2. 若尚未解压备份，执行：
   ```bash
   tar -xzvf db_backup.tgz
   ```
   解压后应看到 `raidar-master/` 与 `tenant_mp_test_tenant/` 目录。
3. 在本机执行 **mongorestore**，将数据导入 VM 上的 Mongo（将 `<VM IP>` 替换为实际 IP）：
   ```bash
   mongorestore --host <VM IP> --port 27017 -d raidar-master ./raidar-master
   mongorestore --host <VM IP> --port 27017 -d tenant_mp_test_tenant ./tenant_mp_test_tenant
   ```
4. 在 mongosh 会话中执行：
    ```bash
   show dbs
   use raidar-master
   db.getCollectionNames()
   use tenant_mp_test_tenant
   db.getCollectionNames()
    ```
   若两个库下都能看到大量业务集合（而不只是 `shedLock`），说明导入成功。

---

## 五、验收

- **Dokploy 界面**：在 Project → Service 的 **Deployments**、**Logs** 中确认状态为成功、无报错；侧边栏 **Docker** 中所有容器为 Running。
- **数据库**：在 mongosh 中执行 `rs.status().ok` 返回 `1`。
- **RabbitMQ**：浏览器访问 `http://<VM IP>:15672`，使用 `guest/guest` 登录成功并看到 RabbitMQ Management Dashboard，说明 Raidar HTTP 与 RabbitMQ 管理端口均已对外可用。
- **Redis**：执行 `redis-cli -h <VM IP> -p 6379 ping`，应返回 `PONG`，确认 Dokploy VM 上的 Redis 实例可从外部连接且正常响应。
- **Swagger API**：浏览器访问 `http://<VM IP>:8080/swagger-ui/index.html`，能打开并加载接口文档，说明 Raidar HTTP 与 Swagger UI 端口均已对外可用。在完成数据导入后，通过 Swagger UI 调用 `GET /api/spa/page-config` 返回 HTTP 200，响应中 `authenticated=false` 且 `tenantInfo.name="Raidar Software"`，说明在当前 Dokploy 部署下，Raidar 的公开 API 已能正常读到配置数据。

---

## 六、后续使用（Multipass VM）

- **再次进入 VM**：`multipass shell dokploy-vm`
- **查看 VM 状态 / IP**：`multipass list`
- **停止 VM**：`multipass stop dokploy-vm`
- **启动 VM**：`multipass start dokploy-vm`
- **删除 VM**（会删除其中所有数据）：`multipass delete dokploy-vm && multipass purge`

---

*本文档随实际操作持续更新；若某步与当前 Dokploy 界面不一致，可在该步骤下注明版本并更新文档。*
