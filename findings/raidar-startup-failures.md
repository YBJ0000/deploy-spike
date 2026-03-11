# Raidar 容器启动失败排查（Dokploy）

本文档用于排查 Dokploy 上 `raidar` 容器 **启动后立刻退出**，导致「Open Terminal」提示 `container ... is not running` 的问题。

> 结论先行：当前遇到的失败通常不是 Docker Hub/网络问题，而是 **应用启动阶段 Spring 依赖注入冲突**（同类型 Bean 存在多个）。需要通过 **修改 medical-server 代码**解决，并重新构建/推送镜像（建议使用带 tag 的 multi-arch），再 Deploy。

---

## 如何快速判断是哪一类问题

- **镜像/缓存问题**（较少见）：日志里看不到 Spring 报错，或者报错与最新代码不一致；Compose 使用 `server-latest` 这种 tag 时更容易混淆。
- **应用启动失败**（本次属于这个）：日志明确出现 `APPLICATION FAILED TO START`，并指出 `expected single matching bean but found 2`。

---

## 本次最新错误（2026-03-11 08:11）

日志文件：`deploy-spike/raidar-20260311_081135.log.txt`

关键信息：

- `APPLICATION FAILED TO START`
- `ThymeleafService required a single bean, but 2 were found`
- `templateFormattingConversionService` 与 `mvcConversionService`

含义：

- Spring 容器里有 **两个** `ConversionService`，而 `ThymeleafService` 的构造器注入点没有被 Spring 正确识别为“要哪个”。
- 报错里提到 `-parameters` 只是提示之一；更稳妥的是直接用 `@Qualifier` 指定。

修复方式（已在 medical-server 中提交）：

- 在 `ThymeleafService` 改成 **显式构造器注入**，并在构造器参数上标注：
  - `@Qualifier("templateFormattingConversionService") ConversionService conversionService`

---

## 另一个常见错误：MongoDB rs0 未初始化

同一份日志也出现：

- `Server mongodb:27017 does not appear to be a member of an initiated replica set.`
- `type=REPLICA_SET_GHOST`

这会导致应用后续访问 Mongo 时失败（即使 Spring 启动问题修好，也必须初始化 rs0）。

处理方式见 `deploy-spike/operating-guide.md` 中 rs0 初始化步骤（如果该步骤还没做，必须先做）。

---

## 推荐的“最小可控”重试流程（避免误删镜像）

不建议一上来就“删除本地+DockerHub 的所有镜像”。更快更可控的做法：

1. **修复代码**（medical-server）并提交。
2. 生成新 tag（例如 `server-20260311-2`），并用 `buildx` 推送 multi-arch 镜像到 Docker Hub。
3. 更新 `deploy-spike/configs/docker-compose-medical-server.yml` 使用新 tag。
4. push `deploy-spike` 后在 Dokploy 重新 Deploy。
5. 只要应用仍退出：下载 `raidar` logs，看是否出现新的 `APPLICATION FAILED TO START` 线索。

---

## 如果你仍想“清理镜像”，建议优先清理 Dokploy 侧

清理优先级（从温和到激进）：

- **优先**：更换 image tag（最稳定、最可审计）
- 其次：在 Dokploy 上对该 Compose 做一次 Down/Up（避免旧容器残留）
- 最后：清理 Dokploy 服务器上的 Docker images（可能影响其他服务，不建议作为第一选择）

