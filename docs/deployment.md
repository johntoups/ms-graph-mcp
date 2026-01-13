---
title: MCP Server Deployment Guide
type: operational-guide
created: 2025-11-23
status: active
---

# MCP Server Deployment Guide

Comprehensive deployment procedures for Microsoft Graph MCP server.

**Last Updated:** 2025-12-25

---

## Production Deployment: Hermes (Current)

**As of 2025-12-25, production M365 MCP runs on Hermes via WireGuard tunnel.**

| Property | Value |
|----------|-------|
| **Location** | `/opt/services/m365-mcp/` |
| **Port** | 10.100.0.1:8000 (WireGuard interface) |
| **Container** | `m365-mcp` |
| **Primary Access** | WireGuard tunnel from Croft network |
| **Fallback Access** | SSH tunnel (`ssh hermes-mcp`) |
| **Claude Code URL** | `http://10.100.0.1:8000/mcp` |

### Access Architecture

```
Hermes VPS (77.42.26.29)
├── WireGuard: 10.100.0.1 (wg0 interface)
│   └── MCP bound to 10.100.0.1:8000
│
└── WireGuard Tunnel (UDP 51820)
    │
    ▼
UDM Pro (Croft)
├── WireGuard: 10.100.0.2
│
└── Croft Network (172.30.0.0/16)
    ├── Laptop (via Teleport or direct)
    ├── Athena Workspace (atlas)
    └── Overwatch Workspace (atlas)
```

### Client Configuration

**Claude Code MCP config (`~/.mcp.json`):**
```json
{
  "mcpServers": {
    "microsoft-graph-email": {
      "url": "http://10.100.0.1:8000/mcp",
      "transport": "http"
    }
  }
}
```

**Required:** Client must be on Croft network (direct or via Teleport)

### Quick Commands (Hermes)

**Check status:**
```bash
ssh hermes 'docker ps | grep m365'
```

**View logs:**
```bash
ssh hermes 'docker logs m365-mcp --tail 50'
```

**Restart:**
```bash
ssh hermes 'cd /opt/services/m365-mcp && docker compose restart'
```

**Test WireGuard connectivity:**
```bash
ping 10.100.0.1  # Should respond ~20-50ms from Croft LAN
curl -s http://10.100.0.1:8000/mcp  # Returns JSON-RPC error (expected - needs SSE client)
```

**Test MCP tools (in Claude Code):**
```
mcp__microsoft-graph-email__test_connection
```

### Fallback: SSH Tunnel Access

When not on Croft network and WireGuard unavailable:

```bash
# Terminal 1: Establish tunnel
ssh hermes-mcp  # Forwards localhost:8000 → 127.0.0.1:8000

# Update ~/.mcp.json temporarily:
# "url": "http://localhost:8000/mcp"
```

### Deploy Code Update to Hermes

```bash
# From laptop (on Croft network or with Hermes SSH access)
rsync -av /home/iris/Athena/project/mcp/microsoft-graph/code/ \
  hermes:/opt/services/m365-mcp/code/ \
  --exclude .env --exclude __pycache__ --exclude "*.backup*"

# Restart container
ssh hermes 'cd /opt/services/m365-mcp && docker compose restart'
```

### Migration History

- **2025-12-25**: WireGuard tunnel established as primary access
- **2025-12-22**: Migrated from atlas to Hermes
- OAuth credentials preserved via Docker volume

---

## Development Environments: atlas (Legacy)

The following section documents the atlas multi-environment setup.
These environments may be used for development/testing but production has moved to Hermes.

---

## Quick Reference

### Environment Overview

| Name | Path | Port | Container | Use For |
|------|------|------|-----------|---------|
| **exp** | `/opt/overwatch/mcp-servers/m365-exp/` | 8002 | `m365-mcp-exp` | Testing experiments, new features |
| **dev** | `/opt/overwatch/mcp-servers/m365-dev/` | 8001 | `m365-mcp-dev` | Staging, validation before production |
| **prod** | `/opt/overwatch/mcp-servers/m365/` | 8000 | `m365-mcp` | Live production (Claude Code default) |

### Environment Selection Guide

| Scenario | Environment | Rationale |
|----------|-------------|-----------|
| Testing new feature | **experimental** (8002) | Isolated, won't break dev/prod |
| Bug fix (small) | **development** (8001) | Validate before production |
| Bug fix (critical) | **dev first, then prod** | Quick path with validation |
| Production hotfix | **production** (8000) directly | Only when downtime unacceptable |
| Feature promotion | **exp → dev → prod** | Full validation pipeline |

**Default rule:** When in doubt, use **development** (8001).

### Source Code Locations

**Workspace (Development):**
- Path: `~/athena/project/mcp/microsoft-graph/code/`
- Main file: `mcp_server_v2.py` (4953 lines)
- Dependencies: `requirements.txt`
- Config: `.env.template`

**Production Server:**
- Base: `/opt/overwatch/mcp-servers/`
- Each environment: `m365/`, `m365-dev/`, `m365-exp/`
- Code directory: `{env}/code/`

### Shared Resources

- **OAuth tokens:** `m365-mcp-credentials` Docker volume (shared across all environments)
- **Network:** `mcp-network` bridge
- **Image:** `python:3.12-slim`

---

## Copy-Paste Commands

### Status Check

**View all MCP containers:**
```bash
ssh atlas 'docker ps --filter "name=m365-mcp" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'
```

**Check specific container logs:**
```bash
# Experimental
ssh atlas 'docker logs m365-mcp-exp --tail 50'

# Development
ssh atlas 'docker logs m365-mcp-dev --tail 50'

# Production
ssh atlas 'docker logs m365-mcp --tail 50'
```

### Backup (Before Editing)

**Experimental environment:**
```bash
ssh atlas 'cd /opt/overwatch/mcp-servers/m365-exp/code && \
  cp mcp_server_v2.py mcp_server_v2.py.backup-$(date +%Y%m%d-%H%M%S) && \
  echo "✅ Backup created: $(ls -lt mcp_server_v2.py.backup-* | head -1 | awk '\''{print $9}'\'')"'
```

**Development environment:**
```bash
ssh atlas 'cd /opt/overwatch/mcp-servers/m365-dev/code && \
  cp mcp_server_v2.py mcp_server_v2.py.backup-$(date +%Y%m%d-%H%M%S) && \
  echo "✅ Backup created: $(ls -lt mcp_server_v2.py.backup-* | head -1 | awk '\''{print $9}'\'')"'
```

**Production environment:**
```bash
ssh atlas 'cd /opt/overwatch/mcp-servers/m365/code && \
  cp mcp_server_v2.py mcp_server_v2.py.backup-$(date +%Y%m%d-%H%M%S) && \
  echo "✅ Backup created: $(ls -lt mcp_server_v2.py.backup-* | head -1 | awk '\''{print $9}'\'')"'
```

### Restart Container

**Experimental:**
```bash
ssh atlas 'cd /opt/overwatch/mcp-servers/m365-exp && docker compose restart'
```

**Development:**
```bash
ssh atlas 'cd /opt/overwatch/mcp-servers/m365-dev && docker compose restart'
```

**Production:**
```bash
ssh atlas 'cd /opt/overwatch/mcp-servers/m365 && docker compose restart'
```

### Health Check

**Note:** Health check validation depends on MCP proxy response format. Verify actual endpoint behavior.

**Experimental (port 8002):**
```bash
ssh atlas 'curl -s http://localhost:8002 | head -10 || echo "❌ Server not responding"'
```

**Development (port 8001):**
```bash
ssh atlas 'curl -s http://localhost:8001 | head -10 || echo "❌ Server not responding"'
```

**Production (port 8000):**
```bash
ssh atlas 'curl -s http://localhost:8000 | head -10 || echo "❌ Server not responding"'
```

**TODO:** Verify if mcp-proxy exposes `/health` endpoint for better validation.

### Full Validation Pipeline

**Development (restart + wait + logs + health):**
```bash
ssh atlas 'cd /opt/overwatch/mcp-servers/m365-dev && \
  docker compose restart && \
  sleep 3 && \
  docker logs m365-mcp-dev --tail 20 && \
  curl -s http://localhost:8001 | head -5'
```

**Production:**
```bash
ssh atlas 'cd /opt/overwatch/mcp-servers/m365 && \
  docker compose restart && \
  sleep 3 && \
  docker logs m365-mcp --tail 20 && \
  curl -s http://localhost:8000 | head -5'
```

### Sync from Workspace

**To experimental:**
```bash
ssh athena.coder 'rsync -av ~/athena/project/mcp/microsoft-graph/code/ \
  atlas:/opt/overwatch/mcp-servers/m365-exp/code/ \
  --exclude .env --exclude __pycache__ --exclude "*.backup*"'
```

**To development:**
```bash
ssh athena.coder 'rsync -av ~/athena/project/mcp/microsoft-graph/code/ \
  atlas:/opt/overwatch/mcp-servers/m365-dev/code/ \
  --exclude .env --exclude __pycache__ --exclude "*.backup*"'
```

**To production:**
```bash
ssh athena.coder 'rsync -av ~/athena/project/mcp/microsoft-graph/code/ \
  atlas:/opt/overwatch/mcp-servers/m365/code/ \
  --exclude .env --exclude __pycache__ --exclude "*.backup*"'
```

### Quick Rollback

**Development (rollback to most recent backup):**
```bash
ssh atlas 'cd /opt/overwatch/mcp-servers/m365-dev/code && \
  LATEST_BACKUP=$(ls -t mcp_server_v2.py.backup-* 2>/dev/null | head -1) && \
  if [ -n "$LATEST_BACKUP" ]; then \
    cp "$LATEST_BACKUP" mcp_server_v2.py && \
    echo "✅ Rolled back to: $LATEST_BACKUP"; \
  else \
    echo "❌ No backups found"; \
  fi'
```

**Production:**
```bash
ssh atlas 'cd /opt/overwatch/mcp-servers/m365/code && \
  LATEST_BACKUP=$(ls -t mcp_server_v2.py.backup-* 2>/dev/null | head -1) && \
  if [ -n "$LATEST_BACKUP" ]; then \
    cp "$LATEST_BACKUP" mcp_server_v2.py && \
    echo "✅ Rolled back to: $LATEST_BACKUP"; \
  else \
    echo "❌ No backups found"; \
  fi'
```

### Log Viewing (Debugging)

**Follow logs in real-time (Ctrl+C to exit):**
```bash
# Development
ssh atlas 'docker logs -f m365-mcp-dev'

# Production
ssh atlas 'docker logs -f m365-mcp'
```

**View logs with timestamps:**
```bash
ssh atlas 'docker logs m365-mcp-dev --timestamps --tail 50'
```

**Search logs for errors:**
```bash
ssh atlas 'docker logs m365-mcp-dev 2>&1 | grep -i error'
```

**View container startup logs (first 100 lines):**
```bash
ssh atlas 'docker logs m365-mcp-dev --tail 100'
```

---

## Deployment Procedures

### Scenario A: Quick Fix (Direct Server Edit)

**Use when:** Small bug fix, typo, quick change
**Environment:** Development first, then production

**Steps:**

- [ ] **1. Backup current code**
  ```bash
  ssh atlas 'cd /opt/overwatch/mcp-servers/m365-dev/code && \
    cp mcp_server_v2.py mcp_server_v2.py.backup-$(date +%Y%m%d-%H%M%S) && \
    echo "✅ Backup created"'
  ```

- [ ] **2. Edit file directly on server**
  Use Read/Edit tools on:
  `/opt/overwatch/mcp-servers/m365-dev/code/mcp_server_v2.py` (via SSH to atlas)

- [ ] **3. Restart container**
  ```bash
  ssh atlas 'cd /opt/overwatch/mcp-servers/m365-dev && docker compose restart'
  ```

- [ ] **4. Wait for startup**
  ```bash
  sleep 3
  ```

- [ ] **5. Check logs for errors**
  ```bash
  ssh atlas 'docker logs m365-mcp-dev --tail 20'
  ```

- [ ] **6. Validate server responding**
  ```bash
  ssh atlas 'curl -s http://localhost:8001 | head -5'
  ```

- [ ] **7. Test functionality**
  Use MCP tool to verify fix works (e.g., `test_connection` or relevant tool)

- [ ] **8. If successful: Deploy to production**
  Repeat steps 1-7 for production environment:
  - Path: `/opt/overwatch/mcp-servers/m365/code/`
  - Container: `m365-mcp`
  - Port: `8000`

- [ ] **9. Sync changes back to workspace Git** (See "Git Sync-Back" section below)

**Common pitfall:** Forgetting step 9 means Git becomes stale and next workspace deploy overwrites your fixes.

---

### Scenario B: Feature Development (Workspace First)

**Use when:** New feature, major change, needs testing
**Environment:** Experimental → Development → Production

**Steps:**

- [ ] **1. Develop in workspace**
  Edit: `~/athena/project/mcp/microsoft-graph/code/mcp_server_v2.py` (in Athena workspace)

- [ ] **2. Commit to experimental branch**
  ```bash
  ssh athena.coder 'cd ~/athena && \
    git checkout experimental && \
    git add project/mcp/microsoft-graph/code/ && \
    git commit -m "Feature: [description]"'
  ```

- [ ] **3. Backup experimental environment**
  ```bash
  ssh atlas 'cd /opt/overwatch/mcp-servers/m365-exp/code && \
    cp mcp_server_v2.py mcp_server_v2.py.backup-$(date +%Y%m%d-%H%M%S)'
  ```

- [ ] **4. Sync workspace → experimental**
  ```bash
  ssh athena.coder 'rsync -av ~/athena/project/mcp/microsoft-graph/code/ \
    atlas:/opt/overwatch/mcp-servers/m365-exp/code/ \
    --exclude .env --exclude __pycache__ --exclude "*.backup*"'
  ```

- [ ] **5. Restart experimental container**
  ```bash
  ssh atlas 'cd /opt/overwatch/mcp-servers/m365-exp && docker compose restart'
  ```

- [ ] **6. Validate experimental (port 8002)**
  ```bash
  ssh atlas 'sleep 3 && \
    docker logs m365-mcp-exp --tail 20 && \
    curl -s http://localhost:8002 | head -5'
  ```

- [ ] **7. Test functionality thoroughly**
  Use MCP tools via port 8002 to validate new feature

- [ ] **8. If successful: Promote to development**
  ```bash
  # Merge experimental → development branch
  ssh athena.coder 'cd ~/athena && \
    git checkout development && \
    git merge experimental && \
    git push origin development'

  # Deploy to dev environment (repeat steps 3-6 for m365-dev)
  ```

- [ ] **9. If dev successful: Promote to production**
  ```bash
  # Merge development → main branch
  ssh athena.coder 'cd ~/athena && \
    git checkout main && \
    git merge development && \
    git push origin main'

  # Deploy to production (repeat steps 3-6 for m365)
  ```

---

### Scenario C: Rollback (Something Broke)

**Use when:** Deployment failed, server not responding, critical error

**Steps:**

- [ ] **1. Quick rollback (automatic - finds most recent backup)**
  ```bash
  # Development
  ssh atlas 'cd /opt/overwatch/mcp-servers/m365-dev/code && \
    LATEST_BACKUP=$(ls -t mcp_server_v2.py.backup-* 2>/dev/null | head -1) && \
    cp "$LATEST_BACKUP" mcp_server_v2.py && \
    echo "✅ Rolled back to: $LATEST_BACKUP"'

  # Production
  ssh atlas 'cd /opt/overwatch/mcp-servers/m365/code && \
    LATEST_BACKUP=$(ls -t mcp_server_v2.py.backup-* 2>/dev/null | head -1) && \
    cp "$LATEST_BACKUP" mcp_server_v2.py && \
    echo "✅ Rolled back to: $LATEST_BACKUP"'
  ```

- [ ] **2. Restart container**
  ```bash
  ssh atlas 'cd /opt/overwatch/mcp-servers/m365-dev && docker compose restart'
  ```

- [ ] **3. Validate restoration**
  ```bash
  ssh atlas 'sleep 3 && \
    docker logs m365-mcp-dev --tail 20 && \
    curl -s http://localhost:8001'
  ```

- [ ] **4. Investigate failure**
  Review logs, identify what went wrong:
  ```bash
  ssh atlas 'docker logs m365-mcp-dev --tail 100 | grep -i error'
  ```

**Manual rollback (if you need specific backup):**
```bash
# List available backups
ssh atlas 'ls -lt /opt/overwatch/mcp-servers/m365-dev/code/*.backup*'

# Restore specific backup
ssh atlas 'cd /opt/overwatch/mcp-servers/m365-dev/code && \
  cp mcp_server_v2.py.backup-YYYYMMDD-HHMMSS mcp_server_v2.py'
```

---

### Git Sync-Back (After Direct Server Edit)

**When to use:** After Scenario A (direct server edit), sync changes back to workspace Git

**Critical:** Prevents Git drift. If you skip this, next workspace deployment overwrites your production fixes.

**Steps:**

- [ ] **1. Copy changed files to workspace**
  ```bash
  # From development environment
  ssh athena.coder 'ssh atlas "cat /opt/overwatch/mcp-servers/m365-dev/code/mcp_server_v2.py" > ~/athena/project/mcp/microsoft-graph/code/mcp_server_v2.py'

  # Or from production
  ssh athena.coder 'ssh atlas "cat /opt/overwatch/mcp-servers/m365/code/mcp_server_v2.py" > ~/athena/project/mcp/microsoft-graph/code/mcp_server_v2.py'
  ```

- [ ] **2. Review changes in workspace**
  ```bash
  ssh athena.coder 'cd ~/athena && git diff project/mcp/microsoft-graph/code/'
  ```

- [ ] **3. Commit to appropriate branch**
  ```bash
  ssh athena.coder 'cd ~/athena && \
    git checkout development && \
    git add project/mcp/microsoft-graph/code/ && \
    git commit -m "Fix: [description of fix made on server]" && \
    git push origin development'
  ```

  **Branch selection:**
  - Quick fix → `development` or `main` (depending on urgency)
  - Experimental work → `experimental` first
  - Production hotfix → `main` directly (then backport to dev/exp if needed)

- [ ] **4. Verify sync complete**
  ```bash
  ssh athena.coder 'cd ~/athena && git log --oneline -3'
  ```

**Alternative (copy entire code directory):**
```bash
ssh athena.coder 'rsync -av atlas:/opt/overwatch/mcp-servers/m365-dev/code/ \
  ~/athena/project/mcp/microsoft-graph/code/ \
  --exclude .env --exclude __pycache__ --exclude "*.backup*"'
```

---

## Troubleshooting

### Container Won't Start

**Symptoms:** `docker ps` shows container not running or constantly restarting

**Diagnosis:**
```bash
# View full startup logs
ssh atlas 'docker logs m365-mcp-dev'

# Check container status
ssh atlas 'docker ps -a --filter "name=m365-mcp-dev"'
```

**Common causes:**
- Syntax error in Python code (check logs for traceback)
- Missing dependency in `requirements.txt`
- Port conflict (another service on 8000/8001/8002)
- Volume mount issue (credentials volume not accessible)

**Fix:**
1. Review logs for Python exceptions
2. Rollback to last backup
3. Fix issue in workspace
4. Redeploy

---

### Server Not Responding After Restart

**Symptoms:** Health check fails, curl times out

**Diagnosis:**
```bash
# Container running?
ssh atlas 'docker ps --filter "name=m365-mcp-dev"'

# Logs showing errors?
ssh atlas 'docker logs m365-mcp-dev --tail 50'

# Port exposed correctly?
ssh atlas 'docker port m365-mcp-dev'
```

**Common causes:**
- Container started but Python crashed (check logs)
- MCP proxy failed to start
- Network configuration issue

**Fix:**
1. Check logs for Python exceptions
2. Verify docker-compose.yml port mappings
3. Test port accessibility: `curl http://172.30.100.100:8001`
4. If MCP proxy issue, check mcp-proxy installation in logs

---

### Changes Not Reflected After Deployment

**Symptoms:** Old behavior persists after deploy

**Diagnosis:**
```bash
# Container actually restarted? (check uptime)
ssh atlas 'docker ps --filter "name=m365-mcp-dev" --format "{{.Status}}"'

# Code actually synced? (check file timestamp)
ssh atlas 'ls -lh /opt/overwatch/mcp-servers/m365-dev/code/mcp_server_v2.py'

# Compare workspace vs production file size
ssh athena.coder 'ls -lh ~/athena/project/mcp/microsoft-graph/code/mcp_server_v2.py'
```

**Common causes:**
- Container didn't actually restart (Docker cached)
- Rsync didn't sync (check rsync output)
- Testing wrong port (8000 vs 8001 vs 8002)
- Python bytecode cache (old .pyc files)

**Fix:**
1. Force container restart: `docker compose down && docker compose up -d`
2. Verify file contents: `head -20 /path/to/mcp_server_v2.py`
3. Clear Python cache: `rm -rf __pycache__`
4. Verify correct port in Claude Code MCP config

---

### OAuth Tokens Lost/Broken

**Symptoms:** Authentication errors, "token not found", device code prompts

**Diagnosis:**
```bash
# Volume mounted correctly?
ssh atlas 'docker inspect m365-mcp-dev | grep m365-mcp-credentials'

# Files exist in volume?
ssh atlas 'docker exec m365-mcp-dev ls -la /root/.athena/credentials/'
```

**Common causes:**
- Volume not mounted (check docker-compose.yml)
- Credentials expired (90-day refresh token limit)
- File permissions issue

**Critical:** OAuth tokens are shared via Docker volume across all environments. If broken in one, may affect all.

**Fix:**
1. Check volume mount in docker-compose.yml
2. Verify volume exists: `docker volume ls | grep m365-mcp-credentials`
3. If needed, re-authenticate (device code flow will prompt)
4. Check logs for specific auth errors

---

### Backup/Restore Issues

**Symptoms:** Backup command fails, rollback can't find backups

**Diagnosis:**
```bash
# List all backups
ssh atlas 'ls -lt /opt/overwatch/mcp-servers/m365-dev/code/*.backup*'

# Check disk space
ssh atlas 'df -h /opt/overwatch/mcp-servers/'

# Verify backup file integrity
ssh atlas 'wc -l /opt/overwatch/mcp-servers/m365-dev/code/mcp_server_v2.py.backup-*'
```

**Common causes:**
- Disk space full (backups can't write)
- Permission issues (can't write to directory)
- Backup filename pattern changed

**Fix:**
1. Clean old backups: `rm /path/to/code/*.backup-2024*` (keep recent only)
2. Check directory permissions
3. Manually verify backup is valid Python file

---

## Best Practices

### Before Every Deployment

1. ✅ **Always backup first** - Even for "tiny changes"
2. ✅ **Deploy to dev/exp first** - Never deploy untested code to production
3. ✅ **Check logs after restart** - Verify clean startup
4. ✅ **Test functionality** - Don't assume it works, verify it
5. ✅ **Sync back to Git** - Prevent drift when editing servers directly

### During Development

1. ✅ **Use branches properly** - experimental → development → main
2. ✅ **Commit before deploying** - Git is source of truth
3. ✅ **Test in isolation** - experimental environment for risky changes
4. ✅ **Document changes** - Update this file if deployment process changes

### Emergency Situations

1. ✅ **Production down? Rollback immediately** - Fix later
2. ✅ **Can't rollback? Check backup list** - Manual restore if needed
3. ✅ **Logs are your friend** - Don't guess, check logs
4. ✅ **Test rollback procedure periodically** - Ensure backups work

---

## Quick Reference Card

**Most common operations:**

```bash
# Deploy from workspace to dev
ssh athena.coder 'rsync -av ~/athena/project/mcp/microsoft-graph/code/ atlas:/opt/overwatch/mcp-servers/m365-dev/code/ --exclude .env --exclude __pycache__' && ssh atlas 'cd /opt/overwatch/mcp-servers/m365-dev && docker compose restart'

# Quick rollback dev
ssh atlas 'cd /opt/overwatch/mcp-servers/m365-dev/code && cp $(ls -t mcp_server_v2.py.backup-* | head -1) mcp_server_v2.py && cd .. && docker compose restart'

# Check all container status
ssh atlas 'docker ps --filter "name=m365-mcp"'

# View recent logs (dev)
ssh atlas 'docker logs m365-mcp-dev --tail 20'
```

---

**Document Status:** Active
**Last Updated:** 2025-11-23
**Maintained By:** Athena system documentation
**Related:** [README.md](README.md), [implementation.md](implementation.md)
