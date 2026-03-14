# Authentik Setup Guide
## Cross-Compatible: Portainer & Docker Compose CLI

This guide covers deploying Authentik using either Portainer or docker-compose CLI with the same files.

---

## Key Improvements in This Setup

✅ **Named volumes** instead of relative paths (./media → authentik-media)
✅ **Works in both Portainer and CLI** without modifications
✅ **Container names** for easier management
✅ **Default values** for optional variables
✅ **Clear environment variable structure**
✅ **Email configuration included**

---

## Prerequisites

- Ubuntu 24.04 VM with Docker installed
- Portainer (if using GUI method) OR terminal access for CLI method
- Nginx Proxy Manager with wildcard SSL certificate
- Chosen subdomain (e.g., auth.yourdomain.com)

---

## Method 1: Deploy with Portainer (GUI)

### Step 1: Prepare Environment Variables

On your Ubuntu VM, generate the required secrets:

```bash
# Generate PostgreSQL password
echo "PG_PASS=$(openssl rand -base64 36 | tr -d '\n')"

# Generate Authentik secret key
echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')"
```

**Copy these values** - you'll need them in Portainer.

### Step 2: Create Stack in Portainer

1. Log into Portainer
2. Go to **Stacks** → **Add stack**
3. Name: `authentik`
4. Choose **Web editor**
5. Paste the entire `docker-compose.yml` content

### Step 3: Add Environment Variables

Scroll to **Environment variables** section and add these:

**Required Variables:**
```
PG_PASS=<paste-your-generated-password>
AUTHENTIK_SECRET_KEY=<paste-your-generated-secret>
```

**Optional Variables:**
```
AUTHENTIK_ERROR_REPORTING__ENABLED=true
PG_USER=authentik
PG_DB=authentik
```

**Optional Email Variables (if configuring email):**
```
AUTHENTIK_EMAIL__HOST=smtp.gmail.com
AUTHENTIK_EMAIL__PORT=587
AUTHENTIK_EMAIL__USERNAME=your-email@gmail.com
AUTHENTIK_EMAIL__PASSWORD=your-app-password
AUTHENTIK_EMAIL__USE_TLS=true
AUTHENTIK_EMAIL__FROM=authentik@yourdomain.com
```

### Step 4: Deploy

1. Click **Deploy the stack**
2. Wait 2-3 minutes for containers to start
3. Check container logs for any errors
4. Verify all 4 containers are running and healthy

---

## Method 2: Deploy with Docker Compose CLI

### Step 1: Prepare Files

```bash
# Create directory
mkdir -p /opt/authentik
cd /opt/authentik

# Download or copy the docker-compose.yml file here
# Then create your .env file
```

### Step 2: Create .env File

```bash
cd /opt/authentik

# Generate and save secrets
echo "PG_PASS=$(openssl rand -base64 36 | tr -d '\n')" > .env
echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')" >> .env

# Add optional settings
echo "AUTHENTIK_ERROR_REPORTING__ENABLED=true" >> .env
echo "PG_USER=authentik" >> .env
echo "PG_DB=authentik" >> .env

# Verify .env file
cat .env
```

### Step 3: Optional Email Configuration

If you want email functionality, add to your .env:

```bash
nano .env
```

Add these lines (adjust for your provider):
```
AUTHENTIK_EMAIL__HOST=smtp.gmail.com
AUTHENTIK_EMAIL__PORT=587
AUTHENTIK_EMAIL__USERNAME=your-email@gmail.com
AUTHENTIK_EMAIL__PASSWORD=your-app-password
AUTHENTIK_EMAIL__USE_TLS=true
AUTHENTIK_EMAIL__FROM=authentik@yourdomain.com
```

### Step 4: Deploy

```bash
cd /opt/authentik

# Pull images
docker-compose pull

# Start in background
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f
```

---

## Configure Nginx Proxy Manager (Both Methods)

### Step 1: Add Proxy Host

1. Log into NPM
2. Go to **Proxy Hosts** → **Add Proxy Host**

### Step 2: Details Tab

- **Domain Names:** `auth.yourdomain.com`
- **Scheme:** `http`
- **Forward Hostname/IP:** `<your-ubuntu-vm-ip>`
- **Forward Port:** `9000`
- ✅ **Cache Assets**
- ✅ **Block Common Exploits**
- ✅ **Websockets Support**

### Step 3: SSL Tab

- **SSL Certificate:** Select your wildcard certificate
- ✅ **Force SSL**
- ✅ **HTTP/2 Support**
- ✅ **HSTS Enabled**

### Step 4: Save

Click **Save** and wait 10-15 seconds.

---

## Initial Authentik Setup

1. Navigate to: `https://auth.yourdomain.com/if/flow/initial-setup/`
   
   ⚠️ **Important:** Include the trailing slash `/`

2. Set password for the `akadmin` user

3. Log in with:
   - Username: `akadmin`
   - Password: what you just set

4. Go to **Admin Interface** → **System** → **Settings**
   - Set **Authentik Domain:** `https://auth.yourdomain.com`
   - Click **Save**

---

## Verification Commands

### Check Container Status

**Portainer Method:**
- View in Portainer's Containers page
- Check logs in Portainer UI

**CLI Method:**
```bash
cd /opt/authentik

# Check status
docker-compose ps

# Should show all 4 containers as "Up" and "healthy"
```

### Check Logs

```bash
# View all logs
docker-compose logs

# Follow logs in real-time
docker-compose logs -f

# Check specific service
docker-compose logs server
docker-compose logs postgresql
```

### Check Port

```bash
# Verify port 9000 is listening
sudo ss -tlnp | grep 9000

# Test locally
curl http://localhost:9000
```

---

## Container Management

### Using Portainer:
- Start/Stop/Restart: Use Portainer UI buttons
- View logs: Click container → Logs
- Console access: Click container → Console

### Using CLI:
```bash
cd /opt/authentik

# Stop all services
docker-compose stop

# Start all services
docker-compose start

# Restart all services
docker-compose restart

# Stop and remove containers (keeps data)
docker-compose down

# Stop, remove containers AND volumes (deletes data)
docker-compose down -v

# Update to latest version
docker-compose pull
docker-compose up -d
```

---

## Backup Important Data

### Backup Secrets

```bash
# Your .env file contains critical secrets
sudo cp /opt/authentik/.env /opt/authentik/.env.backup

# Store securely - consider encrypting
```

### Backup Database

```bash
# Backup PostgreSQL database
docker exec authentik-postgresql pg_dump -U authentik authentik > authentik-backup-$(date +%Y%m%d).sql

# Restore if needed
cat authentik-backup-YYYYMMDD.sql | docker exec -i authentik-postgresql psql -U authentik -d authentik
```

### Backup Volumes

```bash
# List volumes
docker volume ls | grep authentik

# Backup a volume (example)
docker run --rm -v authentik-media:/data -v $(pwd):/backup alpine tar czf /backup/authentik-media-backup.tar.gz -C /data .
```

---

## Troubleshooting

### 502 Bad Gateway from NPM

**Check:**
1. Containers are running: `docker-compose ps`
2. Port 9000 is listening: `sudo ss -tlnp | grep 9000`
3. Correct VM IP in NPM
4. Firewall allows traffic from NPM to VM

### Containers Not Starting

**Check logs:**
```bash
docker-compose logs postgresql
docker-compose logs server
```

**Common issues:**
- Missing environment variables
- Invalid PG_PASS or AUTHENTIK_SECRET_KEY
- Port conflicts (9000 or 9443 already in use)

### Environment Variables Not Working

**Verify they're set:**
```bash
# Check in running container
docker exec authentik-server env | grep AUTHENTIK
docker exec authentik-server env | grep PG_PASS
```

### Can't Access Initial Setup

**Make sure:**
1. URL includes trailing slash: `/if/flow/initial-setup/`
2. Containers are healthy: `docker-compose ps`
3. No errors in logs: `docker-compose logs server`

---

## Security Recommendations

1. ✅ **Backup your .env file** securely
2. ✅ **Use strong passwords** for PG_PASS
3. ✅ **Never commit .env to git** - add to .gitignore
4. ✅ **Limit access** to the VM
5. ✅ **Use HTTPS** (via NPM) for all access
6. ✅ **Enable HSTS** in NPM
7. ✅ **Regular backups** of database and config
8. ✅ **Keep Authentik updated** regularly

---

## Next Steps

After Authentik is running:

1. **Configure Applications** - Add apps for SSO/SAML
2. **Set up Users & Groups** - Create your user directory
3. **Configure Flows** - Customize authentication flows
4. **Add Sources** - Connect LDAP, OAuth providers, etc.
5. **Set up Outposts** - For proxy/LDAP functionality
6. **Test Email** - Verify email configuration works

---

## Additional Resources

- Official Documentation: https://docs.goauthentik.io/
- Integrations: https://integrations.goauthentik.io/
- Community Discord: https://discord.gg/jg33eMhnj6
- GitHub: https://github.com/goauthentik/authentik

---

## Support

If you encounter issues:

1. Check logs: `docker-compose logs -f`
2. Review this troubleshooting section
3. Search Authentik documentation
4. Ask in Authentik Discord community
5. Check GitHub issues

---

**Setup Complete!** 🎉

Your Authentik instance should now be accessible at `https://auth.yourdomain.com`
