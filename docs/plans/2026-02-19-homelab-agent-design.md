# Homelab Manager Agent — Design & Implementation

**Date:** 2026-02-19
**Status:** Implemented

---

## Overview

A Claude Code-native homelab manager agent. Lives as a project directory opened in Claude Code. Each service has a dedicated skill file that the agent invokes when asked about that service. Claude Code's built-in Bash approval is the only safety gate — no custom tooling required.

---

## Architecture

### Why Claude Code Skills?

- Zero infrastructure: no server, no API, no daemon
- Skills are just Markdown — easy to update as the homelab evolves
- Bash approval gives the user a confirmation step for every SSH command
- CLAUDE.md gives the agent context that persists across every session automatically

### Project Layout

```
Home-Server-Agent/
├── CLAUDE.md                          # Agent brain: SSH config, service inventory, golden rules
├── .claude/
│   └── settings.json                  # Pre-approved tools (ssh, dig, ping)
├── skills/
│   ├── _template.md                   # Blank template for new services
│   ├── unbound.md                     # Unbound recursive resolver
│   ├── adguard.md                     # AdGuard Home DNS filtering
│   └── docker.md                      # Docker / Portainer stack management
└── docs/
    └── plans/
        └── 2026-02-19-homelab-agent-design.md
```

---

## Live Infrastructure (as of 2026-02-19)

### Server

| Property | Value |
|----------|-------|
| IP | `192.168.178.26` |
| SSH alias | `docker` |
| SSH user | `dominik` |
| SSH key | `~/.ssh/id_ed25519` |

### DNS Architecture

```
LAN clients
    ↓  port 53 (public)
AdGuard Home (192.168.178.26)
    ↓  upstream: 172.24.255.254:53 (internal Docker network)
Unbound (mvance/unbound:latest)
    ↓  recursive queries
Root DNS servers
```

**DNS rewrite:** `*.home.dschmidt95.de` → `192.168.178.26`

### Docker Stacks

| Stack | Purpose | Containers |
|-------|---------|-----------|
| `adguard` | DNS (AdGuard + Unbound) | adguardhome, unbound |
| `paperlessngx` | Document management | paperless-ngx, paperless-db, paperless-redis |
| `n8n` | Workflow automation | n8n |
| `bitwarden` | Password manager (Vaultwarden) | bitwarden-bitwarden-1, bitwarden-db-1 |
| `portainer` | Docker management UI | portainer |
| `ngnix` | Nginx Proxy Manager | ngnix-app-1, ngnix-db-1 |
| `beszel` | Server monitoring | beszel, beszel-agent |
| `cloudflare-ddns` | Dynamic DNS | cloudflare-ddns |

---

## Skill File Format

Each `skills/<service>.md` follows the Claude Code skill format:

```
---
name: homelab-<service>
description: Use when [trigger conditions]
---
```

Sections in every skill:
1. **Service Overview** — what it does, dependencies, consumers
2. **Infrastructure** — host, paths, ports, container names
3. **Common Operations** — named operations with exact SSH commands
4. **Troubleshooting Playbook** — step-by-step diagnostics for common failures
5. **Configuration Reference** — actual config documented inline
6. **Related Services** — cross-references to other skills

---

## Agent Golden Rules (from CLAUDE.md)

1. Backup before modifying (`cp file file.bak`)
2. Verify service health after every change
3. Confirm before any network/firewall changes
4. Use exact SSH commands from the active skill file
5. Report what was run and what the output was
6. One change at a time with intermediate verification

---

## How to Use

```bash
# Open the agent
cd /path/to/Home-Server-Agent
claude

# Example queries the agent can handle:
# "Check if unbound is running"
# "Show me the last 50 AdGuard log lines"
# "Add a DNS rewrite for mynas.home.dschmidt95.de → 192.168.178.50"
# "List all running Docker containers"
# "Restart the paperless-ngx container"
# "Show me the Unbound cache stats"
```

---

## Adding New Services

1. Copy `skills/_template.md` → `skills/<service>.md`
2. Fill in all sections with real values (SSH in to gather facts)
3. Add row to the Service Inventory table in `CLAUDE.md`

---

## Future Skills

- `pihole.md` — if switching from AdGuard
- `opnsense.md` — firewall/routing (if added)
- `wireguard.md` — VPN
- `grafana.md` — monitoring dashboards
- `traefik.md` — reverse proxy (if replacing Nginx PM)
- `homeassistant.md` — home automation
- `nginx-proxy-manager.md` — Nginx PM config and proxy hosts
