# Authentik Setup Guide

## Cross-Compatible: Dockhand & Docker Compose CLI

This guide covers deploying Authentik using either Dockhand or the Docker Compose CLI with the same files.

---

## Prerequisites

- Ubuntu 24.04 VM with Docker installed
- Dockhand (if using the GUI method) or terminal access for the CLI method
- Nginx Proxy Manager with a wildcard SSL certificate
- A chosen subdomain (e.g. `auth.yourdomain.com`)

---

## Installing Dockhand

[Dockhand](https://dockhand.pro) is a modern Docker management UI — free for homelabs, with OIDC/SSO included at no cost. If you plan to use Method 1 (GUI deployment), install Dockhand first.

### Option A: Docker Compose (recommended)

```yaml
# dockhand/docker-compose.yml
services:
  dockhand:
    image: fnsys/dockhand:latest
    container_name: dockhand
    restart: unless-stopped
    ports:
      - 3000:3000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - dockhand_data:/app/data

volumes:
  dockhand_data:
```

```bash
docker compose up -d
```

### Option B: Docker run (quick start)

```bash
docker run -d \
  --name dockhand \
  --restart unless-stopped \
  -p 3000:3000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v dockhand_data:/app/data \
  fnsys/dockhand:latest
```

### Option C: With PostgreSQL (multi-host / team use)

```yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: dockhand
      POSTGRES_PASSWORD: changeme
      POSTGRES_DB: dockhand
    volumes:
      - postgres_data:/var/lib/postgresql/data

  dockhand:
    image: fnsys/dockhand:latest
    restart: unless-stopped
    ports:
      - 3000:3000
    environment:
      DATABASE_URL: postgres://dockhand:changeme@postgres:5432/dockhand
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - dockhand_data:/app/data
    depends_on:
      - postgres

volumes:
  postgres_data:
  dockhand_data:
```

After deploying, open `http://localhost:3000` (or your VM's IP) to complete initial setup. Put it behind NPM with SSL for production use.

> For managing Docker hosts behind NAT or a firewall, Dockhand uses the open-source [Hawser agent](https://github.com/Finsys/hawser) — see the [Dockhand docs](https://dockhand.pro/manual/) for multi-host setup.

---

---

## Method 1: Deploy with Dockhand (GUI)

### Step 1: Generate secrets

On your Ubuntu VM, generate the required secrets:

```bash
# Generate PostgreSQL password
echo "PG_PASS=$(openssl rand -base64 36 | tr -d '\n')"

# Generate Authentik secret key
echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')"
```

Copy these values — you'll need them when creating the stack in Dockhand.

### Step 2: Create stack in Dockhand

1. Log into Dockhand
2. Navigate to **Stacks** → **New Stack**
3. Name: `authentik`
4. Paste the entire `docker-compose.yml` content into the editor

### Step 3: Add environment variables

Add the following in the stack's environment variable section:

**Required:**
```
PG_PASS=<paste-your-generated-password>
AUTHENTIK_SECRET_KEY=<paste-your-generated-secret>
```

**Optional (defaults shown):**
```
PG_USER=authentik
PG_DB=authentik
AUTHENTIK_ERROR_REPORTING__ENABLED=false
```

**Email (if configuring — Fastmail example):**
```
AUTHENTIK_EMAIL__HOST=smtp.fastmail.com
AUTHENTIK_EMAIL__PORT=587
AUTHENTIK_EMAIL__USERNAME=your-email@fastmail.com
AUTHENTIK_EMAIL__PASSWORD=your-app-specific-password
AUTHENTIK_EMAIL__USE_TLS=true
AUTHENTIK_EMAIL__FROM=authentik@yourdomain.com
```

> **Fastmail:** Use an app-specific password generated under **Settings → Privacy & Security → Connected Apps & API Tokens**. Do not use your main account password.

### Step 4: Deploy

1. Click **Deploy**
2. Wait 2–3 minutes for all containers to start and become healthy
3. Check container logs for errors
4. Confirm all 4 containers (postgresql, redis, server, worker) are running and healthy

---

## Method 2: Deploy with Docker Compose CLI

### Step 1: Prepare the directory

```bash
mkdir -p /opt/authentik
cd /opt/authentik
# Copy or clone docker-compose.yml and env.template here
```

Before deploying, review `docker-compose.yml` and update the following to match your setup:

- **`AUTHENTIK_HOST`** — set to your public Authentik URL (e.g. `https://auth.yourdomain.com`) in both `server` and `worker`
- **`AUTHENTIK_CSRF_TRUSTED_ORIGINS`** — add your AWS SSO region endpoint if using AWS Identity Center (e.g. `https://us-east-1.sso.signin.aws`); present in both `server` and `worker`
- **Volume paths** — the compose file uses relative `./data`, `./media`, `./certs`, and `./custom-templates` paths by default; bind-mount alternatives pointing to absolute host paths (e.g. `/opt/authentik/data`) are included as commented lines alongside each volume entry
- **`AUTHENTIK_TAG`** — the image version is pinned to `2026.2.1` and controlled via `.env`; update `AUTHENTIK_TAG` in your `.env` file when upgrading

### Step 2: Create .env file

```bash
cd /opt/authentik
cp env.template .env

# Generate and insert secrets
PG_PASS=$(openssl rand -base64 36 | tr -d '\n')
AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')

sed -i "s/REPLACE_WITH_GENERATED_PASSWORD/${PG_PASS}/" .env
sed -i "s/REPLACE_WITH_GENERATED_SECRET/${AUTHENTIK_SECRET_KEY}/" .env

# Verify
cat .env
```

Or generate and write manually:

```bash
echo "PG_PASS=$(openssl rand -base64 36 | tr -d '\n')" >> .env
echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')" >> .env
```

### Step 3: Configure email (optional)

`env.template` includes pre-commented email configuration blocks for Fastmail, Gmail, Office 365, and SendGrid. Open `.env` and uncomment the block for your provider rather than typing the variables from scratch:

```bash
nano .env
```

Fastmail example:
```
AUTHENTIK_EMAIL__HOST=smtp.fastmail.com
AUTHENTIK_EMAIL__PORT=587
AUTHENTIK_EMAIL__USERNAME=your-email@fastmail.com
AUTHENTIK_EMAIL__PASSWORD=your-app-specific-password
AUTHENTIK_EMAIL__USE_TLS=true
AUTHENTIK_EMAIL__FROM=authentik@yourdomain.com
```

> **Fastmail:** Generate an app-specific password under **Settings → Privacy & Security → Connected Apps & API Tokens**. Port 587 with STARTTLS is recommended. Port 465 with SSL/TLS is also supported — use `AUTHENTIK_EMAIL__USE_SSL=true` and `AUTHENTIK_EMAIL__PORT=465` in that case.

### Step 4: Deploy

```bash
cd /opt/authentik

docker compose pull
docker compose up -d
docker compose ps
docker compose logs -f
```

---

## Configure Nginx Proxy Manager

### Step 1: Add proxy host

1. Log into NPM
2. Go to **Proxy Hosts** → **Add Proxy Host**

### Step 2: Details tab

| Setting | Value |
|---------|-------|
| Domain Names | `auth.yourdomain.com` |
| Scheme | `http` |
| Forward Hostname/IP | your VM's LAN IP |
| Forward Port | `9000` |
| Cache Assets | ✅ |
| Block Common Exploits | ✅ |
| Websockets Support | ✅ |

### Step 3: SSL tab

| Setting | Value |
|---------|-------|
| SSL Certificate | your wildcard certificate |
| Force SSL | ✅ |
| HTTP/2 Support | ✅ |
| HSTS Enabled | ✅ |

### Step 4: Save

Click **Save** and wait 10–15 seconds.

---

## Initial Authentik Setup

1. Navigate to:
   ```
   https://auth.yourdomain.com/if/flow/initial-setup/
   ```
   > ⚠️ The trailing slash is required.

2. Set a password for the `akadmin` user

3. Log in:
   - Username: `akadmin`
   - Password: what you just set

4. Go to **Admin Interface → System → Settings**
   - Set **Authentik Domain:** `https://auth.yourdomain.com`
   - Click **Save**

---

## AWS Identity Center

To use Authentik as a SAML/OIDC identity provider with AWS Identity Center, `AUTHENTIK_CSRF_TRUSTED_ORIGINS` must include your AWS SSO sign-in endpoint. This is required in **both** the `server` and `worker` services:

```yaml
AUTHENTIK_CSRF_TRUSTED_ORIGINS: "https://auth.yourdomain.com,https://us-east-1.sso.signin.aws"
```

Replace `us-east-1` with the AWS region your Identity Center instance is configured in. Without this, SAML and OIDC flows initiated from the AWS console will fail with CSRF verification errors.

After updating, restart both services:

```bash
docker compose restart server worker
```

---

## Verification

### Check container status

**Dockhand:** view container status and health directly in the stack view.

**CLI:**
```bash
docker compose ps
# All 4 containers should show as "Up" and "healthy"
```

### Check logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f server
docker compose logs -f worker
docker compose logs -f postgresql
docker compose logs -f redis
```

### Check port

```bash
# Confirm port 9000 is listening
ss -tlnp | grep 9000

# Quick test
curl -sI http://localhost:9000
# Expect: HTTP/1.1 302 Found (redirect to /if/flow/default-authentication-flow/)
```

---

## Container Management

### Dockhand

Use the Dockhand stack controls to start, stop, restart, and view logs. The exec/console feature provides shell access if needed.

### CLI

```bash
cd /opt/authentik

docker compose stop              # stop all services
docker compose start             # start stopped services
docker compose restart           # restart all services
docker compose down              # stop and remove containers (data preserved)
docker compose down -v           # stop, remove containers AND volumes — deletes data
docker compose pull              # pull latest images
docker compose up -d             # start (or update after pull)
```

Restart a single service:

```bash
docker compose restart server
docker compose restart worker
```

---

## Backup

### Secrets

```bash
cp .env .env.backup
# Store this file securely — it contains your database password and secret key
# Consider encrypting: gpg -c .env.backup
```

### Database

```bash
# Backup
docker exec authentik-postgresql pg_dump -U authentik authentik \
  > authentik-backup-$(date +%Y%m%d).sql

# Restore
cat authentik-backup-YYYYMMDD.sql | \
  docker exec -i authentik-postgresql psql -U authentik -d authentik
```

### Volumes

```bash
# List Authentik volumes
docker volume ls | grep authentik

# Backup media volume
docker run --rm \
  -v authentik-media:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/authentik-media-$(date +%Y%m%d).tar.gz -C /data .

# Restore media volume
docker run --rm \
  -v authentik-media:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/authentik-media-YYYYMMDD.tar.gz -C /data
```

---

## Troubleshooting

### 502 Bad Gateway from NPM

1. Confirm containers are running: `docker compose ps`
2. Check port 9000 is listening: `ss -tlnp | grep 9000`
3. Verify the correct VM IP is set in NPM
4. Check firewall — NPM must be able to reach the VM on port 9000
5. Check NPM logs — connection refused vs timeout point to different root causes

### AWS / CSRF errors

- `AUTHENTIK_CSRF_TRUSTED_ORIGINS` must be set in **both** `server` and `worker`
- The value must include the exact `https://` origin — no trailing slashes
- Check your Identity Center region matches the URL: `https://<region>.sso.signin.aws`
- After any change: `docker compose restart server worker`

### Containers not starting

```bash
docker compose logs postgresql
docker compose logs redis
docker compose logs server
```

Common causes:
- Missing or malformed `PG_PASS` or `AUTHENTIK_SECRET_KEY`
- Port 9000 or 9443 already in use on the host
- Redis not healthy before server/worker attempt to start

### Environment variables not applied

```bash
docker exec authentik-server env | grep AUTHENTIK
docker exec authentik-server env | grep PG
```

If variables are missing, confirm `.env` exists in the same directory as `docker-compose.yml` and re-run `docker compose up -d`.

### Can't reach initial setup page

- URL must have trailing slash: `/if/flow/initial-setup/`
- Wait at least 60 seconds after first deploy — Authentik takes time to fully initialize
- Check for errors: `docker compose logs -f server`

### Email not sending (Fastmail)

1. Confirm you are using an **app-specific password**, not your Fastmail account password
2. `AUTHENTIK_EMAIL__USERNAME` must be your full Fastmail address (`you@fastmail.com`)
3. Port 587 + `USE_TLS=true` (STARTTLS) or port 465 + `USE_SSL=true` (SSL/TLS)
4. Send a test email from **Admin Interface → System → Email**

---

## Security Recommendations

- Never commit `.env` to version control — add `.env` to [`.gitignore`](https://github.com/variablenix/dotfiles/blob/main/common/.gitignore_global)
- Use strong randomly generated values for `PG_PASS` and `AUTHENTIK_SECRET_KEY`
- Serve Authentik behind HTTPS only (NPM + valid certificate)
- Enable HSTS in NPM
- Restrict VM access — only NPM and trusted internal hosts should reach port 9000
- Back up `.env` and the database regularly
- Pin `AUTHENTIK_TAG` in `.env` and update intentionally — don't use `latest`

---

## Next Steps

1. **Applications** — configure SAML or OIDC apps for SSO
2. **Users & Groups** — build your user directory
3. **Flows** — customize authentication, enrollment, and recovery flows
4. **Sources** — connect LDAP, Google, GitHub, or other OAuth/SAML sources
5. **Outposts** — deploy the forward auth proxy or LDAP outpost for non-SAML apps
6. **AWS Identity Center** — set up SAML federation with the correct `CSRF_TRUSTED_ORIGINS` region
7. **Test email** — send a test from **Admin Interface → System → Email**

---

## Additional Resources

- [Authentik Documentation](https://docs.goauthentik.io/)
- [Authentik Integrations](https://integrations.goauthentik.io/)
- [Authentik GitHub](https://github.com/goauthentik/authentik)
- [Authentik Discord](https://discord.gg/jg33eMhnj6)
- [Dockhand](https://github.com/Finsys/dockhand)
- [Nginx Proxy Manager](https://nginxproxymanager.com/)
