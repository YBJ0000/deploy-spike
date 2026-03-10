# Mac + Multipass 安装 Dokploy 逐步命令清单

在 Mac 上通过 Multipass 跑一台 Ubuntu 虚拟机，在该 VM 内安装自托管版 Dokploy，用于符合 issue #157 的部署（self-hosted, not a paid hosted solution）。

> **说明**：第一步（安装 Multipass）须在**本机终端**中执行，安装系统包时可能提示输入本机管理员密码，无法在无交互环境（如 CI/脚本）中自动完成。

---

## 环境要求

- macOS（Intel 或 Apple Silicon 均可；Apple Silicon 上 Multipass 使用 QEMU）
- 至少 4GB 可用内存、约 35GB 可用磁盘（VM 需 2GB 内存 + 30GB 磁盘，Dokploy 官方建议）
- 网络可访问 GitHub、dokploy.com 等

---

## 命名约定（避免歧义）

为避免把「Dokploy（产品/面板）」和「Multipass VM」混淆，本文统一约定：

- **VM 名称**：`dokploy-vm`（表示“跑 Dokploy 的虚拟机”）
- **Dokploy（产品）**：一律写作 Dokploy
- **Dokploy Cloud**：写作 Dokploy Cloud（付费托管版）

建议避免在同一套文档里把 VM、项目、应用都起名为 `dokploy`，否则后续排错时很难描述“到底是哪一个 dokploy”。

---

## 第一步：安装 Multipass

在 Mac 终端执行：

```bash
brew install --cask multipass
```

**验证**：

```bash
multipass --version
```

若出现版本号（如 `multipass 1.16.1`）即安装成功。

**若本步出错**：在本文档末尾「已遇到的问题与处理」中记录错误信息与解决方式。

---

## 第二步：创建 Ubuntu 虚拟机

创建一台满足 Dokploy 要求的 VM（2GB 内存、30GB 磁盘，Ubuntu 22.04）：

```bash
multipass launch 22.04 --name dokploy-vm --memory 2G --disk 30G --cpus 2
```

- `--name dokploy-vm`：实例名称，后续用 `multipass shell dokploy-vm` 进入。
- `--memory 2G`：Dokploy 官方最低要求 2GB。
- `--disk 30G`：官方建议至少 30GB。
- `--cpus 2`：建议 2 核，便于构建与运行。

**验证**：

```bash
multipass list
```

应看到 `dokploy-vm` 状态为 `Running`。

**若本步出错**：记录错误（例如 QEMU/ARM 相关报错），并更新本文档「已遇到的问题与处理」。

---

## 第三步：进入 VM 并安装 Dokploy

进入虚拟机：

```bash
multipass shell dokploy-vm
```

此时终端提示符会变为 Ubuntu 用户（如 `ubuntu@dokploy-vm`）。在 **VM 内**执行：

```bash
curl -sSL https://dokploy.com/install.sh | sudo sh
```

脚本会检测最新稳定版、安装 Docker（若未安装）、初始化 Docker Swarm、部署 Dokploy + PostgreSQL + Redis。根据网络情况可能需要数分钟。

**成功判据（示例输出）**：

- 看到类似以下内容即可认为安装完成（实际 IP 以你的输出为准）：

```text
Congratulations, Dokploy is installed!
Wait 15 seconds for the server to start
Please go to http://<IP>:3000
```

**注意：脚本输出的 `<IP>` 可能不可用（尤其是本机 Multipass VM 场景）**：

- 在 **VPS** 场景，脚本通常能探测到 VPS 的公网 IP，这个地址一般可直接从浏览器访问。
- 在 **Mac + Multipass** 场景，VM 往往处于 NAT/私网中，脚本可能打印“探测到的公网 IP”（例如你的路由器/出口公网 IP），但这并不代表端口 3000 已做端口映射，因此用该公网 IP 访问可能一直转圈。
- **本机 Multipass 推荐访问方式**：一律以 `multipass list` 显示的 VM 私网 IP 为准（例如 `192.168.64.x`），访问 `http://<VM私网IP>:3000`。

安装完成后，在 VM 内可先退出（后续用浏览器访问即可）：

```bash
exit
```

**若本步出错**：记录完整错误输出（例如端口占用、网络失败、权限错误），并更新本文档「已遇到的问题与处理」。

---

## 第四步：获取 VM IP 并访问 Dokploy

在 **Mac 终端**（不在 VM 内）执行：

```bash
multipass list
```

记下 `dokploy-vm` 对应的 **IPv4**（例如 `192.168.64.2`）。

在浏览器中打开：

```
http://<上一步的 IPv4>:3000
```

例如：`http://192.168.64.2:3000`。

首次访问会进入 Dokploy 初始化页面，按提示创建管理员账号。创建项目时**不会**再出现「No servers available, Please subscribe to a plan」（那是 Dokploy Cloud 的提示）；自托管版下当前机器即为一台可用 Server。

---

## 第五步：后续使用

- **再次进入 VM**：`multipass shell dokploy-vm`
- **查看 VM 状态 / IP**：`multipass list`
- **停止 VM**：`multipass stop dokploy-vm`
- **启动 VM**：`multipass start dokploy-vm`
- **删除 VM**（会删除其中所有数据）：`multipass delete dokploy-vm && multipass purge`

Dokploy 的 Web 终端：登录后 **Settings → Servers → 对应服务器 → Enter Terminal**，无需每次 SSH 或 `multipass shell`。

---

## 已遇到的问题与处理

（执行过程中若某步报错，在此记录错误现象、原因与解决方式，便于后续复现或他人排错。）

| 日期       | 步骤     | 错误现象 | 原因/处理 |
|------------|----------|----------|-----------|
| 2026-03-10 | 第一步   | `brew install --cask multipass` 报错：`sudo: a terminal is required to read the password`，安装器退出 | Homebrew Cask 安装 .pkg 时会调用 `sudo installer`，需要在本机终端中输入管理员密码。在无交互环境（如 Cursor/自动化脚本）中无法代输密码。**处理**：请在本机打开「终端」或 iTerm，手动执行 `brew install --cask multipass`，按提示输入密码完成安装；安装成功后执行 `multipass --version` 验证，再继续第二步。 |
| 2026-03-10 | 第四步   | 安装脚本提示访问 `http://<公网IP>:3000`，但浏览器一直转圈；改用 `http://192.168.64.x:3000` 可正常打开 | 在 Mac + Multipass 场景中，VM 处于私网/NAT，脚本可能打印“探测到的出口公网 IP”，但端口 3000 未做公网映射。**处理**：以 `multipass list` 的 VM 私网 IP 为准，访问 `http://<VM私网IP>:3000`。 |
| （示例）   | 第二步   | 启动失败 | 若为 Apple Silicon 且报 QEMU/CPU 相关错误，可查 [Multipass GitHub Issues](https://github.com/canonical/multipass/issues) 或尝试更新 Multipass。 |

---

## 与操作指南的衔接

完成本文档第四步后，即可按 [operating-guide.md](../operating-guide.md) 从 **第二步：登录并创建项目** 继续（在浏览器打开 `http://<VM IP>:3000`，用刚创建的管理员账号登录并建项目）。
