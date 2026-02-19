---
name: homelab-adguard
description: Use when troubleshooting, configuring, or managing AdGuard Home DNS filtering, ad blocking, DNS rewrites, client stats, or upstream DNS settings
---

## Service Overview

AdGuard Home is the public-facing DNS server for the homelab. All LAN clients send DNS queries to AdGuard Home, which filters ads/trackers and forwards remaining queries upstream to Unbound.

- **Purpose:** DNS ad blocking, filtering, local DNS rewrites, query logging
- **Public-facing?** Yes — port 53 bound to host, receives all LAN DNS queries
- **Dependencies:** Docker, Unbound (upstream resolver at `172.24.255.254:53`)
- **Consumers:** All LAN devices use this as their DNS server

---

## Infrastructure

| Property | Value |
|----------|-------|
| Container name | `adguardhome` |
| Image | `adguard/adguardhome` |
| Host port | `53/tcp`, `53/udp` (public DNS) |
| Web UI | `http://192.168.178.26:80` or `https://192.168.178.26:443` |
| Docker network | `adguard_unbound_dns-network`, `proxy_network` |
| Container IP (dns-net) | `172.24.0.2` |
| Config volume | `adguard_adguard_conf` → `/opt/adguardhome/conf/` |
| Work volume | `adguard_adguard_work` → `/opt/adguardhome/work/` |
| Config file (inside) | `/opt/adguardhome/conf/AdGuardHome.yaml` |
| Upstream resolver | `172.24.255.254:53` (Unbound) |
| Bootstrap DNS | `9.9.9.10` (Quad9, no filtering) |

**SSH command base:**
```bash
ssh docker
```

---

## Common Operations

### Check Service Status
```bash
ssh docker "docker ps --filter name=adguardhome --format 'Status: {{.Status}}'"
```

### View Logs
```bash
# Last 50 lines
ssh docker "docker logs adguardhome --tail 50"

# Follow live
ssh docker "docker logs adguardhome -f"
```

### Restart Container
```bash
ssh docker "docker restart adguardhome && docker ps --filter name=adguardhome"
```

### Open Web UI
Navigate to: `http://192.168.178.26:80`

### View Current DNS Rewrites
```bash
ssh docker "docker exec adguardhome cat /opt/adguardhome/conf/AdGuardHome.yaml | grep -A 20 'rewrites:'"
```

### Add a DNS Rewrite (local hostname)

**Method 1: Via Web UI (preferred)**
1. Go to `http://192.168.178.26:80` → Filters → DNS rewrites → Add rewrite
2. Domain: `hostname.home.dschmidt95.de`, Answer: `192.168.178.X`

**Method 2: Via config file**

Backup first:
```bash
ssh docker "docker exec adguardhome cp /opt/adguardhome/conf/AdGuardHome.yaml /opt/adguardhome/conf/AdGuardHome.yaml.bak"
```

Edit the rewrites section in config (requires restart to apply):
```bash
ssh docker "docker exec -it adguardhome sh -c 'vi /opt/adguardhome/conf/AdGuardHome.yaml'"
ssh docker "docker restart adguardhome"
```

### View Upstream DNS Config
```bash
ssh docker "docker exec adguardhome cat /opt/adguardhome/conf/AdGuardHome.yaml | grep -A 10 'upstream_dns:'"
```

### Change Upstream DNS

Edit via config file (backup first):
```bash
ssh docker "docker exec adguardhome cp /opt/adguardhome/conf/AdGuardHome.yaml /opt/adguardhome/conf/AdGuardHome.yaml.bak"
```

The upstream is currently `172.24.255.254:53` (Unbound). Change only if intentional — this routes around Unbound.

### Test DNS Resolution (from host through AdGuard)
```bash
ssh docker "dig @127.0.0.1 google.com"
ssh docker "dig @127.0.0.1 home.dschmidt95.de"
```

---

## Troubleshooting Playbook

### DNS Not Resolving for LAN Clients

1. Confirm AdGuard is running and healthy:
   ```bash
   ssh docker "docker ps --filter name=adguardhome"
   ```
2. Test resolution from the server:
   ```bash
   ssh docker "dig @127.0.0.1 google.com"
   ```
3. Check if it can reach Unbound (upstream):
   ```bash
   ssh docker "docker exec adguardhome nslookup google.com 172.24.255.254"
   ```
4. Check AdGuard logs for errors:
   ```bash
   ssh docker "docker logs adguardhome --tail 100"
   ```
5. If Unbound is down, DNS will fail — check Unbound skill first:
   ```bash
   ssh docker "docker exec unbound unbound-control status"
   ```

### Specific Domain Blocked (shouldn't be)

1. Check in Web UI: Filters → Check host → enter domain
2. Or check logs in Web UI → Query Log → search for domain
3. Whitelist via Web UI: Filters → Custom filtering rules → add `@@||domain.com^`

### DNS Rewrite Not Working

1. Verify rewrite exists:
   ```bash
   ssh docker "docker exec adguardhome cat /opt/adguardhome/conf/AdGuardHome.yaml | grep -A 5 'rewrites:'"
   ```
2. Test the rewrite:
   ```bash
   ssh docker "dig @127.0.0.1 test.home.dschmidt95.de"
   ```
3. Check `rewrites_enabled: true` in config
4. Restart AdGuard if config was manually edited:
   ```bash
   ssh docker "docker restart adguardhome"
   ```

### Web UI Not Accessible

1. Check container is running:
   ```bash
   ssh docker "docker ps --filter name=adguardhome"
   ```
2. Check port 80 is bound:
   ```bash
   ssh docker "ss -tlnp | grep :80"
   ```
3. Check Nginx Proxy Manager isn't conflicting on port 80 (it is also on port 80):
   ```bash
   ssh docker "docker ps --filter name=ngnix --format '{{.Names}}: {{.Ports}}'"
   ```

---

## Configuration Reference

Config file: `/opt/adguardhome/conf/AdGuardHome.yaml` (inside container, in named volume)
Last documented: 2026-02-19

### Key DNS Settings

```yaml
dns:
  bind_hosts:
    - 0.0.0.0
  port: 53
  upstream_dns:
    - 172.24.255.254:53      # Unbound recursive resolver
  bootstrap_dns:
    - 9.9.9.10               # Quad9 (no filtering) for bootstrap
    - 149.112.112.10
  upstream_mode: parallel
  cache_enabled: true
  cache_size: 33554432       # 32 MB
  cache_ttl_min: 300
  cache_ttl_max: 86400
  enable_dnssec: false       # DNSSEC done by Unbound instead
```

### DNS Rewrites

| Domain Pattern | Answer | Notes |
|---------------|--------|-------|
| `*.home.dschmidt95.de` | `192.168.178.26` | Wildcard → Docker host |

### Filtering Settings

| Setting | Value |
|---------|-------|
| `filtering_enabled` | `true` |
| `blocking_mode` | `refused` |
| `safe_search` | `false` |
| `parental_enabled` | `false` |
| `safebrowsing_enabled` | `false` |
| `protection_enabled` | `true` |
| Filter update interval | 24 hours |

---

## Related Services

- **Unbound** (`skills/unbound.md`): Backend recursive resolver — AdGuard sends all upstream queries here
- **Nginx Proxy Manager** (`skills/docker.md`): Reverse proxy that serves AdGuard UI via `*.home.dschmidt95.de`
- **Docker** (`skills/docker.md`): Container lifecycle management
