## 🚀 Secure V2Ray Deployment Guide

VLESS + WebSocket + TLS + Cloudflare CDN + Nginx (WARP Enhanced)

### 🧰 1. Prerequisites

| Requirement | Description                                                   |
| ----------- | ------------------------------------------------------------- |
| VPS         | Offshore VPS (Debian 11+/Ubuntu 20.04+ recommended)           |
| Domain      | Managed via Cloudflare, pointed to VPS IP                     |
| Cloudflare  | CDN proxy enabled (orange cloud), SSL mode: **Full (Strict)** |
| Open Ports  | **TCP 80, 443**                                               |

### 🔐 2. Generate a Cloudflare API Token

1. Log in to Cloudflare → **Profile** → **API Tokens**
2. Click **Create Token** → **Edit zone DNS**
3. Scope: Your domain → Save token

### ⚙️ 3. Install Required Packages

```bash
apt update && apt upgrade -y
apt install -y nginx curl socat uuid-runtime unzip fail2ban
```

### 📜 4. Install acme.sh & Issue TLS Certificate

```bash
curl https://get.acme.sh | sh -s email=you@example.com
source ~/.bashrc

export CF_Token="your_cloudflare_api_token"

# Issue ECC certificate
acme.sh --issue --dns dns_cf -d yourdomain.com --keylength ec-256

# Install & auto-renew
mkdir -p /etc/nginx/ssl
acme.sh --install-cert -d yourdomain.com \
--key-file /etc/nginx/ssl/yourdomain.key \
--fullchain-file /etc/nginx/ssl/yourdomain.crt \
--reloadcmd "systemctl reload nginx"
```

### 📦 5. Install V2Ray

```bash
bash <(curl -Ls https://github.com/v2fly/fhs-install-v2ray/raw/master/install-release.sh)
```

### ⚡ 6. Configure V2Ray (`/usr/local/etc/v2ray/config.json`)

```json
{
  "log": {
    "loglevel": "warning"
  },
  "dns": {
    "servers": ["1.1.1.1", "8.8.8.8", "8.8.4.4"]
  },
  "inbounds": [{
    "port": 10000,
    "listen": "127.0.0.1",
    "protocol": "vless",
    "settings": {
      "clients": [{
        "id": "your_uuid",
        "flow": ""
      }],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "ws",
      "security": "none",
      "wsSettings": {
        "path": "/raypath"
      }
    }
  }],
  "outbounds": [
    {
      "tag": "warp-out",
      "protocol": "freedom",
      "settings": {},
      "streamSettings": {
        "sockopt": {
          "interface": "wgcf"
        }
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field",
        "ip": ["geoip:cn", "geoip:private"],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "domain": ["geosite:cn"],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "network": "tcp,udp",
        "outboundTag": "warp-out"
      }
    ]
  }
}
```

### 🌐 7. Configure Nginx (with Reverse Proxy & Rate Limiting)

```nginx
server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /etc/nginx/ssl/yourdomain.crt;
    ssl_certificate_key /etc/nginx/ssl/yourdomain.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location /api/status {
        proxy_pass http://127.0.0.1:10000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        limit_req zone=req_limit burst=10 nodelay;
    }

    location / {
        root /var/www/html;
        index index.html;
    }

    limit_req_zone $binary_remote_addr zone=req_limit:10m rate=1r/s;
}

server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$host$request_uri;
}
```

### 🌍 8. WARP WireGuard Config (`/etc/wireguard/wgcf.conf`)

```ini
[Interface]
PrivateKey = 6Cld6l3EnFQFiiKLEBzhcLb88oKjD9D3tH9wZol7ZFE=
Address = 172.16.0.2/32, 2606:4700:110:8f8b:dde9:e82a:c46b:34ed/128
DNS = 1.1.1.1,8.8.8.8,8.8.4.4
MTU = 1420
PostUp = ip -4 rule add from 107.172.82.25 lookup main prio 18
PostDown = ip -4 rule delete from 107.172.82.25 lookup main prio 18

[Peer]
PublicKey = bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = engage.cloudflareclient.com:2408
```

### 🟢 9. Start Services

```bash
systemctl restart v2ray
systemctl enable v2ray
systemctl restart wg-quick@wgcf
systemctl enable wg-quick@wgcf
systemctl restart nginx
```

### 🛡️ 10. Security Hardening

- Fail2Ban
```bash
systemctl enable fail2ban
systemctl start fail2ban
```
- Cloudflare Bot Fight Mode
- WAF & Rate Limiting

### 📲 11. Client Settings (V2RayN/V2RayNG)

| Field      | Value                                |
| ---------- | ------------------------------------ |
| Address    | yourdomain.com                       |
| Port       | 443                                  |
| UUID       | your_uuid                            |
| Encryption | none (VLESS)                         |
| Transport  | WebSocket                            |
| Path       | /raypath                             |
| TLS        | Enabled                              |
| Host (SNI) | yourdomain.com                       |


### 🔗 12. VLESS Link

```perl
vless://your_uuid@yourdomain.com:443?encryption=none&security=tls&type=ws&host=yourdomain.com&path=%2Fapi%2Fstatus#VLESS-CDN
```

### ✅ Why This Setup Is Secure

- **VLESS + WS + TLS**: Encrypted, stealthy
- **Cloudflare CDN**: Real IP hidden
- **Nginx Reverse Proxy**: Legitimate cover page
- **Fail2Ban** + Rate Limiting: Bot & scanner defense
- **WARP**: Foreign traffic accelerated & DNS poisoning bypass
