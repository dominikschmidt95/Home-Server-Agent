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

1. Check if it's a specific domain:
   ```bash
   ssh docker "docker exec unbound drill -D @127.0.0.1 <domain>"
   ```
2. Try with DNSSEC disabled (diagnostic only):
   ```bash
   ssh docker "docker exec unbound drill @127.0.0.1 -k /dev/null <domain>"
   ```
3. Check trust anchor is correct in config (see Configuration Reference)
4. Flush cache and retry:
   ```bash
   ssh docker "docker exec unbound unbound-control flush_zone ."
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
