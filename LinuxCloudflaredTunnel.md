# Cloudflare Tunnel on Linux (No Nginx)
Securely expose local dashboards, services, or Docker apps to the internet using HTTPS, with **no port forwarding** and **no extra login layer**. Your app keeps full control of authentication (Grafana login, your own auth, etc.).

---

## ✅ What This Setup Gives You
- Public URL like `https://app.yourdomain.com`
- Free HTTPS via Cloudflare
- No inbound firewall holes (tunnel is outbound-only)
- No nginx, no Basic Auth gate, no extra auth layer
- Works for:
  - Docker containers
  - Host services (localhost / server ports)
  - LAN devices (cameras, miners, dashboards on other machines)

---

## 🧠 Architecture

Browser
↓ HTTPS
Cloudflare Edge (DNS + TLS)
↓ Cloudflare Tunnel
cloudflared (Linux host or Docker container)
↓
Your service (Docker container, localhost, or LAN IP)

yaml
Copy code

---

## ⚠️ Security Note (Important)
If the app you expose has **no authentication**, you are making it public on the internet.

If you want “public but safer” without adding a login wall:
- Use Cloudflare WAF / Firewall Rules (IP allowlist, geo blocks)
- Rate limiting
- Bot controls
- Separate hostnames for admin-only dashboards

---

## ✅ Prereqs
- Domain on Cloudflare (nameservers pointing to Cloudflare)
- A service you want to expose:
  - Docker container(s), or
  - Host service, or
  - LAN device at `http://192.168.x.x:port`
- Linux server or VM (Ubuntu/Debian assumed for install steps)

---

# Recommended Pattern
## ✅ Option A: Docker Compose (App + Tunnel in One Repo)
Best for repeatable deployments and “project templates”.
- Everything is defined in one `docker-compose.yml`
- cloudflared and your app share a private Docker network
- Routing is defined in `cloudflared/config.yml`

---

## 1) Install Docker + Compose (if needed)

Quick check:
```bash
docker --version
docker compose version
If you don’t have Docker installed, install it using the official Docker docs for your distro.

2) Install cloudflared on the host (one-time)
This uses Cloudflare’s apt repo for Debian/Ubuntu.

bash
Copy code
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-public-v2.gpg | sudo tee /usr/share/keyrings/cloudflare-public-v2.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-public-v2.gpg] https://pkg.cloudflare.com/cloudflared any main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt-get update && sudo apt-get install cloudflared
3) Authenticate cloudflared
This opens a browser flow so Cloudflare can authorize your machine.

bash
Copy code
cloudflared tunnel login
4) Create a tunnel
Name it whatever you want (example: my-tunnel).

bash
Copy code
cloudflared tunnel create my-tunnel
You’ll get a tunnel UUID and a credentials JSON file.

5) Create DNS routes for each hostname
Examples:

bash
Copy code
cloudflared tunnel route dns my-tunnel grafana.yourdomain.com
cloudflared tunnel route dns my-tunnel nerdqaxe.yourdomain.com
You can add as many hostnames as you want.

6) Create a project folder layout
Example structure:

pgsql
Copy code
project/
  docker-compose.yml
  cloudflared/
    config.yml
    <TUNNEL-UUID>.json
Copy your tunnel credentials JSON into project/cloudflared/.

Where is it?
Usually in:

~/.cloudflared/<TUNNEL-UUID>.json

Copy it:

bash
Copy code
mkdir -p project/cloudflared
cp ~/.cloudflared/<TUNNEL-UUID>.json project/cloudflared/
7) Create cloudflared/config.yml
Replace:

<TUNNEL-UUID>

hostnames

service names/ports

yaml
Copy code
tunnel: <TUNNEL-UUID>
credentials-file: /etc/cloudflared/<TUNNEL-UUID>.json

ingress:
  - hostname: grafana.yourdomain.com
    service: http://grafana:3000

  - hostname: nerdqaxe.yourdomain.com
    service: http://nerdqaxe:80

  - service: http_status:404
Notes:

The final http_status:404 rule is required as a catch-all.

service: can point to Docker container names on the shared network.

8) Create docker-compose.yml
Example with 2 apps + tunnel container:

yaml
Copy code
services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    networks:
      - tunnel

  nerdqaxe:
    image: your/nerdqaxe-image:latest
    container_name: nerdqaxe
    restart: unless-stopped
    networks:
      - tunnel

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel run
    volumes:
      - ./cloudflared:/etc/cloudflared:ro
    networks:
      - tunnel

networks:
  tunnel:
    name: tunnel
Notes:

You do not need to publish ports like 3000:3000 unless you also want local LAN access.

Cloudflare Tunnel will reach containers privately through the Docker network.

9) Start everything
From inside project/:

bash
Copy code
docker compose up -d
docker logs -f cloudflared
If everything is correct, you should see the tunnel connect.

10) Verify externally
Open in an incognito window:

https://grafana.yourdomain.com

https://nerdqaxe.yourdomain.com

You should see the app directly.
If the app has a login page, it should appear normally.

Debug tips (Option A)
Check that the tunnel container can reach your service containers:

bash
Copy code
docker exec -it cloudflared wget -qO- http://grafana:3000 >/dev/null && echo OK
docker exec -it cloudflared wget -qO- http://nerdqaxe:80 >/dev/null && echo OK
View logs:

bash
Copy code
docker logs -f cloudflared
Restart:

bash
Copy code
docker compose restart cloudflared
Option B: Run cloudflared on the Host (Best for LAN devices / host services)
Use this when your target is not inside Docker, like:

Camera UI at http://192.168.1.50:80

Miner dashboard on another machine on your LAN

A host service bound to 127.0.0.1:PORT

1) Install + authenticate + create tunnel
Same steps as above:

bash
Copy code
sudo apt-get update && sudo apt-get install cloudflared
cloudflared tunnel login
cloudflared tunnel create my-tunnel
cloudflared tunnel route dns my-tunnel camera.yourdomain.com
2) Create config file on host
Example: ~/.cloudflared/config.yml

yaml
Copy code
tunnel: <TUNNEL-UUID>
credentials-file: /home/<user>/.cloudflared/<TUNNEL-UUID>.json

ingress:
  - hostname: camera.yourdomain.com
    service: http://192.168.1.50:80

  - hostname: app.yourdomain.com
    service: http://127.0.0.1:8080

  - service: http_status:404
3) Run the tunnel
Manual run:

bash
Copy code
cloudflared tunnel run <TUNNEL-UUID>
4) Install as a system service (recommended)
bash
Copy code
sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl status cloudflared
Logs:

bash
Copy code
sudo journalctl -u cloudflared -f
Restart:

bash
Copy code
sudo systemctl restart cloudflared
Option C: Token-based Tunnel (Fastest, config stored in Cloudflare dashboard)
This is useful if you don’t want to keep config.yml in your repo.
The tunnel rules live in Cloudflare Zero Trust dashboard.

Run with token:

bash
Copy code
docker run cloudflare/cloudflared:latest tunnel --no-autoupdate run --token <TUNNEL_TOKEN>
Tradeoff:

Faster to start

Less portable for repos (config isn’t local)

Quick Checklist
Domain is on Cloudflare

Tunnel exists

DNS routes created for hostnames

Ingress rules point to correct service targets

Tunnel process/container is running

App is reachable internally

Minimal “Project Template” Summary (Copy/Paste)
Create tunnel + DNS routes

bash
Copy code
cloudflared tunnel login
cloudflared tunnel create my-tunnel
cloudflared tunnel route dns my-tunnel app.yourdomain.com
Put tunnel JSON in ./cloudflared/

cloudflared/config.yml

yaml
Copy code
tunnel: <TUNNEL-UUID>
credentials-file: /etc/cloudflared/<TUNNEL-UUID>.json

ingress:
  - hostname: app.yourdomain.com
    service: http://app:3000
  - service: http_status:404
docker-compose.yml

yaml
Copy code
services:
  app:
    image: your/app:latest
    restart: unless-stopped
    networks: [tunnel]

  cloudflared:
    image: cloudflare/cloudflared:latest
    restart: unless-stopped
    command: tunnel run
    volumes:
      - ./cloudflared:/etc/cloudflared:ro
    networks: [tunnel]

networks:
  tunnel:
    name: tunnel
Start:

bash
Copy code
docker compose up -d
docker logs -f cloudflared
Visit:

https://app.yourdomain.com

Author
@0xBerto
A repeatable Cloudflare Tunnel pattern for securely exposing local services without port forwarding or an extra auth gate.
