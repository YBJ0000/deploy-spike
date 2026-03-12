# medical-server 分支 `spike/dokploy-deployment` 代码变更整理

本文档整理 **medical-server** 仓库中 **spike/dokploy-deployment** 分支相对 **main** 的代码变更。该分支在自托管 Dokploy 部署验证过程中因容器启动失败（Spring 依赖注入歧义）被迫修改了部分 Java 代码与构建方式；此处仅作记录，便于后续向 main 回 port 或评审。

---

## 1. 变更概览

| 类型 | 路径 | 说明 |
|------|------|------|
| 新增 | `app/Dockerfile` | 多阶段 Docker 构建，替代 bootBuildImage，避免 EXPORTING 阶段失败 |
| 新增 | `app/.dockerignore` | 镜像构建时排除 build、.gradle、.idea 等 |
| 修改 | `app/README.md` | 补充 Dockerfile 构建说明与使用场景 |
| 修改 | `app/src/main/java/.../file/FileService.java` | 显式构造器 + `@Qualifier` 消除 GridFsTemplate 注入歧义 |
| 修改 | `app/src/main/java/.../utils/ThymeleafService.java` | 显式构造器 + `@Qualifier` 消除 ConversionService 注入歧义 |

**说明**：分支上曾对 `app/build.gradle` 做过修改（如指定 `imagePlatform`、docker username 等），后经 **revert** 恢复为与 main 一致，镜像改为通过 Dockerfile 构建，故当前 **main..spike/dokploy-deployment** 的 diff 中不包含 `build.gradle` 变更。

---

## 2. 提交历史（main 之后）

在 medical-server 仓库中执行 `git log main..spike/dokploy-deployment --oneline` 得到（从新到旧）：

```
e08eec79 fix: disambiguate ConversionService injection for ThymeleafService
ed118606 fix: No qualifying bean of type 'org.springframework.data.mongodb.gridfs.GridFsTemplate' available: expected single matching bean but found 2: masterTenantGridFsTemplate,subTenantGridFsTemplate
aecf7f74 build: add Dockerfile and docs for image build (alternative to bootBuildImage)
5ea20d91 revert: restore app/build.gradle to main (image build via Dockerfile)
308a1712 fix: use linux/amd64 explicitly
6d938e1b specify docker username explicitly
```

---

## 3. 变更背景与对应失败日志

部署到 Dokploy 后，raidar 容器多次在启动阶段退出，日志中出现 `APPLICATION FAILED TO START` 与 `expected single matching bean but found 2`。原因与对应修复如下。

### 3.1 GridFsTemplate 注入歧义（FileService）

- **日志**：`deploy-spike/raidar-20260311_075328.log.txt`  
  `Error creating bean with name 'fileService' ... No qualifying bean of type 'org.springframework.data.mongodb.gridfs.GridFsTemplate' available: expected single matching bean but found 2: masterTenantGridFsTemplate, subTenantGridFsTemplate`
- **原因**：Spring 容器中存在两个 `GridFsTemplate` Bean（master / sub tenant），`FileService` 使用 `@RequiredArgsConstructor` 注入时未指定要哪一个，导致歧义。
- **修复**（commit `ed118606`）：在 **FileService** 中改为显式构造器，并对两个 `GridFsTemplate` 参数分别使用 `@Qualifier("masterTenantGridFsTemplate")` 与 `@Qualifier("subTenantGridFsTemplate")`，移除字段上的 `@Qualifier`、去掉 `@RequiredArgsConstructor`。

### 3.2 ConversionService 注入歧义（ThymeleafService）

- **日志**：`deploy-spike/raidar-20260311_081135.log.txt`  
  `Parameter 2 of constructor in com.raidar.server.utils.ThymeleafService required a single bean, but 2 were found: templateFormattingConversionService, mvcConversionService`
- **原因**：存在两个 `ConversionService` Bean，`ThymeleafService` 的构造器注入未指定使用 `templateFormattingConversionService`。
- **修复**（commit `e08eec79`）：在 **ThymeleafService** 中改为显式构造器，对 `ConversionService` 参数使用 `@Qualifier("templateFormattingConversionService")`，去掉 `@RequiredArgsConstructor`。

### 3.3 构建方式：Dockerfile 与 build.gradle 的取舍

- **背景**：本地使用 `bootBuildImage` 构建镜像时，在 **EXPORTING** 阶段反复出现 `content digest ... not found`（与 Docker Desktop containerd 存储的兼容问题）；另在尝试指定平台时曾改过 `build.gradle`。
- **最终做法**（commit `aecf7f74`、`5ea20d91`）：在 `app/` 下新增 **Dockerfile** 与 **.dockerignore**，用 `docker build` 多阶段构建镜像；并将 **app/build.gradle** 恢复为与 main 一致，不再为镜像构建保留特殊配置。构建与推送步骤见 deploy-spike 的 [docs/build-and-push-raidar-image.md](../docs/build-and-push-raidar-image.md)。

---

## 4. 具体代码差异摘要

### 4.1 FileService.java

- 去掉 `@RequiredArgsConstructor`。
- 去掉字段上的 `@Qualifier`。
- 新增显式构造器，在构造器参数上使用：
  - `@Qualifier("masterTenantGridFsTemplate") GridFsTemplate masterGridFsTemplate`
  - `@Qualifier("subTenantGridFsTemplate") GridFsTemplate subGridFsTemplate`

### 4.2 ThymeleafService.java

- 去掉 `@RequiredArgsConstructor`。
- 去掉字段上的 `@Qualifier("templateFormattingConversionService")`。
- 新增显式构造器，在构造器参数上使用：
  - `@Qualifier("templateFormattingConversionService") ConversionService conversionService`

### 4.3 新增 Dockerfile（app/Dockerfile）

- 多阶段：builder 使用 `eclipse-temurin:21-jdk-jammy`，执行 `./gradlew bootJar -x test`；runtime 使用 `eclipse-temurin:21-jre-jammy`，复制 JAR 并以 `java -jar app.jar` 启动。
- 设置 `JAVA_TOOL_OPTIONS="-XX:MaxDirectMemorySize=256M"`，EXPOSE 8080。

### 4.4 新增 .dockerignore（app/.dockerignore）

- 排除：`build`、`.gradle`、`.idea`、`*.iml`、`.DS_Store`、`.git`、`.gitignore`、`*.md`、`*.http` 等，保留 `gradle/wrapper`。

### 4.5 README.md（app/README.md）

- 在 “Build” 小节中区分 **Spring Boot Buildpacks（默认）** 与 **Dockerfile 构建（替代方案）**，并说明当 `bootBuildImage` 在 EXPORTING 失败时使用 `docker build --platform linux/amd64 -t <image-name> .`，以及参考 deploy-spike 文档。

---

## 5. 相关文档与日志

- 启动失败排查与重试流程：[findings/raidar-startup-failures.md](../findings/raidar-startup-failures.md)
- 时间线与尝试记录：[journal/timeline-and-attempts.md](./timeline-and-attempts.md)（Raidar 容器启动失败、ConversionService / GridFsTemplate 等条目）
- 会话记录：[journal/2026-03-10-session-1.md](./2026-03-10-session-1.md)
- 失败日志（供对照）：`raidar-20260311_073420.log.txt`、`raidar-20260311_075328.log.txt`、`raidar-20260311_081135.log.txt`

---

## 6. 后续建议

- **合并回 main**：两处 Java 修改（FileService、ThymeleafService）均为“消除 Spring 注入歧义”的保守修复，不改变业务逻辑，建议在 main 上做等价修改并跑通现有测试后合并；Dockerfile / .dockerignore / README 的增补可作为可选构建方式一并合入或单独 PR。
- **仅用分支构建镜像**：若短期不合并，部署 Dokploy 时需从 **spike/dokploy-deployment** 分支构建并推送镜像（见 deploy-spike 的 build-and-push 文档），再在 Compose 中引用对应 tag。
