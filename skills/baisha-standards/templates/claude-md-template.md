# CLAUDE.md 模板（拷到新项目根目录）

> 复制下面的内容到新项目根的 `CLAUDE.md`，让 Claude Code 自动加载。
> 把 `{{PROJECT}}` / `{{SERVER_HOST}}` / `{{PORT}}` 替换成实际值。

---

# {{PROJECT}} 协作约定（AI 加载时优先阅读）

| 项 | 值 |
|---|---|
| 后端 | Node 18 + Fastify 5 + better-sqlite3 v10 |
| 前端 | Vue 3 + Vite 5 + Element Plus |
| 数据库 | SQLite，位于 `server/data.db` |
| 部署 | CentOS 7 + systemd + nginx 反代 |
| 生产地址 | `http://{{SERVER_HOST}}:{{PORT}}/`（如已上线）|

## 技术约束（必须遵守）

1. **CentOS 7 部署锁 Node 18 + better-sqlite3 v10**——v11 需新 glibc，跑不动。
2. **SQL 字面量永远用单引号**，双引号在 SQLite 是标识符（列名）。
3. **HTTP baseURL 用相对路径** `/api`，让 Vite proxy / nginx 反代统一处理。
4. **JWT_SECRET 必须 env 注入**，代码里只读 `process.env.JWT_SECRET`，缺就拒启动。
5. **CORS 公网暴露前**改成白名单，不要留 `origin: true`。
6. **诊断脚本写完即删**，不要长期留在 `server/scripts/`；webhook / secret / token 不要写进 git。

## 改动流程

1. **改之前先说改什么**——说"我要改 A 文件 B 函数，新增 C 行、删 D 行"，等点头再改
2. **改完给验证步骤**——不要拍胸脯说"我已经测过"
3. **涉及外部系统写操作**（推 ERP / 真发 IM 消息 / 删生产数据）必须经用户明示同意才能跑

## 启动 / 部署常用命令

```bash
# 本地起服（前后端一起）
pnpm dev

# 本地停服
pkill -9 -f "src/index.js"; pkill -9 -f "vite"; pkill -9 -f "concurrently"

# 部署到生产
bash ./deploy/update.sh

# 拉生产库到本地
bash ./deploy/db-pull.sh

# 看生产日志
ssh root@{{SERVER_HOST}} 'journalctl -u {{PROJECT}}-server -n 200 -f'
```

## 沟通风格偏好

- 不要"邀功"语气（不写"深夜还在""加班修了 X 个 bug"）
- 不写"下一步建议"、不写凭空 KPI 表格
- 时间表述要精准（日期 / 星期对得上）
- 简短回答 > 长文档；改动前**先说改什么再动手**

## 已知通用坑（避免重蹈）

- 路由 `:id` 切换不刷新 → 用 `watch(id, load)` 替代 `onMounted`
- vite preview 不读 `server.proxy` → 配 `preview.proxy` 或用 nginx
- 拉生产 SQLite 前先 `PRAGMA wal_checkpoint(TRUNCATE)`
- systemd 单元里 Node 写绝对路径，不要写 `node`
- Fastify 5 动作端点不带 body 会 400 → `index.js` 里重装 ContentTypeParser
