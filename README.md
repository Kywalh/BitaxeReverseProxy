# Bitaxe Secure Reverse Proxy (NGINX)

Secure HTTPS reverse proxy configuration for Bitaxe miners using NGINX on a Raspberry Pi or Linux server.

This setup allows:

- Secure remote HTTPS access to Bitaxe WebUI
- Read-only REST API exposure over WAN
- Blocking dangerous HTTP methods from the Internet
- Allowing only a specific restart endpoint
- TLS encryption with Let's Encrypt
- Protection against unauthorized remote configuration changes

---

# Hardware Used

This setup was tested using:

| Hardware | Role |
|---|---|
| Raspberry Pi 3 / 4 / 5 | Reverse Proxy Server |
| Bitaxe Ultra / Gamma / Supra | Bitcoin ASIC Miner |
| Freebox Router | Port Forwarding |
| Optional SSD or High-Endurance SD Card | Better reliability |

---

# Recommended Raspberry Pi Setup

## Minimum Recommended

| Component | Recommendation |
|---|---|
| CPU | Raspberry Pi 3 minimum |
| RAM | 1 GB minimum |
| Storage | 16 GB SD Card |
| OS | Raspberry Pi OS Lite |
| Network | Ethernet preferred |

---

## Recommended for Multiple Bitaxes

| Component | Recommendation |
|---|---|
| CPU | Raspberry Pi 4 or 5 |
| RAM | 2 GB+ |
| Storage | SSD preferred |
| Cooling | Passive or active cooling |

---

# Architecture

```text
Internet
    │
    ▼
Router Port Forwarding
    │
    ▼
Raspberry Pi / Linux Reverse Proxy
    │
    ├── HTTPS :10001 → Bitaxe #1
    ├── HTTPS :10002 → Bitaxe #2
    └── HTTPS :10003 → Bitaxe #3
```

Example public URLs:

```text
https://your-domain.example.com:10001
https://your-domain.example.com:10002
```

---

# Why This Configuration Is Important

Bitaxe devices expose a REST API without:

- HTTPS
- Authentication sessions
- CSRF protection
- Advanced access control

If directly exposed to the Internet, an attacker knowing the API endpoints could potentially:

- Change pool configuration
- Change wallet address
- Overclock the miner
- Upload firmware
- Modify system settings
- Reboot the device

This reverse proxy mitigates these risks by:

- Allowing only GET and HEAD
- Allowing only one specific POST endpoint:
  - /api/system/restart

Everything else is blocked.

---

# Features

## Allowed

| Method | Endpoint | Status |
|---|---|---|
| GET | / | ✅ |
| GET | /api/system/info | ✅ |
| GET | /api/system/status | ✅ |
| HEAD | All endpoints | ✅ |
| POST | /api/system/restart | ✅ |

---

## Blocked

| Method | Endpoint | Status |
|---|---|---|
| POST | /api/system | ❌ |
| PATCH | /api/system | ❌ |
| PUT | Any | ❌ |
| DELETE | Any | ❌ |
| POST | OTA/Firmware endpoints | ❌ |
| POST | Pool configuration endpoints | ❌ |

---

# Requirements

- Raspberry Pi or Linux server
- NGINX installed
- Public domain name
- Port forwarding configured
- Let's Encrypt certificate

---

# Installation

## Install NGINX

```bash
sudo apt update
sudo apt install nginx -y
```

---

## Install Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
```

---

## Generate SSL Certificate

Replace:

```text
your-domain.example.com
```

with your own domain name.

Example:

```bash
sudo certbot certonly --nginx -d your-domain.example.com
```

Certificates will be stored in:

```text
/etc/letsencrypt/live/your-domain.example.com/
```

---

# NGINX Configuration

File:

```text
/etc/nginx/sites-available/bitaxe
```

---

# Complete Configuration Example

Replace:

- your-domain.example.com
- 192.168.1.201

with your own values.

```nginx
server {

    listen 10001 ssl;
    listen [::]:10001 ssl;

    server_name your-domain.example.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;

    location ~ ^/api/system/restart/?$ {

        limit_except POST {
            deny all;
        }

        proxy_pass http://192.168.1.201/;

        proxy_http_version 1.1;

        proxy_set_header Host 192.168.1.201;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 90;
    }

    location / {

        limit_except GET HEAD {
            deny all;
        }

        proxy_pass http://192.168.1.201/;

        proxy_http_version 1.1;

        proxy_set_header Host 192.168.1.201;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_read_timeout 90;
    }
}
```

---

# Enable Site

```bash
sudo ln -s /etc/nginx/sites-available/bitaxe /etc/nginx/sites-enabled/
```

---

# Validate Configuration

```bash
sudo nginx -t
```

---

# Reload NGINX

```bash
sudo systemctl reload nginx
```

---

# Router Port Forwarding

| Public Port | Internal IP | Internal Port |
|---|---|---|
| 10001 | RaspberryPiIP | 10001 |
| 10002 | RaspberryPiIP | 10002 |

---

# Testing

## Test HTTPS Access

```bash
curl -k https://your-domain.example.com:10001/api/system/info
```

---

## Test Allowed Restart

```bash
curl -k -X POST https://your-domain.example.com:10001/api/system/restart
```

---

## Test Forbidden POST

```bash
curl -k -X POST https://your-domain.example.com:10001/api/system
```

Expected:

```text
403 Forbidden
```

---

# Security Notes

## This setup protects against:

- Unauthorized configuration changes
- Remote overclocking
- Unauthorized firmware updates
- Wallet hijacking
- Pool hijacking
- Generic REST API abuse

---

## This setup DOES NOT protect against:

- Vulnerabilities inside AxeOS itself
- Compromised LAN devices
- Exposed SSH services
- Physical access attacks

---

# Recommended Additional Security

- Strong router admin password
- Disable UPnP
- Keep Raspberry Pi updated
- Use non-standard public ports
- Restrict access by IP if possible

---

# Useful Commands

## View NGINX Logs

```bash
sudo tail -f /var/log/nginx/access.log
```

```bash
sudo tail -f /var/log/nginx/error.log
```

---

## Restart NGINX

```bash
sudo systemctl restart nginx
```

---

## Check Listening Ports

```bash
sudo ss -tulpn
```

---

# Disclaimer

This configuration is provided as-is without warranty.

The user remains fully responsible for:

- Internet exposure
- Network security
- Reverse proxy configuration
- Bitaxe configuration
- Firmware integrity
- Firewall and router settings

Exposing mining hardware directly to the Internet always carries inherent risks.
