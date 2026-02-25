# Server Deployment Guide (Ubuntu + Nginx + Postgres)

This guide documents a production deployment workflow for this Flowise fork, including channel/webhook concerns and known pitfalls discovered during implementation.

## Scope

- OS: Ubuntu 22.04/24.04
- Reverse proxy: Nginx
- Database: PostgreSQL
- Process manager: systemd
- TLS: Certbot (Let's Encrypt)

## 1. Architecture

- `Nginx (443/80)` terminates TLS and proxies to `Flowise server (127.0.0.1:3000)`.
- `Flowise server` runs as non-root `flowise` user.
- `Postgres` stores app data (recommended for production).

## 2. Prerequisites

- DNS A record pointing your domain to server IP.
- Ubuntu server with ports `22`, `80`, `443` open.
- Node.js `20.x`.
- pnpm `>= 10.26.0` (repo engine requirement).

## 3. Install system packages

```bash
sudo apt update
sudo apt install -y git curl build-essential nginx certbot python3-certbot-nginx postgresql postgresql-contrib
```

Install Node + pnpm:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo corepack enable
corepack prepare pnpm@10.26.0 --activate
node -v
pnpm -v
```

## 4. Create application user

```bash
sudo useradd -m -s /bin/bash flowise
sudo mkdir -p /home/flowise
sudo chown -R flowise:flowise /home/flowise
```

## 5. Clone and build workspace

> Important: run workspace commands from repo root, not only `packages/server`.

```bash
sudo su - flowise
cd /home/flowise
git clone <YOUR_FORK_REPO_URL> Flowise
cd Flowise
pnpm install
pnpm --filter ./packages/components build
pnpm --filter ./packages/ui build
pnpm --filter ./packages/server build
```

## 6. Configure PostgreSQL

Create DB and user:

```bash
sudo -u postgres psql
```

```sql
CREATE USER flowise WITH PASSWORD 'CHANGE_ME_STRONG_PASSWORD';
CREATE DATABASE flowise OWNER flowise;
GRANT ALL PRIVILEGES ON DATABASE flowise TO flowise;
\q
```

Optional connectivity test:

```bash
psql postgresql://flowise:CHANGE_ME_STRONG_PASSWORD@127.0.0.1:5432/flowise -c "SELECT 1;"
```

## 7. Configure server environment

Create `/home/flowise/Flowise/packages/server/.env`:

```env
NODE_ENV=production
HOST=127.0.0.1
PORT=3000

DATABASE_TYPE=postgres
DATABASE_HOST=127.0.0.1
DATABASE_PORT=5432
DATABASE_USER=flowise
DATABASE_PASSWORD=CHANGE_ME_STRONG_PASSWORD
DATABASE_NAME=flowise

FLOWISE_SECRETKEY_OVERWRITE=CHANGE_ME_LONG_RANDOM_VALUE
JWT_AUTH_TOKEN_SECRET=CHANGE_ME_LONG_RANDOM_VALUE
JWT_REFRESH_TOKEN_SECRET=CHANGE_ME_LONG_RANDOM_VALUE

LOG_LEVEL=info

# Optional only while debugging channel webhooks
# CHANNEL_WEBHOOK_DEBUG=true
```

## 8. Run database migrations

```bash
cd /home/flowise/Flowise/packages/server
pnpm typeorm:migration-run
```

If migration fails with TypeScript/workspace errors, return to repo root and run:

```bash
cd /home/flowise/Flowise
pnpm install
pnpm --filter ./packages/components build
pnpm --filter ./packages/server build
```

Then rerun migration.

## 9. Configure systemd

Create `/etc/systemd/system/flowise.service`:

```ini
[Unit]
Description=Flowise Server
After=network.target

[Service]
Type=simple
User=flowise
WorkingDirectory=/home/flowise/Flowise
Environment=NODE_ENV=production
Environment=HOST=127.0.0.1
Environment=PORT=3000
ExecStart=/usr/bin/pnpm start
Restart=always
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable flowise
sudo systemctl start flowise
sudo systemctl status flowise
journalctl -u flowise -f
```

## 10. Configure Nginx reverse proxy

Create `/etc/nginx/sites-available/flowise`:

```nginx
server {
    listen 80;
    server_name your-domain.com;

    client_max_body_size 50m;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 3600;
        proxy_send_timeout 3600;
        proxy_connect_timeout 60;

        proxy_buffering off;
    }
}
```

Enable and reload:

```bash
sudo ln -s /etc/nginx/sites-available/flowise /etc/nginx/sites-enabled/flowise
sudo nginx -t
sudo systemctl reload nginx
```

## 11. Enable HTTPS

```bash
sudo certbot --nginx -d your-domain.com
sudo systemctl reload nginx
```

## 12. Webhook/channel production setup

For WhatsApp/Instagram callbacks:

- Use full webhook URL format:
  - `/api/v1/channel-webhooks/:provider/:webhookPath`
- Example:
  - `https://your-domain.com/api/v1/channel-webhooks/whatsapp/<webhookPath>`
- Ensure callback uses HTTPS.
- Keep app secret, verify token, access token aligned with the same Meta app/business assets.

## 13. Verification checklist

- Service reachable:

```bash
curl -i https://your-domain.com/api/v1/ping
```

- Process and proxy logs:

```bash
journalctl -u flowise -f
sudo tail -f /var/log/nginx/access.log /var/log/nginx/error.log
```

- Confirm webhook challenge succeeds in Meta.
- Send inbound message and verify response path.

## 14. Upgrade/deploy workflow

```bash
sudo su - flowise
cd /home/flowise/Flowise
git pull
pnpm install
pnpm --filter ./packages/components build
pnpm --filter ./packages/ui build
pnpm --filter ./packages/server build
cd packages/server
pnpm typeorm:migration-run
cd ../..
exit
sudo systemctl restart flowise
sudo systemctl status flowise
```

## 14.1 Restart procedures

### Restart service (no code change)

```bash
sudo systemctl restart flowise
sudo systemctl status flowise
journalctl -u flowise -f
```

### Restart after code/config change

```bash
sudo su - flowise
cd /home/flowise/Flowise
pnpm install
pnpm --filter ./packages/components build
pnpm --filter ./packages/ui build
pnpm --filter ./packages/server build
cd packages/server
pnpm typeorm:migration-run
exit
sudo systemctl restart flowise
sudo systemctl status flowise
```

### Reload only Nginx config

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 15. Troubleshooting (issues seen during implementation)

### 15.1 pnpm engine mismatch

Symptom:

- install/build/lint fails with pnpm version mismatch.

Fix:

- Use `pnpm >= 10.26.0`.

### 15.2 `command start not found` / oclif confusion

Symptom:

- server launch via incorrect command path fails.

Fix:

- For production use `pnpm start` from repo root via systemd.
- For local dev use `pnpm oclif-dev` in `packages/server`.

### 15.3 Migration compile errors (`Cannot find module 'flowise-components'`)

Symptom:

- `pnpm typeorm:migration-run` fails compiling TS.

Cause:

- running from `packages/server` without workspace deps/build artifacts.

Fix:

```bash
cd /home/flowise/Flowise
pnpm install
pnpm --filter ./packages/components build
pnpm --filter ./packages/server build
cd packages/server
pnpm typeorm:migration-run
```

### 15.4 Missing UI build artifact

Symptom:

- `ENOENT ... flowise-ui/build/index.html`

Fix:

```bash
cd /home/flowise/Flowise
pnpm install
pnpm --filter ./packages/ui build
pnpm --filter ./packages/server build
```

### 15.5 Register page click shows no network call

Symptom:

- no `/api/v1/account/register` in browser network.

Cause:

- current register page path depends on platform flags and may not fire for open-source mode.

Fix:

- Use initial organization bootstrap flow, or patch register UI branch for open-source mode.

### 15.6 Webhook challenge works but POST returns 401

Cause:

- POST uses Meta HMAC signature validation; challenge token success does not validate POST auth.

Checks:

- correct app secret
- `x-hub-signature-256` preserved through proxy
- raw body not mutated

### 15.7 Outbound WhatsApp send fails (401/190)

Symptom:

- Graph API token errors (`OAuthException`, code `190`).

Fix:

- refresh/replace expired token
- ensure token scope/asset permissions match phone number ID

### 15.8 Duplicate inbound retries

Behavior:

- log like `duplicate inbound ignored provider=whatsapp`.

Explanation:

- idempotency guard is active and suppresses duplicate webhook deliveries.

### 15.9 LLM runtime message order error

Symptom:

- `System messages are only permitted as the first passed message.`

Fix:

- ensure flow message assembly keeps system role first and does not inject later system messages from downstream nodes.

## 16. Security and operations recommendations

- Keep server bound to `127.0.0.1`; expose only via Nginx.
- Disable debug flags in production (`CHANNEL_WEBHOOK_DEBUG=false`).
- Use strong random secrets and rotate periodically.
- Backup Postgres regularly (daily dumps + retention).
- Keep OS and Node dependencies patched.
- Restrict admin SSH access and use keys only.

## 17. Vector DB for Production (Qdrant)

Recommended default for this fork in production: **Qdrant**.

### 17.1 Install Docker

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable docker
sudo systemctl start docker
```

### 17.2 Create persistent storage

```bash
sudo mkdir -p /opt/qdrant/storage
sudo chown -R $USER:$USER /opt/qdrant
```

### 17.3 Run Qdrant

```bash
docker run -d \
  --name qdrant \
  --restart unless-stopped \
  -p 6333:6333 \
  -p 6334:6334 \
  -e QDRANT__SERVICE__API_KEY='CHANGE_ME_STRONG_KEY' \
  -v /opt/qdrant/storage:/qdrant/storage \
  qdrant/qdrant:latest
```

### 17.4 Verify health

```bash
curl http://127.0.0.1:6333/healthz
```

### 17.5 Configure Flowise credentials/nodes

- Endpoint: `http://127.0.0.1:6333` (same host case)
- API Key: `CHANGE_ME_STRONG_KEY`
- Use a stable collection naming convention per environment.
- Keep embedding dimensions and distance metric consistent with your model.

### 17.6 Nginx/network hardening for Qdrant

- Do **not** expose Qdrant publicly unless required.
- Bind and access Qdrant privately (localhost/VPC/internal subnet).
- If remote access is needed, enforce firewall allowlist + TLS termination.

### 17.7 Backup strategy

- Back up `/opt/qdrant/storage` regularly, or use Qdrant snapshot API.
- Store snapshots in object storage with retention policy.
- Test restore procedure periodically in staging.
