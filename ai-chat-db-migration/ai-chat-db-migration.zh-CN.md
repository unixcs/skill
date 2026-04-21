# AI-chat 数据库迁移操作手册

## 文档定位

这份文档是 `ai-chat-db-migration` Skill 的中文操作手册版本，适合直接按步骤执行数据库导出、导入和回滚。

适用目标：

1. 从源服务器在线导出 SQLite 数据库
2. 把 `exported-data.sqlite` 导入目标服务器
3. 导入前备份当前线上数据库
4. 回滚到导入前的数据库备份

不适用目标：

1. 更新 GitHub 代码
2. 部署程序到新服务器
3. 重建不涉及数据库切换的 Docker 服务

如果你的目标是更新或部署程序，请改看 `ai-chat-server-deploy.zh-CN.md`。

## 一句话原则

**先备份，再切换；不验证，不算成功。**

## 固定目录和约定

- 生效数据库目录：`/opt/AI-chat-data/db`
- 备份目录：`/opt/AI-chat-data/backups`
- 临时导入快照路径：`/opt/AI-chat-data/exported-data.sqlite`
- 后端数据库挂载：`/opt/AI-chat-data/db -> /app/data`
- 后端实际读取路径：`/app/data/data.sqlite`

## 使用前总检查

每次切库前，先确认这 3 件事：

### 1. 当前服务健康

```bash
ssh admin@TARGET_IP "curl -s http://127.0.0.1:8181/api/health"
```

### 2. 当前后端挂载是否正确

```bash
ssh admin@TARGET_IP "docker inspect ai-chat-backend --format '{{json .Mounts}}'"
```

### 3. 当前生效数据库路径是否存在

```bash
ssh admin@TARGET_IP "ls -lah /opt/AI-chat-data/db"
```

## 场景一：源服务器在线导出数据库

适合源服务器还在运行、不想停机的情况。

### 步骤 1：确认 sqlite3 可用

```bash
ssh admin@SOURCE_IP "sqlite3 --version"
```

如果没有，再安装：

```bash
ssh admin@SOURCE_IP "sudo apt-get update && sudo apt-get install -y sqlite3"
```

### 步骤 2：在线导出一致性快照

```bash
ssh admin@SOURCE_IP "sqlite3 /opt/AI-chat-data/db/data.sqlite '.backup /opt/AI-chat-data/exported-data.sqlite'"
```

注意：

不要直接复制运行中的：

1. `data.sqlite`
2. `data.sqlite-wal`
3. `data.sqlite-shm`

在线导出时，优先使用 `.backup`。

### 步骤 3：检查导出的快照内容

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

确认这份导出库确实是你想迁移的数据，再继续。

### 步骤 4：传输到目标服务器

```bash
scp /opt/AI-chat-data/exported-data.sqlite admin@TARGET_IP:/opt/AI-chat-data/
```

## 场景二：导入 `exported-data.sqlite` 到目标服务器

### 步骤 1：先检查导入文件

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

如果这里看到的用户数、会话数、消息数就不对，不能继续切库。

### 步骤 2：对比当前线上数据库和导入文件

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

目的：避免把一份和当前线上库几乎一样的数据再切一遍。

### 步骤 3：备份当前生效数据库

```bash
ssh admin@TARGET_IP "backup_dir=/opt/AI-chat-data/backups/import-snapshot-$(date +%Y%m%d-%H%M%S); mkdir -p \"$backup_dir\"; cp -a /opt/AI-chat-data/db/data.sqlite* \"$backup_dir\"/; echo $backup_dir"
```

把输出的备份目录记下来，用于后续回滚。

### 步骤 4：停止后端

```bash
ssh admin@TARGET_IP "cd /opt/AI-chat && docker compose stop backend"
```

### 步骤 5：替换生效数据库

如果导入的是 `.backup` 生成的单文件快照，正确替换方式是：

```bash
ssh admin@TARGET_IP "rm -f /opt/AI-chat-data/db/data.sqlite /opt/AI-chat-data/db/data.sqlite-shm /opt/AI-chat-data/db/data.sqlite-wal && cp -a /opt/AI-chat-data/exported-data.sqlite /opt/AI-chat-data/db/data.sqlite && chown admin:admin /opt/AI-chat-data/db/data.sqlite"
```

注意：

1. 不要把 `exported-data.sqlite-shm` 和 `exported-data.sqlite-wal` 一起复制进生效目录
2. 启动后如果 SQLite 需要，它会自己重新生成新的 `-wal` 和 `-shm`

### 步骤 6：重新启动服务

```bash
ssh admin@TARGET_IP "cd /opt/AI-chat && docker compose up -d backend frontend"
```

### 步骤 7：验证导入是否成功

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

## 场景三：导入后回滚

如果导入后发现数据不对，就使用导入前的备份目录回滚。

### 回滚步骤

#### 1. 停止后端

```bash
ssh admin@TARGET_IP "cd /opt/AI-chat && docker compose stop backend"
```

#### 2. 删除当前生效库

```bash
ssh admin@TARGET_IP "rm -f /opt/AI-chat-data/db/data.sqlite /opt/AI-chat-data/db/data.sqlite-shm /opt/AI-chat-data/db/data.sqlite-wal"
```

#### 3. 用备份目录恢复

```bash
ssh admin@TARGET_IP "cp -a /opt/AI-chat-data/backups/BACKUP_DIR_NAME/data.sqlite* /opt/AI-chat-data/db/"
```

#### 4. 重新启动服务

```bash
ssh admin@TARGET_IP "cd /opt/AI-chat && docker compose up -d backend frontend"
```

#### 5. 再次验证

```bash
ssh admin@TARGET_IP "curl -s http://127.0.0.1:8181/api/health"
ssh admin@TARGET_IP "python3 - <<'PY'
import sqlite3
conn = sqlite3.connect('/opt/AI-chat-data/db/data.sqlite')
cur = conn.cursor()
print('users =', cur.execute('SELECT COUNT(*) FROM users').fetchone()[0])
print('conversations =', cur.execute('SELECT COUNT(*) FROM conversations').fetchone()[0])
print('messages =', cur.execute('SELECT COUNT(*) FROM messages').fetchone()[0])
conn.close()
PY"
```

## 常见问题处理

### 1. 直接复制运行中的 `data.sqlite*`

不推荐。在线导出时优先使用 `.backup`。

### 2. 后端没停就直接覆盖生效库

这是高风险操作，容易造成数据库状态不一致。

### 3. 导入前没检查导入文件内容

这样很可能把错误的库“成功导入”。

### 4. 切换后短暂出现 502

这通常是后端重启窗口。请等待后端完全 `Up` 后再次检查 `/api/health`。

## 完成判定

只有同时满足下面几点，才算数据库迁移成功：

1. 导入前备份目录已经创建
2. 后端已经正常重启
3. `/api/health` 返回正常
4. 当前生效库数量和导入快照一致
5. 后端挂载仍然是 `/opt/AI-chat-data/db -> /app/data`
