---
name: homelab-nginx
description: Use when managing Nginx Proxy Manager on the homelab — proxy hosts, SSL certificates, Let's Encrypt renewals, adding new services behind the proxy, diagnosing 502 errors, or any nginx/reverse-proxy operation
---

## Service Overview

Nginx Proxy Manager (NPM) is the reverse proxy gateway for all homelab services. Every external HTTPS request to `*.dschmidt95.de` and `*.home.dschmidt95.de` comes through NPM. It handles SSL termination (Let's Encrypt via Cloudflare DNS), HTTP→HTTPS redirects, and routes traffic to backend containers or LAN hosts by name or IP.

- **Purpose:** TLS termination + reverse proxy for all homelab services
- **Backend connectivity:** Uses `proxy_network` Docker bridge — containers on that network are reachable by container name (no IPs needed)
- **SSL:** Let's Encrypt certificates, auto-renewed by Certbot with Cloudflare DNS-01 challenge
- **Database:** MariaDB (`ngnix-db-1`) stores proxy host config, users, certificates metadata
- **Admin UI:** `http://192.168.178.26:82`

---

## Infrastructure

| Property | Value |
|----------|-------|
| SSH alias | `docker` |
| App container | `ngnix-app-1` |
| DB container | `ngnix-db-1` |
| Image | `jc21/nginx-proxy-manager:latest` |
| Admin UI | `http://192.168.178.26:82` |
| HTTP port | `80` (public) |
| HTTPS port | `443` (public) |
| Data dir | `/data/compose/14/data/` |
| SSL/certs dir | `/data/compose/14/letsencrypt/` |
| MariaDB dir | `/data/compose/14/mysql/` |
| Nginx configs | `/data/compose/14/data/nginx/proxy_host/` |
| Log dir | `/data/compose/14/data/logs/` |

### Networks

| Network | Subnet | Role |
|---------|--------|------|
| `ngnix_default` | `172.20.0.0/16` | Internal: app↔db only |
| `proxy_network` | `172.23.0.0/16` | Shared with all proxied containers |

**Key:** NPM routes to Docker containers by container name because both NPM (`ngnix-app-1`) and the target containers are on `proxy_network`. No IP addresses needed for Docker-hosted services.

---

## Proxy Host Inventory

All currently configured proxy hosts. Config files live at `/data/compose/14/data/nginx/proxy_host/<id>.conf`.

| Config | Domain | Backend | Port | Scheme | Notes |
|--------|--------|---------|------|--------|-------|
| `4.conf` | `ha.dschmidt95.de` | `192.168.178.171` | `8123` | http | Home Assistant (LAN) |
| `5.conf` | `bw.dschmidt95.de` | `bitwarden-bitwarden-1` | `8080` | http | Vaultwarden |
| `6.conf` | `zm.dschmidt95.de` | `192.168.178.11` | `80` | http | ZoneMinder (LAN, public) |
| `14.conf` | `adguard.home.dschmidt95.de` | `adguardhome` | `80` | http | AdGuard Home |
| `15.conf` | `beszel.dschmidt95.de` | `beszel` | `8090` | http | Beszel monitoring |
| `18.conf` | `ngnix.home.dschmidt95.de` | `192.168.178.26` | `82` | http | NPM admin UI itself |
| `19.conf` | `pve.home.dschmidt95.de` | `192.168.178.242` | `8006` | https | Proxmox VE (LAN) |
| `20.conf` | `docker.home.dschmidt95.de` | `192.168.178.26` | `9443` | https | Portainer UI |
| `21.conf` | `zm.home.dschmidt95.de` | `192.168.178.11` | `80` | http | ZoneMinder (LAN, home) |
| `22.conf` | `n8n.home.dschmidt95.de` | `n8n` | `5678` | http | n8n automation |
| `24.conf` | `paperless.home.dschmidt95.de` | `paperless-ngx` | `8000` | http | Paperless-ngx |

### proxy_network Container IPs (for reference)

| Container | IP |
|-----------|----|
| `beszel` | `172.23.0.2` |
| `n8n` | `172.23.0.3` |
| `ngnix-app-1` | `172.23.0.4` |
| `paperless-ngx` | `172.23.0.5` |
| `adguardhome` | `172.23.0.6` |
| `bitwarden-bitwarden-1` | `172.23.0.7` |

---

## Common Operations

### Check NPM Container Status
```bash
ssh docker "docker ps --filter name=ngnix --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'"
```

### View NPM Application Logs
```bash
# Last 50 lines
ssh docker "docker logs ngnix-app-1 --tail 50"

# Follow live (shows SSL renewals, reloads, errors)
ssh docker "docker logs ngnix-app-1 -f"

# Last 30 minutes
ssh docker "docker logs ngnix-app-1 --since 30m"
```

### Restart NPM (app only, no DB downtime)
```bash
ssh docker "docker restart ngnix-app-1"
# Verify it's back healthy:
ssh docker "docker ps --filter name=ngnix-app-1"
```

### Restart Full Stack (app + DB)
```bash
ssh docker "docker restart ngnix-db-1 && sleep 5 && docker restart ngnix-app-1"
ssh docker "docker ps --filter name=ngnix"
```

### Force Nginx Config Reload (without restart)
NPM auto-reloads nginx when you save changes in the UI. To force reload manually:
```bash
ssh docker "docker exec ngnix-app-1 nginx -s reload"
```

### Test Nginx Config Validity
```bash
ssh docker "docker exec ngnix-app-1 nginx -t"
```

### Open Admin UI
Navigate to: `http://192.168.178.26:82`

---

## Adding a New Proxy Host

### Rule: Which backend type to use?

```
Is the service a Docker container?
  Yes → Is it on proxy_network?
    Yes → Use container name as backend (e.g. "my-container")
    No  → Add it to proxy_network first (see below), then use container name
  No  → Use LAN IP as backend (e.g. "192.168.178.X")
```

### Path A: New Docker container (join proxy_network first)

1. Add `proxy_network` to the container's compose file:
   ```yaml
   networks:
     proxy_network:
       external: true

   services:
     my-service:
       networks:
         - proxy_network
   ```
2. Redeploy the stack via Portainer or `docker compose up -d`
3. Verify the container joined the network:
   ```bash
   ssh docker "docker network inspect proxy_network --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{println}}{{end}}'"
   ```
4. In NPM admin UI (`http://192.168.178.26:82`):
   - Proxy Hosts → Add Proxy Host
   - Domain: `service.home.dschmidt95.de`
   - Forward Hostname: `<container-name>` (no IP needed)
   - Forward Port: the container's internal port
   - Enable "Block Common Exploits", "Websockets Support" if needed
   - SSL tab → Request a new SSL certificate → Let's Encrypt → enable "Force SSL"

### Path B: LAN device (IP-based backend)

1. In NPM admin UI:
   - Forward Hostname: `192.168.178.X`
   - Forward Port: the service port
   - If the backend uses self-signed HTTPS (e.g. Proxmox, Portainer): set Forward Scheme to `https`, enable "Ignore invalid certificate"

---

## SSL / Let's Encrypt Operations

### Check Certificate Expiry for All Hosts
```bash
ssh docker "ls /data/compose/14/letsencrypt/live/ && for cert in /data/compose/14/letsencrypt/live/*/cert.pem; do echo \"=== \$cert ===\"; openssl x509 -noout -enddate -in \"\$cert\"; done"
```

### View Certbot Logs (SSL renewal activity)
```bash
ssh docker "cat /data/compose/14/data/logs/letsencrypt.log | tail -50"
```

### Force SSL Renewal (all certs due within 30 days)
NPM auto-renews, but to trigger manually:
```bash
ssh docker "docker exec ngnix-app-1 certbot renew --force-renewal"
```

### Check Which Certs Exist
```bash
ssh docker "ls /data/compose/14/letsencrypt/live/"
```

Each NPM certificate is stored as `npm-<id>/` (e.g. `npm-58/`). The ID matches the certificate record in the NPM UI (SSL Certificates tab).

---

## Log Reference

All logs are in `/data/compose/14/data/logs/` on the Docker host.

| Log file | Contents |
|----------|---------|
| `<domain>_access.log` | HTTP access log per proxy host |
| `<domain>_error.log` | Nginx errors per proxy host |
| `default-host_access.log` | Requests hitting the default catch-all |
| `fallback_error.log` | Errors from the fallback server block |
| `fallback_http_access.log` | HTTP (port 80) access before redirect |
| `letsencrypt.log` | Certbot certificate renewal activity |

### Read access log for a proxy host
```bash
# List available logs
ssh docker "ls /data/compose/14/data/logs/"

# Read last 50 lines of a specific proxy's access log
ssh docker "tail -50 /data/compose/14/data/logs/adguard.home.dschmidt95.de_access.log"

# Follow live
ssh docker "tail -f /data/compose/14/data/logs/n8n.home.dschmidt95.de_access.log"

# Check for errors
ssh docker "tail -50 /data/compose/14/data/logs/fallback_error.log"
```

---

## Config File Reference

NPM generates nginx server block configs automatically. Each proxy host has its own file:

```
/data/compose/14/data/nginx/proxy_host/<id>.conf
```

### Read a proxy host's generated nginx config
```bash
ssh docker "cat /data/compose/14/data/nginx/proxy_host/22.conf"
```

### Key variables in each config
```nginx
set $forward_scheme http;          # http or https to backend
set $server         "n8n";         # backend hostname or IP
set $port           5678;          # backend port
server_name n8n.home.dschmidt95.de; # public domain
ssl_certificate /etc/letsencrypt/live/npm-XX/fullchain.pem;
```

**Never edit these files directly** — NPM overwrites them. Make changes in the admin UI.

---

## Troubleshooting Playbook

### 502 Bad Gateway

NPM reached the backend hostname/IP but got no response.

1. Identify which proxy host is failing (check domain in browser, find its config):
   ```bash
   ssh docker "grep -l 'server_name failing.domain.de' /data/compose/14/data/nginx/proxy_host/*.conf"
   ```
2. Check what backend it's pointing to:
   ```bash
   ssh docker "grep 'set \$server\|set \$port\|set \$forward_scheme' /data/compose/14/data/nginx/proxy_host/<id>.conf"
   ```
3. If backend is a Docker container — is it running?
   ```bash
   ssh docker "docker ps --filter name=<container-name>"
   ```
4. If backend is a Docker container — is it on `proxy_network`?
   ```bash
   ssh docker "docker inspect <container-name> --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool | grep proxy_network"
   ```
5. Test connectivity from inside NPM to the backend:
   ```bash
   ssh docker "docker exec ngnix-app-1 wget -qO- http://<container-name>:<port>/ 2>&1 | head -5"
   ```
6. If backend is a LAN IP — is the device reachable?
   ```bash
   ssh docker "ping -c 3 192.168.178.X"
   ssh docker "curl -sk https://192.168.178.X:PORT/ -o /dev/null -w '%{http_code}'"
   ```

### 504 Gateway Timeout

NPM connected but the backend is too slow to respond.

1. Check backend container health and resource usage:
   ```bash
   ssh docker "docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}' | grep <container-name>"
   ```
2. Increase proxy timeout in NPM admin UI:
   - Edit the proxy host → Advanced tab → add:
     ```
     proxy_read_timeout 300s;
     proxy_connect_timeout 300s;
     ```

### SSL Certificate Error / Expired

1. Check cert expiry:
   ```bash
   ssh docker "openssl x509 -noout -enddate -in /data/compose/14/letsencrypt/live/npm-<id>/cert.pem"
   ```
2. Check renewal logs:
   ```bash
   ssh docker "tail -100 /data/compose/14/data/logs/letsencrypt.log"
   ```
3. Force renewal:
   ```bash
   ssh docker "docker exec ngnix-app-1 certbot renew --force-renewal"
   ```
4. Reload nginx after renewal:
   ```bash
   ssh docker "docker exec ngnix-app-1 nginx -s reload"
   ```

### NPM Won't Start

1. Check logs:
   ```bash
   ssh docker "docker logs ngnix-app-1 --tail 100"
   ```
2. Check if DB is up (NPM requires DB on startup):
   ```bash
   ssh docker "docker ps --filter name=ngnix-db-1"
   ```
3. If DB is down, start DB first then NPM:
   ```bash
   ssh docker "docker start ngnix-db-1 && sleep 5 && docker start ngnix-app-1"
   ssh docker "docker logs ngnix-app-1 --tail 20"
   ```
4. Check for port conflicts (80/443 must be free):
   ```bash
   ssh docker "sudo ss -tlnp | grep -E ':80|:443'"
   ```

### Config Change Not Taking Effect

NPM applies changes immediately when saved in the UI (reloads nginx). If changes aren't showing:
1. Test nginx config validity:
   ```bash
   ssh docker "docker exec ngnix-app-1 nginx -t"
   ```
2. Check NPM logs for errors:
   ```bash
   ssh docker "docker logs ngnix-app-1 --tail 30"
   ```
3. Force reload:
   ```bash
   ssh docker "docker exec ngnix-app-1 nginx -s reload"
   ```

### New Container Not Reachable by Name

If NPM returns 502 when using a container name as backend:
```bash
# Confirm the container is on proxy_network
ssh docker "docker network inspect proxy_network --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{println}}{{end}}' | grep <container-name>"

# If missing — connect it manually (temporary, fix compose file too)
ssh docker "docker network connect proxy_network <container-name>"

# Verify DNS resolution from NPM
ssh docker "docker exec ngnix-app-1 nslookup <container-name>"
```

---

## Update NPM

```bash
# Step 1: Pull new image
ssh docker "docker pull jc21/nginx-proxy-manager:latest"

# Step 2: Recreate app container (DB is not updated this way)
ssh docker "docker stop ngnix-app-1 && docker rm ngnix-app-1"
# Then redeploy via Portainer: Stacks → ngnix → Update the stack

# Step 3: Verify
ssh docker "docker logs ngnix-app-1 --tail 30"
ssh docker "docker ps --filter name=ngnix"
```

---

## Related Services

- **Docker / Portainer** (`skills/docker.md`): Container management, stack redeployment, proxy_network inspection
- **AdGuard Home** (`skills/adguard.md`): DNS rewrites that point `*.home.dschmidt95.de` → `192.168.178.26` (required for NPM to receive requests)
- **Cloudflare DDNS** (`cloudflare-ddns` container): Keeps public DNS A records current for external access
- **All proxied services:** Must be on `proxy_network` to be reachable by container name from NPM
