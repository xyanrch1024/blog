---
title: 'Overplus 安装使用教程：5 分钟搭建你自己的代理服务'
date: 2026-07-18T12:00:00+08:00
draft: false
categories: ["教程"]
tags: ["代理", "教程", "C++", "科学上网"]
summary: "手把手教你用 Overplus 搭建自己的代理服务器。一键安装脚本 + Windows GUI 客户端，全程 5 分钟搞定。"
---

## 什么是 Overplus

[Overplus](https://github.com/xyanrch1024/overplus) 是一个用 C++17 编写的轻量高性能代理服务器，支持 SOCKS5、HTTPS、Trojan 协议。

亮点：

- **极致性能**：146 个并发连接仅占 25MB 内存，CPU < 1%
- **零依赖**：静态编译，无需安装任何运行库
- **Windows GUI 客户端**：开箱即用，一键连接
- **一键安装**：服务端一条命令搞定

## 前置准备

你需要：

1. 一台境外 VPS（推荐 Ubuntu/Debian，2 核 2G 即可）
2. 一个域名（可选，用于申请证书）
3. SSH 登录 VPS 的终端

## 第一步：服务端安装

### 方式一：一键安装（推荐）

SSH 登录 VPS 后，执行以下命令：

```bash
curl -O https://raw.githubusercontent.com/xyanrch1024/overplus/master/install.sh && chmod +x install.sh && sudo ./install.sh
```

脚本会引导你完成以下步骤：

1. **输入密码**：设置你的代理连接密码
2. **选择端口**：默认 443，建议保持默认（伪装成正常 HTTPS 流量）

安装完成后，脚本会自动：
- 生成 TLS 证书（有效期 10 年）
- 下载 Overplus 二进制文件
- 配置并启动 systemd 服务

验证服务是否正常运行：

```bash
systemctl status overplus
```

看到 `active (running)` 就说明成功了。

### 方式二：手动安装

如果你更喜欢手动操作：

```bash
# 1. 下载
wget https://github.com/xyanrch1024/overplus/releases/latest/download/overplus-linux-x86_64.zip
unzip overplus-linux-x86_64.zip -d /tmp/overplus

# 2. 安装二进制文件
cp /tmp/overplus/overplus /usr/bin/overplus
chmod +x /usr/bin/overplus

# 3. 生成 TLS 证书（需要已安装 certbot 或手动用 openssl）
# 这里以自签证书为例：
mkdir -p /etc/overplus
openssl req -x509 -nodes -newkey ec:<(openssl ecparam -name prime256v1) \
  -keyout /etc/overplus/server.key \
  -out /etc/overplus/server.crt \
  -subj "/CN=your_server_ip" -days 3650

# 4. 创建配置文件
cat > /etc/overplus/server.json << 'EOF'
{
    "run_type": "server",
    "local_addr": "0.0.0.0",
    "local_port": "443",
    "allowed_passwords": ["your_password_here"],
    "log_level": "NOTICE",
    "log_dir": "",
    "ssl": {
        "cert": "/etc/overplus/server.crt",
        "key": "/etc/overplus/server.key"
    },
    "websocketEnabled": false,
    "dns_cache_ttl": 600,
    "dns_cleanup_interval": 600
}
EOF

# 5. 创建 systemd 服务
cat > /etc/systemd/system/overplus.service << 'EOF'
[Unit]
Description=overplus proxy
After=network.target

[Service]
User=root
ExecStart=/usr/bin/overplus -c /etc/overplus/server.json
Restart=on-failure
RestartSec=10s
LimitNOFILE=infinity

[Install]
WantedBy=multi-user.target
EOF

# 6. 启动服务
systemctl daemon-reload
systemctl start overplus
systemctl enable overplus
```

### 开启 BBR 加速（强烈推荐）

BBR 能显著提升 TCP 传输性能：

```bash
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
```

验证是否生效：

```bash
sysctl net.ipv4.tcp_congestion_control
# 输出 net.ipv4.tcp_congestion_control = bbr
```

## 第二步：Windows 客户端

### 下载

前往 [Release 页面](https://github.com/xyanrch1024/overplus/releases/latest) 下载 `overplus-client-windows-x64.zip`，解压到任意目录。

### 配置

解压后会看到以下文件：

```
overplus_client.exe    # 主程序
Qt5Core.dll           # Qt 运行库
Qt5Gui.dll
Qt5Widgets.dll
libcrypto-3-x64.dll   # OpenSSL
libssl-3-x64.dll
plugins/              # Qt 插件
```

双击 `overplus_client.exe` 启动客户端，会看到一个简洁的 GUI 界面：

1. 在 **Host Name** 输入你的 VPS IP 地址
2. 在 **Host Port** 输入端口号（默认 443）
3. 在 **Password** 输入你设置的密码
4. 点击 **SAVE** 保存配置
5. 点击 **CONNECT** 连接

状态栏显示 **CONNECTED** 且托盘图标变绿，就说明连接成功了。

### 配置系统代理

连接成功后，Overplus 会自动设置 Windows 系统 SOCKS 代理。关闭程序时会自动恢复。你也可以手动配置：

- 打开 **Windows 设置** → **网络和 Internet** → **代理**
- 手动设置代理服务器：`127.0.0.1`，端口 `1080`

### 使用其他客户端

Overplus 兼容 Trojan 协议，你也可以使用其他支持 Trojan 的客户端：

| 客户端 | 平台 | 下载 |
|--------|------|------|
| Clash | Windows/Mac/Linux | [GitHub](https://github.com/clash-clash-clash/clash) |
| Nekoray | Windows/Mac/Linux | [GitHub](https://github.com/MatsuriDayo/nekoray) |
| Shadowrocket | iOS | App Store |
| Quantumult X | iOS | App Store |

配置示例（以 Clash 为例）：

```yaml
proxies:
  - name: "Overplus"
    type: trojan
    server: your_server_ip
    port: 443
    password: your_password
    sni: your_server_ip
    skip-cert-verify: true
```

## 第三步：验证连接

### 检查服务端日志

```bash
# 查看 Overplus 日志
journalctl -u overplus -f

# 或直接查看日志文件
cat /var/log/overplus.*.log
```

正常连接时会看到类似输出：

```
accept incoming connection :1.2.3.4
connected to example.com:443
session destroyed
```

### 测试代理

连接成功后，访问 [https://ipinfo.io](https://ipinfo.io) 确认 IP 已切换到 VPS 的 IP。

## 常见问题

### 连不上怎么办？

1. **检查防火墙**：确保 VPS 的 443 端口开放
   ```bash
   # Ubuntu/Debian
   ufw allow 443/tcp
   # 或关闭防火墙
   ufw disable
   ```

2. **检查服务状态**：
   ```bash
   systemctl status overplus
   journalctl -u overplus -n 20
   ```

3. **检查端口占用**：443 端口不能被其他服务占用（如 nginx、apache）

### 速度慢怎么办？

1. 确认已开启 BBR
2. 尝试更换 VPS 的机房/线路
3. 检查 VPS 的带宽和流量是否充足

### 如何更换密码？

编辑服务端配置文件：

```bash
nano /etc/overplus/server.json
# 修改 allowed_passwords 字段
systemctl restart overplus
```

## 性能数据

在 2 核 Xeon / 2GB RAM / Ubuntu 24.04 上的实测数据：

| 指标 | 数值 |
|------|------|
| 并发连接数 | 146 |
| 内存占用 | 25 MB |
| 单连接内存 | ~170 KB |
| CPU 占用 | < 1% |
| TLS 重连 | < 1ms |

## 相关链接

- **GitHub**: https://github.com/xyanrch1024/overplus
- **Release 下载**: https://github.com/xyanrch1024/overplus/releases
- **Telegram 群**: https://t.me/+JfKOqh2wH25kMWFl
