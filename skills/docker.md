---
name: homelab-docker
description: Use when managing Docker containers, stacks, logs, restarts, image updates, or inspecting any service running on the homelab Docker host
---

## Service Overview

All homelab services run as Docker containers on a single host, managed via Portainer (EE). Stacks are defined in Portainer and use Docker Compose format. The host is accessed via SSH alias `docker`.

- **Host IP:** `192.168.178.26`
- **SSH alias:** `docker` (`dominik@192.168.178.26`, key `~/.ssh/id_ed25519`)
- **Portainer UI:** `https://192.168.178.26:9443`
- **Compose files:** Managed through Portainer UI (stacks)
- **Config/data paths:** Under `/home/dominik/docker/<stack>/` for bind mounts

---

## Infrastructure

| Property | Value |
|----------|-------|
| Docker host | `192.168.178.26` |
| SSH user | `dominik` |
| SSH key | `~/.ssh/id_ed25519` |
| SSH alias | `docker` |
| Portainer | `https://192.168.178.26:9443` (Portainer EE) |
| Docker data | `/var/lib/docker/` |
| User compose/config | `/home/dominik/docker/<stack>/` |

---

## Running Stacks

| Stack | Containers | Networks | Purpose |
|-------|-----------|---------|---------|
| `adguard` | adguardhome, unbound | adguard_unbound_dns-network, proxy_network | DNS (AdGuard + Unbound) |
| `paperlessngx` | paperless-ngx, paperless-db, paperless-redis | paperless_internal | Document management |
| `n8n` | n8n | proxy_network | Workflow automation |
| `bitwarden` | bitwarden-bitwarden-1, bitwarden-db-1 | bitwarden_default | Password manager (Vaultwarden) |
| `portainer` | portainer | bridge | Docker management UI |
| `ngnix` | ngnix-app-1, ngnix-db-1 | ngnix_default, proxy_network | Nginx Proxy Manager |
| `beszel` | beszel, beszel-agent | proxy_network, host | Server monitoring |
| `cloudflare-ddns` | cloudflare-ddns | host | Cloudflare Dynamic DNS |

---

## Common Operations

### List All Running Containers
```bash
ssh docker "docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'"
```

### List All Containers (including stopped)
```bash
ssh docker "docker ps -a --format 'table {{.Names}}\t{{.Status}}'"
```

### View Container Logs
```bash
# Last 50 lines
ssh docker "docker logs <container-name> --tail 50"

# Follow live
ssh docker "docker logs <container-name> -f"

# With timestamps
ssh docker "docker logs <container-name> --tail 50 -t"
```

Replace `<container-name>` with: `adguardhome`, `unbound`, `paperless-ngx`, `n8n`, `portainer`, `ngnix-app-1`, `bitwarden-bitwarden-1`, `beszel`, `beszel-agent`, `cloudflare-ddns`

### Restart a Container
```bash
ssh docker "docker restart <container-name>"
# Then verify it's back up:
ssh docker "docker ps --filter name=<container-name>"
```

### Stop / Start a Container
```bash
ssh docker "docker stop <container-name>"
ssh docker "docker start <container-name>"
```

### Pull Latest Image for a Container
```bash
# Pull new image
ssh docker "docker pull <image-name>"
# Then recreate the container (preferably via Portainer → Stack → Update)
```

### Update a Stack via Portainer (preferred)
1. Go to `https://192.168.178.26:9443`
2. Stacks → select stack → Update the stack (pulls new images and recreates containers)

### Inspect a Container
```bash
# Full JSON inspect
ssh docker "docker inspect <container-name>"

# Just IP addresses and networks
ssh docker "docker inspect <container-name> --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool"

# Environment variables
ssh docker "docker inspect <container-name> --format '{{json .Config.Env}}' | python3 -m json.tool"

# Mounts/volumes
ssh docker "docker inspect <container-name> --format '{{json .Mounts}}' | python3 -m json.tool"
```

### Run a Command Inside a Container
```bash
ssh docker "docker exec <container-name> <command>"

# Interactive shell
ssh docker "docker exec -it <container-name> sh"
# or bash if available:
ssh docker "docker exec -it <container-name> bash"
```

### Check Disk Usage
```bash
# Docker overall disk usage
ssh docker "docker system df"

# Per-volume usage
ssh docker "docker system df -v"
```

### List Docker Networks
```bash
ssh docker "docker network ls"
```

### Check Host System Resources
```bash
# CPU and memory
ssh docker "top -bn1 | head -20"

# Disk space
ssh docker "df -h"

# Docker stats (CPU/memory per container)
ssh docker "docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}'"
```

---

## Troubleshooting Playbook

### Container Keeps Restarting

1. Check restart count and status:
   ```bash
   ssh docker "docker ps -a --filter name=<container-name>"
   ```
2. Read logs for the crash reason:
   ```bash
   ssh docker "docker logs <container-name> --tail 100"
   ```
3. Inspect the exit code:
   ```bash
   ssh docker "docker inspect <container-name> --format '{{.State.ExitCode}} {{.State.Error}}'"
   ```
4. Fix the underlying cause (missing volume, bad config, port conflict)
5. Restart after fix:
   ```bash
   ssh docker "docker restart <container-name>"
   ```

### Container Won't Start (port conflict)

1. Check what's using the port:
   ```bash
   ssh docker "sudo ss -tlnp | grep :<port>"
   ```
2. Find the conflicting container:
   ```bash
   ssh docker "docker ps --format '{{.Names}}: {{.Ports}}' | grep <port>"
   ```

### Out of Disk Space

1. Check disk usage:
   ```bash
   ssh docker "df -h && docker system df"
   ```
2. Clean up unused images/containers (safe cleanup):
   ```bash
   ssh docker "docker system prune -f"
   ```
3. Clean up unused volumes (careful — verify first):
   ```bash
   ssh docker "docker volume ls -f dangling=true"
   ssh docker "docker volume prune -f"
   ```

### Stack Update Fails in Portainer

1. Check Portainer logs:
   ```bash
   ssh docker "docker logs portainer --tail 50"
   ```
2. Try pulling the image manually:
   ```bash
   ssh docker "docker pull <image>"
   ```
3. Check for container conflicts:
   ```bash
   ssh docker "docker ps -a | grep <container-name>"
   ```

### Container Has No Internet Access

1. Check DNS from inside the container:
   ```bash
   ssh docker "docker exec <container-name> nslookup google.com"
   ```
2. Check network connectivity:
   ```bash
   ssh docker "docker exec <container-name> ping -c 3 8.8.8.8"
   ```
3. Inspect container networks:
   ```bash
   ssh docker "docker inspect <container-name> --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool"
   ```

---

## Per-Service Quick Reference

### paperless-ngx
```bash
# Status
ssh docker "docker ps --filter name=paperless"
# Logs
ssh docker "docker logs paperless-ngx --tail 50"
# Data path: /var/lib/docker/volumes/paperless*/
```

### n8n
```bash
ssh docker "docker logs n8n --tail 50"
# UI: http://192.168.178.26:5678 (or via proxy)
```

### bitwarden (Vaultwarden)
```bash
ssh docker "docker logs bitwarden-bitwarden-1 --tail 50"
# DB: bitwarden-db-1 (MariaDB)
```

### nginx proxy manager
```bash
ssh docker "docker logs ngnix-app-1 --tail 50"
# Admin UI: http://192.168.178.26:82
# Proxy: port 80 (HTTP), port 443 (HTTPS)
```

### beszel monitoring
```bash
ssh docker "docker logs beszel --tail 50"
ssh docker "docker logs beszel-agent --tail 50"
# UI: http://192.168.178.26:8090 (or via proxy)
```

### cloudflare-ddns
```bash
ssh docker "docker logs cloudflare-ddns --tail 50"
# Runs on host network — updates public IP in Cloudflare DNS
```

---

## Related Services

- **AdGuard Home** (`skills/adguard.md`): DNS filtering — runs in `adguard` stack
- **Unbound** (`skills/unbound.md`): Recursive resolver — runs in `adguard` stack
- **Portainer UI:** `https://192.168.178.26:9443` for visual stack management
