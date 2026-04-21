---
name: ai-chat-server-deploy
description: Use when deploying AI-chat to a new server, updating AI-chat code on the existing server, rebuilding Docker services, or redeploying the website without changing database contents. Use for safe server-side code deployment that preserves /opt/AI-chat-data and verifies health on port 8181.
---

# AI-chat Server Deploy

## Overview

Deploy or update the AI-chat application on the server without touching live database contents.

Core rule: only change program files under `/opt/AI-chat`; never overwrite or delete `/opt/AI-chat-data`.

## Fixed Assumptions

- Code directory: `/opt/AI-chat`
- Active database directory: `/opt/AI-chat-data/db`
- Backup directory: `/opt/AI-chat-data/backups`
- Public HTTP port: `8181`
- Backend container mount: `/opt/AI-chat-data/db -> /app/data`
- Backend database env: `SQLITE_PATH=/app/data/data.sqlite`

## Use This Skill For

- First deployment on a new server
- Updating AI-chat from GitHub
- Replacing code with a new uploaded package
- Rebuilding Docker services after code/config changes
- Checking whether the deployment is still wired to the isolated data directory

## Do Not Use This Skill For

- Importing a different SQLite database snapshot
- Exporting data from another server
- Rolling back to a previous database backup

Use `ai-chat-db-migration` for any database-content change.

## Safety Rules

- Never delete `/opt/AI-chat-data`
- Never replace files inside `/opt/AI-chat-data/db`
- Never point the backend back to `/opt/AI-chat/backend/data.sqlite`
- If `docker-compose.yml` or `.env` would break the isolated database path, stop and fix that first
- If the current server state is unclear, inspect before rebuilding

## Standard Flow

### 1. Inspect Current State

Run these checks first:

```bash
ssh admin@SERVER_IP "cd /opt/AI-chat && docker compose ps"
ssh admin@SERVER_IP "curl -s http://127.0.0.1:8181/api/health"
ssh admin@SERVER_IP "docker inspect ai-chat-backend --format '{{json .Mounts}}'"
```

Confirm the backend is mounted from `/opt/AI-chat-data/db`.

### 2. Confirm Required Paths

Ensure these exist:

```bash
ssh admin@SERVER_IP "ls -lah /opt/AI-chat /opt/AI-chat-data /opt/AI-chat-data/db /opt/AI-chat-data/backups"
```

If `/opt/AI-chat-data/db` is missing, stop and repair the deployment structure before updating code.

### 3. Update Code

Choose one source of truth.

#### Option A: Pull from GitHub

```bash
ssh admin@SERVER_IP "cd /opt/AI-chat && git pull --ff-only origin main"
```

#### Option B: Replace with uploaded code package

Use this only when GitHub is unavailable or you intentionally deploy from a local archive.

Rules:

- Unpack into `/opt/AI-chat`
- Preserve server `.env`
- Preserve `/opt/AI-chat-data`
- Re-check `docker-compose.yml` still mounts `/opt/AI-chat-data/db:/app/data`

### 4. Rebuild Services

```bash
ssh admin@SERVER_IP "cd /opt/AI-chat && docker compose up -d --build"
```

### 5. Verify Deployment

Run all of these before claiming success:

```bash
ssh admin@SERVER_IP "cd /opt/AI-chat && docker compose ps"
ssh admin@SERVER_IP "curl -s http://127.0.0.1:8181/api/health"
ssh admin@SERVER_IP "python3 - <<'PY'
import sqlite3
conn = sqlite3.connect('/opt/AI-chat-data/db/data.sqlite')
cur = conn.cursor()
print('users =', cur.execute('SELECT COUNT(*) FROM users').fetchone()[0])
print('conversations =', cur.execute('SELECT COUNT(*) FROM conversations').fetchone()[0])
print('messages =', cur.execute('SELECT COUNT(*) FROM messages').fetchone()[0])
conn.close()
PY"
```

Optional external check:

```bash
curl http://SERVER_IP:8181/api/health
```

## New Server Deployment Flow

Use this only for the first deployment on a clean machine.

1. Install Docker and Docker Compose
2. Create `/opt/AI-chat`
3. Create `/opt/AI-chat-data/db` and `/opt/AI-chat-data/backups`
4. Put code under `/opt/AI-chat`
5. Ensure backend config points to `SQLITE_PATH=/app/data/data.sqlite`
6. Ensure `docker-compose.yml` mounts `/opt/AI-chat-data/db:/app/data`
7. Start with `docker compose up -d --build`
8. Run the full verification block above

## Common Failures

### Health check returns 502 right after restart

Usually a short backend restart window. Re-run:

```bash
ssh admin@SERVER_IP "cd /opt/AI-chat && docker compose ps && docker compose logs --tail=50 backend"
```

If backend is `Up` and `/api/health` returns OK on retry, deployment is healthy.

### Deployment accidentally points back to code-directory database

Check both:

```bash
ssh admin@SERVER_IP "grep '^SQLITE_PATH=' /opt/AI-chat/backend/.env"
ssh admin@SERVER_IP "docker inspect ai-chat-backend --format '{{json .Mounts}}'"
```

Expected state:

- `SQLITE_PATH=/app/data/data.sqlite`
- mount source `/opt/AI-chat-data/db`

### Code update wants to replace server `.env`

Do not accept that blindly. Keep the server `.env` unless you intentionally change secrets or deployment variables.

## Completion Criteria

Do not say deployment succeeded until all are true:

- `docker compose ps` shows services running
- `http://127.0.0.1:8181/api/health` returns OK
- backend mount still points at `/opt/AI-chat-data/db`
- current database counts can be read from `/opt/AI-chat-data/db/data.sqlite`
