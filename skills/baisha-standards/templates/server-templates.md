# 后端模板（Fastify 5 + better-sqlite3 v10 + JWT）

> 纯架构骨架，不带业务字段。复制到新项目改改就能跑。

## 导览

- `0-2`：package、Fastify 入口、SQLite 初始化
- `3-5`：JWT/bcrypt、通用工具、种子账号
- `6-7`：认证路由、标准 CRUD 路由模板
- `8-9`：权限守卫模式、关键决策回放

## 0. server/package.json

```json
{
  "name": "{{PROJECT}}-server",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "node --no-warnings --watch src/index.js",
    "start": "node --no-warnings src/index.js",
    "seed": "node --no-warnings src/seed.js"
  },
  "dependencies": {
    "@fastify/cors": "^10.0.1",
    "@fastify/jwt": "^9.0.1",
    "bcryptjs": "^2.4.3",
    "better-sqlite3": "^10.1.0",
    "dayjs": "^1.11.13",
    "fastify": "^5.1.0",
    "nanoid": "^5.0.8"
  }
}
```

**⚠️ better-sqlite3 锁 v10**——v11 需要更新的 glibc，CentOS 7 跑不动。

如果要 cron 调度，再加 `"node-cron": "^4.2.1"`。

## 1. src/index.js（Fastify 入口）

```js
import Fastify from 'fastify';
import cors from '@fastify/cors';
import jwt from '@fastify/jwt';

import './db.js';                    // 初始化 schema + safeExec 迁移
import { authHook } from './auth.js';
import authRoutes from './routes/auth.js';
import personsRoutes from './routes/persons.js';
import rolesRoutes from './routes/roles.js';
import accountsRoutes from './routes/accounts.js';

const app = Fastify({ logger: { level: 'info' } });

await app.register(cors, { origin: true, credentials: true });   // 公网暴露前改白名单
await app.register(jwt, { secret: process.env.JWT_SECRET || '{{PROJECT}}-dev-secret-change-me' });

// 容忍空 JSON body：很多动作端点（submit/cancel）不带 body
app.removeContentTypeParser('application/json');
app.addContentTypeParser('application/json', { parseAs: 'string' }, (req, body, done) => {
  if (!body) return done(null, {});
  try { done(null, JSON.parse(body)); } catch (e) { done(e); }
});

app.decorate('authHook', authHook);   // 让 routes 里 app.addHook('preHandler', app.authHook) 能用

app.get('/api/health', async () => ({ ok: true, ts: Date.now() }));

await app.register(authRoutes,     { prefix: '/api' });
await app.register(personsRoutes,  { prefix: '/api' });
await app.register(rolesRoutes,    { prefix: '/api' });
await app.register(accountsRoutes, { prefix: '/api' });

const port = +(process.env.PORT || 4000);
app.listen({ port, host: '0.0.0.0' })
  .then(() => app.log.info(`server on http://localhost:${port}`))
  .catch((err) => { app.log.error(err); process.exit(1); });
```

**要点**：
- `cors: origin: true` 内网够用；公网暴露前**必须**改白名单
- `JWT_SECRET` 走 env 注入，**别写死**
- 容空 body parser 是 Fastify 5 的小坑，保留这段
- `authHook` 用 decorate 暴露，路由里 `app.addHook('preHandler', app.authHook)` 引用

## 2. src/db.js（SQLite + safeExec 迁移）

```js
import Database from 'better-sqlite3';
import { fileURLToPath } from 'node:url';
import path from 'node:path';

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const dbPath = path.resolve(__dirname, '..', 'data.db');

export const db = new Database(dbPath);
db.pragma('journal_mode = WAL');         // 必开：避免读写阻塞
db.pragma('foreign_keys = ON');

// better-sqlite3 自带 db.transaction(fn)
export function transaction(fn) {
  return db.transaction(fn)();
}

function initSchema() {
  db.exec(`
    CREATE TABLE IF NOT EXISTS persons (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL,
      employee_no TEXT UNIQUE,
      phone TEXT,
      email TEXT,
      status TEXT NOT NULL DEFAULT 'active',
      created_at TEXT NOT NULL DEFAULT (datetime('now','localtime'))
    );

    CREATE TABLE IF NOT EXISTS roles (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      code TEXT UNIQUE NOT NULL,
      name TEXT NOT NULL,
      description TEXT,
      created_at TEXT NOT NULL DEFAULT (datetime('now','localtime'))
    );

    CREATE TABLE IF NOT EXISTS person_roles (
      person_id INTEGER NOT NULL,
      role_id INTEGER NOT NULL,
      PRIMARY KEY (person_id, role_id),
      FOREIGN KEY (person_id) REFERENCES persons(id) ON DELETE CASCADE,
      FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
    );

    CREATE TABLE IF NOT EXISTS accounts (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      username TEXT UNIQUE NOT NULL,
      password_hash TEXT NOT NULL,
      person_id INTEGER UNIQUE NOT NULL,
      enabled INTEGER NOT NULL DEFAULT 1,
      last_login_at TEXT,
      created_at TEXT NOT NULL DEFAULT (datetime('now','localtime')),
      FOREIGN KEY (person_id) REFERENCES persons(id) ON DELETE CASCADE
    );
  `);
}

initSchema();

// 简易迁移：缺列就加上；列已存在时报错被 try-catch 吞掉，重复跑安全
function safeExec(sql) {
  try { db.exec(sql); } catch (e) { /* ignore */ }
}

// 业务新增列时往这里加，例如：
// safeExec(`ALTER TABLE accounts ADD COLUMN password_plain TEXT`);
// safeExec(`ALTER TABLE persons ADD COLUMN avatar_url TEXT`);
```

**要点**：
- `WAL` 必开，否则单进程读写互相阻塞
- `foreign_keys = ON` SQLite 默认关，必须显式开
- `safeExec` 是反复用的迁移模式——`ALTER TABLE ADD COLUMN`，错就吞，幂等
- **不要用 prisma/drizzle 等 ORM**，直接 prepare + run/get/all 三招最快

## 3. src/auth.js（JWT + bcrypt）

```js
import bcrypt from 'bcryptjs';
import { db } from './db.js';

export function hashPassword(plain) { return bcrypt.hashSync(plain, 10); }
export function verifyPassword(plain, hash) { return bcrypt.compareSync(plain, hash); }

// 全局 preHandler：把 req.user 解出来
export async function authHook(req, reply) {
  try { await req.jwtVerify(); } catch { return reply.code(401).send({ error: '未登录' }); }
}

// 路由里拿当前用户（含角色）
export function getCurrentUser(req) {
  const acc = db.prepare('SELECT * FROM accounts WHERE id = ?').get(req.user?.id);
  if (!acc) return null;
  const person = db.prepare('SELECT * FROM persons WHERE id = ?').get(acc.person_id);
  const roles = db.prepare(`
    SELECT r.* FROM roles r
    JOIN person_roles pr ON pr.role_id = r.id
    WHERE pr.person_id = ?
  `).all(acc.person_id);
  return { account: acc, person, roles };
}
```

## 4. src/util.js

```js
import dayjs from 'dayjs';
import { db } from './db.js';

export const now = () => dayjs().format('YYYY-MM-DD HH:mm:ss');
export const touch = (table, id) =>
  db.prepare(`UPDATE ${table} SET updated_at = ? WHERE id = ?`).run(now(), id);

// 业务单号生成（可选）：{{PREFIX}} + 日期 + 当日序号
// 用法：nextSeq('SO') → 'SO202606200001'
const seqCache = {};
export function nextSeq(prefix) {
  const today = dayjs().format('YYYYMMDD');
  const key = `${prefix}${today}`;
  seqCache[key] = (seqCache[key] || 0) + 1;
  return `${key}${String(seqCache[key]).padStart(4, '0')}`;
}
```

## 5. src/seed.js（种子账号）

```js
import { db } from './db.js';
import { hashPassword } from './auth.js';

const ROLES = [
  { code: 'admin', name: '系统管理员', description: '所有权限' },
  // 业务角色按需加
];

const PERSONS = [
  { name: '管理员', employee_no: 'A001', phone: '13800000000', email: 'admin@example.com',
    roles: ['admin'], username: 'admin', password: 'admin123' },
];

const insRole = db.prepare('INSERT INTO roles (code, name, description) VALUES (?, ?, ?)');
for (const r of ROLES) {
  try { insRole.run(r.code, r.name, r.description); } catch {}
}

const getRoleId = db.prepare('SELECT id FROM roles WHERE code = ?');
const insPerson = db.prepare('INSERT INTO persons (name, employee_no, phone, email) VALUES (?, ?, ?, ?)');
const insAccount = db.prepare('INSERT INTO accounts (username, password_hash, person_id, enabled) VALUES (?, ?, ?, 1)');
const insPersonRole = db.prepare('INSERT INTO person_roles (person_id, role_id) VALUES (?, ?)');

for (const p of PERSONS) {
  try {
    const r = insPerson.run(p.name, p.employee_no, p.phone, p.email);
    const pid = r.lastInsertRowid;
    for (const code of p.roles) {
      const role = getRoleId.get(code);
      if (role) insPersonRole.run(pid, role.id);
    }
    insAccount.run(p.username, hashPassword(p.password), pid);
    console.log(`✓ ${p.username} / ${p.password}`);
  } catch (e) {
    console.log(`× ${p.username}: ${e.message}`);
  }
}
```

## 6. src/routes/auth.js（登录 + 修改密码）

```js
import { db } from '../db.js';
import { verifyPassword, hashPassword, getCurrentUser } from '../auth.js';
import { now } from '../util.js';

export default async function authRoutes(app) {
  app.post('/auth/login', async (req, reply) => {
    const { username, password } = req.body || {};
    if (!username || !password) return reply.code(400).send({ error: '账号密码必填' });
    const acc = db.prepare('SELECT * FROM accounts WHERE username = ? AND enabled = 1').get(username);
    if (!acc || !verifyPassword(password, acc.password_hash))
      return reply.code(401).send({ error: '账号不存在或密码错误' });
    db.prepare('UPDATE accounts SET last_login_at = ? WHERE id = ?').run(now(), acc.id);
    const token = app.jwt.sign({ id: acc.id, username: acc.username }, { expiresIn: '7d' });
    const ctx = getCurrentUser({ user: { id: acc.id } });
    return { token, user: { id: acc.id, username: acc.username, person: ctx.person, roles: ctx.roles } };
  });

  app.get('/auth/me', { preHandler: app.authHook }, async (req) => {
    const ctx = getCurrentUser(req);
    return { id: ctx.account.id, username: ctx.account.username, person: ctx.person, roles: ctx.roles };
  });

  app.post('/auth/change-password', { preHandler: app.authHook }, async (req, reply) => {
    const { old_password, new_password } = req.body || {};
    const acc = db.prepare('SELECT * FROM accounts WHERE id = ?').get(req.user.id);
    if (!verifyPassword(old_password, acc.password_hash))
      return reply.code(400).send({ error: '旧密码错误' });
    db.prepare('UPDATE accounts SET password_hash = ? WHERE id = ?').run(hashPassword(new_password), acc.id);
    return { ok: true };
  });
}
```

## 7. src/routes/persons.js（标准 CRUD 模板）

照这套写新业务路由：

```js
import { db } from '../db.js';
import { now } from '../util.js';

export default async function personsRoutes(app) {
  app.addHook('preHandler', app.authHook);   // 全模块要登录

  // GET /persons?keyword=&page=1&page_size=10
  app.get('/persons', async (req) => {
    const { keyword = '', page = 1, page_size = 10 } = req.query || {};
    const offset = (page - 1) * page_size;
    let where = '';
    const args = [];
    if (keyword) {
      where = ' WHERE name LIKE ? OR employee_no LIKE ?';
      args.push(`%${keyword}%`, `%${keyword}%`);
    }
    const total = db.prepare(`SELECT COUNT(*) AS n FROM persons${where}`).get(...args).n;
    const list = db.prepare(`SELECT * FROM persons${where} ORDER BY id DESC LIMIT ? OFFSET ?`)
      .all(...args, +page_size, +offset);
    return { list, total, page: +page, page_size: +page_size };
  });

  // GET /persons/:id
  app.get('/persons/:id', async (req, reply) => {
    const row = db.prepare('SELECT * FROM persons WHERE id = ?').get(+req.params.id);
    if (!row) return reply.code(404).send({ error: '未找到' });
    return row;
  });

  // POST /persons
  app.post('/persons', async (req, reply) => {
    const { name, employee_no, phone, email } = req.body || {};
    if (!name) return reply.code(400).send({ error: 'name 必填' });
    try {
      const r = db.prepare(`
        INSERT INTO persons (name, employee_no, phone, email)
        VALUES (?, ?, ?, ?)
      `).run(name, employee_no || null, phone || null, email || null);
      return { id: r.lastInsertRowid };
    } catch (e) {
      if (String(e.message).includes('UNIQUE')) return reply.code(400).send({ error: '工号已存在' });
      throw e;
    }
  });

  // PUT /persons/:id
  app.put('/persons/:id', async (req, reply) => {
    const id = +req.params.id;
    const row = db.prepare('SELECT * FROM persons WHERE id = ?').get(id);
    if (!row) return reply.code(404).send({ error: '未找到' });
    const { name, employee_no, phone, email } = req.body || {};
    db.prepare(`
      UPDATE persons SET
        name = COALESCE(?, name),
        employee_no = COALESCE(?, employee_no),
        phone = COALESCE(?, phone),
        email = COALESCE(?, email)
      WHERE id = ?
    `).run(name, employee_no, phone, email, id);
    return { ok: true };
  });

  // DELETE /persons/:id
  app.delete('/persons/:id', async (req, reply) => {
    const r = db.prepare('DELETE FROM persons WHERE id = ?').run(+req.params.id);
    if (r.changes === 0) return reply.code(404).send({ error: '未找到' });
    return { ok: true };
  });
}
```

## 8. 权限守卫常见模式（按需用）

如果业务有"非 admin 只能动自己创建的记录"诉求，可以加这个 helper：

```js
import { getCurrentUser } from '../auth.js';

function checkCanOperate(req, row) {
  const ctx = getCurrentUser(req);
  if (!ctx) return { code: 401, body: { error: '未登录' } };
  if (ctx.roles.some((r) => r.code === 'admin')) return null;
  if (row.creator_id === ctx.person.id) return null;
  return { code: 403, body: { error: '只能操作自己创建的记录' } };
}

// 用法
app.put('/biz/:id', async (req, reply) => {
  const row = db.prepare('SELECT * FROM biz WHERE id = ?').get(+req.params.id);
  if (!row) return reply.code(404).send({ error: '未找到' });
  const guard = checkCanOperate(req, row);
  if (guard) return reply.code(guard.code).send(guard.body);
  // ...
});
```

业务表加 `creator_id INTEGER` 列即可。

## 9. 关键决策回放

| 决策 | 选什么 | 不选什么 | 原因 |
|---|---|---|---|
| ORM | 无（裸 SQL）| Prisma / Drizzle / TypeORM | better-sqlite3 同步 API 已经够 |
| 鉴权 | JWT + localStorage | session + cookie | 内网 + axios 拦截 401，简单 |
| 权限 | 硬编码 admin + 业务规则 | Casbin / 自写 RBAC | 5-15 人系统过度设计 |
| 迁移 | safeExec ALTER | knex / typeorm migration | 单文件、幂等 |
| 单号 | nextSeq 工具 | nanoid / uuid | 业务方要看得懂的单号 |
| 时区 | 字符串 + datetime('now','localtime') | UTC + 转换 | SQLite 本地时间 + `substr(t,1,10)` 比较 |
