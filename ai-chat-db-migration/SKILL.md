---
name: ai-chat-db-migration
description: Use when exporting AI-chat SQLite data from one server, importing an exported SQLite snapshot into another server, switching the live AI-chat database, verifying database contents before and after migration, or rolling back to a previous database backup. Use for any operation that changes database contents under /opt/AI-chat-data/db.
---

# AI-chat DB Migration

## Overview

Safely export, import, switch, and roll back AI-chat database contents.

Core rule: database changes are explicit, backed up first, and verified with counts after the switch.

## Fixed Assumptions

- Active database directory: `/opt/AI-chat-data/db`
- Backup directory: `/opt/AI-chat-data/backups`
- Temporary imported snapshot path: `/opt/AI-chat-data/exported-data.sqlite`
- Backend mount: `/opt/AI-chat-data/db -> /app/data`
- Backend reads `/app/data/data.sqlite`

## Use This Skill For

- Online export from a source server that must stay running
- Importing `exported-data.sqlite` into the target server
- Backing up the current live SQLite database before replacement
- Checking whether imported data actually differs from current live data
- Rolling back to a previous backup if the imported data is wrong

## Do Not Use This Skill For

- Updating application code only
- New server setup without a database switch
- Docker rebuilds that do not touch database contents

Use `ai-chat-server-deploy` for code deployment work.

## Safety Rules

- Never switch databases without creating a backup directory first
- Never overwrite the live database while backend is running
- If importing a `.backup` snapshot, replace only `data.sqlite`
- Remove old live `data.sqlite-wal` and `data.sqlite-shm` before starting backend again
- Always compare counts before and after import

## Flow A: Online Export From Source Server

Use this when the source server must stay online.

### 1. Ensure sqlite3 Exists

```bash
ssh admin@SOURCE_IP "sqlite3 --version"
```

If missing:

```bash
ssh admin@SOURCE_IP "sudo apt-get update && sudo apt-get install -y sqlite3"
```

### 2. Export a Consistent Snapshot

```bash
ssh admin@SOURCE_IP "sqlite3 /opt/AI-chat-data/db/data.sqlite '.backup /opt/AI-chat-data/exported-data.sqlite'"
```

Do not copy running `data.sqlite`, `data.sqlite-wal`, and `data.sqlite-shm` directly from the source server.

### 3. Inspect the Exported Snapshot

```bash
ssh admin@SOURCE_IP "python3 - <<'PY'
import sqlite3
conn = sqlite3.connect('/opt/AI-chat-data/exported-data.sqlite')
cur = conn.cursor()
print('users =', cur.execute('SELECT COUNT(*) FROM users').fetchone()[0])
print('conversations =', cur.execute('SELECT COUNT(*) FROM conversations').fetchone()[0])
print('messages =', cur.execute('SELECT COUNT(*) FROM messages').fetchone()[0])
for row in cur.execute('SELECT phone, nickname, role FROM users ORDER BY phone LIMIT 10'):
    print(row)
conn.close()
PY"
```

Only proceed if the exported snapshot contains the data you expect.

### 4. Transfer the Snapshot

```bash
scp /opt/AI-chat-data/exported-data.sqlite admin@TARGET_IP:/opt/AI-chat-data/
```

## Flow B: Import Snapshot Into Target Server

### 1. Inspect Uploaded Snapshot

```bash
ssh admin@TARGET_IP "python3 - <<'PY'
import sqlite3
conn = sqlite3.connect('/opt/AI-chat-data/exported-data.sqlite')
cur = conn.cursor()
print('users =', cur.execute('SELECT COUNT(*) FROM users').fetchone()[0])
print('conversations =', cur.execute('SELECT COUNT(*) FROM conversations').fetchone()[0])
print('messages =', cur.execute('SELECT COUNT(*) FROM messages').fetchone()[0])
for row in cur.execute('SELECT phone, nickname, role FROM users ORDER BY phone LIMIT 10'):
    print(row)
conn.close()
PY"
```

If this does not match your expected source data, stop.

### 2. Compare Against Current Live Data

```bash
ssh admin@TARGET_IP "python3 - <<'PY'
import sqlite3
for name, path in [('incoming', '/opt/AI-chat-data/exported-data.sqlite'), ('active', '/opt/AI-chat-data/db/data.sqlite')]:
    conn = sqlite3.connect(path)
    cur = conn.cursor()
    print('===', name)
    print('users =', cur.execute('SELECT COUNT(*) FROM users').fetchone()[0])
    print('conversations =', cur.execute('SELECT COUNT(*) FROM conversations').fetchone()[0])
    print('messages =', cur.execute('SELECT COUNT(*) FROM messages').fetchone()[0])
    conn.close()
PY"
```

This prevents pointless switches when the incoming database is effectively the same.

### 3. Backup Current Live Database

```bash
ssh admin@TARGET_IP "backup_dir=/opt/AI-chat-data/backups/import-snapshot-$(date +%Y%m%d-%H%M%S); mkdir -p \"$backup_dir\"; cp -a /opt/AI-chat-data/db/data.sqlite* \"$backup_dir\"/; echo $backup_dir"
```

Save that backup path. It is the rollback source.

### 4. Stop Backend

```bash
ssh admin@TARGET_IP "cd /opt/AI-chat && docker compose stop backend"
```

### 5. Replace the Live Database

For `.backup` snapshots, replace the live database with the single exported file:

```bash
ssh admin@TARGET_IP "rm -f /opt/AI-chat-data/db/data.sqlite /opt/AI-chat-data/db/data.sqlite-shm /opt/AI-chat-data/db/data.sqlite-wal && cp -a /opt/AI-chat-data/exported-data.sqlite /opt/AI-chat-data/db/data.sqlite && chown admin:admin /opt/AI-chat-data/db/data.sqlite"
```

Do not copy `exported-data.sqlite-shm` or `exported-data.sqlite-wal` into the live directory.

### 6. Start Services Again

```bash
ssh admin@TARGET_IP "cd /opt/AI-chat && docker compose up -d backend frontend"
```

### 7. Verify Import Succeeded

Run all checks:

```bash
ssh admin@TARGET_IP "curl -s http://127.0.0.1:8181/api/health"
ssh admin@TARGET_IP "python3 - <<'PY'
import sqlite3
conn = sqlite3.connect('/opt/AI-chat-data/db/data.sqlite')
cur = conn.cursor()
print('users =', cur.execute('SELECT COUNT(*) FROM users').fetchone()[0])
print('conversations =', cur.execute('SELECT COUNT(*) FROM conversations').fetchone()[0])
print('messages =', cur.execute('SELECT COUNT(*) FROM messages').fetchone()[0])
for row in cur.execute('SELECT phone, nickname, role FROM users ORDER BY phone LIMIT 10'):
    print(row)
conn.close()
PY"
ssh admin@TARGET_IP "docker inspect ai-chat-backend --format '{{json .Mounts}}'"
```

Import is complete only when:

- health check returns OK
- live counts match the imported snapshot
- backend still mounts `/opt/AI-chat-data/db`

## Rollback Flow

Use this when the imported data is wrong.

1. Stop backend
2. Replace `/opt/AI-chat-data/db/data.sqlite*` with files from the backup directory created before import
3. Start backend and frontend again
4. Verify health and counts

Example:

```bash
ssh admin@TARGET_IP "cd /opt/AI-chat && docker compose stop backend && rm -f /opt/AI-chat-data/db/data.sqlite /opt/AI-chat-data/db/data.sqlite-shm /opt/AI-chat-data/db/data.sqlite-wal && cp -a /opt/AI-chat-data/backups/BACKUP_DIR_NAME/data.sqlite* /opt/AI-chat-data/db/ && docker compose up -d backend frontend"
```

Then re-run the verification block.

## Common Mistakes

### Copying live `data.sqlite*` directly from a running source server

Do not do that. Use `.backup` instead.

### Replacing the live database while backend is still running

This is unsafe. Stop backend first.

### Importing the snapshot but forgetting to inspect its contents

Always check counts before switching, or you may import the wrong database successfully.

### Thinking a 502 during restart means import failed

A brief `502 Bad Gateway` can happen while backend restarts. Re-check health after backend is fully `Up`.

## Completion Criteria

Do not say migration succeeded until all are true:

- backup directory exists for the pre-import live database
- backend restarted successfully
- `/api/health` returns OK
- live database counts match the imported snapshot
- backend mount still points at `/opt/AI-chat-data/db`
