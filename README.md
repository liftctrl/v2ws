# 🚀 Secure V2Ray Deployment Guide

VLESS + WebSocket + TLS + CDN + Nginx (Cloudflare + WARP Enhanced)

---

## 🧰 1. Prerequisites

| Requirement | Description                                                    |
| ----------- | -------------------------------------------------------------- |
| VPS         | Offshore VPS (Debian 11+ recommended)                          |
| Domain      | Managed via Cloudflare, pointed to VPS IP                      |
| Cloudflare  | CDN proxy enabled (orange cloud), SSL set to **Full (strict)** |
| Open Ports  | TCP ports **80** and **443**                                   |

---

## 🔐 2. Generate a Cloudflare API Token

1. Log in to Cloudflare → Profile → **API Tokens**
2. Click "**Create Token**" and select "**Edit zone DNS**"
3. Set the token's scope to your domain
3. Save the token for use with `acme.sh`

---

## ⚙️ 3. Install Required Packages

```bash
apt update && apt upgrade -y
apt install -y nginx curl socat uuid-runtime unzip fail2ban
```

---

## 📜 4. Install acme.sh & Issue TLS Certificate

```bash
curl https://get.acme.sh | sh -s email=youremail@example.com
source ~/.bashrc

export CF_Token="your_cloudflare_api_token"

# Issue ECC certificate
acme.sh --issue --dns dns_cf -d yourdomain.com --keylength ec-256

# Install and configure auto-renew
mkdir -p /etc/nginx/ssl
acme.sh --install-cert -d yourdomain.com \
--key-file /etc/nginx/ssl/yourdomain.key \
--fullchain-file /etc/nginx/ssl/yourdomain.crt \
--reloadcmd "systemctl reload nginx"
```

---

## 📦 5. Install V2Ray (Official Script)

```bash
bash <(curl -Ls https://github.com/v2fly/fhs-install-v2ray/raw/master/install-release.sh)
```

---

## ⚡ 6. Configure V2Ray

Generate a UUID:

```bash
uuidgen
```

Edit `/usr/local/etc/v2ray/config.json`:

```json
{
  "inbounds": [{
    "port": 10000,
    "listen": "127.0.0.1",
    "protocol": "vless",
    "settings": {
      "clients": [{
        "id": "your-uuid-here",
        "flow": ""
      }],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "ws",
      "security": "none",
      "wsSettings": {
        "path": "/api/status"
      }
    }
  }],
  "outbounds": [{
    "protocol": "freedom",
    "settings": {}
  }]
}
```

---

## 🌐 7. Configure Nginx (with Rate Limiting & Obfuscation)

Create a dummy site:

```bash
mkdir -p /var/www/html
echo "<h1>Welcome</h1>" > /var/www/html/index.html
```

Edit `/etc/nginx/sites-available/default`:

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

---

## 🟢 8. Start Services

```bash
systemctl restart v2ray
systemctl restart nginx
systemctl enable v2ray
systemctl enable nginx
```

---

## 🛡️ 9. Harden Security with Fail2Ban

```bash
systemctl enable fail2ban
systemctl start fail2ban
```

> Protects against port scanning and brute-force attacks.

---

## ☁️ 10. Cloudflare DNS Settings

- **A record**: Points to VPS IP
- **Proxy status**: Enabled (orange cloud)
- **SSL/TLS**: Full (strict)
- Optional:
  - Enable Bot Fight Mode
  - Enable Rate Limiting, WAF rules

---

## 📲 11. V2RayN / V2RayNG Client Settings

| Field      | Value                 |
| ---------- | --------------------- |
| Address    | yourdomain.com        |
| Port       | 443                   |
| UUID       | (your generated UUID) |
| Encryption | none (VLESS)          |
| Transport  | WebSocket             |
| Path       | `/api/status`         |
| TLS        | Enabled               |
| Host (SNI) | yourdomain.com        |

---

## 🔗 12. Generate VLESS Link

```bash
vless://your-uuid@yourdomain.com:443?encryption=none&security=tls&type=ws&host=yourdomain.com&path=%2Fapi%2Fstatus#VLESS-CDN
```

---

## 🌍 13. Optional: Install WARP (for DNS Poisoning Protection)

Install via Script:

```bash
bash <(curl -fsSL https://git.io/warp.sh)
```

- Add WARP IPv4/IPv6
- Enable full outbound via WARP
- Great for solving acme.sh DNS issues

Verify:

```bash
curl https://www.cloudflare.com/cdn-cgi/trace | grep warp
# Should return: warp=on
```

---

## ✅ Summary: Why This Setup is Hardened

| Feature             | Benefit                         |
| ------------------- | ------------------------------- |
| VLESS + WS + TLS    | Obfuscated + Encrypted traffic  |
| Nginx Reverse Proxy | Front-facing domain camouflage  |
| Cloudflare CDN      | Hides real server IP            |
| Real Website Cover  | Fake landing page via Nginx     |
| Fail2Ban            | Brute-force and scanner defense |
| Rate Limiting       | Basic anti-bot protection       |
| Optional WARP       | Avoids DNS pollution/censorship |


---
