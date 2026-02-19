# Nginx Proxy Manager Skill — Design Document

**Date:** 2026-02-19
**Status:** Implemented
**Deliverable:** `skills/nginx.md`

---

## Context

The homelab already has skills for AdGuard Home, Unbound, and Docker/Portainer. Nginx Proxy Manager is the reverse proxy gateway for all services but had no skill. The goal was to create a skill that makes the agent a self-sufficient NPM operator — no guessing, no improvised commands.

---

## Discovery Findings

Everything below was confirmed live against the server before writing the skill.

### Infrastructure

| Property | Value |
|----------|-------|
| Image | `jc21/nginx-proxy-manager:latest` |
| App container | `ngnix-app-1` |
| DB container | `ngnix-db-1` (MariaDB) |
| Admin UI | `http://192.168.178.26:82` |
| Ports | 80 (HTTP), 443 (HTTPS), 82→81 (admin) |
| Data | `/data/compose/14/data/` |
| SSL certs | `/data/compose/14/letsencrypt/` |

### Network Architecture

NPM sits on two Docker networks:
- `ngnix_default` (`172.20.0.0/16`): internal app↔db communication only
- `proxy_network` (`172.23.0.0/16`): shared with all proxied Docker containers

The `proxy_network` is the key insight: any container on this network is reachable from NPM by container name (Docker embedded DNS). No IP addresses needed. LAN devices (Proxmox, Home Assistant, ZoneMinder) are reached by their LAN IP instead.

### All Configured Proxy Hosts (11 total)

| Domain | Backend | Port | Scheme |
|--------|---------|------|--------|
| `ha.dschmidt95.de` | `192.168.178.171` | 8123 | http |
| `bw.dschmidt95.de` | `bitwarden-bitwarden-1` | 8080 | http |
| `zm.dschmidt95.de` | `192.168.178.11` | 80 | http |
| `adguard.home.dschmidt95.de` | `adguardhome` | 80 | http |
| `beszel.dschmidt95.de` | `beszel` | 8090 | http |
| `ngnix.home.dschmidt95.de` | `192.168.178.26` | 82 | http |
| `pve.home.dschmidt95.de` | `192.168.178.242` | 8006 | https |
| `docker.home.dschmidt95.de` | `192.168.178.26` | 9443 | https |
| `zm.home.dschmidt95.de` | `192.168.178.11` | 80 | http |
| `n8n.home.dschmidt95.de` | `n8n` | 5678 | http |
| `paperless.home.dschmidt95.de` | `paperless-ngx` | 8000 | http |

### SSL

Let's Encrypt via Certbot with Cloudflare DNS-01 challenge plugin. Certificates stored under `/data/compose/14/letsencrypt/live/npm-<id>/`. Auto-renewal handled by NPM on a timer.

---

## Skill Sections

The skill (`skills/nginx.md`) covers:

1. **Service Overview** — role of NPM, SSL approach, MariaDB dependency
2. **Infrastructure** — all paths, ports, container names
3. **Proxy Host Inventory** — complete table of all 11 proxy hosts + `proxy_network` IPs
4. **Network Architecture** — how `proxy_network` enables container-name routing
5. **Common Operations** — status check, logs, restart, reload, config test
6. **Adding a New Proxy Host** — decision tree (Docker vs LAN), step-by-step for both paths
7. **SSL / Let's Encrypt** — check expiry, view renewal logs, force renewal
8. **Log Reference** — which log file belongs to which proxy host, how to read them
9. **Config File Reference** — where generated nginx configs live, key variables
10. **Troubleshooting Playbook** — 502, 504, cert errors, NPM won't start, config not applying, container not reachable by name
11. **Update NPM** — safe pull & recreate workflow
12. **Related Services** — docker.md, adguard.md, and their relationship to NPM
