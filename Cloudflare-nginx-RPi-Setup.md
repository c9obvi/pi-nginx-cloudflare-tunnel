# 🧠 Cloudflare Tunnel + nginx Secure Access on Raspberry Pi  
### 🔒 Securely expose local dashboards or devices (like NerdQaxe++, Grafana, cameras, etc.) to the internet — with HTTPS, password protection, and **no port forwarding**.

---

## 🚀 Overview

This setup lets you access any local service (e.g., `http://192.168.x.x:80`) through a **public HTTPS domain** using:
- **Cloudflare Tunnel (cloudflared)** – encrypted outbound connection (no firewall holes)
- **nginx** – local reverse proxy with Basic Auth security
- **Cloudflare Edge** – free TLS certificates and global CDN routing

---

## ⚙️ Architecture

```
Browser (you, anywhere)
     ↓ HTTPS (TLS)
Cloudflare Edge (yourdomain.com)
     ↓ Cloudflare Tunnel
cloudflared (on your Raspberry Pi)
     ↓ localhost proxy
nginx (auth gate)
     ↓
Local device (e.g. NerdQaxe++ dashboard)
```

---

## 🧩 Prerequisites
- Raspberry Pi (any version)
- Local web service running (e.g. NerdQaxe++ on `192.168.5.106:80`)
- Domain on [Cloudflare.com](https://dash.cloudflare.com)
- Tunnel already authenticated via `cloudflared tunnel login`

---

## 🪜 Step-by-Step Setup

### **1️⃣ Install nginx and cloudflared**
```bash
sudo apt update && sudo apt install -y nginx apache2-utils cloudflared
```

---

### **2️⃣ Create a password file for Basic Auth**
```bash
sudo htpasswd -c /etc/nginx/.htpasswd user
# enter password when prompted
```

---

### **3️⃣ Create nginx site config**
```bash
sudo nano /etc/nginx/sites-available/projFolder
```

Paste this:
```nginx
server {
  listen 127.0.0.1:8080;
  server_name {yourdomain}.com;

  auth_basic "Restricted";
  auth_basic_{user}_file /etc/nginx/.htpasswd;

  location / {
    proxy_pass http://192.168.5.106:80;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # prevent large cookie/header issues
    proxy_set_header Cookie "";
  }
}
```

Enable and reload:
```bash
sudo ln -s /etc/nginx/sites-available/{yourProject} /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

### **4️⃣ Edit Cloudflare Tunnel config**
```bash
sudo nano /etc/cloudflared/config.yml
```

Use this format:
```yaml
tunnel: 11bfad09-a8ac-4a63-bd9a-14d63cf5ab68
credentials-file: /etc/cloudflared/11bfad09-a8ac-4a63-bd9a-14d63cf5ab68.json

ingress:
  - hostname: {yourdomain}.com
    service: http://127.0.0.1:8080
    originRequest:
      noTLSVerify: true
  - service: http_status:404
```

Restart and check logs:
```bash
sudo systemctl restart cloudflared
sudo journalctl -u cloudflared -f
```
Look for:  
`Route propagating: {yourDomain}.com → http://127.0.0.1:8080`

---

### **5️⃣ Test locally**
```bash
curl -I http://127.0.0.1:8080
# expect 401 Unauthorized (auth challenge)
curl -I -u {user} http://127.0.0.1:8080
# expect 200 OK
```

---

### **6️⃣ Verify externally**
Open an **incognito browser** and visit:
```
https://{yourdomain}.tech619.com
```
✅ You should see:  
- A **padlock** (Cloudflare HTTPS)  
- A **login prompt** from nginx  
- Your **dashboard** after login  

---

## 🧱 Optional Hardening

### Larger headers & buffers
To avoid “Header fields too large”:
```bash
sudo nano /etc/nginx/nginx.conf
```
Inside the `http {}` block:
```nginx
large_client_header_buffers 8 32k;
proxy_buffer_size 16k;
proxy_buffers 8 16k;
proxy_busy_buffers_size 32k;
```

---

### Disable Bot Fight Mode for this domain  
**(Cloudflare Dashboard → Security → Bots)**  
Create a “Skip” rule for `{yourdomain}.com` to avoid extra cookies.

---

### Allow only certain IPs (optional)
Inside your nginx `server {}` block:
```nginx
allow 12.34.56.78;  # your home/work IP
deny all;
```

---

## 🔄 Restart Services Anytime
```bash
sudo systemctl restart nginx
sudo systemctl restart cloudflared
sudo journalctl -u cloudflared -f
```

---

## 🧠 Summary

| Component | Purpose |
|------------|----------|
| **nginx** | Password-protected proxy, hides internal IP |
| **cloudflared** | Maintains tunnel to Cloudflare edge |
| **Cloudflare Edge** | Provides global HTTPS + DNS routing |
| **Your service** | Stays private, accessible only through authenticated tunnel |

---

## 💡 GPT Prompt to Rebuild This Setup

```text
Act as a Raspberry Pi sysadmin.  
I want to expose a local web service (like a miner dashboard) to the public internet securely using Cloudflare Tunnel and nginx Basic Auth.  
Walk me step-by-step like a cleanroom setup — assume I have a Pi, a domain on Cloudflare, and SSH access.  
Include every command, config file, and reload step so I can copy/paste my way through.  
```

---

### 🧩 Author
**@0xBerto**  
Built for repeatable, secure deployments of local apps to the web using Cloudflare’s edge and nginx on Raspberry Pi.
