# AI-chat 服务器部署操作手册

## 文档定位

这份文档是 `ai-chat-server-deploy` Skill 的中文操作手册版本，适合直接照步骤执行。

适用目标：

1. 新服务器首次部署 AI-chat
2. 从 GitHub 更新线上代码
3. 使用本地上传的代码包更新线上程序
4. 重新构建 Docker 服务

不适用目标：

1. 导出数据库
2. 导入 `exported-data.sqlite`
3. 切换线上数据库内容
4. 回滚数据库备份

如果要处理数据库迁移，请改看 `ai-chat-db-migration.zh-CN.md`。

## 一句话原则

**只更新程序，不改数据库内容。**

## 固定目录和约定

- 代码目录：`/opt/AI-chat`
- 生效数据库目录：`/opt/AI-chat-data/db`
- 数据备份目录：`/opt/AI-chat-data/backups`
- 对外访问端口：`8181`
- 后端容器数据库挂载：`/opt/AI-chat-data/db -> /app/data`
- 后端数据库环境变量：`SQLITE_PATH=/app/data/data.sqlite`

## 部署前检查

每次更新代码前，先执行以下检查。

### 1. 检查容器状态

```bash
ssh admin@SERVER_IP "cd /opt/AI-chat && docker compose ps"
```

### 2. 检查健康接口

```bash
ssh admin@SERVER_IP "curl -s http://127.0.0.1:8181/api/health"
```

### 3. 检查数据库挂载是否正确

```bash
ssh admin@SERVER_IP "docker inspect ai-chat-backend --format '{{json .Mounts}}'"
```

确认挂载源目录仍然是：

```text
/opt/AI-chat-data/db
```

### 4. 检查关键目录是否存在

```bash
ssh admin@SERVER_IP "ls -lah /opt/AI-chat /opt/AI-chat-data /opt/AI-chat-data/db /opt/AI-chat-data/backups"
```

如果 `/opt/AI-chat-data/db` 不存在，不要继续更新代码，先修复目录结构。

## 场景一：新服务器首次部署

### 准备条件

1. 服务器可 SSH 登录
2. 已安装 Docker 和 Docker Compose
3. 已准备好项目代码

### 操作步骤

#### 1. 创建基础目录

```bash
sudo mkdir -p /opt/AI-chat
sudo mkdir -p /opt/AI-chat-data/db
sudo mkdir -p /opt/AI-chat-data/backups
sudo chown -R admin:admin /opt/AI-chat /opt/AI-chat-data
```

#### 2. 放置项目代码

代码最终应位于：

```text
/opt/AI-chat
```

#### 3. 确认数据库路径配置正确

检查：

```bash
grep '^SQLITE_PATH=' /opt/AI-chat/backend/.env
```

期望值：

```text
SQLITE_PATH=/app/data/data.sqlite
```

#### 4. 确认 compose 挂载正确

检查 `docker-compose.yml` 必须包含：

```text
/opt/AI-chat-data/db:/app/data
```

#### 5. 启动服务

```bash
cd /opt/AI-chat
docker compose up -d --build
```

### 验证标准

```bash
cd /opt/AI-chat
docker compose ps
curl -s http://127.0.0.1:8181/api/health
python3 - <<'PY'
import sqlite3
conn = sqlite3.connect('/opt/AI-chat-data/db/data.sqlite')
cur = conn.cursor()
print('users =', cur.execute('SELECT COUNT(*) FROM users').fetchone()[0])
print('conversations =', cur.execute('SELECT COUNT(*) FROM conversations').fetchone()[0])
print('messages =', cur.execute('SELECT COUNT(*) FROM messages').fetchone()[0])
conn.close()
PY
```

## 场景二：从 GitHub 更新线上代码

### 使用前提

1. 当前项目已经部署在 `/opt/AI-chat`
2. 数据目录已经独立到 `/opt/AI-chat-data/db`

### 操作步骤

#### 1. 拉取最新代码

```bash
cd /opt/AI-chat
git pull --ff-only origin main
```

#### 2. 重建服务

```bash
cd /opt/AI-chat
docker compose up -d --build
```

### 验证标准

```bash
cd /opt/AI-chat
docker compose ps
curl -s http://127.0.0.1:8181/api/health
docker inspect ai-chat-backend --format '{{json .Mounts}}'
```

确认两件事：

1. 健康检查正常
2. 后端挂载仍然指向 `/opt/AI-chat-data/db`

## 场景三：用本地上传代码包更新程序

### 使用前提

1. GitHub 不可用，或你明确希望从本地压缩包部署
2. 你只想替换程序目录，不想动数据目录

### 操作步骤

#### 1. 上传代码包到服务器临时位置

例如上传到：

```text
/opt/
```

#### 2. 替换程序目录时遵守这 4 条规则

1. 只替换 `/opt/AI-chat`
2. 保留服务器上的 `.env`
3. 保留 `/opt/AI-chat-data`
4. 替换后重新检查 `docker-compose.yml`

#### 3. 重建服务

```bash
cd /opt/AI-chat
docker compose up -d --build
```

### 验证标准

与 GitHub 更新后的验证相同：

```bash
cd /opt/AI-chat
docker compose ps
curl -s http://127.0.0.1:8181/api/health
docker inspect ai-chat-backend --format '{{json .Mounts}}'
```

## 常见问题处理

### 1. 刚重启就出现 502

这通常是后端重启窗口，不一定代表失败。

复查命令：

```bash
cd /opt/AI-chat
docker compose ps
docker compose logs --tail=50 backend
curl -s http://127.0.0.1:8181/api/health
```

如果 backend 状态是 `Up` 且健康接口恢复正常，就可以视为成功。

### 2. 数据库路径被改回代码目录

重点检查：

```bash
grep '^SQLITE_PATH=' /opt/AI-chat/backend/.env
docker inspect ai-chat-backend --format '{{json .Mounts}}'
```

正确状态必须是：

1. `SQLITE_PATH=/app/data/data.sqlite`
2. 挂载源目录是 `/opt/AI-chat-data/db`

### 3. 更新代码时想覆盖 `.env`

正常情况下不要覆盖。线上 `.env` 属于服务器专属配置，除非你明确要改环境变量，否则应该保留。

## 完成判定

只有同时满足下面几点，才算部署或更新成功：

1. `docker compose ps` 显示服务运行中
2. `http://127.0.0.1:8181/api/health` 返回正常
3. 后端仍然挂载 `/opt/AI-chat-data/db`
4. 能正常读取 `/opt/AI-chat-data/db/data.sqlite` 的内容
