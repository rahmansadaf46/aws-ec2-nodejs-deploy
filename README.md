
# 🚀 Node.js Deployment Template (AWS EC2)

This repository documents a **production-ready deployment pattern** for running a Node.js application on an **AWS EC2 Ubuntu server** using:

- **systemd** (process manager)
- **Nginx** (reverse proxy)
- **Environment-based configuration**
- **Optional HTTPS (SSL)**
- **Safe restarts & crash recovery**

---

## 🧱 Architecture Overview

```

Internet
|
|  http(s)://SERVER_IP or DOMAIN:PUBLIC_PORT
v
Nginx (public-facing)
|
|  proxy_pass
v
Node.js App (localhost only)
|
v
Database / External APIs

```

---

## 📁 Directory Structure (Flexible)

This template supports **common Node.js layouts**.

### Option A — Build Output Inside `/dist/server`
```

/var/www/
      └── your-app/
              ├── dist/
              │     └── server/
              │           └── index.js
              ├── package.json
              ├── node_modules/
              └── .env.production

````

**ExecStart**
```ini
ExecStart=/usr/bin/node dist/server/index.js
````

---

### Option B — Build Output Inside `/dist`
```

/var/www/
      └── your-app/
              ├── dist/
              │     └── index.js
              ├── package.json
              ├── node_modules/
              └── .env.production

````

**ExecStart**
```ini
ExecStart=/usr/bin/node dist/index.js
````

### Option C — `index.js` in Project Root

```
/var/www/
      └── your-app/
              ├── dist/
              ├── index.js
              ├── package.json
              ├── node_modules/
              └── .env.production
```

**ExecStart**

```ini
ExecStart=/usr/bin/node index.js
```

👉 Choose **only one** pattern.
The rest of the setup stays **exactly the same**.

---

## 🔐 Environment Variables

Create the production env file:

```bash
sudo nano /var/www/your-app/.env.production
```

Example:

```env
NODE_ENV=production

# Node internal server
PORT=3005
HOST=127.0.0.1

# Public access (Nginx)
ALLOWED_ORIGINS=http://SERVER_IP:PUBLIC_PORT

LOG_LEVEL=INFO

DB_HOST=...
DB_USER=...
DB_PASSWORD=...
DB_NAME=...
```

### ⚠️ Critical Rules

* `HOST` **must always be `127.0.0.1`**
* Node **must never bind to public IP**
* Public access is handled by **Nginx only**

---

## ⚙️ systemd Service (Node Process Manager)

### Create Service File

```bash
sudo nano /etc/systemd/system/your-app.service
```

### Service Definition

```ini
[Unit]
Description=Node.js Application
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/var/www/your-app

EnvironmentFile=/var/www/your-app/.env.production

ExecStart=/usr/bin/node dist/server/index.js
# OR
# ExecStart=/usr/bin/node dist/index.js
# OR
# ExecStart=/usr/bin/node index.js

Restart=always
RestartSec=5

StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### Enable & Start

```bash
sudo systemctl daemon-reload
sudo systemctl enable your-app
sudo systemctl start your-app
```

### Useful Commands

```bash
systemctl status your-app
sudo systemctl restart your-app
journalctl -u your-app -f
```

---

## 🌐 Nginx Reverse Proxy (HTTP)

### Create Nginx Site

```bash
sudo nano /etc/nginx/sites-available/your-app
```

### HTTP Configuration

```nginx
server {
    listen PUBLIC_PORT;
    server_name SERVER_IP;

    client_max_body_size 20M;

    location / {
        proxy_pass http://127.0.0.1:APP_PORT;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto http;
    }
}
```

Enable it:

```bash
sudo ln -s /etc/nginx/sites-available/your-app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🔐 HTTPS / SSL Setup (Optional but Recommended)

### Requirements

* A **domain name** (SSL does not work reliably with raw IPs)
* DNS pointing to your EC2 public IP

### Install Certbot

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

### Issue SSL Certificate

```bash
sudo certbot --nginx -d yourdomain.com
```

Certbot will:

* Create HTTPS server block
* Redirect HTTP → HTTPS (optional)
* Auto-renew certificates

---

### Final HTTPS Nginx Example

```nginx
server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    client_max_body_size 20M;

    location / {
        proxy_pass http://127.0.0.1:APP_PORT;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

👉 Update `.env.production`:

```env
ALLOWED_ORIGINS=https://yourdomain.com
```

---

## 🔎 Verification Checklist

```bash
# Node app (local only)
ss -tulnp | grep APP_PORT

# Nginx public port
ss -tulnp | grep PUBLIC_PORT

# Test
curl http://SERVER_IP:PUBLIC_PORT
curl https://yourdomain.com
```

---

## 🚫 Common Mistakes (Avoid These)

### ❌ Binding Node to Public IP

```env
HOST=SERVER_IP ❌
```

✅ Correct:

```env
HOST=127.0.0.1
```

---

### ❌ Exposing Node Port in AWS Security Group

Only allow:

* `80`, `443`, or chosen `PUBLIC_PORT`

Never allow:

* `APP_PORT`

---

### ❌ Forcing HTTPS Without SSL

Do not redirect to HTTPS unless Certbot is installed and active.

---

## 🔁 Recommended Deployment Flow

1. Build locally
2. Upload `dist/` (or root files)
3. Install prod dependencies
4. Restart systemd service

Example:

```bash
rsync dist package.json bun.lock server:/var/www/your-app
ssh server "cd /var/www/your-app && rm -rf node_modules && npm install --omit=dev && sudo systemctl restart your-app"
```

---

## ✅ Result

* Stable Node process (systemd)
* Auto-restart on crash
* Clean logs via `journalctl`
* Nginx handles public traffic
* SSL-ready
* Multiple apps on one EC2 supported

---

## 📌 Reuse This Template

Works for:

* Express / Fastify / NestJS
* Monorepo or single-file apps
* Dev / Stage / Production

Just change:

* App name
* Ports
* Domain
* Paths

---



```
Happy deploying 🚀
```
