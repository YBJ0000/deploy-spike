# 自托管 Dokploy vs Dokploy Cloud（与 issue #157 的符合性）

## 结论摘要

- **issue #157 要求**：使用 **open-source, self-hosted** 版 Dokploy，**not a paid hosted solution**。
- **截图中的「Create project 失败 + 需订阅」**：来自 **Dokploy Cloud**（付费托管版），**不符合** issue #157。不应通过订阅 Hobby/Startup 来继续。
- **符合 issue #157 的做法**：在**自己的 Linux 环境**上安装 **自托管版 Dokploy**（免费、无需向 Dokploy 付费）。可用「远程 Linux VPS」或「本机 Mac 上的 Linux 虚拟机」获得该环境。

---

## 两种产品区别

| 项目 | 自托管版（Self-Hosted） | Dokploy Cloud（托管版） |
|------|-------------------------|--------------------------|
| 是什么 | 你在自己的 Linux 服务器上运行 Dokploy 软件 | 使用 Dokploy 官方的托管服务（app.dokploy.com 等） |
| 费用 | 软件免费；你只付服务器成本（若用 VPS 则付给 VPS 商） | 按订阅付费（如 Hobby $13.5/月、Startup $15/月） |
| 创建项目时 | 项目跑在你安装 Dokploy 的那台机器上，无需「订阅」 | 需要先「Subscribe to a plan」才能有可用 Server，才会出现「Create project」成功 |
| 与 issue #157 | **符合**（self-hosted, not paid hosted） | **不符合**（paid hosted solution） |

你当前在界面上看到的 “No servers available, Please subscribe to a plan” 是 **Dokploy Cloud** 的流程：在托管版里要先订阅计划、分配 server 才能建项目。  
若走 issue #157 的路线，应**不订阅** Dokploy Cloud，改为在**自己的 Linux 上装自托管版**。

---

## 没有远程 Linux VPS 时怎么办？Mac 能否获得同等效果？

可以。**不需要**一定有一台远程的、可 SSH 的 Linux VPS，只要有一个 **Linux 环境** 即可：

- **方式一：租用 Linux VPS**  
  SSH 登录后在那台机器上执行官方安装脚本，Dokploy 就装在那台 VPS 上。你只付 VPS 费用，不付 Dokploy 订阅费。

- **方式二：在 Mac 上跑 Linux 虚拟机（推荐用于本地验证）**  
  用 **Multipass** 或 **UTM** 在 Mac 上起一个 Linux 虚拟机，在虚拟机里执行同样的安装脚本，效果等同于「有一台 Linux 服务器」：
  - **Multipass**（Canonical）：`multipass launch docker` 即可得到带 Docker 的 Ubuntu，适合本机快速起一个「本地 Linux 服务器」。
  - **UTM**（Apple Silicon 可用）：可装完整 Linux，资源占用相对更大。

装好后，Dokploy Web 界面跑在虚拟机里，浏览器访问 `http://<虚拟机IP>:3000` 即可。**不需要**在 Mac 上 SSH 到「远程」VPS，但**需要**能访问到「这台 Linux」（本机 VM 或远程 VPS 均可）。

**本机 VM 与 VPS 的差别（访问与持久化）**：

- **本机 VM**：使用私网 IP，仅本机（及同局域网）可访问；Mac 休眠（合盖）后 VM 会挂起，服务不可用。适合**探索可行方案、本地验证**；若要让外部/公网持续访问，需做端口映射或隧道，或改用 VPS。
- **Linux VPS**：有公网 IP，可 7×24 运行，外部可直接访问；适合**持久化、对外提供服务**。

---

## 是否必须 SSH 到 Linux 才能用 Dokploy？

- **安装阶段**：必须在「这台 Linux」上执行安装脚本，所以需要能在这台机器上执行命令：  
  - 远程 VPS → 用 SSH 连上去执行；  
  - 本机 Linux VM → 用 `multipass shell` 或 UTM 的终端在 VM 里执行。
- **安装完成后**：Dokploy 自带 Web 界面和**终端能力**：
  - 在 **Settings → Servers → 对应服务器 → Enter Terminal** 可进该服务器的 shell；
  - 对部署的应用/容器也有终端或「Run Command」类入口。  
  日常操作不必每次都 SSH 到主机，用 Dokploy 网页里的终端即可；但**首次安装**必须在 Linux 上跑脚本（SSH 或 VM 终端）。

---

## 是否需要开 Startup（或 Hobby）计划？

**不需要。**  
Hobby/Startup/Enterprise 是 **Dokploy Cloud** 的订阅计划。issue #157 明确要求 **not a paid hosted solution**，即不用托管版、不订阅。  
应使用 **自托管版**：在自己的 Linux（VPS 或 Mac 上的 Linux VM）安装，不购买 Dokploy 的 plan。

---

## 若无法获得任何 Linux 环境，是否还有必要做下去？

若既没有、也不打算租 VPS，且不在 Mac 上起 Linux VM，则**无法安装自托管 Dokploy**（官方脚本仅支持 Linux）。  
此时可以：

- **汇报结论**：自托管 Dokploy 需至少一台 Linux 环境（远程 VPS 或本机虚拟机）；在仅有 macOS 且不提供 Linux 环境的前提下，无法完成 issue #157 的部署目标。
- **若愿意在本机起 Linux VM**：可按上述「方式二」用 Multipass 在 Mac 上获得与 Linux VPS 同等效果，再按操作指南在 VM 内安装 Dokploy，继续后续步骤。

---

## 建议的下一步（符合 issue #157）

1. **不要**在 Dokploy 官网订阅 Hobby/Startup。
2. 二选一：
   - 租一台 Linux VPS，SSH 上去执行：`curl -sSL https://dokploy.com/install.sh | sudo sh`；或  
   - 在 Mac 上安装 Multipass，起一个 Linux VM，在 VM 内执行同一条命令。
3. 安装完成后在浏览器打开 `http://<该 Linux 的 IP>:3000`，在**自托管实例**里创建项目、部署 medical-server，并记录在 deploy-spike 的 journal 与操作指南中。
