# creioonescu.ro – Portfolio Site

A lightweight, container-packaged personal portfolio that serves static HTML/CSS/JS through **NGINX**, is published to **GitHub Container Registry (GHCR)**, and is exposed to the public web via a **Cloudflare Tunnel**—so no inbound ports or router configuration are required. I built this to run on an old mini-PC which I transformed into a home-server with Debian as the OS of choice ![WhatsApp Image 2025-08-06 at 20 13 24_8da715ff](https://github.com/user-attachments/assets/45e7b7c4-a015-4f4a-bdf5-48c8c4a6d9b3) I plan to turn this into a fully operational self-hosted e-commerce website.


---

## ✨  Key Features

| Feature                       | Notes                                                                                   |
| ----------------------------- | --------------------------------------------------------------------------------------- |
| **Single-stage Dockerfile**   | Tiny Alpine-based NGINX image (~110 MB) with static assets copied in.                  |
| **CI/CD with GitHub Actions** | Every push to `main` builds & tags `ghcr.io/alexander-drg/k8s-site:latest`.             |
| **Auto-pull on the server**   | A systemd timer pulls the latest image nightly and restarts the container.              |
| **Zero router changes**       | Cloudflare Tunnel keeps an outbound connection open; visitors hit HTTPS via Cloudflare. |
| **Let’s Encrypt optional**    | Native TLS provided by Cloudflare; no Certbot or port-forward needed.                   |

---

## 🗂  Repo Layout

k8s-site/
├─ Dockerfile            # NGINX + site assets (COPY site/ …)
├─ .dockerignore         # keeps image small (.git, README, etc.)
├─ site/                 # exported Webflow / static files
│   ├─ index.html
│   └─ css/, js/, images/
├─ .github/workflows/
│   └─ deploy.yml        # Build & push to GHCR on each commit
└─ README.md             # You are here

🚀 Quick Start (Local)

# build and tag locally
docker build -t k8s-site:dev .

# run on http://localhost:8080
docker run --rm -p 8080:80 k8s-site:dev


🐳 Build & Push to GHCR

# authenticate once per machine
export CR_PAT="<classic PAT with write:packages>"
docker login ghcr.io -u alexander-drg -p "$CR_PAT"

TAG=$(git rev-parse --short HEAD)  # e.g. 51f0f49

docker build -t ghcr.io/alexander-drg/k8s-site:$TAG \
             -t ghcr.io/alexander-drg/k8s-site:latest .
docker push ghcr.io/alexander-drg/k8s-site:$TAG
docker push ghcr.io/alexander-drg/k8s-site:latest

🌐 Deploy on Home Server
Run the container

docker run -d --name portfolio \
  --restart unless-stopped \
  -p 8080:80 \
  ghcr.io/alexander-drg/k8s-site:latest
Install & configure Cloudflare Tunnel


sudo apt install -y cloudflared
cloudflared tunnel login
cloudflared tunnel create portfolio
cloudflared tunnel route dns portfolio creioonescu.ro
cloudflared tunnel route dns portfolio www.creioonescu.ro

# ~/.cloudflared/config.yml
tunnel: <TUNNEL-ID>
credentials-file: ~/.cloudflared/<TUNNEL-ID>.json
ingress:
  - hostname: creioonescu.ro
    service: http://127.0.0.1:8080
  - hostname: www.creioonescu.ro
    service: http://127.0.0.1:8080
  - service: http_status:404

sudo cloudflared service install
Verify – browse https://creioonescu.ro (padlock present).

🤖 Auto-Update Service (Systemd)

# /etc/systemd/system/portfolio-update.timer
[Unit]
Description=Daily pull & restart portfolio container

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target

# /etc/systemd/system/portfolio-update.service
[Unit]
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
ExecStart=/usr/bin/docker pull ghcr.io/alexander-drg/k8s-site:latest
ExecStart=/usr/bin/docker rm -f portfolio
ExecStart=/usr/bin/docker run -d --name portfolio --restart unless-stopped -p 8080:80 ghcr.io/alexander-drg/k8s-site:latest
Enable with:

sudo systemctl daemon-reload
sudo systemctl enable --now portfolio-update.timer

🛠 Developing / Updating Content
Edit files under site/, rebuild the image, and push. CI publishes the new tag; the nightly timer pulls it automatically, or trigger manually:

sudo systemctl start portfolio-update.service

📜 License
MIT © 2025 Alexandru Drăghici

