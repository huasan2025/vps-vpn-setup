---
name: vps-vpn-setup
description: |
  一键部署 VPS + VPN + Clash 代理环境。适合已购买 VMiss 或 DMIT 服务器、想通过干净 IP 访问 Claude/ChatGPT 等 AI 工具的用户。

  触发场景：
  - 用户提供了 VPS 的 IP、密码或 PEM 密钥，要求部署代理环境
  - 用户说"帮我装 VPN"、"帮我配置服务器"、"我买了 VMiss/DMIT"
  - 用户提供了形如 /skill VPS信息: IP: xxx 的格式

  覆盖能力：SSH 远程执行、v2ray-agent 安装（VLESS Reality）、订阅链接获取、Clash Verge 配置指导、DNS 防泄漏验证。

  只要用户提供了 VPS 凭据并表达想配代理，ALWAYS invoke this skill。
---

# VPS + VPN + Clash 一键部署

## 角色定位

你是一个自动化部署 Agent。目标是：用户提供 VPS 信息 → 你接管所有技术操作 → 最终用户只需在 Clash Verge 里点几下。

整体流程分为 **4 个阶段**，前两个阶段完全自动化，后两个阶段需要用户在图形界面操作（你提供清单）。

---

## 阶段零：解析用户信息

从用户消息中提取以下信息，缺什么问什么，确认后再继续：

| 字段 | 必填 | 说明 |
|------|------|------|
| `VPS_IP` | ✅ | 服务器 IPv4 地址 |
| `AUTH_TYPE` | ✅ | `password`（VMiss）或 `pem`（DMIT） |
| `VPS_PASSWORD` | 密码登录必填 | SSH 登录密码 |
| `PEM_PATH` | PEM登录必填 | 本地 PEM 文件路径，如 `~/Downloads/id_rsa.pem` |
| `OS` | 可选 | 默认 Ubuntu/Debian，差异不大 |

提取完后向用户确认一次（密码只显示 `***`），然后进入阶段一。

---

## 阶段一：VPS 服务端自动化部署

**目标：** SSH 进入服务器 → 安装环境 → 安装 v2ray-agent → 获取 clashMeta 订阅链接

### 1.0 本机前置依赖检查

密码登录需要 `sshpass`，先检查是否安装：

```bash
which sshpass 2>/dev/null || echo "NOT_FOUND"
```

如果输出 `NOT_FOUND`，自动安装：

```bash
brew install hudochenkov/sshpass/sshpass
```

> 如果没有 Homebrew，先安装：`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
> PEM 登录（DMIT）不需要 sshpass，可跳过此步。

### 1.1 SSH 连通性测试

先测试能否连上服务器。根据 `AUTH_TYPE` 选择方式：

**密码登录（VMiss）：**
```bash
# 用 sshpass 执行一个简单命令确认连通
sshpass -p 'VPS_PASSWORD' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 root@VPS_IP "echo SSH_OK"
```

**PEM 登录（DMIT）：**
```bash
# 先设置权限，再测试连通
chmod 600 PEM_PATH
ssh -i PEM_PATH -o StrictHostKeyChecking=no -o ConnectTimeout=10 root@VPS_IP "echo SSH_OK"
```

如果输出 `SSH_OK`，继续。如果失败，向用户报告错误信息，常见原因：
- 密码错误 → 让用户重新确认
- 端口不是 22 → 询问端口号
- PEM 权限问题 → 检查文件路径

### 1.2 安装基础依赖

```bash
sshpass -p 'VPS_PASSWORD' ssh -o StrictHostKeyChecking=no root@VPS_IP "apt update && apt install -y wget curl sudo expect"
```

PEM 版本替换 `sshpass -p '...' ssh` 为 `ssh -i PEM_PATH`（后续同理，不再重复说明）。

### 1.3 关闭自动更新（防止后续断连）

```bash
sshpass -p 'VPS_PASSWORD' ssh -o StrictHostKeyChecking=no root@VPS_IP "
systemctl disable --now apt-daily.timer 2>/dev/null || true
systemctl disable --now apt-daily-upgrade.timer 2>/dev/null || true
systemctl disable --now unattended-upgrades.service 2>/dev/null || true
echo AUTO_UPDATE_DISABLED
"
```

### 1.4 安装 v2ray-agent（自动化菜单交互）

这一步用 `expect` 脚本驱动安装菜单，选择 **VLESS Reality 无域名版**。

先把 expect 脚本上传到服务器：

```bash
sshpass -p 'VPS_PASSWORD' ssh -o StrictHostKeyChecking=no root@VPS_IP "cat > /tmp/install_vasma.exp << 'EXPECT_EOF'
#!/usr/bin/expect -f
set timeout 300

# 下载并运行安装脚本
spawn bash -c {wget -P /root -N --no-check-certificate "https://raw.githubusercontent.com/mack-a/v2ray-agent/master/install.sh" && chmod 700 /root/install.sh && /root/install.sh}

# 处理 sudoers 提示 → 选 N
expect {
    -re {sudoers.*\[Y/n\]} { send \"N\r\"; exp_continue }
    -re {sudoers.*\[y/N\]} { send \"N\r\"; exp_continue }
    -re {请选择|Please select} { }
    timeout { puts \"TIMEOUT_WAITING_FOR_MENU\"; exit 1 }
}

# 主菜单：选 1（安装）
send \"1\r\"

# 选择内核：从菜单文本里找含 Xray 的行号，动态发送
expect {
    -re {(\d+)[^\n]*[Xx]ray} {
        set choice \$expect_out(1,string)
        send \"\$choice\r\"
    }
    timeout { puts \"TIMEOUT_KERNEL\"; exit 1 }
}

# 选择协议：从菜单文本里找含 Reality 无域名的行号，动态发送
# 等菜单完整显示后（出现"请选择"提示），再从缓冲区提取行号
expect {
    -re {请选择|Please select|input} { }
    timeout { puts \"TIMEOUT_PROTOCOL_PROMPT\"; exit 1 }
}
# 回溯缓冲区找 Reality 无域名的编号
set buf \$expect_out(buffer)
if {[regexp {(\d+)[^\n]*Reality[^\n]*无域名} \$buf m choice]} {
    send \"\$choice\r\"
} elseif {[regexp {(\d+)[^\n]*无域名[^\n]*Reality} \$buf m choice]} {
    send \"\$choice\r\"
} else {
    puts \"MENU_PARSE_FAILED: \$buf\"
    exit 1
}

# 后续提示一路回车，直到安装完成
set done 0
while {\$done == 0} {
    expect {
        -re {\[Y/n\]|\[y/N\]|回车|press Enter|continue} { send \"\r\" }
        -re {安装完成|install.*complete|vasma} { set done 1 }
        timeout { puts \"TIMEOUT_INSTALL\"; exit 1 }
    }
}
puts \"INSTALL_COMPLETE\"
EXPECT_EOF
chmod +x /tmp/install_vasma.exp
echo EXPECT_SCRIPT_READY
"
```

> ⚠️ **注意：** 如果输出 `MENU_PARSE_FAILED` 或其他错误，跳转到 **[手动安装备用方案](#手动安装备用方案)**。`MENU_PARSE_FAILED` 后面会打印菜单内容，分析后可以判断是哪里出了问题。

运行 expect 脚本：

```bash
sshpass -p 'VPS_PASSWORD' ssh -o StrictHostKeyChecking=no root@VPS_IP "expect /tmp/install_vasma.exp 2>&1"
```

等待输出 `INSTALL_COMPLETE`（最多 5 分钟）。如果失败，走备用方案。

### 1.5 获取 clashMeta 订阅链接

安装完成后，用 expect 驱动 `vasma` 菜单（7 → 2 → 回车）：

```bash
sshpass -p 'VPS_PASSWORD' ssh -o StrictHostKeyChecking=no root@VPS_IP "
cat > /tmp/get_sub.exp << 'EOF'
#!/usr/bin/expect -f
set timeout 60
spawn vasma
expect -re {请选择|Please select|>} { send \"7\r\" }
expect -re {请选择|Please select|>} { send \"2\r\" }
# 一路回车直到看到订阅链接
set done 0
while {\$done == 0} {
    expect {
        -re {(http://\S+clashMeta\S+)} {
            puts \"SUB_URL:\$expect_out(1,string)\"
            set done 1
        }
        -re {回车|Enter|请按} { send \"\r\" }
        -re {请选择|>} { send \"\r\" }
        timeout { puts \"TIMEOUT_SUB\"; exit 1 }
    }
}
expect eof
EOF
chmod +x /tmp/get_sub.exp
expect /tmp/get_sub.exp 2>&1
"
```

从输出中提取 `SUB_URL:http://...` 那一行，这就是 **clashMeta 订阅链接**。

格式通常为：
```
http://VPS_IP:端口/s/clashMetaProfiles/xxxxxxxxxxxxxxxx
```

**把这个链接展示给用户，让他们复制保存。**

### 1.6 验证 VPN 服务状态

```bash
sshpass -p 'VPS_PASSWORD' ssh -o StrictHostKeyChecking=no root@VPS_IP "
systemctl is-active xray 2>/dev/null || systemctl is-active v2ray 2>/dev/null
"
```

输出 `active` 说明服务正在运行。

---

## 手动安装备用方案

如果 expect 脚本失败（菜单版本变化），切换为"Claude 提示 + 用户手动执行"模式：

1. 告诉用户在自己的终端执行：
   ```bash
   ssh root@VPS_IP -p 22
   ```
2. 登录后执行：
   ```bash
   wget -P /root -N --no-check-certificate "https://raw.githubusercontent.com/mack-a/v2ray-agent/master/install.sh" && chmod 700 /root/install.sh && /root/install.sh
   ```
3. 在菜单中选择：**安装 → Xray 内核 → VLESS Reality（无域名版）**，之后一路回车
4. 安装完后执行 `vasma`，选 `7 → 2 → 回车`，把显示的 `http://...` 订阅链接发给你

获取订阅链接后，继续阶段二。

---

## 阶段二：Clash Verge 配置指引

这一阶段是图形界面操作，你无法自动化，**生成清单给用户执行**。

告诉用户：

---

### Clash Verge 配置清单

**第一步：安装 Clash Verge Rev**（如已安装跳过）
- 下载地址：https://github.com/clash-verge-rev/clash-verge-rev/releases
- Mac 下载 `.dmg`，Windows 下载 `.exe`

**第二步：导入订阅链接**
1. 打开 Clash Verge Rev
2. 点击左侧「**订阅**」
3. 粘贴订阅链接（刚才复制的 `http://VPS_IP:...` 那条）
4. 点击「**导入**」，等待导入成功

**第三步：网络设置**（进入「设置」）

| 项目 | 操作 | 目的 |
|------|------|------|
| 虚拟网卡模式（TUN） | **开启** | 接管所有流量，包括 CLI 工具 |
| 系统代理 | **关闭** | 避免和 TUN 重叠，减少故障 |
| 代理模式 | 选「**规则**」 | 国内直连，海外走代理 |
| IPv6 | **关闭** | 减少 IPv6 泄漏路径 |

> 开启 TUN 时会提示输入系统密码，正常授权即可。

**第四步：DNS 防泄漏（推荐）**

在浏览器中操作：
- Chrome 地址栏输入 `chrome://flags/#enable-quic` → 设置为 **Disabled**（防止 UDP 绕过代理）
- 安装扩展 **WebRTC Control** 并开启（防止 WebRTC 泄露真实 IP）

**完成后告诉我，我帮你验证。**

---

## 阶段三：自动化验证

用户告知配置完成后，执行以下验证。

### 3.1 基础 IP 验证

在用户本机运行（通过 Bash 工具）：

```bash
curl -s https://api.ipify.org
```

如果返回的 IP 是 VPS 的 IP，代理生效。

### 3.2 VPS 服务健康检查

```bash
sshpass -p 'VPS_PASSWORD' ssh -o StrictHostKeyChecking=no root@VPS_IP "
echo '=== Xray 服务状态 ==='
systemctl is-active xray 2>/dev/null || systemctl is-active v2ray 2>/dev/null
echo '=== 监听端口 ==='
ss -tlnp | grep -E 'xray|v2ray' | head -5
"
```

### 3.3 DNS 泄漏快速检查（CLI 侧）

```bash
# 检查 Claude Code 是否走 Clash 代理端口
lsof -nP -i | grep -E 'claude|7897|7890' | head -10
```

如果能看到本地到 `127.0.0.1:7897`（或 7890）的连接，说明 CLI 流量已经过 Clash。

### 3.4 输出验证摘要

根据以上检查结果，向用户报告：

```
✅ VPS 服务运行正常
✅ 出口 IP：xxx.xxx.xxx.xxx（你的 VPS IP）
✅/⚠️ CLI 流量：已/未检测到 Clash 代理连接
```

如果有异常，根据症状提供排查建议（参考下方排查指南）。

---

## 常见问题排查

### SSH 无法连接
- 确认 IP 和密码正确
- 尝试端口：`-p 22`（默认）或 `-p 2222`
- DMIT PEM 文件权限：`chmod 600 ~/.ssh/id_rsa.pem`

### 服务器过几天断连
- 已在 1.3 步关闭自动更新，如果跳过了，重新执行那段命令

### Clash 订阅导入失败
- 确认订阅链接完整（从 `http://` 到最后一个字符）
- 检查 VPS 防火墙是否放行了对应端口（通常 v2ray-agent 会自动配置）

### DNS 仍然泄漏
排查顺序：
1. 确认 Clash Verge 的 DNS 覆写已开启
2. 检查当前生效配置文件（Settings → 查看内核启动参数 `-f` 后的路径）
3. 在终端运行：`ps aux | grep -E 'mihomo|clash-verge'`，确认 `-f` 指向哪个文件
4. 不要直接改 `config.yaml`，要改当前真正加载的那个文件

---

## 执行约定

1. **每步执行前**说明在做什么（一句话）
2. **命令执行后**解读关键输出，确认是否符合预期
3. **遇到错误**：先读错误信息，针对性处理，不盲目重试
4. **阶段一全部自动执行完**再给用户 Clash 配置清单
5. 密码等敏感信息在输出中用 `***` 替代，不要打印明文
