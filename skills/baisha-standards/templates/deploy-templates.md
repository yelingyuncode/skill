# 部署模板（CentOS 7 + systemd + nginx 反代）

> 全部用 `{{PROJECT}}` / `{{SERVER_HOST}}` / `{{REMOTE_APP}}` 占位符，新项目自己替换。

## 导览

- 环境约束和目录约定：先看顶部两节
- `1-3`：更新脚本、数据库拉取、数据库推送
- `4-5`：后端 systemd、前端部署方式
- `6`：首次部署完整步骤
- `7`：公网映射前安全检查

## CentOS 7 部署约束（一定要看）

| 项 | 锁定 | 原因 |
|---|---|---|
| Node | 18 LTS | glibc 2.17 跑不动 Node 22 |
| better-sqlite3 | v10 | v11 需新 glibc |
| 编译工具链 | `devtoolset-9` | better-sqlite3 native 编译需要 |
| 包管理（服务器侧）| npm（不装 pnpm）| 服务器装的越少越好 |
| 进程管理 | systemd | 标配，比 pm2 更省事 |

## 推荐目录约定

```
/usr/{{PROJECT}}/                              项目根
/usr/{{PROJECT}}/server/data.db                SQLite 库
/etc/systemd/system/{{PROJECT}}-server.service 后端 systemd 单元
/etc/systemd/system/{{PROJECT}}-web.service    前端 systemd 单元（或用 nginx 直 serve dist）
/etc/nginx/conf.d/{{PROJECT}}.conf             nginx 反代（推荐）
~/{{PROJECT}}-db/                              本地拉生产库的备份目录（开发机用）
```

## 1. deploy/{{PROJECT}}-update.sh

本地一键打包 + scp + 远程解压 + 重启 + 自检。

```bash
#!/usr/bin/env bash
# 本地改完代码 → 一键打包 + 上传 + 远程重启
# 用法：bash ./deploy/{{PROJECT}}-update.sh
set -e

PROJECT_DIR="$(cd "$(dirname "$0")/.." && pwd)"   # 自动定位项目根
SERVER=root@{{SERVER_HOST}}
REMOTE_APP=/usr/{{PROJECT}}
TMP_TGZ=/tmp/{{PROJECT}}-update-$(date +%s).tar.gz

_GRN=$(tput setaf 2 2>/dev/null); _OFF=$(tput sgr0 2>/dev/null)
log() { printf "${_GRN}[ %s ]${_OFF} %s\n" "$(date +%H:%M:%S)" "$*"; }

log "1. 打包代码（不含 node_modules / dist / *.db）..."
tar -czf "$TMP_TGZ" \
  --exclude='**/node_modules' --exclude='**/.git' --exclude='**/dist' \
  --exclude='**/data.db*' --exclude='**/*.log' --exclude='**/.DS_Store' \
  --exclude='**/.claude' \
  -C "$PROJECT_DIR" \
  server/src server/scripts server/package.json \
  web/src web/public web/index.html web/package.json web/vite.config.js 2>/dev/null
ls -lh "$TMP_TGZ"

log "2. 上传到 $SERVER:/tmp/ ..."
scp "$TMP_TGZ" "$SERVER":/tmp/update.tar.gz
echo

log "3. 远程：解压 + 智能 npm install + 构建 + 重启 ..."
ssh "$SERVER" 'bash -s' << REMOTE_EOF
set -e

cd $REMOTE_APP

# 解压前记录依赖指纹，决定要不要 npm install
SERVER_PKG_BEFORE=\$(md5sum server/package.json 2>/dev/null | awk '{print \$1}')
WEB_PKG_BEFORE=\$(md5sum web/package.json 2>/dev/null | awk '{print \$1}')

tar -xzf /tmp/update.tar.gz -C $REMOTE_APP

# 后端依赖变了才 npm install（用 devtoolset-9 编译 better-sqlite3）
SERVER_PKG_AFTER=\$(md5sum server/package.json | awk '{print \$1}')
if [ "\$SERVER_PKG_BEFORE" != "\$SERVER_PKG_AFTER" ]; then
  echo "  → 后端 package.json 改了，npm install ..."
  [ -f /opt/rh/devtoolset-9/enable ] && source /opt/rh/devtoolset-9/enable
  cd $REMOTE_APP/server
  npm install --omit=dev --no-audit --no-fund
else
  echo "  → 后端依赖未变，跳过 npm install"
fi

# 前端依赖变了才 npm install
WEB_PKG_AFTER=\$(md5sum $REMOTE_APP/web/package.json | awk '{print \$1}')
if [ "\$WEB_PKG_BEFORE" != "\$WEB_PKG_AFTER" ]; then
  echo "  → 前端 package.json 改了，npm install ..."
  cd $REMOTE_APP/web
  npm install --no-audit --no-fund
fi

# 永远重新 build 前端
echo "  → 重新构建前端 ..."
cd $REMOTE_APP/web
npm run build

echo "  → 重启 systemd 服务 ..."
systemctl restart {{PROJECT}}-server
sleep 1
systemctl restart {{PROJECT}}-web    # 如果用 nginx 直 serve dist，去掉这行
sleep 2

# 自检
echo
echo "=== 自检 ==="
B=\$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:4000/api/health 2>/dev/null)
W=\$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:5173/ 2>/dev/null)
echo "后端: HTTP \$B  (期望 200)"
echo "前端: HTTP \$W  (期望 200)"

rm -f /tmp/update.tar.gz
REMOTE_EOF

rm -f "$TMP_TGZ"
log "✓ 更新完成"
```

**要点**：
- `PROJECT_DIR="$(cd "$(dirname "$0")/.." && pwd)"` 自动定位项目根，**不要硬编码绝对路径**
- 排除 `node_modules / dist / data.db / .claude` —— 这些不能被覆盖或不该传
- md5 diff `package.json` 决定是否 npm install —— 90% 的迭代不动依赖，跳过装包能省几十秒
- 用 `devtoolset-9` 启用现代 g++ 编译 better-sqlite3
- 永远重 build 前端（vite 增量很快）
- curl 自检：用 `/api/health` 而不是 `/api/auth/me`（后者要 token 不便自检）

## 2. deploy/{{PROJECT}}-db-pull.sh

从生产服务器拉 SQLite 库到本地（带时间戳快照）。

```bash
#!/usr/bin/env bash
# 从服务器拉数据库到本地（带时间戳备份）
# 用法：bash ./deploy/{{PROJECT}}-db-pull.sh
set -e

SERVER=root@{{SERVER_HOST}}
REMOTE_DB=/usr/{{PROJECT}}/server/data.db
# 项目根/db-snapshots/，与本地开发库 server/data.db 区分开
LOCAL_DIR="$(cd "$(dirname "$0")/.." && pwd)/db-snapshots"
TS=$(date +%Y%m%d-%H%M%S)

_GRN=$(tput setaf 2 2>/dev/null); _OFF=$(tput sgr0 2>/dev/null)
log() { printf "${_GRN}[ %s ]${_OFF} %s\n" "$(date +%H:%M:%S)" "$*"; }

mkdir -p "$LOCAL_DIR"

log "服务器先 checkpoint WAL（确保所有数据写入主文件）..."
ssh "$SERVER" "sqlite3 $REMOTE_DB 'PRAGMA wal_checkpoint(TRUNCATE);'"

log "下载到 $LOCAL_DIR/{{PROJECT}}-prod.db ..."
scp "$SERVER:$REMOTE_DB" "$LOCAL_DIR/{{PROJECT}}-prod.db"

# 同时保留一份带时间戳的快照
cp "$LOCAL_DIR/{{PROJECT}}-prod.db" "$LOCAL_DIR/{{PROJECT}}-prod-$TS.db"

ls -lh "$LOCAL_DIR/{{PROJECT}}-prod.db" "$LOCAL_DIR/{{PROJECT}}-prod-$TS.db"
log "✓ 完成。Navicat 打开 $LOCAL_DIR/{{PROJECT}}-prod.db"
```

**关键**：`PRAGMA wal_checkpoint(TRUNCATE)` 必须先跑——SQLite WAL 模式下 `data.db` 主文件不含未 checkpoint 的事务，不做 checkpoint 直接 scp 会丢数据。

## 3. deploy/{{PROJECT}}-db-push.sh

本地改完库推回服务器（停后端 5-10s、二次确认）。

```bash
#!/usr/bin/env bash
# 把本地改完的数据库推回服务器（会停后端 + 清 WAL + 重启）
# 用法：bash ./deploy/{{PROJECT}}-db-push.sh
set -e

SERVER=root@{{SERVER_HOST}}
REMOTE_DB=/usr/{{PROJECT}}/server/data.db
LOCAL_DB="$(cd "$(dirname "$0")/.." && pwd)/db-snapshots/{{PROJECT}}-prod.db"

_GRN=$(tput setaf 2 2>/dev/null); _YLW=$(tput setaf 3 2>/dev/null); _OFF=$(tput sgr0 2>/dev/null)
log()  { printf "${_GRN}[ %s ]${_OFF} %s\n" "$(date +%H:%M:%S)" "$*"; }
warn() { printf "${_YLW}[WARN]${_OFF} %s\n" "$*"; }

[ -f "$LOCAL_DB" ] || { echo "找不到 $LOCAL_DB"; exit 1; }

echo
warn "⚠️  这会覆盖服务器的 $REMOTE_DB"
warn "⚠️  服务器后端会被短暂停掉（5-10 秒）"
warn "⚠️  期间生产侧的写入会丢"
echo
read -p "确认推送？输 yes 继续: " ans
[ "$ans" = "yes" ] || { echo "已取消"; exit 0; }

log "服务器先备份当前 DB ..."
ssh "$SERVER" "cp $REMOTE_DB $REMOTE_DB.bak-\$(date +%s)"

log "停后端 ..."
ssh "$SERVER" 'systemctl stop {{PROJECT}}-server'

log "上传本地 DB 覆盖 ..."
scp "$LOCAL_DB" "$SERVER:$REMOTE_DB"

log "清掉旧 WAL/SHM 文件 + 重启后端 ..."
ssh "$SERVER" "
  rm -f $REMOTE_DB-shm $REMOTE_DB-wal
  systemctl start {{PROJECT}}-server
  sleep 2
  systemctl is-active {{PROJECT}}-server
"

log "✓ 推送完成，旧 DB 已备份为 $REMOTE_DB.bak-<时间戳>"
```

**警告**：推回去前生产侧的写入会被覆盖。**这操作风险大**，不要日常用——仅在「只有自己一个人在改数据」的修数据场景用。

## 4. deploy/{{PROJECT}}-server.service（后端 systemd 单元）

```ini
# 放到服务器 /etc/systemd/system/{{PROJECT}}-server.service
[Unit]
Description={{PROJECT}} Backend (Fastify)
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/usr/{{PROJECT}}/server

# JWT 通过环境变量注入，不要写进代码
Environment=NODE_ENV=production
Environment=PORT=4000
Environment=JWT_SECRET=替换成一段长随机串_至少_32_位

# Node 路径按服务器实际改：which node
ExecStart=/root/.nvm/versions/node/v18.20.4/bin/node --no-warnings src/index.js

Restart=on-failure
RestartSec=5

StandardOutput=journal
StandardError=journal
SyslogIdentifier={{PROJECT}}-server

[Install]
WantedBy=multi-user.target
```

激活：

```bash
systemctl daemon-reload
systemctl enable {{PROJECT}}-server
systemctl start {{PROJECT}}-server
systemctl status {{PROJECT}}-server
journalctl -u {{PROJECT}}-server -n 200 -f      # 实时日志
```

## 5. 前端两种形态二选一

### 5.1 简单：`{{PROJECT}}-web.service` 跑 `vite preview`

```ini
[Unit]
Description={{PROJECT}} Frontend (vite preview)
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/usr/{{PROJECT}}/web
Environment=NODE_ENV=production
ExecStart=/root/.nvm/versions/node/v18.20.4/bin/npm run preview -- --host 0.0.0.0 --port 5173
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**前提**：`vite.config.js` 必须配 `preview.proxy`，否则 `/api` 404。

### 5.2 推荐：nginx 直 serve dist + 反代 `/api`

```nginx
# /etc/nginx/conf.d/{{PROJECT}}.conf
server {
    listen 80;
    server_name {{SERVER_HOST}};

    root /usr/{{PROJECT}}/web/dist;
    index index.html;

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 反向代理 API
    location /api/ {
        proxy_pass http://127.0.0.1:4000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 60s;
    }

    client_max_body_size 20m;
}
```

激活：

```bash
nginx -t                          # 测试配置
systemctl reload nginx
```

**优点**：
- 不用占两个 systemd 服务
- nginx 静态文件性能比 vite preview 好得多
- HTTPS 加证书在 nginx 里改一处即可

## 6. 首次部署完整步骤

```bash
# 服务器侧：装 Node 18 + npm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 18

# 服务器侧：装 devtoolset-9（CentOS 7 编译 better-sqlite3）
yum install -y centos-release-scl
yum install -y devtoolset-9

# 服务器侧：装 sqlite3 命令行（pull/push 脚本要用）
yum install -y sqlite

# 本地：SSH 免密
ssh-copy-id root@{{SERVER_HOST}}

# 本地：首次同步整个项目
rsync -avz --exclude='node_modules' --exclude='.git' --exclude='dist' \
  ./ root@{{SERVER_HOST}}:/usr/{{PROJECT}}/

# 服务器：装依赖
ssh root@{{SERVER_HOST}} '
  source /opt/rh/devtoolset-9/enable
  cd /usr/{{PROJECT}}/server && npm install --no-audit --no-fund
  cd /usr/{{PROJECT}}/web && npm install --no-audit --no-fund && npm run build
'

# 服务器：装 systemd 单元
scp deploy/{{PROJECT}}-server.service root@{{SERVER_HOST}}:/etc/systemd/system/
ssh root@{{SERVER_HOST}} '
  # 编辑 JWT_SECRET 和 node 路径
  vi /etc/systemd/system/{{PROJECT}}-server.service
  systemctl daemon-reload
  systemctl enable {{PROJECT}}-server && systemctl start {{PROJECT}}-server
'

# 服务器：装 nginx 反代（推荐）
scp deploy/{{PROJECT}}-web.nginx.conf root@{{SERVER_HOST}}:/etc/nginx/conf.d/{{PROJECT}}.conf
ssh root@{{SERVER_HOST}} 'nginx -t && systemctl reload nginx'

# 之后日常更新一行命令
bash ./deploy/{{PROJECT}}-update.sh
```

## 7. 公网映射前的安全检查（重要）

把这套对外暴露前，**逐项过**：

| 项 | 默认 | 公网要求 |
|---|---|---|
| `JWT_SECRET` | dev 字面量 | systemd `Environment=` 注入长随机串 |
| admin 默认密码 | `admin123` | 强制改强密码 |
| CORS `origin: true` 全开 | 内网 OK | 改白名单 |
| 速率限制 | 无 | 加 `@fastify/rate-limit` |
| HTTPS | HTTP 明文 | nginx + Let's Encrypt |
| 第三方 API 密钥 | 硬编码 | `Environment=` 注入 |
| 后端日志 | 含 token 片段 | 上线前 grep 一遍 |

**最稳的做法**：不暴露公网，员工连公司 VPN 访问内网 IP。
