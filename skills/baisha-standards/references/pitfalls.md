# 通用踩坑（12 条）

> 都是真实在「活动盖板订单」系统里踩过的。新项目复用这套架构前先扫一遍。
> 业务相关的坑（外部 API 集成、群消息渲染、单位换算等）不在这——这里只留**通用框架坑**。

## 前端类

### 1. 路由 `:id` 切换不刷新

**症状**：详情页里跳"下一条"（路由 `:id` 变了）后页面内容没变。

**根因**：Vue Router 默认复用组件实例，**不触发 onMounted**。

**修法**：

```js
import { watch } from 'vue';
const id = computed(() => +route.params.id);
watch(id, load, { immediate: true });   // 替代 onMounted
```

### 2. 编辑入口被 URL 直接访问绕过权限

**症状**：用户 A 在地址栏直接输 `/biz/123/edit` 能编辑用户 B 的记录。

**修法**：`Edit.vue` 在 `load()` 里再校验一次

```js
async function load() {
  const data = await api.detail(id.value);
  if (mode.value === 'edit' && !userStore.isAdmin && data.creator_id !== userStore.user.person.id) {
    ElMessage.error('只能编辑自己创建的记录');
    router.replace(`/biz/${id.value}`);
    return;
  }
}
```

后端**同时**要在 PUT 接口做同样的守卫（双保险）。

### 3. 表格横向滚动条只在底部

**症状**：列多的表格横向滚动条出现在表格底部，鼠标得拖到底部才能滑。

**修法**：`el-table` 加 `max-height`

```vue
<el-table :data="list" max-height="600" border>
```

## SQL 类

### 4. SQL 字面量双引号被当列名

**症状**：搜索接口报 `no such column: ""`。

**根因**：

```js
sql += ' WHERE name LIKE ? OR COALESCE(code, "") LIKE ?';   // 双引号 ❌
```

SQLite 把 `""` 当作零长度列名，找不到就报错。

**修法**：用单引号

```js
sql += " WHERE name LIKE ? OR COALESCE(code, '') LIKE ?";   // 单引号 ✓
```

**通用教训**：SQL 里字符串字面量**永远用单引号**，双引号在 SQLite 里是标识符（列名 / 表名）。

### 5. SQLite 字符串日期比较

**症状**：`WHERE created_at > '2026-06-15'` 漏掉当天的记录。

**根因**：`created_at` 形如 `'2026-06-15 14:30:00'`，字符串字典序比较时，`'2026-06-15 14:30:00' = '2026-06-15'` 是 false。

**修法**：用 `substr` 取日期前缀

```sql
WHERE substr(created_at, 1, 10) = '2026-06-15'
```

**不要用** `DATE(created_at) = ...` —— SQLite 的 `DATE()` 按 UTC 转，跟存的 localtime 错位。

### 6. WAL 模式下 `data.db` 主文件不包含未 checkpoint 的事务

**症状**：直接 `scp data.db` 把库拉到本地，发现少了最近几小时的写入。

**根因**：WAL 模式下新写入先进 `data.db-wal`，`data.db` 主文件只有 checkpoint 后才包含。

**修法**：拉库前先 `checkpoint`

```bash
sqlite3 data.db 'PRAGMA wal_checkpoint(TRUNCATE);'
scp ...
```

部署脚本 `db-pull.sh` 已经做了。

## 后端类

### 7. Fastify 5 空 JSON body 报 400

**症状**：动作端点（如 `POST /biz/:id/submit`）不带 body 就报 `Body cannot be empty`。

**修法**：在 `index.js` 重装内容类型解析器

```js
app.removeContentTypeParser('application/json');
app.addContentTypeParser('application/json', { parseAs: 'string' }, (req, body, done) => {
  if (!body) return done(null, {});
  try { done(null, JSON.parse(body)); } catch (e) { done(e); }
});
```

### 8. `cors: { origin: true }` 在公网下是漏洞

**症状**：内网用没事，对外映射后任意域名能调你的 API（CSRF）。

**根因**：`origin: true` 表示"反射来源域名"，等于全开。

**修法**：白名单

```js
const allowedOrigins = ['https://your-domain.com', 'http://192.168.x.x'];
await app.register(cors, {
  origin: (origin, cb) => {
    if (!origin || allowedOrigins.includes(origin)) return cb(null, true);
    return cb(new Error('Not allowed by CORS'), false);
  },
  credentials: true,
});
```

### 9. JWT_SECRET 默认值留在 prod

**症状**：暴露公网后，攻击者用代码里的 `'dev-secret-change-me'` 字面量签出任意账号 JWT。

**修法**：systemd 单元里 `Environment=JWT_SECRET=<32 位随机串>`。代码里只读 env，缺了就拒启动：

```js
if (!process.env.JWT_SECRET) {
  console.error('JWT_SECRET 未配置，拒绝启动');
  process.exit(1);
}
await app.register(jwt, { secret: process.env.JWT_SECRET });
```

部署前先生成：

```bash
openssl rand -base64 48
```

## 部署类

### 10. CentOS 7 装 better-sqlite3 v11 失败

**症状**：`npm install better-sqlite3` 报 `GLIBC_2.28 not found`。

**根因**：CentOS 7 的 glibc 是 2.17，better-sqlite3 v11 链接的是 2.28。

**修法**：锁 v10

```json
"better-sqlite3": "^10.1.0"
```

并装 `devtoolset-9` 让 g++ 能编：

```bash
yum install -y centos-release-scl devtoolset-9
source /opt/rh/devtoolset-9/enable
npm install
```

### 11. `vite preview` 不读 `server.proxy`

**症状**：生产服务器 `vite preview` 跑前端，`/api` 请求全 404。

**根因**：`vite.config.js` 里的 `server.proxy` 只对 `vite dev` 生效，`vite preview` 要单独配 `preview.proxy`。

**修法**：

```js
export default defineConfig({
  server:  { proxy: { '/api': { target: 'http://localhost:4000' } } },
  preview: { proxy: { '/api': { target: 'http://localhost:4000' } } },   // ← 加这段
});
```

**更稳的做法**：生产用 **nginx 反代**，不用 vite preview。

### 12. CentOS 7 默认 Node 跑不动现代依赖

**症状**：服务器 `node` 是 v8 / v10，跑不动 `import.meta` 或者 ES modules。

**修法**：用 nvm 装 Node 18

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 18
nvm alias default 18
```

systemd 单元里 `ExecStart` 写 nvm 装的绝对路径（不要写 `node`，PATH 在 systemd 里不一样）：

```ini
ExecStart=/root/.nvm/versions/node/v18.20.4/bin/node --no-warnings src/index.js
```

---

## 反模式速查

| 一句话总结 | 不要 | 要 |
|---|---|---|
| SQL 字面量 | 双引号 | 单引号 |
| SQLite 日期比较 | `DATE()` | `substr(t,1,10)` |
| 路由 `:id` 切换 | 靠 `onMounted` | `watch(id, load)` |
| 前端权限 | 只前端 v-if | 前端 v-if + 后端守卫双保险 |
| 部署 | better-sqlite3 v11 | 锁 v10 + devtoolset-9 |
| 前端生产 | `vite preview` 不配 proxy | 配 `preview.proxy` 或上 nginx |
| CORS | `origin: true` 上公网 | 白名单 |
| JWT secret | 代码里写死 | systemd env 注入，缺就拒启动 |
| HTTP baseURL | 绝对 URL | 相对 `/api`，proxy / nginx 转 |
| 拉生产 DB | 直接 scp | 先 `PRAGMA wal_checkpoint(TRUNCATE)` |
| Node 路径 | systemd 里写 `node` | 写绝对路径 `/root/.nvm/.../node` |
