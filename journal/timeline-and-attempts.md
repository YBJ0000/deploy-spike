# 时间线与尝试记录

按「时间 | 目标 | 操作 | 结果 | 证据 | 结论 | 下一步」记录每次操作；并标注结论类型：**Confirmed**（已验证）/ **Inferred**（推断）/ **Unknown**（未确认）。

---

## 记录模板（每条可复制使用）

```markdown
### YYYY-MM-DD HH:MM — 简短标题

- **目标**：
- **操作**：
- **结果**：（成功 / 失败 / 部分成功）
- **证据**：（截图路径、错误日志片段、链接）
- **结论**：Confirmed / Inferred / Unknown
- **下一步**：
```

---

## 前置条件检查结果（2026-03-10）

| 检查项       | 结果说明 |
|--------------|----------|
| 当前环境     | macOS (Darwin)，非 Linux VPS |
| Docker       | 已安装 v28.0.1；沙箱内无法访问 daemon，正常终端可用 |
| 端口 3000/80/443 | 本机未被监听，可用于将来在 VPS 上安装 |
| 官方安装脚本 | 仅支持 Linux、须 root；在 Mac 上执行会报错退出 |
| 结论         | 安装须在 Linux VPS 上执行；Mac 仅能做文档与配置准备 |

---

## 已尝试过的路径与错误方案

（随实际操作在此补充：哪些方案试过、报什么错、为什么放弃。）

| 日期       | 方案简述           | 结果   | 放弃/保留原因 |
|------------|--------------------|--------|----------------|
| 2026-03-10 | 在 Mac 本机运行官方安装脚本 `curl ... \| sh` | 失败   | 脚本要求 root，且明确拒绝 Darwin，仅支持 Linux |
| 2026-03-10 | 在 Dokploy Cloud 界面直接 Create project | 失败   | 托管版需订阅才有 Server；与 issue #157「not a paid hosted solution」不符，应改用自托管 |
| 2026-03-10 | 在 Cursor 环境自动执行 `brew install --cask multipass` | 未完成 | 安装需 sudo 密码，无交互环境无法输入；已记录于 [docs/mac-multipass-dokploy.md](../docs/mac-multipass-dokploy.md)，需在本机终端手动执行第一步 |
| 2026-03-10 | 在 Multipass VM 内执行安装脚本安装 Dokploy | 成功 | 输出 “Congratulations, Dokploy is installed!”；进入自托管面板继续后续步骤 |
| 2026-03-10 | Compose 部署时从 GitHub Select repository 选 medical-server | 不可见 | medical-server 为公司组织私有库，个人 GitHub 连接后看不到；改用个人有权限的 repo（如 deploy-spike）作 compose 来源，或 Raw/组织授权/Fork，见 operating-guide |
| （示例）   | 用内置 MongoDB     | 失败   | 无法开 rs0     |

---

## 最终推荐方案摘要

（在探索结束后填写：采用哪种部署方式、关键配置、文档索引。）

- 推荐方式：
- 关键配置/文档：
- 已知 blocker / 后续事项：
