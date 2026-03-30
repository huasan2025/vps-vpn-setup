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

## 前置要求

- 已购买 VPS（推荐 [VMiss](https://app.vmiss.com/aff.php?aff=4686) 或 [DMIT](https://www.dmit.io/aff.php?aff=19146)）
- Mac 或 Linux（Windows 未测试）
- 已安装 [Claude Code](https://claude.ai/code)

## 安装

```bash
claude skill install https://github.com/huasan2025/vps-vpn-clash-setup/tree/main/skill
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
