---
title: 'Overplus Getting Started: Set Up Your Own Proxy in 5 Minutes'
date: 2026-07-18T12:00:00+08:00
draft: false
categories: ["Tutorial"]
tags: ["proxy", "tutorial", "C++", "networking"]
summary: "Step-by-step guide to set up your own proxy server with Overplus. One-click install script + Windows GUI client, up and running in 5 minutes."
---

## What is Overplus

[Overplus](https://github.com/xyanrch1024/overplus) is a lightweight, high-performance proxy server written in C++17, supporting SOCKS5, HTTPS, and Trojan protocols.

Highlights:

- **Blazing fast**: 146 concurrent connections use only 25MB memory, CPU < 1%
- **Zero dependencies**: Statically compiled, no runtime libraries needed
- **Windows GUI client**: Works out of the box, one-click connect
- **One-click install**: Single command to set up the server

## Prerequisites

You need:

1. A VPS outside your region (Ubuntu/Debian recommended, 2 cores / 2GB RAM is enough)
2. A domain name (optional, for certificate issuance)
3. SSH access to your VPS

## Step 1: Server Installation

### Option A: One-Click Install (Recommended)

SSH into your VPS and run:

```bash
curl -O https://raw.githubusercontent.com/xyanrch1024/overplus/master/install.sh && chmod +x install.sh && sudo ./install.sh
```

The script will guide you through:

1. **Set a password**: Your proxy connection password
2. **Choose a port**: Default is 443 (recommended — disguises traffic as normal HTTPS)

After installation, the script automatically:
- Generates a TLS certificate (valid for 10 years)
- Downloads the Overplus binary
- Configures and starts a systemd service

Verify the service is running:

```bash
systemctl status overplus
```

If you see `active (running)`, you're all set.

### Option B: Manual Install

If you prefer manual setup:

```bash
# 1. Download
wget https://github.com/xyanrch1024/overplus/releases/latest/download/overplus-linux-x86_64.zip
unzip overplus-linux-x86_64.zip -d /tmp/overplus

# 2. Install binary
cp /tmp/overplus/overplus /usr/bin/overplus
chmod +x /usr/bin/overplus

# 3. Generate TLS certificate (self-signed example)
mkdir -p /etc/overplus
openssl req -x509 -nodes -newkey ec:<(openssl ecparam -name prime256v1) \
  -keyout /etc/overplus/server.key \
  -out /etc/overplus/server.crt \
  -subj "/CN=your_server_ip" -days 3650

# 4. Create config file
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

# 5. Create systemd service
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

# 6. Start service
systemctl daemon-reload
systemctl start overplus
systemctl enable overplus
```

### Enable BBR (Highly Recommended)

BBR significantly improves TCP throughput:

```bash
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
```

Verify it's active:

```bash
sysctl net.ipv4.tcp_congestion_control
# Output: net.ipv4.tcp_congestion_control = bbr
```

## Step 2: Windows Client

### Download

Go to the [Release page](https://github.com/xyanrch1024/overplus/releases/latest) and download `overplus-client-windows-x64.zip`. Extract to any directory.

### Setup

After extraction, you'll see:

```
overplus_client.exe    # Main program
Qt5Core.dll           # Qt runtime
Qt5Gui.dll
Qt5Widgets.dll
libcrypto-3-x64.dll   # OpenSSL
libssl-3-x64.dll
plugins/              # Qt plugins
```

Double-click `overplus_client.exe` to launch the GUI client:

1. Enter your **VPS IP address** in the Host Name field
2. Enter the **port** (default 443)
3. Enter your **password**
4. Click **SAVE** to store settings
5. Click **CONNECT** to connect

When the status bar shows **CONNECTED** and the tray icon turns green, you're connected.

### System Proxy

Once connected, Overplus automatically sets the Windows system SOCKS proxy. It restores the original settings when you close the program. You can also configure manually:

- Open **Windows Settings** → **Network & Internet** → **Proxy**
- Manual proxy setup: `127.0.0.1`, port `1080`

### Alternative Clients

Overplus is compatible with the Trojan protocol. You can use any Trojan-compatible client:

| Client | Platform | Download |
|--------|----------|----------|
| Clash | Windows/Mac/Linux | [GitHub](https://github.com/clash-clash-clash/clash) |
| Nekoray | Windows/Mac/Linux | [GitHub](https://github.com/MatsuriDayo/nekoray) |
| Shadowrocket | iOS | App Store |
| Quantumult X | iOS | App Store |

Example configuration (Clash):

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

## Step 3: Verify

### Check Server Logs

```bash
# View Overplus logs
journalctl -u overplus -f

# Or check log files directly
cat /var/log/overplus.*.log
```

Normal connection output looks like:

```
accept incoming connection :1.2.3.4
connected to example.com:443
session destroyed
```

### Test Your Connection

Visit [https://ipinfo.io](https://ipinfo.io) to confirm your IP has changed to your VPS IP.

## Troubleshooting

### Can't Connect?

1. **Check firewall**: Make sure port 443 is open on your VPS
   ```bash
   # Ubuntu/Debian
   ufw allow 443/tcp
   # Or disable firewall
   ufw disable
   ```

2. **Check service status**:
   ```bash
   systemctl status overplus
   journalctl -u overplus -n 20
   ```

3. **Check port conflict**: Port 443 must not be used by other services (nginx, apache, etc.)

### Slow Speed?

1. Make sure BBR is enabled
2. Try a different VPS datacenter/region
3. Check your VPS bandwidth and traffic quota

### How to Change Password?

Edit the server config:

```bash
nano /etc/overplus/server.json
# Modify the allowed_passwords field
systemctl restart overplus
```

## Benchmarks

Tested on 2-core Xeon Skylake / 2GB RAM / Ubuntu 24.04:

| Metric | Value |
|--------|-------|
| Concurrent connections | 146 |
| Memory usage | 25 MB |
| Memory per connection | ~170 KB |
| CPU usage | < 1% |
| TLS reconnect | < 1ms |

## Links

- **GitHub**: https://github.com/xyanrch1024/overplus
- **Releases**: https://github.com/xyanrch1024/overplus/releases
- **Telegram**: https://t.me/+JfKOqh2wH25kMWFl
