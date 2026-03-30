# vps-vpn-setup

一个 Claude Code skill，帮你一键完成 VPS + VPN + Clash 代理环境的部署。

## 这个 skill 做什么

你只需要提供服务器信息，Claude 会接管所有技术操作：

1. SSH 远程连入服务器
2. 安装 v2ray-agent（VLESS Reality 无域名版）
3. 获取 clashMeta 订阅链接
4. 给你 Clash Verge 的配置清单（图形界面部分由你操作）
5. 验证配置是否生效（出口 IP、服务状态、DNS 检查）

**适用场景：** 已购买 VMiss 或 DMIT 服务器，想通过干净的海外 IP 访问 Claude / ChatGPT 等 AI 工具。

## 购买 VPS

推荐两个服务商（以下为推广链接，对你没有额外费用）：

| 服务商 | 价格 | 备注 |
|--------|------|------|
| **[VMiss](https://app.vmiss.com/aff.php?aff=4686)** | $5 / 月起 | 自用推荐，IP 质量好，线路选择多 |
| [DMIT](https://www.dmit.io/aff.php?aff=19146) | $9.9 / 月起 | 备选，稳定性好 |

## 前置要求

- 已购买 VPS
- 已安装 [Claude Code](https://claude.ai/code)
- Mac / Linux，或 Windows 用户见下方说明

## 安装

```bash
claude skill install https://github.com/huasan2025/vps-vpn-setup
```

或手动复制 `skill/` 目录到 `~/.claude/skills/vps-vpn-setup/`。

## 使用方法

在 Claude Code 会话中输入：

```
/vps-vpn-setup 我的 VPS 服务器信息：
- IP地址：1.2.3.4
- 密码：your_password
- 服务商：VMiss
```

或 DMIT（PEM 登录）：

```
/vps-vpn-setup 我的 VPS 服务器信息：
- IP地址：1.2.3.4
- PEM文件：~/Downloads/id_rsa.pem
- 服务商：DMIT
```

Claude 会开始自动部署，全程告诉你每步在做什么。

## Windows 用户说明

skill 的自动化部分依赖 `sshpass` 和 `expect`，这两个工具在 Windows 原生环境下不可用。有两条路：

**推荐：安装 WSL2（一劳永逸）**

WSL2 在 Windows 里装一个 Ubuntu 子系统，之后所有命令和 Mac/Linux 完全一致。

```powershell
# 在 PowerShell（管理员）里执行
wsl --install
```

重启后打开"Ubuntu"应用，再在其中安装 Claude Code，skill 就能正常跑。

**备选：DMIT 用户可以跳过 WSL**

如果你用的是 DMIT（PEM 私钥登录），Windows 10/11 内置的 SSH 客户端支持 `-i` 参数，可以直接连服务器：

```powershell
# 在 PowerShell 或 Windows Terminal 里
ssh -i C:\Users\你的用户名\.ssh\id_rsa.pem root@服务器IP
```

但 skill 的自动化菜单交互（expect 部分）仍然需要 WSL。如果不想装 WSL，可以参考[原始教程](https://github.com/huasan2025/vps-vpn-clash-setup)手动完成服务端配置，Clash 客户端部分 Windows 和 Mac 完全一样。

## 目录结构

```
vps-vpn-setup/
├── SKILL.md     # skill 主体，Claude 的执行指令
└── README.md    # 本文件
```

## 关联教程

这个 skill 基于教程：[用干净 IP 访问 Claude：VPS + VPN + Clash 完整配置教程](https://github.com/huasan2025/vps-vpn-clash-setup)

如果 skill 的自动化步骤失败，可以参考原始教程手动完成。

## 已知限制

- v2ray-agent 的安装菜单会随版本更新变化。skill 采用动态解析菜单的方式提高兼容性，但如果解析失败会自动切换为手动指引模式。
- Clash Verge 的图形界面配置无法自动化，skill 会生成清单由用户手动操作。
- 密码登录需要本机安装 `sshpass`（skill 会自动检测并提示安装）。

## License

MIT
