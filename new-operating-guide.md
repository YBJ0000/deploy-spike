# Dokploy 部署 Spike 操作指南

前置任务：本地构建并推送 medical-server 镜像到 Docker Hub，供 Dokploy 拉取

Dokploy 官方安装脚本仅支持 Linux
方式 A：租用一台 Linux VPS，SSH 登录后在该机上安装
方式 B：在 Mac 上用 Multipass 跑 Linux 虚拟机，在 VM 内安装。逐步命令见

选了方式B：
安装 Multipass
创建 Ubuntu 虚拟机：4GB 内存、40GB 磁盘
进入 VM 并安装 Dokploy
获取 VM IP 并在浏览器访问 Dokploy

浏览器成功访问 Dokploy之后：
创建管理员账号，进入 dashboard
点Create Project
在创建好的 Project 里，点 Create service，选 Compose 类型：
* Name： medical-server-stack 
* App Name：medical-server
* Compose Type：选 Docker Compose。 
* Description：可选，如 Mongo + RabbitMQ + Redis + Raidar 


Service 创建成功后，去到 Settings 里连接 GitHub 账户并授权 GitHub App：
在Dokploy侧边栏找到 Git ，点进去选 GitHub，（不要勾选Organization），点击Create GitHub App。连接你的 GitHub 账号并按提示安装/授权 Dokploy 的 GitHub App（选择允许访问包含 compose 文件的仓库）

回到Project - Service -  General 界面，填表：
- repository选deploy-spike （要去GitHub先把 https://github.com/YBJ0000/deploy-spike fork到自己账号里）
- Branch选main
- Compose Path填：configs/docker-compose-medical-server.yml（这里面有docker hub的镜像地址，会拉取medical-server镜像）
- Trigger Type先选On Push（目前还是手动deploy，还没有研究过On Tag触发流程）
- Watch Paths可不填
填完后点 Save

去到当前Service的Environment 中设置环境变量，例如RAIDAR_TAG=server-20260311-2 ，确保镜像 tag 与 Docker Hub一致

点击 Deploy，等待部署完成
若部署失败，请查看 Service 的 Deployments 或 Logs 页面，排查原因

操作数据库：
在本机终端输入`mongosh "mongodb://<Dokploy VM 私网 IP>:27017”`访问数据库
（若看到rs0 [direct: primary]则连接正常）
在mongosh输入rs.initiate()，查看是否成功初始化
在mongosh输入rs.status().ok，查看是否返回1

数据导入：
确保上一步完成（可以从本机连上 Dokploy VM 的 Mongo）
去到medical-server/db目录：
- 若未解压db_backup.tgz，则解压，然后查看db是否新增raidar-master/和tenant_mp_test_tenant/
- 执行mongorestore命令，把数据导入 Dokploy VM 上的 Mongo
- 验证导入结果

验收：
Dokploy 界面快速验收
数据库验收
RabbitMQ / Redis 验收
swagger api验收