# authentik-setup

A self-hosted [Authentik](https://goauthentik.io/) deployment guide for Docker Compose and [Dockhand](https://github.com/fnsys/dockhand), with Nginx Proxy Manager reverse proxy, Fastmail SMTP, and AWS Identity Center compatibility.

---

## Repository Contents

| File | Purpose |
|------|---------|
| [docker-compose.yml](docker-compose.yml) | Production-ready compose file with Redis, resource limits, Prometheus metrics, and AWS Identity Center CSRF origins |
| [env.template](env.template) | Environment variable template — copy to `.env` and fill in required secrets; includes pre-commented email examples for Fastmail, Gmail, Office 365, and SendGrid |
| [SETUP.md](SETUP.md) | Full step-by-step deployment guide |

---

## What's Included

- **PostgreSQL 16** + **Redis** — Authentik's required backing services
- **Named volumes** and bind-mount alternatives for persistent data
- **Resource limits** on all services (memory reservations + caps)
- **Prometheus metrics** — server on `:9300`, worker on `:9301`
- **CSRF trusted origins** pre-configured for AWS Identity Center compatibility
- **Email configuration** examples — Fastmail, Gmail, Office 365, SendGrid
- Dual deployment methods: **Dockhand GUI** and **Docker Compose CLI**

---

## Quick Start

### 1. Get the files

Clone this repo or download the files directly:

```bash
git clone https://github.com/variablenix/authentik-setup.git
cd authentik-setup
```

Or download the upstream Authentik compose file and use this repo's version as a reference:

```bash
wget https://goauthentik.io/docker-compose.yml
```

### 2. Generate secrets

```bash
# PostgreSQL password
echo "PG_PASS=$(openssl rand -base64 36 | tr -d '\n')"

# Authentik secret key
echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')"
```

### 3. Create your .env file

```bash
cp env.template .env
# Edit .env and paste in the generated values
nano .env
```

### 4. Deploy

```bash
docker compose pull
docker compose up -d
docker compose ps
```

### 5. Initial setup

Navigate to:

```
https://auth.yourdomain.com/if/flow/initial-setup/
```

> ⚠️ The trailing slash is required.

Set a password for `akadmin`, then go to **Admin Interface → System → Settings** and set the **Authentik Domain** to your full URL.

---

## AWS Identity Center

To use Authentik as a SAML/OIDC identity provider with AWS Identity Center, `AUTHENTIK_CSRF_TRUSTED_ORIGINS` must include your AWS SSO sign-in endpoint. This is set in both the `server` and `worker` services in `docker-compose.yml`:

```yaml
AUTHENTIK_CSRF_TRUSTED_ORIGINS: "https://auth.yourdomain.com,https://us-east-1.sso.signin.aws"
```

Replace `us-east-1` with the AWS region your Identity Center instance is configured in. Without this entry, SAML and OIDC flows initiated from the AWS console will fail with CSRF verification errors.

---

## Nginx Proxy Manager

Point NPM at the Authentik server container:

| Setting | Value |
|---------|-------|
| Scheme | `http` |
| Forward Hostname/IP | your VM's LAN IP |
| Forward Port | `9000` |
| Websockets Support | ✅ enabled |
| Force SSL | ✅ enabled |
| HTTP/2 Support | ✅ enabled |
| HSTS | ✅ enabled |

---

## Prometheus Metrics

The compose file exposes Authentik metrics on two ports for scraping:

| Port | Service | Metric path |
|------|---------|-------------|
| `9300` | server | `/metrics` |
| `9301` | worker | `/metrics` |

Add both to your Prometheus scrape config to get full coverage of Authentik's internals.

---

## Backup

### Secrets

```bash
cp .env .env.backup
# Store this securely — it contains your database password and secret key
```

### Database

```bash
# Backup
docker exec authentik-postgresql pg_dump -U authentik authentik > authentik-backup-$(date +%Y%m%d).sql

# Restore
cat authentik-backup-YYYYMMDD.sql | docker exec -i authentik-postgresql psql -U authentik -d authentik
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
```

---

## Troubleshooting

### 502 Bad Gateway from NPM
1. Confirm containers are running: `docker compose ps`
2. Check port 9000 is listening: `ss -tlnp | grep 9000`
3. Verify the VM IP in NPM is correct
4. Check NPM logs for connection refused vs timeout — they point to different problems

### Broadcaster / AWS CSRF errors
- Ensure `AUTHENTIK_CSRF_TRUSTED_ORIGINS` is set in **both** `server` and `worker`
- The value must match the exact origin URL including scheme (`https://`)
- Restart both containers after any change: `docker compose restart server worker`

### Containers not starting
```bash
docker compose logs postgresql
docker compose logs server
```
Common causes: missing `PG_PASS` or `AUTHENTIK_SECRET_KEY`, port 9000/9443 already in use, Redis not healthy before server starts.

### Environment variables not applied
```bash
docker exec authentik-server env | grep AUTHENTIK
docker exec authentik-server env | grep PG
```

### Can't reach initial setup page
- Confirm URL has trailing slash: `/if/flow/initial-setup/`
- Wait 60+ seconds after first start — Authentik (PHP-FPM + Liquidsoap workers) takes time to initialize
- Check: `docker compose logs -f server`

### Email not sending
- Fastmail requires an **app-specific password** — not your account password
- Generate at: **Settings → Privacy & Security → Connected Apps & API Tokens**
- Port 587 + `USE_TLS=true` (STARTTLS) or port 465 + `USE_SSL=true` (SSL/TLS)
- Test under **Admin Interface → System → Email**

---

## Security

- Never commit `.env` to version control — add it to `.gitignore`
- Use a strong randomly generated `PG_PASS` and `AUTHENTIK_SECRET_KEY`
- Serve Authentik behind HTTPS only (NPM + valid SSL cert)
- Enable HSTS in NPM
- Back up `.env` and the database on a schedule
- Keep the Authentik image version pinned in `.env` (`AUTHENTIK_TAG`) and update intentionally

---

## Next Steps

1. **Applications** — add SAML/OIDC apps for SSO
2. **Users & Groups** — build your directory
3. **Flows** — customize enrollment, authentication, and recovery flows
4. **Sources** — connect LDAP, Google, GitHub, or other OAuth providers
5. **Outposts** — deploy the proxy or LDAP outpost for non-SAML apps
6. **AWS Identity Center** — configure SAML federation with the correct `CSRF_TRUSTED_ORIGINS` region

---

## Resources

- [Authentik Documentation](https://docs.goauthentik.io/)
- [Authentik Integrations](https://integrations.goauthentik.io/)
- [Authentik GitHub](https://github.com/goauthentik/authentik)
- [Authentik Discord](https://discord.gg/jg33eMhnj6)
- [Dockhand](https://github.com/fnsys/dockhand)
