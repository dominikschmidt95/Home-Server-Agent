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

**Method 1: REST API (preferred — live, no restart)**
```bash
ssh docker "curl -s -u 'Dominik:<password>' -X POST http://localhost:80/control/rewrite/add \
  -H 'Content-Type: application/json' \
  -d '{\"domain\": \"mynas.home.dschmidt95.de\", \"answer\": \"192.168.178.50\"}'"
```

**Method 2: Via Web UI**
Go to `http://192.168.178.26:80` → Filters → DNS rewrites → Add rewrite

**Method 3: Via config file (requires restart)**
See "Operational Runbook → Safe Config Edit Procedure" — YAML must not be edited while AdGuard is running.

### Remove a DNS Rewrite

```bash
ssh docker "curl -s -u 'Dominik:<password>' -X POST http://localhost:80/control/rewrite/delete \
  -H 'Content-Type: application/json' \
  -d '{\"domain\": \"mynas.home.dschmidt95.de\", \"answer\": \"192.168.178.50\"}'"
```

### List All DNS Rewrites (via API)

```bash
ssh docker "curl -s -u 'Dominik:<password>' http://localhost:80/control/rewrite/list"
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

## REST API Quick Reference

Base URL: `http://localhost:80/control` (from SSH into Docker host)
Auth: HTTP Basic (`Dominik:<password>`)

> **Prefer the API over editing YAML.** AdGuard Home overwrites its config file while running — if you edit the YAML without stopping the container first, your changes will be lost on the next write cycle.

### Status & Protection

```bash
# Get overall status
ssh docker "curl -s -u 'Dominik:<password>' http://localhost:80/control/status"

# Check DNS config (upstreams, cache, etc.)
ssh docker "curl -s -u 'Dominik:<password>' http://localhost:80/control/dns_info"

# Temporarily disable filtering (e.g. for troubleshooting) — pauses for 30 min
ssh docker "curl -s -u 'Dominik:<password>' -X POST http://localhost:80/control/protection \
  -H 'Content-Type: application/json' \
  -d '{\"enabled\": false, \"duration\": 1800000}'"

# Re-enable filtering
ssh docker "curl -s -u 'Dominik:<password>' -X POST http://localhost:80/control/protection \
  -H 'Content-Type: application/json' \
  -d '{\"enabled\": true}'"
```

### DNS Rewrites (live, no restart)

```bash
# List all rewrites
ssh docker "curl -s -u 'Dominik:<password>' http://localhost:80/control/rewrite/list"

# Add a rewrite
ssh docker "curl -s -u 'Dominik:<password>' -X POST http://localhost:80/control/rewrite/add \
  -H 'Content-Type: application/json' \
  -d '{\"domain\": \"hostname.home.dschmidt95.de\", \"answer\": \"192.168.178.X\"}'"

# Remove a rewrite
ssh docker "curl -s -u 'Dominik:<password>' -X POST http://localhost:80/control/rewrite/delete \
  -H 'Content-Type: application/json' \
  -d '{\"domain\": \"hostname.home.dschmidt95.de\", \"answer\": \"192.168.178.X\"}'"
```

DNS rewrite syntax notes:
- `domain`: supports wildcards — `*.home.dschmidt95.de` matches one level of subdomains only
- `answer`: IPv4/IPv6 address, or `"A"` / `"AAAA"` to inherit from another rewrite
- **Rewrites have higher priority than filter rules** — a rewrite always wins over a blocklist entry

### Filtering & Blocking (live, no restart)

```bash
# Check if a domain is blocked and why
ssh docker "curl -s -u 'Dominik:<password>' 'http://localhost:80/control/filtering/check_host?name=ads.example.com'"

# Force refresh all filter lists now
ssh docker "curl -s -u 'Dominik:<password>' -X POST http://localhost:80/control/filtering/refresh \
  -H 'Content-Type: application/json' \
  -d '{\"whitelist\": false}'"

# Get current filtering status and user rules
ssh docker "curl -s -u 'Dominik:<password>' http://localhost:80/control/filtering/status"

# Add a single block rule (appends to user_rules)
# First get existing rules, then append:
ssh docker "curl -s -u 'Dominik:<password>' http://localhost:80/control/filtering/status | grep user_rules"
# Then set all rules including new one:
ssh docker "curl -s -u 'Dominik:<password>' -X POST http://localhost:80/control/filtering/set_rules \
  -H 'Content-Type: application/json' \
  -d '{\"rules\": \"||newbad.example.com^\n@@||safe.example.com^\"}'"

# Clear DNS cache
ssh docker "curl -s -u 'Dominik:<password>' -X POST http://localhost:80/control/cache_clear"
```

### Test Upstream DNS

```bash
# Test that Unbound (upstream) is reachable
ssh docker "curl -s -u 'Dominik:<password>' -X POST http://localhost:80/control/test_upstream_dns \
  -H 'Content-Type: application/json' \
  -d '{\"upstream_dns\": [\"172.24.255.254:53\"]}'"
# Expected response: {"172.24.255.254:53": "OK"}
# Error response:    {"172.24.255.254:53": "Error: ..."}
```

### Query Log & Stats

```bash
# Recent query log (last 100 queries)
ssh docker "curl -s -u 'Dominik:<password>' 'http://localhost:80/control/querylog?limit=100'"

# Query log filtered to one domain
ssh docker "curl -s -u 'Dominik:<password>' 'http://localhost:80/control/querylog?limit=50&search=example.com'"

# Statistics summary
ssh docker "curl -s -u 'Dominik:<password>' http://localhost:80/control/stats"
```

---

## Log Messages Reference

### Enable Verbose Logging

Must stop the container, edit config, restart (verbose logging can't be toggled live):

```bash
# 1. Stop
ssh docker "docker stop adguardhome"

# 2. Edit config — add/change log section
ssh docker "docker exec adguardhome ..."  # not possible while stopped, use:
# Access via temp container (volume is named, not bind-mounted):
ssh docker "docker run --rm -v adguard_adguard_conf:/conf busybox \
  sed -i 's/verbose: false/verbose: true/' /conf/AdGuardHome.yaml"

# 3. Start
ssh docker "docker start adguardhome"

# 4. Follow logs
ssh docker "docker logs adguardhome -f"

# Reset when done (same sed command, true → false)
```

### Common Log Messages and Their Meaning

| Log Message Pattern | Meaning | Action |
|--------------------|---------|--------|
| `[dns] cache cleared` | DNS cache was cleared via API or UI | None |
| `[filtering] applied N rules` | Filter list loaded with N rules | None |
| `upstream DNS is not responding` | Unbound (upstream) not reachable | Check Unbound skill |
| `bind: address already in use` | Port 53 or 80 already taken by another process | Check port conflicts: `ss -tlnp` |
| `Blocked by CNAME or IP` | CNAME-cloaking detected — a blocked domain appeared in a CNAME chain | Normal filtering behavior |
| `Error: control/version.json` | Can't reach AdGuard update check server | Cosmetic — doesn't affect DNS |
| `upstream: no answer` | Upstream returned empty response | Check Unbound logs |
| `[filters] filtering: failed to load` | Filter list URL unreachable or returned error | Check filter URL reachability |

---

## DNS Rewrite Config Syntax (YAML)

For reference when editing the config file directly. **Stop AdGuard before editing YAML** — a running process will overwrite your changes on its next write cycle.

```yaml
filtering:
  rewrites:
    # Simple A record
    - domain: "nas.home.dschmidt95.de"
      answer: "192.168.178.50"

    # Wildcard — matches one level (foo.home.dschmidt95.de, NOT foo.bar.home.dschmidt95.de)
    - domain: "*.home.dschmidt95.de"
      answer: "192.168.178.26"

    # AAAA record (IPv6)
    - domain: "nas.home.dschmidt95.de"
      answer: "fd00::50"

    # Inherit another rewrite's answer
    - domain: "alias.home.dschmidt95.de"
      answer: "A"   # resolves to same IP as the A rewrite for this name
```

### Custom Filter Rule Syntax (user_rules)

```
# Block a domain and all subdomains
||ads.example.com^

# Allow / unblock a domain (override blocklists)
@@||safe.example.com^

# Block with highest priority (overrides allow rules)
||ads.example.com^$important

# Block only for a specific client
||tracker.com^$client=192.168.178.100

# Block only AAAA queries (let A queries through)
||ipv6only.example.com^$dnstype=AAAA
```

---

## Operational Runbook

### Update AdGuard Home Image

```bash
# 1. Pull new image
ssh docker "docker pull adguard/adguardhome"

# 2. Restart container (Portainer stack update recommended — recreates both adguardhome + unbound together)
# Via CLI:
ssh docker "docker restart adguardhome"

# 3. Verify it came back
ssh docker "docker ps --filter name=adguardhome && docker logs adguardhome --tail 20"

# 4. Test DNS still works
ssh docker "dig @127.0.0.1 google.com"
```

**Preferred:** Use Portainer → adguard stack → Redeploy — this pulls new images and recreates both containers (adguardhome + unbound) in the correct order.

### Backup Config

The config lives in a named Docker volume (`adguard_adguard_conf`). Back it up as a file:

```bash
# Backup to home dir on server
ssh docker "docker exec adguardhome cp /opt/adguardhome/conf/AdGuardHome.yaml \
  /opt/adguardhome/conf/AdGuardHome.yaml.$(date +%Y%m%d)"

# Copy to local Mac
scp docker:/var/lib/docker/volumes/adguard_adguard_conf/_data/AdGuardHome.yaml \
  ~/Desktop/AdGuardHome.yaml.backup
```

### Restore Config from Backup

```bash
# 1. Stop container first — CRITICAL (running process overwrites config on writes)
ssh docker "docker stop adguardhome"

# 2. Copy backup into volume
ssh docker "docker run --rm -v adguard_adguard_conf:/conf -v /home/dominik:/src busybox \
  cp /src/AdGuardHome.yaml.bak /conf/AdGuardHome.yaml"

# 3. Start container
ssh docker "docker start adguardhome"

# 4. Verify
ssh docker "docker logs adguardhome --tail 30"
```

### Safe Config Edit Procedure

**NEVER edit the YAML while AdGuard is running.** The running process writes config to disk periodically and will overwrite your changes.

```bash
# 1. Backup first
ssh docker "docker exec adguardhome cp /opt/adguardhome/conf/AdGuardHome.yaml \
  /opt/adguardhome/conf/AdGuardHome.yaml.bak"

# 2. Stop container
ssh docker "docker stop adguardhome"

# 3. Edit config (use a temp container to access the named volume)
ssh docker "docker run --rm -it -v adguard_adguard_conf:/conf alpine vi /conf/AdGuardHome.yaml"

# 4. Start container
ssh docker "docker start adguardhome"

# 5. Check it started correctly
ssh docker "docker logs adguardhome --tail 30"

# 6. Test DNS
ssh docker "dig @127.0.0.1 google.com"

# If something broke — restore:
ssh docker "docker stop adguardhome"
ssh docker "docker run --rm -v adguard_adguard_conf:/conf busybox \
  cp /conf/AdGuardHome.yaml.bak /conf/AdGuardHome.yaml"
ssh docker "docker start adguardhome"
```

### Things That Can Be Changed Without Restart (via API)

- DNS rewrites (add/remove)
- Custom filter rules (user_rules)
- Temporarily disable/enable filtering protection
- Force filter list refresh
- Clear DNS cache
- Check if a domain is blocked

### Things That Require Restart

- Changing `upstream_dns` servers
- Changing `dns.port` or `dns.bind_hosts`
- Changing `http.address` (web UI port)
- Any other change to `AdGuardHome.yaml`

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
3. Test that upstream (Unbound) is reachable via API:
   ```bash
   ssh docker "curl -s -u 'Dominik:<password>' -X POST http://localhost:80/control/test_upstream_dns \
     -H 'Content-Type: application/json' \
     -d '{\"upstream_dns\": [\"172.24.255.254:53\"]}'"
   # Expected: {"172.24.255.254:53": "OK"}
   ```
4. Check AdGuard logs for upstream errors:
   ```bash
   ssh docker "docker logs adguardhome --tail 100"
   # Look for: "upstream DNS is not responding"
   ```
5. If Unbound is down — DNS will completely fail (no fallback_dns configured):
   ```bash
   ssh docker "docker exec unbound unbound-control status"
   ```
   Fix: restart Unbound: `ssh docker "docker restart unbound"`

### Specific Domain Blocked (shouldn't be)

1. Check exactly why it's blocked via API:
   ```bash
   ssh docker "curl -s -u 'Dominik:<password>' 'http://localhost:80/control/filtering/check_host?name=example.com'"
   ```
2. The response shows: which filter list matched, the rule that triggered, and the result
3. Unblock via API (add allow rule):
   ```bash
   # Get current user_rules first, then append allowlist entry
   ssh docker "curl -s -u 'Dominik:<password>' -X POST http://localhost:80/control/filtering/set_rules \
     -H 'Content-Type: application/json' \
     -d '{\"rules\": \"@@||example.com^\"}'"
   ```
4. Or via Web UI: Filters → Custom filtering rules → add `@@||domain.com^`

### DNS Rewrite Not Working

1. List all rewrites via API:
   ```bash
   ssh docker "curl -s -u 'Dominik:<password>' http://localhost:80/control/rewrite/list"
   ```
2. Test the rewrite resolves:
   ```bash
   ssh docker "dig @127.0.0.1 test.home.dschmidt95.de"
   ```
3. Check `rewrites_enabled: true` in config (if false, rewrites are silently ignored)
4. Note: **rewrites win over blocklist rules** — if a domain is both rewritten and blocked, the rewrite applies

### Domain Resolves to Wrong IP (stale rewrite or cache)

1. Clear DNS cache:
   ```bash
   ssh docker "curl -s -u 'Dominik:<password>' -X POST http://localhost:80/control/cache_clear"
   ```
2. Verify current rewrites:
   ```bash
   ssh docker "curl -s -u 'Dominik:<password>' http://localhost:80/control/rewrite/list"
   ```
3. Test after cache clear:
   ```bash
   ssh docker "dig @127.0.0.1 <domain>"
   ```

### Upstream DNS Unreachable (Unbound is down)

AdGuard has **no fallback_dns configured** — if Unbound is unreachable, all DNS fails.

1. Confirm Unbound is down:
   ```bash
   ssh docker "docker ps --filter name=unbound"
   ```
2. Restart Unbound:
   ```bash
   ssh docker "docker restart unbound && docker exec unbound unbound-control status"
   ```
3. Test DNS recovers:
   ```bash
   ssh docker "dig @127.0.0.1 google.com"
   ```
4. Consider adding a fallback for resilience (edit YAML, requires restart):
   ```yaml
   dns:
     fallback_dns:
       - "9.9.9.9"   # only used if Unbound is completely unreachable
   ```

### Web UI Not Accessible

1. Check container is running:
   ```bash
   ssh docker "docker ps --filter name=adguardhome"
   ```
2. Check port 80 is bound to the right container:
   ```bash
   ssh docker "ss -tlnp | grep :80"
   ssh docker "docker ps --format '{{.Names}}: {{.Ports}}' | grep :80"
   ```
3. Note: Nginx Proxy Manager also uses port 80 — check for conflicts:
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
