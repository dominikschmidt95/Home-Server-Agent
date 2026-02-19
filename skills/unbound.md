---
name: homelab-unbound
description: Use when troubleshooting, configuring, or managing the Unbound recursive DNS resolver on the homelab server
---

## Service Overview

Unbound is a DNSSEC-validating recursive resolver running as a Docker container. It acts as the upstream resolver for AdGuard Home — clients never talk to Unbound directly. Unbound queries root servers directly (no forwarding), validating DNSSEC along the way.

- **Purpose:** Private, validating recursive DNS resolver (backend)
- **Public-facing?** No — only reachable from AdGuard Home inside Docker network
- **Dependencies:** Docker, `adguard_unbound_dns-network`
- **Consumers:** AdGuard Home sends all upstream DNS queries here

---

## Infrastructure

| Property | Value |
|----------|-------|
| Container name | `unbound` |
| Image | `mvance/unbound:latest` |
| Docker network | `adguard_unbound_dns-network` |
| Container IP | `172.24.255.254` |
| Port | `53/tcp` and `53/udp` (internal only, not host-exposed) |
| Config file (host) | `/home/dominik/docker/unbound/unbound.conf` |
| root.hints (host) | `/home/dominik/docker/unbound/root.hints` |
| Config inside container | `/opt/unbound/etc/unbound/unbound.conf` (read-only mount) |
| Unbound version | `1.22.0` |

**SSH command base:**
```bash
ssh docker
```

---

## Common Operations

### Check Service Status
```bash
ssh docker "docker ps --filter name=unbound --format 'Status: {{.Status}}'"
```

### Full Health Check (status + control)
```bash
ssh docker "docker exec unbound unbound-control status"
```

### View Logs
```bash
# Last 50 lines
ssh docker "docker logs unbound --tail 50"

# Follow live
ssh docker "docker logs unbound -f"
```

### Validate Config (before applying)
```bash
ssh docker "docker exec unbound unbound-checkconf /opt/unbound/etc/unbound/unbound.conf"
```

### Reload Config (without restart)
```bash
ssh docker "docker exec unbound unbound-control reload"
```

### Restart Container
```bash
ssh docker "docker restart unbound && docker ps --filter name=unbound"
```

### Check Cache Statistics
```bash
ssh docker "docker exec unbound unbound-control stats_noreset"
```

### Flush Cache (full)
```bash
ssh docker "docker exec unbound unbound-control flush_zone ."
```

### Test DNS Resolution from inside Unbound container
```bash
# Test against Unbound directly
ssh docker "docker exec unbound drill @127.0.0.1 google.com"

# Test DNSSEC validation
ssh docker "docker exec unbound drill -D @127.0.0.1 google.com"
```

### Test DNS Resolution from host (via AdGuard)
```bash
ssh docker "dig @127.0.0.1 google.com"
```

### Edit Config

**Always backup first:**
```bash
ssh docker "cp /home/dominik/docker/unbound/unbound.conf /home/dominik/docker/unbound/unbound.conf.bak"
```

Edit on the host:
```bash
ssh docker "nano /home/dominik/docker/unbound/unbound.conf"
```

After editing, validate and reload:
```bash
ssh docker "docker exec unbound unbound-checkconf /opt/unbound/etc/unbound/unbound.conf && docker exec unbound unbound-control reload"
```

### Update root.hints
```bash
ssh docker "cd /home/dominik/docker/unbound && wget -O root.hints.new https://www.internic.net/domain/named.cache && mv root.hints.new root.hints"
ssh docker "docker exec unbound unbound-control reload"
```

---

## Log Messages Reference

### Verbosity Levels

Set in config (`verbosity: N`) or live with `unbound-control verbosity N` (resets on next reload).

| Level | What you see |
|-------|-------------|
| `0` | Errors only — minimal, use in production |
| `1` | Operational info — server start/stop, config reload (default) |
| `2` | Short per-query info — good for debugging without flooding |
| `3` | Full per-query detail — one entry per query |
| `4` | Algorithm-level — DNSSEC crypto operations |
| `5` | Client identification on cache misses |

Enable query logging temporarily (live, no reload):
```bash
ssh docker "docker exec unbound unbound-control set_option log-queries: yes"
ssh docker "docker exec unbound unbound-control verbosity 2"
# Reset when done:
ssh docker "docker exec unbound unbound-control set_option log-queries: no"
ssh docker "docker exec unbound unbound-control verbosity 0"
```

### Common Log Messages and Their Meaning

| Log Message Pattern | Meaning | Action |
|--------------------|---------|--------|
| `info: start of service` | Unbound started successfully | None |
| `info: reload of config` | Config reloaded via `unbound-control reload` | None |
| `info: cache memory msg=N rrset=N` | Memory report at reload | None |
| `info: generate keytag query` | RFC 5011 trust anchor probing | Normal DNSSEC behavior |
| `info: validated` | DNSSEC validation passed | None |
| `validator: validation failure <name> <type>` | DNSSEC validation failed | See DNSSEC troubleshooting |
| `validator: bogus` | Record failed signature verification | Flush bogus, check domain |
| `warning: SERVFAIL` | Query returned SERVFAIL — usually a DNSSEC failure | Check val-log-level output |
| `error: ssl handshake failed` | TLS upstream connection problem | Check upstream server |
| `unwanted reply from <IP>` | Possible DNS spoofing attempt | Check upstream resolver |

Enable detailed DNSSEC failure logging (live):
```bash
ssh docker "docker exec unbound unbound-control set_option val-log-level: 2"
# val-log-level 2 = logs exact failure reason per query:
# "validator: validation failure example.com A: signature expired"
# "validator: validation failure example.com A: NSEC3 closest encloser proof failed"
# Reset when done:
ssh docker "docker exec unbound unbound-control set_option val-log-level: 0"
```

---

## unbound-control Quick Reference

### Lifecycle

| Command | Effect |
|---------|--------|
| `unbound-control status` | Show running status, version, uptime |
| `unbound-control reload` | Flush cache + re-read config |
| `unbound-control reload_keep_cache` | Re-read config, preserve cache contents |
| `unbound-control stop` | Stop the daemon |

**Reload vs Restart:**
- **Reload** — sufficient for: local zones, access-control, log settings, forward/stub zones, DNSSEC options. No downtime, cache optionally preserved.
- **Restart** — required only for: changing `interface:` bindings or port numbers. Causes brief outage.

In this Docker setup, "restart" means `docker restart unbound` — config is re-read from the host bind mount.

### Cache Management

| Command | Effect |
|---------|--------|
| `unbound-control stats_noreset` | Print all stats without resetting counters |
| `unbound-control stats` | Print stats and reset counters to zero |
| `unbound-control cache_lookup <name>` | Show cached records for a name |
| `unbound-control lookup <name>` | Show which nameservers would be queried for a name |
| `unbound-control flush <name>` | Remove all cached records for a name |
| `unbound-control flush_type <name> <type>` | Remove specific record type for a name |
| `unbound-control flush_zone <name>` | Expire all records at or below a zone |
| `unbound-control flush_bogus` | Remove all DNSSEC-failed entries from cache |
| `unbound-control flush_negative` | Remove all NXDOMAIN/SERVFAIL entries |
| `unbound-control dump_cache` | Dump full cache contents to stdout |

### Live DNS Record Management

These take effect immediately, **no reload needed**. Note: lost on container restart since config is read-only. For permanent records, edit `unbound.conf` instead.

| Command | Effect |
|---------|--------|
| `unbound-control local_zone "home.lan." static` | Add authoritative local zone |
| `unbound-control local_data "host.home.lan. A 192.168.x.x"` | Add an A record |
| `unbound-control local_data "host.home.lan. AAAA ::1"` | Add an AAAA record |
| `unbound-control local_data_remove "host.home.lan."` | Remove records for a name |
| `unbound-control local_zone_remove "home.lan."` | Remove a zone and all its records |
| `unbound-control list_local_zones` | List all local zones |
| `unbound-control list_local_data` | List all local DNS records |

### DNSSEC Controls

| Command | Effect |
|---------|--------|
| `unbound-control insecure_add <domain>` | Bypass DNSSEC validation for a domain (live) |
| `unbound-control insecure_remove <domain>` | Re-enable DNSSEC for a domain |
| `unbound-control list_insecure` | List all DNSSEC bypass domains |

### Key Stats Counters

```bash
ssh docker "docker exec unbound unbound-control stats_noreset | grep -E 'queries|cache|bogus|rcode'"
```

| Counter | Meaning |
|---------|---------|
| `total.num.queries` | Total queries received |
| `threadX.num.cachehits` | Queries served from cache |
| `threadX.num.cachemiss` | Queries needing full recursion |
| `num.answer.bogus` | Answers that failed DNSSEC |
| `num.answer.rcode.NXDOMAIN` | NXDOMAIN responses sent |
| `unwanted.replies` | Suspicious unsolicited replies (spoofing indicator) |
| `msg.cache.count` | Replies currently in message cache |
| `rrset.cache.count` | RRsets currently in RRset cache |

---

## Adding Local DNS Records

Two methods — use persistent (config) for records you want to survive container restarts.

### Method 1: Persistent (unbound.conf — survives restarts)

Backup first:
```bash
ssh docker "cp /home/dominik/docker/unbound/unbound.conf /home/dominik/docker/unbound/unbound.conf.bak"
```

Add to the `server:` section of `/home/dominik/docker/unbound/unbound.conf`:
```
# Internal zone
local-zone: "home.lan." static
local-data: "mynas.home.lan.    A 192.168.178.50"
local-data: "myserver.home.lan. A 192.168.178.51"
# Reverse DNS
local-data-ptr: "192.168.178.50 mynas.home.lan"
```

Then validate and reload:
```bash
ssh docker "docker exec unbound unbound-checkconf /opt/unbound/etc/unbound/unbound.conf && docker exec unbound unbound-control reload"
```

**Local zone types:**
| Type | Use case |
|------|----------|
| `static` | Authoritative — returns NXDOMAIN for unknown names in zone |
| `transparent` | Returns local data if present, resolves normally otherwise |
| `redirect` | All subdomains return the same record (wildcard sink) |

### Method 2: Live (unbound-control — lost on restart)

```bash
# Add zone + record
ssh docker "docker exec unbound unbound-control local_zone 'home.lan.' static"
ssh docker "docker exec unbound unbound-control local_data 'mynas.home.lan. A 192.168.178.50'"

# Verify
ssh docker "docker exec unbound unbound-control list_local_data"

# Test it
ssh docker "docker exec unbound drill @127.0.0.1 mynas.home.lan"
```

---

## Operational Runbook

### Update Unbound Image

Unbound gets security updates via image updates. Safe to do with near-zero downtime.

```bash
# 1. Pull new image
ssh docker "docker pull mvance/unbound:latest"

# 2. Check what version will be deployed
ssh docker "docker inspect mvance/unbound:latest --format '{{index .RepoDigests 0}}'"

# 3. Restart container (uses new image automatically)
ssh docker "docker restart unbound"

# 4. Verify it came back healthy
ssh docker "docker ps --filter name=unbound && docker exec unbound unbound-control status"

# 5. Test resolution
ssh docker "docker exec unbound drill @127.0.0.1 google.com"
```

**Note:** If the stack is managed in Portainer, use Portainer → adguard stack → Update the stack to pull and recreate both adguardhome and unbound together.

### Backup Config

```bash
# Backup to timestamped file on the server
ssh docker "cp /home/dominik/docker/unbound/unbound.conf /home/dominik/docker/unbound/unbound.conf.$(date +%Y%m%d)"

# Copy to local Mac for safe keeping
scp docker:/home/dominik/docker/unbound/unbound.conf ~/Desktop/unbound.conf.backup
```

### Restore Config from Backup

```bash
# List available backups
ssh docker "ls -la /home/dominik/docker/unbound/"

# Restore a specific backup
ssh docker "cp /home/dominik/docker/unbound/unbound.conf.YYYYMMDD /home/dominik/docker/unbound/unbound.conf"

# Validate before reloading
ssh docker "docker exec unbound unbound-checkconf /opt/unbound/etc/unbound/unbound.conf"

# Reload
ssh docker "docker exec unbound unbound-control reload"
```

### Update root.hints

root.hints contains the 13 root DNS server addresses. Update every 6-12 months or when resolution mysteriously fails.

```bash
# 1. Backup current
ssh docker "cp /home/dominik/docker/unbound/root.hints /home/dominik/docker/unbound/root.hints.bak"

# 2. Download fresh from IANA
ssh docker "wget -q -O /home/dominik/docker/unbound/root.hints https://www.internic.net/domain/named.root"

# 3. Verify it looks right (should start with '. 3600000 IN NS ...')
ssh docker "head -5 /home/dominik/docker/unbound/root.hints"

# 4. Reload (no restart needed)
ssh docker "docker exec unbound unbound-control reload"
```

### Safe Config Change Procedure

Always follow this sequence — never skip validation:

```bash
# 1. Backup
ssh docker "cp /home/dominik/docker/unbound/unbound.conf /home/dominik/docker/unbound/unbound.conf.bak"

# 2. Edit
ssh docker "nano /home/dominik/docker/unbound/unbound.conf"

# 3. Validate (stops here if syntax error — do NOT proceed if this fails)
ssh docker "docker exec unbound unbound-checkconf /opt/unbound/etc/unbound/unbound.conf"

# 4. Reload (preferred — preserves cache)
ssh docker "docker exec unbound unbound-control reload_keep_cache"

# 5. Verify running
ssh docker "docker exec unbound unbound-control status"

# 6. Test resolution
ssh docker "docker exec unbound drill @127.0.0.1 google.com"

# If something broke — restore immediately:
ssh docker "cp /home/dominik/docker/unbound/unbound.conf.bak /home/dominik/docker/unbound/unbound.conf && docker restart unbound"
```

---

## Troubleshooting Playbook

### Unbound Not Responding / DNS Fails

1. Check if container is running:
   ```bash
   ssh docker "docker ps --filter name=unbound"
   ```
2. Check control status:
   ```bash
   ssh docker "docker exec unbound unbound-control status"
   ```
3. Check recent logs for errors:
   ```bash
   ssh docker "docker logs unbound --tail 100"
   ```
4. Test resolution directly:
   ```bash
   ssh docker "docker exec unbound drill @127.0.0.1 google.com"
   ```
5. If container is stopped, restart:
   ```bash
   ssh docker "docker restart unbound"
   ```

### DNSSEC Validation Failures

Symptom: SERVFAIL on domains that should resolve, or `drill -D` shows `BOGUS`

**Step 1 — Confirm DNSSEC is the cause:**
```bash
# If this succeeds but normal query fails → DNSSEC is the problem
ssh docker "dig <domain> +cd @127.0.0.1"
```

**Step 2 — Enable detailed logging and re-query:**
```bash
ssh docker "docker exec unbound unbound-control set_option val-log-level: 2"
ssh docker "docker exec unbound unbound-control verbosity 2"
ssh docker "docker logs unbound --tail 20"
# Reset when done:
ssh docker "docker exec unbound unbound-control set_option val-log-level: 0"
ssh docker "docker exec unbound unbound-control verbosity 0"
```

**Step 3 — Match log output to cause:**

| Log says | Cause | Fix |
|----------|-------|-----|
| `signature expired` | RRSIG expired or server clock wrong | Check NTP: `ssh docker "timedatectl"` |
| `not yet valid` | Clock skew — Unbound time is behind | Sync NTP on Docker host |
| `no signatures` | Zone has DS record in parent but no RRSIG | Zone owner issue; use `insecure_add` |
| `NSEC3 closest encloser proof failed` | NSEC3 chain broken | Zone owner issue; use `insecure_add` |
| `no supported algorithms` | Algorithm not supported by this Unbound version | Update Unbound image |

**Step 4 — Fix actions:**
```bash
# Flush all cached bogus entries
ssh docker "docker exec unbound unbound-control flush_bogus"

# Bypass DNSSEC for one domain (live, lost on restart)
ssh docker "docker exec unbound unbound-control insecure_add <domain>"

# Check what domains are bypassed
ssh docker "docker exec unbound unbound-control list_insecure"

# For permanent bypass, add to unbound.conf:
# domain-insecure: "<domain>"
```

### Config Change Not Taking Effect

1. Validate config syntax first:
   ```bash
   ssh docker "docker exec unbound unbound-checkconf /opt/unbound/etc/unbound/unbound.conf"
   ```
2. Reload (preferred — no downtime):
   ```bash
   ssh docker "docker exec unbound unbound-control reload"
   ```
3. Verify it's running the new config:
   ```bash
   ssh docker "docker exec unbound unbound-control status"
   ```
4. If reload fails, restart container:
   ```bash
   ssh docker "docker restart unbound"
   ```

### High Latency / Slow DNS

1. Check cache hit stats:
   ```bash
   ssh docker "docker exec unbound unbound-control stats_noreset | grep cache"
   ```
2. Check if serve-expired is working (it should serve stale cache on slow upstreams):
   - `serve-expired: yes` is set in config ✓
3. Prefetch is enabled (`prefetch: yes`) — popular records should be warm

---

## Configuration Reference

Config file: `/home/dominik/docker/unbound/unbound.conf`
Last documented: 2026-02-19

```yaml
server:
    # Basic settings
    verbosity: 0
    module-config: "validator iterator"
    interface: 0.0.0.0
    port: 53
    do-ip4: yes
    do-ip6: yes
    do-udp: yes
    do-tcp: yes

    # IPv6 for cable internet - prefer IPv4 for stability
    prefer-ip4: yes
    outgoing-interface: 0.0.0.0
    target-fetch-policy: "2 1 0 0 0"

    # Cable ISP compatibility
    edns-buffer-size: 1232
    so-reuseport: yes
    infra-host-ttl: 900

    # Security - relaxed for ISP compatibility
    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    harden-referral-path: yes
    harden-below-nxdomain: no
    use-caps-for-id: no
    val-permissive-mode: no

    # Privacy - no logging
    logfile: ""
    log-queries: no

    # Performance - caching
    cache-min-ttl: 600
    cache-max-ttl: 86400
    prefetch: yes
    prefetch-key: yes
    ratelimit: 1000
    aggressive-nsec: yes

    # Performance - threading and connections
    msg-cache-size: 100m
    rrset-cache-size: 200m
    num-threads: 2
    outgoing-range: 819
    num-queries-per-thread: 4096
    so-rcvbuf: 4m
    msg-cache-slabs: 2
    rrset-cache-slabs: 2
    infra-cache-slabs: 2
    key-cache-slabs: 2

    # Performance - serve expired cache
    serve-expired: yes
    serve-expired-ttl: 86400
    serve-expired-client-timeout: 300

    # Access control
    access-control: 127.0.0.0/8 allow
    access-control: 10.0.0.0/8 allow
    access-control: 172.16.0.0/12 allow
    access-control: 192.168.0.0/16 allow
    access-control: ::1 allow
    access-control: fd00::/8 allow
    access-control: fe80::/10 allow

    # DNS rebinding protection
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
    private-address: 192.0.2.0/24
    private-address: 198.51.100.0/24
    private-address: 203.0.113.0/24
    private-address: 255.255.255.255/32
    private-address: 2001:db8::/32

    # Root hints
    root-hints: "/opt/unbound/etc/unbound/root.hints"

    # DNSSEC trust anchor (static)
    trust-anchor: ". IN DS 20326 8 2 E06D44B80B8F1D39A95C0B0D7C65D08458E880409BBC683457104237C7F8EC8D"

remote-control:
    control-enable: yes
```

### Key Settings Summary

| Setting | Value | Notes |
|---------|-------|-------|
| `interface` | `0.0.0.0:53` | Listens on all interfaces inside container |
| `prefer-ip4` | `yes` | Cable ISP stability |
| `num-threads` | `2` | Matches host CPU cores |
| `msg-cache-size` | `100m` | Query response cache |
| `rrset-cache-size` | `200m` | Resource record cache |
| `serve-expired` | `yes` | Serves stale cache while refreshing |
| `cache-min-ttl` | `600` | 10 min minimum TTL |
| `prefetch` | `yes` | Pre-warms popular records |
| `log-queries` | `no` | Privacy — no query logging |

---

## Related Services

- **AdGuard Home** (`skills/adguard.md`): Frontend DNS — all client queries go through AdGuard first, then upstream to Unbound
- **Docker** (`skills/docker.md`): Container lifecycle management
