# VPS cloud-init app templates (Debian 13)

**Audience:** KVM / Virtualizor (and similar) VPS providers  
**Version:** 2026-10-11  
**OS:** Debian 13 (trixie) only — amd64 or arm64  
**Status:** Lab-ready prototypes. Statically validated (YAML + `bash -n` + `cloud-init schema`). Live VM soak-testing still required before selling as a finished one-click product.

Each `NN-*.yaml` is a **self-contained cloud-init user-data** file. Deploy **exactly one** template per fresh VPS. There is no shared YAML dependency between apps.

On first boot the template:

1. Enables **root SSH** (`PermitRootLogin yes`, `PasswordAuthentication yes`)  
2. Installs Docker Engine + Compose (official Docker `trixie` apt repo) if needed  
3. Creates app files under `/opt/vps-apps/<template>/`  
4. Starts the app (except OpenClaw and Hermes, which stage for interactive setup)  
5. Installs `sudo /usr/local/sbin/vps-app-test` for a quick health check  

App credentials are generated **on the VPS** into `.env` files (not baked into user-data).

### Root SSH password (set in each YAML)

Every template sets a **default root password** with cloud-init `chpasswd` so SSH works even when the panel password is ignored/overridden by custom user-data:

| Field | Value |
|---|---|
| User | `root` |
| Default password | `VpsApp-ChangeMe-2026` |

**Change it before production / before giving a VPS to a customer.** Near the top of each `NN-*.yaml`:

```yaml
users:
- name: root
  lock_passwd: false
  plain_text_passwd: "YOUR-STRONG-PASSWORD-HERE"
```

Also enabled: `disable_root: false`, `ssh_pwauth: true`, and `PermitRootLogin yes`.

If Virtualizor also injects a root password, cloud-init `chpasswd` in this user-data wins for that boot — use the password from the YAML (or change the YAML to match what you sell in WHMCS).

Bootstrap waits for a sane system clock (ConfigDrive VMs often boot at `1970` until NTP catches up). Without that, `apt` / Docker HTTPS pulls fail and `cloud-init` shows `runcmd` errors.

---

## Ready-to-use options

| Mode | When to use | How customers reach the app |
|---|---|---|
| **A. IP ready (default in these files)** | Fast lab / first boot tests | `http://VPS_IP:PORT` (Portainer also has HTTPS on `:9443`) |
| **B. Domain + HTTPS (recommended for customers)** | Selling / production-like use | `https://app.example.com` via reverse proxy (Caddy or nginx) |
| **C. Interactive agents** | OpenClaw, Hermes | SSH in, run setup, then use IP or domain |

Current templates ship in **Mode A**: app ports are published on all interfaces so you can test from the internet without an SSH tunnel. Claim admin UIs quickly; do not leave unclaimed signup pages open on public IPs longer than necessary.

---

## App catalog

| File | App | Ready after boot? | Public port | Default URL | Min → Rec (vCPU / RAM / SSD) |
|---|---|---|---|---|---|
| `01-n8n.yaml` | n8n | Yes | `5678` | `http://VPS_IP:5678` | 1/2G/15G → 2/4G/30G |
| `02-openclaw.yaml` | OpenClaw | After `sudo openclaw-setup` | `18789` | `http://VPS_IP:18789` | 1/1G/10G → 2/4G/25G |
| `03-wordpress.yaml` | WordPress | Yes (finish wizard) | `8080` | `http://VPS_IP:8080` | 1/1G/15G → 2/2G/30G |
| `04-coolify.yaml` | Coolify | Yes (claim admin ASAP) | `8000` (+ `6001`/`6002`) | `http://VPS_IP:8000` | 2/2G/20G → 4/4G/50G |
| `05-open-webui.yaml` | Open WebUI | Yes (needs LLM backend) | `3000` | `http://VPS_IP:3000` | 1/2G/15G → 2/4G/30G |
| `06-hermes-agent.yaml` | Hermes Agent | After `sudo hermes-setup` | — | No web UI (outbound messaging) | 1/2G/15G → 2/4G/30G |
| `07-woocommerce.yaml` | WooCommerce | Yes (activate plugin) | `8080` | `http://VPS_IP:8080` | 2/4G/25G → 4/8G/50G |
| `08-ollama.yaml` | Ollama | Yes (tiny CPU model) | `11434` | `http://VPS_IP:11434/api/tags` | 2/4G/20G → 4/8G/50G |
| `09-portainer.yaml` | Portainer CE | Yes | `9000` (HTTP), `9443` (HTTPS) | `http://VPS_IP:9000` or `https://VPS_IP:9443` | 1/2G/15G → 2/4G/30G |
| `10-chatwoot.yaml` | Chatwoot CE | Yes (no SMTP by default) | `3000` | `http://VPS_IP:3000` | 4/4G/30G → 4/8G/50G |
| `11-dify.yaml` | Dify | Yes (multi-container warmup) | `8080` (HTTP), `8443` (HTTPS self-signed) | `http://VPS_IP:8080/install` or `https://VPS_IP:8443/install` | 2/4G/30G → 4/8G/60G |
| `12-odoo.yaml` | Odoo 19 | Yes (create DB) | `8069` | `http://VPS_IP:8069` | 2/4G/30G → 4/8G/60G |

**Firewall:** open the app port(s) + `22/tcp` on the provider/hypervisor firewall for Mode A. For Mode B, prefer opening only `80/tcp` and `443/tcp` to the world and keep app ports firewalled to localhost after the reverse proxy is in place.

---

## Quick start (Mode A — IP ready)

1. Create a **new Debian 13** VPS (one app per VPS). Use at least the recommended resources for that app.
2. Paste the **entire** `NN-*.yaml` as cloud-init **user-data** (must start with `#cloud-config`). Do not paste it into a shell.
3. Power on, SSH in, wait for bootstrap:

```bash
sudo cloud-init status --wait --long
sudo /usr/local/sbin/vps-app-test
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
sudo ss -lntp
```

4. From your browser / another host:

```text
http://VPS_IP:PORT
```

5. Finish the app’s first-run wizard. Then reboot once and re-check `vps-app-test`.

### Interactive apps only

```bash
# OpenClaw
sudo openclaw-setup
sudo /usr/local/sbin/vps-app-test

# Hermes
sudo hermes-setup
sudo /usr/local/sbin/vps-app-test
```

### Ollama smoke test

```bash
curl -fsS http://127.0.0.1:11434/api/tags
curl -fsS http://127.0.0.1:11434/api/generate \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen2.5:0.5b","prompt":"Say hello in one sentence","stream":false}'
```

---

## Mode B — Use a domain for any / all apps

Same pattern for every web app: **DNS → reverse proxy on :80/:443 → local app port**.  
Hermes has no HTTP UI; skip domain setup for it unless you later add your own dashboard.

### 1) DNS

For each VPS / app, create an **A** (or **AAAA**) record:

```text
n8n.example.com        →  VPS_IP
blog.example.com       →  VPS_IP
shop.example.com       →  VPS_IP
coolify.example.com    →  VPS_IP
ai.example.com         →  VPS_IP
chat.example.com       →  VPS_IP
dify.example.com       →  VPS_IP
erp.example.com        →  VPS_IP
portainer.example.com  →  VPS_IP
ollama.example.com     →  VPS_IP   # only if you intentionally expose the API
gateway.example.com    →  VPS_IP   # OpenClaw after setup
```

Wait until DNS resolves:

```bash
dig +short n8n.example.com
```

### 2) Suggested hostnames and upstream ports

| App | Example hostname | Upstream on the VPS |
|---|---|---|
| n8n | `n8n.example.com` | `127.0.0.1:5678` |
| OpenClaw | `gateway.example.com` | `127.0.0.1:18789` |
| WordPress | `blog.example.com` | `127.0.0.1:8080` |
| Coolify | `coolify.example.com` | `127.0.0.1:8000` |
| Open WebUI | `ai.example.com` | `127.0.0.1:3000` |
| WooCommerce | `shop.example.com` | `127.0.0.1:8080` |
| Ollama | `ollama.example.com` | `127.0.0.1:11434` |
| Portainer | `portainer.example.com` | `127.0.0.1:9000` (HTTP) or `127.0.0.1:9443` (HTTPS upstream) |
| Chatwoot | `chat.example.com` | `127.0.0.1:3000` |
| Dify | `dify.example.com` | `127.0.0.1:8080` (HTTP) or `127.0.0.1:8443` (HTTPS self-signed) |
| Odoo | `erp.example.com` | `127.0.0.1:8069` |

One domain per VPS is the simple model (these templates are one-app-per-VPS). If you later host a reverse proxy elsewhere, point the hostname at that proxy instead.

### 3) Install Caddy (HTTPS automatic)

Run on the VPS after the app is healthy:

```bash
sudo apt-get update -qq
sudo apt-get install -y -qq debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
  | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt-get update -qq
sudo apt-get install -y -qq caddy
```

Create `/etc/caddy/Caddyfile` (replace hostname + port):

```caddy
n8n.example.com {
        reverse_proxy 127.0.0.1:5678
}
```

Portainer (HTTP upstream on `:9000`, simplest):

```caddy
portainer.example.com {
        reverse_proxy 127.0.0.1:9000
}
```

Or Portainer HTTPS upstream on `:9443` (self-signed):

```caddy
portainer.example.com {
        reverse_proxy https://127.0.0.1:9443 {
                transport http {
                        tls_insecure_skip_verify
                }
        }
}
```

Reload:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl enable --now caddy
sudo systemctl reload caddy
```

Open **80** and **443** on the provider firewall. After Caddy works, optionally re-bind the Docker publish to loopback only (`127.0.0.1:PORT:PORT`) and recreate the stack so the raw app port is no longer public.

### 4) Point each app at the public domain

Apps that generate links, cookies, webhooks, or assets need their **canonical public URL** updated. Edit the VPS `.env` (or app setting), then recreate the stack.

#### n8n — `/opt/vps-apps/01-n8n/.env`

```bash
N8N_HOST=n8n.example.com
N8N_PROTOCOL=https
N8N_PORT=443
N8N_EDITOR_BASE_URL=https://n8n.example.com/
WEBHOOK_URL=https://n8n.example.com/
N8N_SECURE_COOKIE=true
N8N_ENCRYPTION_KEY=...keep existing...
```

```bash
cd /opt/vps-apps/01-n8n
# also set in compose environment if you keep N8N_PROTOCOL/PORT there:
sudo sed -i 's/N8N_PROTOCOL: http/N8N_PROTOCOL: https/' compose.yaml
sudo sed -i 's/N8N_PORT: 5678/N8N_PORT: 443/' compose.yaml
sudo sed -i "s/N8N_SECURE_COOKIE: 'false'/N8N_SECURE_COOKIE: 'true'/" compose.yaml
sudo docker compose up -d
```

#### WordPress / WooCommerce

Complete (or re-run) the install using the domain, or update siteurl/home:

```bash
cd /opt/vps-apps/03-wordpress   # or 07-woocommerce
sudo docker compose exec -T wordpress wp option update siteurl 'https://blog.example.com' --allow-root || true
sudo docker compose exec -T wordpress wp option update home 'https://blog.example.com' --allow-root || true
```

If `wp` CLI is unavailable inside the image, set **Settings → General** in the WordPress admin, or update `wp_options` in MariaDB.

#### Coolify

Open `https://coolify.example.com`, finish onboarding, and set the instance domain in Coolify’s settings. Coolify can also issue certificates for apps it deploys.

#### Open WebUI — optional compose env

```yaml
environment:
  WEBUI_URL: https://ai.example.com
```

Then `docker compose up -d` in `/opt/vps-apps/05-open-webui`.

#### Chatwoot — `/opt/vps-apps/10-chatwoot/.env`

Bootstrap sets `FRONTEND_URL=http://<IPv4>:3000` (IPv4 only — bare IPv6 breaks Chatwoot with `URI::InvalidURIError`).  
For a domain:

```bash
FRONTEND_URL=https://chat.example.com
```

```bash
cd /opt/vps-apps/10-chatwoot
sudo docker compose up -d
sudo docker compose restart rails sidekiq
```

#### Dify — `/opt/vps-apps/11-dify/source/docker/.env`

Lab IP access (open provider firewall **8080** and **8443**):

- HTTP: `http://VPS_IP:8080/install`
- HTTPS: `https://VPS_IP:8443/install` (self-signed cert; accept the browser warning)

Template sets `EXPOSE_NGINX_PORT=8080`, `EXPOSE_NGINX_SSL_PORT=8443`, and `NGINX_HTTPS_ENABLED=true` with a generated `nginx/ssl/dify.crt` + `dify.key`.

For a real domain (preferred in production), set at least:

```bash
CONSOLE_API_URL=https://dify.example.com
CONSOLE_WEB_URL=https://dify.example.com
SERVICE_API_URL=https://dify.example.com
APP_API_URL=https://dify.example.com
APP_WEB_URL=https://dify.example.com
```

(Exact keys can vary slightly by Dify release; keep names that already exist in `.env.example`.)

```bash
cd /opt/vps-apps/11-dify/source/docker
sudo docker compose up -d
```

#### Odoo

Use `https://erp.example.com` in the browser. If you terminate TLS at Caddy, keep Odoo on HTTP upstream (`proxy_mode` may need enabling for production reverse proxies — see Odoo docs).

Master password (DB manager):

```bash
sudo grep ODOO_MASTER_PASSWORD /opt/vps-apps/12-odoo/.env
```

#### Portainer

Lab IP test: `http://VPS_IP:9000` (or `https://VPS_IP:9443` with the self-signed warning).  
With a domain/Caddy: `https://portainer.example.com`. Create the admin user on first visit.

#### OpenClaw

After `sudo openclaw-setup`, put Caddy in front of `127.0.0.1:18789` and use `https://gateway.example.com` in the dashboard / token config.

#### Ollama

Domain is optional. If exposed, put auth or IP allowlists in front — the API has **no** built-in login. Prefer keeping it on private network / tunnel for production.

#### Hermes

No HTTP service to put behind a domain by default.

### 5) One-shot helper: bind app to localhost after domain works

Example for n8n after Caddy is serving HTTPS:

```bash
cd /opt/vps-apps/01-n8n
sudo sed -i "s/- '5678:5678'/- '127.0.0.1:5678:5678'/" compose.yaml
sudo docker compose up -d
```

Repeat the same idea for each app’s published port. Provider firewall should then allow **22, 80, 443** only.

### 6) Optional: nginx instead of Caddy

```nginx
server {
    listen 80;
    server_name n8n.example.com;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl http2;
    server_name n8n.example.com;
    # ssl_certificate / ssl_certificate_key via certbot
    location / {
        proxy_pass http://127.0.0.1:5678;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Issue certs with Certbot (`python3-certbot-nginx`), then reload nginx.

---

## Provider integration (Virtualizor / WHMCS)

1. Build a Debian 13 template with working cloud-init.  
2. Map each marketplace product to **one** `NN-*.yaml` as user-data.  
3. Document the product’s port (Mode A) or require a hostname field (Mode B).  
4. Never print `/opt/vps-apps/**/.env` secrets into tickets, emails, or panel logs.  
5. Acceptance checklist before marking an app “READY” for sale:

- `cloud-init status: done` on a cold boot  
- `vps-app-test` passes  
- Real functional test (login + one business action), not only HTTP 200  
- Reboot persistence  
- Domain + HTTPS path tested if you sell domains  
- Backup / restore of volumes tested on a disposable VPS  
- License / acceptable-use review for upstream software  

---

## Paths and logs

| Item | Location |
|---|---|
| App root | `/opt/vps-apps/<NN-name>/` |
| Compose (most apps) | `/opt/vps-apps/<NN-name>/compose.yaml` |
| Dify compose | `/opt/vps-apps/11-dify/source/docker/` |
| Coolify data | `/data/coolify/` |
| OpenClaw source | `/opt/vps-apps/02-openclaw/source/` |
| Bootstrap log | `/var/log/vps-apps/<NN-name>-bootstrap.log` |
| cloud-init log | `/var/log/cloud-init-output.log` |
| Health helper | `/usr/local/sbin/vps-app-test` |
| Retry bootstrap | `sudo bash /usr/local/sbin/vps-app-bootstrap` |

```bash
sudo cloud-init status --long
sudo tail -n 150 /var/log/vps-apps/*-bootstrap.log
sudo journalctl -u docker -n 80 --no-pager
cd /opt/vps-apps/01-n8n && sudo docker compose logs --tail=100
```

Do **not** run `docker compose down -v` unless you intend to destroy data. Do **not** cloud-init clean/reset a live customer VM.

---

## Security notes

- Mode A is for **lab / controlled testing**. Signup pages (Coolify, Portainer, Open WebUI, Chatwoot, Dify, etc.) are claimable by anyone who reaches the port first.  
- For customer delivery, prefer **Mode B** (domain + HTTPS) and lock app ports to localhost.  
- Ollama and similar APIs should not be public without authentication.  
- Rotate any secret that may have appeared in console logs.  
- Keep provider firewall rules in sync with the mode you sell.

---

## Official references

- Docker on Debian: https://docs.docker.com/engine/install/debian/  
- n8n: https://docs.n8n.io/hosting/installation/docker/  
- OpenClaw: https://docs.openclaw.ai/install/docker-vm-runtime  
- WordPress: https://hub.docker.com/_/wordpress  
- WooCommerce: https://woocommerce.com/documentation/  
- Coolify: https://coolify.io/docs/get-started/introduction  
- Open WebUI: https://docs.openwebui.com/getting-started/quick-start/  
- Hermes: https://hermes-agent.nousresearch.com/docs/user-guide/docker  
- Ollama: https://docs.ollama.com/  
- Portainer: https://docs.portainer.io/start/install-ce/server/docker/linux  
- Chatwoot: https://developers.chatwoot.com/self-hosted/deployment/docker  
- Dify: https://docs.dify.ai/en/self-host/quick-start/docker-compose  
- Odoo: https://hub.docker.com/_/odoo  
- Caddy: https://caddyserver.com/docs/  
