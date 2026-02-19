---
name: homelab-SERVICE_NAME
description: Use when troubleshooting, configuring, or managing SERVICE_DISPLAY_NAME on the homelab server
---

## Service Overview

Brief description of what this service does and why it's running in the homelab.

- **Purpose:** What problem does this service solve?
- **Dependencies:** What does it depend on? (network, other services, storage)
- **Consumers:** What depends on this service?

---

## Infrastructure

| Property | Value |
|----------|-------|
| Host | `YOUR_SERVER_HOSTNAME_OR_IP` |
| SSH User | `YOUR_SSH_USER` |
| SSH Key | `~/.ssh/YOUR_KEY_NAME` |
| Service Name | `systemd-service-name` or `docker-container-name` |
| Config File(s) | `/etc/service/config.conf` |
| Data Directory | `/var/lib/service/` |
| Log Location | `journalctl -u service` or `/var/log/service/` |

**SSH base command:**
```bash
ssh -i ~/.ssh/YOUR_KEY_NAME YOUR_SSH_USER@YOUR_SERVER_HOSTNAME_OR_IP
```

---

## Common Operations

### Check Service Status
```bash
ssh -i ~/.ssh/YOUR_KEY_NAME YOUR_SSH_USER@YOUR_SERVER_HOSTNAME_OR_IP \
  "systemctl status SERVICE_NAME"
```

### View Logs
```bash
# Last 50 lines
ssh -i ~/.ssh/YOUR_KEY_NAME YOUR_SSH_USER@YOUR_SERVER_HOSTNAME_OR_IP \
  "journalctl -u SERVICE_NAME -n 50 --no-pager"

# Follow live
ssh -i ~/.ssh/YOUR_KEY_NAME YOUR_SSH_USER@YOUR_SERVER_HOSTNAME_OR_IP \
  "journalctl -u SERVICE_NAME -f"
```

### Restart Service
```bash
ssh -i ~/.ssh/YOUR_KEY_NAME YOUR_SSH_USER@YOUR_SERVER_HOSTNAME_OR_IP \
  "sudo systemctl restart SERVICE_NAME && systemctl status SERVICE_NAME"
```

### Reload Config (without restart)
```bash
ssh -i ~/.ssh/YOUR_KEY_NAME YOUR_SSH_USER@YOUR_SERVER_HOSTNAME_OR_IP \
  "sudo systemctl reload SERVICE_NAME"
```

### Edit Config
```bash
# Always backup first
ssh -i ~/.ssh/YOUR_KEY_NAME YOUR_SSH_USER@YOUR_SERVER_HOSTNAME_OR_IP \
  "sudo cp /etc/service/config.conf /etc/service/config.conf.bak"

# Open editor (or use echo/tee for specific changes)
ssh -i ~/.ssh/YOUR_KEY_NAME YOUR_SSH_USER@YOUR_SERVER_HOSTNAME_OR_IP \
  "sudo nano /etc/service/config.conf"
```

### Validate Config
```bash
ssh -i ~/.ssh/YOUR_KEY_NAME YOUR_SSH_USER@YOUR_SERVER_HOSTNAME_OR_IP \
  "SERVICE_NAME --configtest"
```

---

## Troubleshooting Playbook

### Service Won't Start

1. Check status for error message:
   ```bash
   ssh ... "systemctl status SERVICE_NAME"
   ```
2. Check logs for details:
   ```bash
   ssh ... "journalctl -u SERVICE_NAME -n 100 --no-pager"
   ```
3. Validate config syntax:
   ```bash
   ssh ... "SERVICE_NAME --configtest"
   ```
4. Check for port conflicts:
   ```bash
   ssh ... "sudo ss -tlnp | grep PORT_NUMBER"
   ```
5. Check file permissions:
   ```bash
   ssh ... "ls -la /etc/service/"
   ```

### Service Starts But Doesn't Work

1. Verify it's listening on the expected port:
   ```bash
   ssh ... "sudo ss -tlnp | grep PORT_NUMBER"
   ```
2. Test the service locally from the server:
   ```bash
   ssh ... "SERVICE_CLIENT_COMMAND test"
   ```
3. Check firewall allows the port:
   ```bash
   ssh ... "sudo ufw status | grep PORT_NUMBER"
   ```

### Config Change Didn't Take Effect

1. Validate config:
   ```bash
   ssh ... "SERVICE_NAME --configtest"
   ```
2. Reload (preferred, no downtime):
   ```bash
   ssh ... "sudo systemctl reload SERVICE_NAME"
   ```
3. If reload not supported, restart:
   ```bash
   ssh ... "sudo systemctl restart SERVICE_NAME"
   ```
4. Verify change is live:
   ```bash
   ssh ... "SERVICE_NAME --verify-command"
   ```

---

## Configuration Reference

Document the actual running configuration here. Update this section whenever the config changes.

```
# /etc/service/config.conf
# Last updated: YYYY-MM-DD

# Paste actual config here
```

### Key Settings

| Setting | Value | Description |
|---------|-------|-------------|
| `setting_name` | `value` | What this controls |

---

## Known Issues

Document any known bugs, quirks, or recurring problems here.

### Issue: [Title]
- **Symptom:** What you see
- **Cause:** Why it happens
- **Fix:** Exact commands to resolve it

---

## Related Services

- **Service A:** How it relates
- **Service B:** How it relates
