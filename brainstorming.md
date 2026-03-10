把 medical-server 各部分(app/redis/mongodb/rabbit)研究清楚，确定如何部署到“自托管版 Dokploy”上，并且产出一份可以直接照着实施的部署方案。

（在Cursor里使用Gemini模型，搜索能力强，这个任务几乎不需要写代码）

给AI看readme和issue#157 

用Gemini、Grok（搜索能力强的AI）查有没有网友部署成功类似项目的，有的话请看看他的经验是什么，有什么注意的点，有什么不可行的地方：
MongoDB replica set（rs0） 是最大的注意点，其他部分（Spring Boot + RabbitMQ + Redis）都很顺手：
MongoDB replica set 是唯一大坑
Dokploy 内置 MongoDB 服务不支持直接开 rs0（或开了也容易 hostname 不匹配导致连不上）。
必须用自定义 Mongo 服务（推荐方式下面会给）。
连接字符串一定要加?replicaSet=rs0（否则事务会报错，和你本地 dev 一样）。
服务名必须叫 mongodb（这样 rs.initiate 里的 hostname 才对）

整体部署or分开部署
用 Compose 一键部署整个 stack（app + mongo + rabbit + redis）？
分开部署（App + 3 个 Database 服务），Mongo 用 Advanced → Custom Command？
推荐整体部署：Dokploy 原生支持 docker-compose.yml，直接导入就行；直接复制 Checkmate 的 Mongo 配置。

RabbitMQ & Redis 超级简单
- 直接用 Dokploy 内置 Database 服务（RabbitMQ / Redis）。
- 默认 guest/guest 和 redis://redis:6379 就行（内部网络，不暴露公网，很安全）。
- 项目 Docker run 示例里也是外部连接，完全兼容。

从GitHub仓库拉取代码还是Docker Hub拉取镜像？
推荐GitHub

还要考虑：环境变量、内存、数据导入、swaggerUI测试…

时间规划：
0.5～1 小时：快速过 repo 结构，锁定关键文件
1～2 小时：读运行文档、Docker/构建配置、release workflow
1～2 小时：查 Dokploy 自托管支持方式，确定最佳部署路径
1～2 小时：整理具体配置项、依赖、网络、域名/TLS、存储
0.5～1 小时：写交付文档并列风险/缺口

本地medical-server上一级目录创建一个和medical-server同级别文件夹/deploy-spike，把它变成一个git repo，作为实验日志整理器、配置生成器，和 medical-server 主仓库解耦。因为部署过程中产生的临时脚本、文档，很容易把主仓库弄脏

用cursor打开 和 /medical-server 同级别文件夹/deploy-spike ，记录做的事情的时间线：
* 尝试过的路径
* 每一步操作
* 配置草案
* 错误截图/错误日志摘录
* 为什么放弃某个方案
* 最终推荐方案

用cursor IDE打开这个目录，用浏览器打开dokploy，用鼠标操作，在dokploy的每一步尝试都和cursor IDE说，让cursor生成相关md文档，每次更新都commit。
（gpt：我支持频繁 commit，但建议按“一个实验单元”来 commit，而不是每点一下网页就 commit 一次。不然历史会太碎，自己回头也难看。）

建议目录
deploy-spike/
├─ README.md
├─ journal/
│  ├─ 2026-03-10-session-1.md
│  ├─ 2026-03-10-session-2.md
├─ findings/
│  ├─ repo-runtime-summary.md
│  ├─ dokploy-options.md
│  ├─ recommended-setup.md
│  ├─ blockers-and-risks.md
├─ configs/
│  ├─ env-template.md
│  ├─ dokploy-app-config.md
│  ├─ docker-notes.md
├─ evidence/
│  ├─ screenshots/
│  ├─ logs/
│  ├─ error-snippets.md
└─ decisions/
   ├─ 001-use-dockerfile-build.md
   ├─ 002-require-postgres-managed-separately.md

journal/：记录时间线和每次尝试
repo-runtime-summary.md：从 medical-server 仓库读出来的运行要求总结

findings/recommended-setup.md：最终推荐的 Dokploy 自托管部署方案:
结构可以很简单：
* 推荐部署方式
* Dokploy 中的应用类型
* 构建方式
* 运行命令
* 依赖服务
* 环境变量
* 端口/域名/TLS
* 存储
* 上线前检查项
* 已知 blocker
blockers-and-risks.md：已知风险、缺口、后续动作

先定义“记录格式”，不然还是会乱，例如每条记录都包含：
* 时间
* 目标
* 操作
* 结果
* 证据
* 结论
* 下一步

不只是记录“操作”，还要记录“为什么”：
* 为什么选这个方案
* 为什么排除另一个方案
* 哪个结论来自 repo，哪个结论来自 Dokploy UI 实测
* 哪个只是推断，还没验证

建议把结论分成三类：
* Confirmed：已验证
* Inferred：根据代码/文档推断
* Unknown：仍未确认

提前想好“敏感信息管理”:

deploy-spike repo 最好一开始就加上：
* .gitignore
* 明确禁止提交真实密钥
* 截图前注意别把 secrets 拍进去
* 日志里把敏感值替换成占位符

建议固定约定：
* 文档里只写 `DB_PASSWORD=<set in Dokploy secret>`
* 不写真实值
* 截图如果带 secrets，先打码再放

还要记录“不是 Dokploy UI 内能完成的前置条件”:
如可能包括：
* VPS 基础规格够不够
* 防火墙端口是否开放
* DNS 是否已指向
* 反向代理/TLS 由谁接管
* 医疗服务依赖的数据库/Redis/对象存储由谁提供
* 备份策略有没有
* 日志/监控有没有
即使你这次不把它们都做完，也要在文档里标注：
* “Dokploy 内完成”
* “VPS/infra 前置完成”
* “需应用团队补充”

最后要区分“可运行”与“可上线”:
Minimum workable setup
* Dokploy 自托管
* 一个 app 服务
* 一个 Postgres
* 域名/TLS
* 环境变量完整
* 健康检查可用
Production hardening follow-ups
* 备份
* 监控告警
* Secret rotation
* 多环境隔离
* 灰度/回滚
* 资源限制与扩缩容策略

还要写一个“实际操作流程”文档，指导一步步操作，随着探索深入，文档也要更新。
内容包括（设想）：
安装 Dokploy：操作SSH
部署 Raidar Server 到 Dokploy：浏览器操作，填配置信息，点击deploy按钮
MongoDB replica set 配置（难点）
数据导入：部署完mongodb后，在浏览器dokploy终端执行mongosh命令

