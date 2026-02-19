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

# Logs since a relative time (e.g., last 30 minutes)
ssh docker "docker logs <container-name> --since 30m"

# Logs between two timestamps
ssh docker "docker logs <container-name> --since '2024-01-15T10:00:00' --until '2024-01-15T10:30:00'"
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

## docker logs — Reference

### Flags

| Flag | Purpose |
|------|---------|
| `-f, --follow` | Stream new output continuously (Ctrl+C to stop) |
| `-t, --timestamps` | Prefix each line with RFC3339Nano timestamp |
| `-n, --tail N` | Show last N lines only (default: all) |
| `--since DURATION` | Show logs after this time; accepts `30m`, `2h`, `2024-01-15T10:00:00`, unix timestamp |
| `--until DURATION` | Show logs before this time; same format as `--since` |
| `--details` | Show extra metadata (env vars, labels) attached to log entries |

### Important: When `docker logs` Shows Nothing

Some containers write to files instead of stdout/stderr. This happens with:
- Nginx (writes to `/var/log/nginx/`) — fixed by symlinking to `/dev/stdout` and `/dev/stderr`
- Apache httpd — writes to `/proc/self/fd/1` and `/proc/self/fd/2`
- Any container using a non-default logging driver (e.g., `json-file` is default; `syslog`, `journald`, `fluentd` redirect logs elsewhere)

If `docker logs` returns nothing, check the logging driver:
```bash
ssh docker "docker inspect <container-name> --format '{{.HostConfig.LogConfig.Type}}'"
```

---

## docker inspect — Format String Reference

### Syntax
```bash
docker inspect [--type=container|image|volume|network] --format 'TEMPLATE' NAME_OR_ID
```

The `--format` flag uses Go text/template syntax. Use `{{json .FIELD}}` to dump a subsection as JSON.

### Most Useful Format Strings

```bash
# Container IP address (on all attached networks)
ssh docker "docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}' <container>"

# Container IP on a specific network
ssh docker "docker inspect --format '{{.NetworkSettings.Networks.proxy_network.IPAddress}}' <container>"

# All port bindings (container-port -> host-port)
ssh docker "docker inspect --format '{{range \$p, \$conf := .NetworkSettings.Ports}}{{if \$conf}}{{$p}} -> {{(index \$conf 0).HostPort}}{{println}}{{end}}{{end}}' <container>"

# Host port for a specific container port
ssh docker "docker inspect --format '{{(index (index .NetworkSettings.Ports \"8080/tcp\") 0).HostPort}}' <container>"

# Exit code and error message
ssh docker "docker inspect --format '{{.State.ExitCode}} {{.State.Error}}' <container>"

# Full container state (running, exited, OOMKilled, etc.)
ssh docker "docker inspect --format '{{json .State}}' <container> | python3 -m json.tool"

# OOMKilled flag specifically
ssh docker "docker inspect --format '{{.State.OOMKilled}}' <container>"

# Restart count
ssh docker "docker inspect --format '{{.RestartCount}}' <container>"

# Image used by container
ssh docker "docker inspect --format '{{.Config.Image}}' <container>"

# Image digest (pinned version actually running)
ssh docker "docker inspect --format '{{.Image}}' <container>"

# Restart policy
ssh docker "docker inspect --format '{{.HostConfig.RestartPolicy.Name}}' <container>"

# All environment variables
ssh docker "docker inspect --format '{{json .Config.Env}}' <container> | python3 -m json.tool"

# All mounts (volumes + bind mounts)
ssh docker "docker inspect --format '{{json .Mounts}}' <container> | python3 -m json.tool"

# Log file path on host
ssh docker "docker inspect --format '{{.LogPath}}' <container>"

# Container start time
ssh docker "docker inspect --format '{{.State.StartedAt}}' <container>"

# Container finish time (if exited)
ssh docker "docker inspect --format '{{.State.FinishedAt}}' <container>"

# MAC address per network
ssh docker "docker inspect --format '{{range .NetworkSettings.Networks}}{{.MacAddress}} {{end}}' <container>"

# Subsection as formatted JSON
ssh docker "docker inspect --format '{{json .Config}}' <container> | python3 -m json.tool"
ssh docker "docker inspect --format '{{json .HostConfig}}' <container> | python3 -m json.tool"
ssh docker "docker inspect --format '{{json .NetworkSettings}}' <container> | python3 -m json.tool"

# Total container size on disk (requires --size flag)
ssh docker "docker inspect --size --format '{{.SizeRootFs}} bytes total, {{.SizeRw}} bytes written' <container>"

# Volume inspect: get host mountpoint
ssh docker "docker volume inspect --format '{{.Mountpoint}}' <volume-name>"
```

---

## docker stats — Column Reference

### Command
```bash
# Live stream (all running containers)
ssh docker "docker stats"

# Snapshot only (no stream)
ssh docker "docker stats --no-stream"

# Custom table
ssh docker "docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}\t{{.BlockIO}}'"

# Single container as JSON
ssh docker "docker stats <container> --no-stream --format '{{json .}}'"
```

### Column Meanings

| Column | Placeholder | Meaning |
|--------|-------------|---------|
| CONTAINER ID | `.Container` | Short container ID |
| NAME | `.Name` | Container name |
| CPU % | `.CPUPerc` | Percentage of host CPU used |
| MEM USAGE / LIMIT | `.MemUsage` | Current RAM used / RAM limit set for container |
| MEM % | `.MemPerc` | RAM used as percentage of container limit |
| NET I/O | `.NetIO` | Total network bytes received / sent since container start |
| BLOCK I/O | `.BlockIO` | Total disk bytes read / written since container start |
| PIDs | `.PIDs` | Number of processes and kernel threads inside the container |

### Format Placeholders
`.Container`, `.Name`, `.ID`, `.CPUPerc`, `.MemUsage`, `.NetIO`, `.BlockIO`, `.MemPerc`, `.PIDs`

### Diagnostics via Stats
- **High CPU %**: Runaway process, stuck loop, or legitimate load
- **MEM USAGE near LIMIT**: Container approaching memory limit; may trigger OOMKill
- **MEM LIMIT = 0B**: No memory limit set (container can use all host RAM)
- **NET I/O = 0B / 0B**: Container has no network traffic (possible connectivity issue or idle)
- **BLOCK I/O very high**: Heavy disk read/write — check for log spam or indexing

---

## docker system df — Output Reference

### Commands
```bash
# Summary view
ssh docker "docker system df"

# Detailed breakdown per image/container/volume
ssh docker "docker system df -v"
```

### Summary Output Columns

| Column | Meaning |
|--------|---------|
| TYPE | Resource category: `Images`, `Containers`, `Local Volumes`, `Build Cache` |
| TOTAL | Total count of that resource type |
| ACTIVE | Count currently in use (images used by running containers, volumes mounted, etc.) |
| SIZE | Total disk space consumed by all resources of that type |
| RECLAIMABLE | Space that can be freed with `docker system prune` (shown as bytes and %) |

### Verbose Mode (-v) Adds

**Images section:** REPOSITORY, TAG, IMAGE ID, CREATED, SIZE (virtual), SHARED SIZE (shared with other images), UNIQUE SIZE (only in this image), CONTAINERS (how many containers use it)

**Containers section:** CONTAINER ID, IMAGE, COMMAND, LOCAL VOLUMES, SIZE (writable layer only), CREATED, STATUS, NAMES

**Volumes section:** NAME, LINKS (containers using it), SIZE

**Note:** Networks are not shown — they consume no disk space.

**SIZE = SHARED SIZE + UNIQUE SIZE** for images.

---

## docker events — Real-Time Event Monitoring

### Command
```bash
# Stream all events live
ssh docker "docker events"

# Last 10 minutes of events
ssh docker "docker events --since 10m"

# Events in a time window
ssh docker "docker events --since '2024-01-15T10:00:00' --until '2024-01-15T10:05:00'"

# Filter to specific event type
ssh docker "docker events --filter 'event=die'"
ssh docker "docker events --filter 'event=oom'"
ssh docker "docker events --filter 'event=restart'"
ssh docker "docker events --filter 'event=stop'"

# Filter to a specific container
ssh docker "docker events --filter 'container=<container-name>'"

# Combine filters (AND logic across different keys)
ssh docker "docker events --filter 'type=container' --filter 'event=die'"

# JSON output format
ssh docker "docker events --format json"

# Custom format
ssh docker "docker events --format 'Type={{.Type}} Status={{.Status}} ID={{.ID}}'"
```

Note: Only the last 256 log events are retained without filtering.

### Container Event Types and Their Meanings

| Event | Meaning |
|-------|---------|
| `create` | Container was created (not yet started) |
| `start` | Container process started |
| `restart` | Container was restarted (by restart policy or manually) |
| `stop` | Container was gracefully stopped (SIGTERM + wait) |
| `kill` | Container received a kill signal |
| `die` | Container process exited (any reason — check exit code separately) |
| `oom` | Container was killed by the kernel due to out-of-memory |
| `pause` / `unpause` | Container execution was paused/resumed |
| `destroy` | Container was removed |
| `attach` / `detach` | Terminal attached/detached |
| `exec_create` / `exec_start` / `exec_die` | `docker exec` was run inside the container |
| `health_status` | Health check result changed (healthy / unhealthy / starting) |
| `rename` | Container was renamed |
| `update` | Container resource limits were updated |
| `copy` | File was copied from the container |
| `commit` | Container was committed to a new image |
| `export` | Container filesystem was exported |

### Image, Volume, Network Events

| Object | Events |
|--------|--------|
| Images | `pull`, `push`, `tag`, `untag`, `delete`, `import`, `load`, `save` |
| Volumes | `create`, `mount`, `unmount`, `destroy` |
| Networks | `create`, `connect`, `disconnect`, `destroy`, `remove` |
| Plugins | `install`, `enable`, `disable`, `remove` |
| Daemon | `reload` |

### Filter Keys
`config`, `container`, `daemon`, `event`, `image`, `label`, `network`, `node`, `plugin`, `scope`, `secret`, `service`, `type`, `volume`

**Logic:** Multiple identical filter keys = OR; different filter keys = AND.

---

## Container Failure Patterns

### Exit Codes

| Exit Code | Meaning |
|-----------|---------|
| `0` | Clean exit — process finished successfully |
| `1` | General application error (check logs for details) |
| `2` | Misuse of shell command / invalid argument |
| `125` | Docker daemon error (invalid CLI flag or Docker itself failed) |
| `126` | Container command found but cannot be executed (permissions issue) |
| `127` | Container command not found (wrong path, missing binary) |
| `128+N` | Process killed by signal N (e.g., `137` = killed by SIGKILL = exit 128+9) |
| `130` | Killed by SIGINT (Ctrl+C) |
| `137` | SIGKILL — usually OOMKilled or manually killed |
| `139` | Segmentation fault (SIGSEGV) |
| `143` | Graceful SIGTERM |

### OOMKilled

OOMKilled means the Linux kernel killed the container process because it exceeded its memory limit (or the host ran out of memory).

**Detect OOMKilled:**
```bash
# Check OOMKilled flag
ssh docker "docker inspect --format '{{.State.OOMKilled}}' <container>"

# Full state details
ssh docker "docker inspect --format '{{json .State}}' <container> | python3 -m json.tool"

# Check events for oom event
ssh docker "docker events --since 1h --filter 'event=oom' --filter 'container=<container>'"
```

**OOMKilled indicators:**
- `State.OOMKilled: true` in inspect output
- Exit code `137` in `docker ps -a` STATUS column (e.g., `Exited (137)`)
- `oom` event in `docker events`
- Log lines from kernel: `Out of memory: Kill process`

**Remediation:**
- Increase memory limit in compose file (`mem_limit:` or `deploy.resources.limits.memory:`)
- Investigate memory leak in the application
- Add swap to the host: `ssh docker "free -h"`

### Restart Loop Detection

```bash
# Check restart count
ssh docker "docker inspect --format '{{.RestartCount}}' <container>"

# Container status showing restart loop
ssh docker "docker ps -a --filter name=<container>"
# STATUS column will show: "Restarting (1) X seconds ago"

# Get the last exit code driving the restarts
ssh docker "docker inspect --format 'ExitCode={{.State.ExitCode}} OOMKilled={{.State.OOMKilled}} Error={{.State.Error}}' <container>"

# Read logs from before the crash
ssh docker "docker logs <container> --tail 100"
```

**Common restart loop causes:**
1. Bad configuration file (syntax error, missing required env var)
2. Missing dependency (database not ready, secret file missing)
3. Port already in use — container starts then immediately fails
4. OOMKilled immediately after start (memory limit too low)
5. Permission denied on a mounted volume or file
6. Application exits cleanly with code 0 but restart policy is `always` or `unless-stopped`

### Restart Policies

| Policy | Behavior |
|--------|----------|
| `no` (default) | Never restart automatically |
| `on-failure[:N]` | Restart only if exit code is non-zero; optionally limit to N retries |
| `always` | Always restart regardless of exit code; restarts on daemon reboot |
| `unless-stopped` | Like `always` but does NOT restart if manually stopped before daemon reboot |

**Important caveats:**
- Restart policy only activates after the container runs for at least 10 seconds (to prevent immediate restart loops from being amplified)
- If manually stopped, restart policy is ignored until daemon restarts or manual `docker start`
- `always` will restart a container even after a clean `exit 0`

```bash
# Check a container's restart policy
ssh docker "docker inspect --format '{{.HostConfig.RestartPolicy.Name}} maxRetry={{.HostConfig.RestartPolicy.MaximumRetryCount}}' <container>"
```

---

## Safe Image Update Workflow (pull → recreate)

### Via Portainer (preferred for Compose stacks)
1. Go to `https://192.168.178.26:9443`
2. Stacks → select stack → click "Pull and redeploy" (or "Update the stack")
3. Portainer pulls new images, then recreates containers in dependency order

### Via CLI (Compose stack)
```bash
# Step 1: Pull new images without stopping anything
ssh docker "docker compose -f /home/dominik/docker/<stack>/docker-compose.yml pull"

# Step 2: Recreate containers with new images (brief downtime per container)
ssh docker "docker compose -f /home/dominik/docker/<stack>/docker-compose.yml up -d --force-recreate"

# Optional: remove orphaned containers not in compose file
ssh docker "docker compose -f /home/dominik/docker/<stack>/docker-compose.yml up -d --force-recreate --remove-orphans"
```

### Update a Single Service in a Compose Stack
```bash
# Pull only this service's image
ssh docker "docker compose -f /path/to/compose.yml pull <service-name>"

# Recreate only this service (without restarting dependencies)
ssh docker "docker compose -f /path/to/compose.yml up -d --force-recreate --no-deps <service-name>"
```

### Via CLI (standalone container)
```bash
# Step 1: Pull new image
ssh docker "docker pull <image>:<tag>"

# Step 2: Stop current container
ssh docker "docker stop <container-name>"

# Step 3: Remove current container (data in volumes is safe)
ssh docker "docker rm <container-name>"

# Step 4: Start new container with same run parameters
ssh docker "docker run -d --name <container-name> [all original flags] <image>:<tag>"

# Step 5: Verify it's running
ssh docker "docker ps --filter name=<container-name>"
ssh docker "docker logs <container-name> --tail 20"
```

### Clean Up Old Images After Update
```bash
# Remove dangling images (old layers no longer referenced)
ssh docker "docker image prune -f"

# See what images are taking space
ssh docker "docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedSince}}'"
```

### docker compose up Flags Reference

| Flag | Meaning |
|------|---------|
| `-d` | Run in background (detached) |
| `--build` | Build images from Dockerfile before starting |
| `--pull always` | Always pull image from registry before starting |
| `--pull missing` | Pull only if image not present locally (default behavior) |
| `--pull never` | Never pull; use local image only |
| `--force-recreate` | Recreate containers even if config/image unchanged |
| `--no-recreate` | Never recreate containers that already exist |
| `--no-deps` | Start only named service, not its dependencies |
| `--remove-orphans` | Remove containers for services no longer in compose file |
| `--wait` | Wait for services to reach healthy/running state before returning |
| `--abort-on-container-exit` | Stop all services if any container stops |

---

## docker network — Inspection and Diagnostics

### List Networks
```bash
ssh docker "docker network ls"
# Columns: NETWORK ID, NAME, DRIVER, SCOPE

# Filter by driver
ssh docker "docker network ls --filter driver=bridge"

# User-defined networks only (not built-ins)
ssh docker "docker network ls --filter type=custom"

# Full IDs
ssh docker "docker network ls --no-trunc"

# Custom format
ssh docker "docker network ls --format 'table {{.ID}}\t{{.Name}}\t{{.Driver}}\t{{.Scope}}'"
```

### Network Drivers and Their Meaning

| Driver | Scope | Meaning |
|--------|-------|---------|
| `bridge` | local | Default isolated network; containers on same bridge can communicate |
| `host` | local | Container shares host's network stack directly (no isolation) |
| `overlay` | swarm | Multi-host networking for Docker Swarm |
| `null` / `none` | local | No networking; completely isolated container |
| `macvlan` | local | Container gets its own MAC/IP on the physical network |

### Inspect a Network
```bash
# Full JSON output (shows all connected containers with their IPs)
ssh docker "docker network inspect <network-name>"

# Get subnet and gateway
ssh docker "docker network inspect --format '{{range .IPAM.Config}}subnet={{.Subnet}} gateway={{.Gateway}}{{end}}' <network-name>"

# List all containers on a network with their IPs
ssh docker "docker network inspect --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{println}}{{end}}' <network-name>"

# Verbose diagnostics
ssh docker "docker network inspect -v <network-name>"
```

### Diagnose Container Connectivity Issues

```bash
# Step 1: Confirm both containers are on the same network
ssh docker "docker inspect --format '{{json .NetworkSettings.Networks}}' <container-a> | python3 -m json.tool"
ssh docker "docker inspect --format '{{json .NetworkSettings.Networks}}' <container-b> | python3 -m json.tool"

# Step 2: Get IPs of both containers
ssh docker "docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}' <container-a>"
ssh docker "docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}' <container-b>"

# Step 3: Test DNS resolution from inside container
ssh docker "docker exec <container-a> nslookup <container-b>"
ssh docker "docker exec <container-a> getent hosts <container-b>"

# Step 4: Test IP connectivity
ssh docker "docker exec <container-a> ping -c 3 <container-b>"
ssh docker "docker exec <container-a> ping -c 3 <ip-of-container-b>"

# Step 5: Test port reachability
ssh docker "docker exec <container-a> nc -zv <container-b> <port>"
# or with wget:
ssh docker "docker exec <container-a> wget -qO- http://<container-b>:<port>/health"

# Step 6: Check if a container can reach the internet
ssh docker "docker exec <container-name> ping -c 3 8.8.8.8"
ssh docker "docker exec <container-name> nslookup google.com"
```

**Common network issues:**
- Containers on different Compose stacks are on different default networks — must explicitly share a network
- `host` network containers bypass Docker networking; they see host's IP directly
- DNS between containers works by container name only within the same user-defined network (not on default `bridge`)

---

## docker volume — Inspection and Backup

### List Volumes
```bash
ssh docker "docker volume ls"

# Show only dangling volumes (not attached to any container)
ssh docker "docker volume ls -f dangling=true"

# Custom format
ssh docker "docker volume ls --format 'table {{.Name}}\t{{.Driver}}\t{{.Mountpoint}}'"
```

### Inspect a Volume
```bash
# Full JSON
ssh docker "docker volume inspect <volume-name>"

# Get host mountpoint path only
ssh docker "docker volume inspect --format '{{.Mountpoint}}' <volume-name>"

# Get driver
ssh docker "docker volume inspect --format '{{.Driver}}' <volume-name>"

# Created time
ssh docker "docker volume inspect --format '{{.CreatedAt}}' <volume-name>"
```

### Volume Fields

| Field | Meaning |
|-------|---------|
| `Name` | Volume identifier used in compose/run commands |
| `Driver` | Storage driver (`local` = default, stored at Docker data root) |
| `Mountpoint` | Absolute host path where data is stored (e.g., `/var/lib/docker/volumes/<name>/_data`) |
| `CreatedAt` | When the volume was first created |
| `Labels` | Custom metadata labels |
| `Options` | Driver-specific options |
| `Scope` | `local` (single host) or `global` (swarm) |

### Backup a Named Volume
```bash
# Method 1: tar via a temporary container (volume must not require exclusive access)
# Backs up to /home/dominik/ on the Docker host
ssh docker "docker run --rm -v <volume-name>:/data -v /home/dominik:/backup alpine tar czf /backup/<volume-name>-$(date +%Y%m%d).tar.gz -C /data ."

# Method 2: Copy directly from mountpoint (requires sudo on host)
MOUNT=$(ssh docker "docker volume inspect --format '{{.Mountpoint}}' <volume-name>")
ssh docker "sudo tar czf /home/dominik/<volume-name>-backup.tar.gz -C $MOUNT ."

# Restore from backup
ssh docker "docker run --rm -v <volume-name>:/data -v /home/dominik:/backup alpine tar xzf /backup/<volume-name>-backup.tar.gz -C /data"
```

### Clean Up Unused Volumes
```bash
# Preview dangling volumes before removing
ssh docker "docker volume ls -f dangling=true"

# Remove all unused volumes (CAUTION: verify the list first)
ssh docker "docker volume prune -f"

# Remove a specific volume
ssh docker "docker volume rm <volume-name>"
```

---

## docker ps — Filter and Format Reference

### Useful Filters

```bash
# Show containers that exited with a specific code
ssh docker "docker ps -a --filter 'exited=1'"
ssh docker "docker ps -a --filter 'exited=137'"   # OOMKilled or SIGKILL

# Show only running containers
ssh docker "docker ps --filter 'status=running'"

# Show containers in restart loop
ssh docker "docker ps --filter 'status=restarting'"

# Show exited containers
ssh docker "docker ps -a --filter 'status=exited'"

# Filter by image
ssh docker "docker ps --filter 'ancestor=nginx:latest'"

# Filter by network
ssh docker "docker ps --filter 'network=proxy_network'"

# Filter by health status
ssh docker "docker ps --filter 'health=unhealthy'"
ssh docker "docker ps --filter 'health=starting'"

# Filter by name substring
ssh docker "docker ps -a --filter 'name=paperless'"
```

### Status Values

| Status | Meaning |
|--------|---------|
| `created` | Container exists but has never been started |
| `running` | Container is running normally |
| `restarting` | Container is in a restart loop (restart policy active) |
| `paused` | Container execution suspended with `docker pause` |
| `exited` | Container has stopped; check exit code for reason |
| `removing` | Container is being deleted |
| `dead` | Container could not be removed properly; needs `docker rm -f` |

### Format Placeholders
`.ID`, `.Image`, `.Command`, `.CreatedAt`, `.RunningFor`, `.Ports`, `.State`, `.Status`, `.Size`, `.Names`, `.Labels`, `.Label "key"`, `.Mounts`, `.Networks`

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
3. Inspect the exit code and OOMKilled flag:
   ```bash
   ssh docker "docker inspect --format 'ExitCode={{.State.ExitCode}} OOMKilled={{.State.OOMKilled}} RestartCount={{.RestartCount}} Error={{.State.Error}}' <container-name>"
   ```
4. Check recent events for the container:
   ```bash
   ssh docker "docker events --since 1h --filter 'container=<container-name>'"
   ```
5. Fix the underlying cause (missing volume, bad config, port conflict, OOM)
6. Restart after fix:
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
2. See what's consuming space in detail:
   ```bash
   ssh docker "docker system df -v"
   ```
3. Clean up unused images/containers (safe cleanup):
   ```bash
   ssh docker "docker system prune -f"
   ```
4. Clean up unused volumes (careful — verify first):
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

### Container Using Too Much Memory

1. Check current usage:
   ```bash
   ssh docker "docker stats --no-stream --format 'table {{.Name}}\t{{.MemUsage}}\t{{.MemPerc}}'"
   ```
2. Check if OOM events occurred recently:
   ```bash
   ssh docker "docker events --since 24h --filter 'event=oom'"
   ```
3. Check memory limit:
   ```bash
   ssh docker "docker inspect --format '{{.HostConfig.Memory}}' <container-name>"
   # 0 = no limit
   ```

### Health Check Failing

```bash
# See health status
ssh docker "docker ps --filter 'name=<container>' --format '{{.Names}}: {{.Status}}'"

# Detailed health check history (last 5 results)
ssh docker "docker inspect --format '{{json .State.Health}}' <container-name> | python3 -m json.tool"

# Watch health events in real time
ssh docker "docker events --filter 'event=health_status' --filter 'container=<container-name>'"
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
