# 审计与追溯

**为什么这条特别重要**：白鲨的场景里，车间会说"我明明改过啊"、"这单不是我下的"、"那批货怎么变了"。**没有 audit_log 你无法反驳/查明**。所以这是硬规矩。

## §1 audit_log 表结构

```sql
CREATE TABLE audit_log (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  ts           INTEGER NOT NULL,               -- 秒级 unix ts
  actor_id     INTEGER NOT NULL,               -- 谁做的（账号 ID）
  actor_name   TEXT NOT NULL,                  -- 冗余存名字，账号删了也能查
  entity       TEXT NOT NULL,                  -- 'sales_order' / 'work_order' / 'stock' 等
  entity_id    TEXT NOT NULL,                  -- 该实体的主键，转成字符串
  action       TEXT NOT NULL,                  -- 'create' / 'update' / 'delete' / 'status_change' / 'login' / 'export'
  before_json  TEXT,                           -- 变更前快照（UPDATE / status_change 时）
  after_json   TEXT,                           -- 变更后快照
  meta_json    TEXT,                           -- 附加信息（IP、UA、单号、原因）
  remark       TEXT DEFAULT ''
);

CREATE INDEX idx_audit_entity        ON audit_log(entity, entity_id, ts DESC);
CREATE INDEX idx_audit_actor         ON audit_log(actor_id, ts DESC);
CREATE INDEX idx_audit_ts            ON audit_log(ts DESC);
```

## §2 什么时候记

### 必记（漏一个 = review 打回）

- **所有 INSERT** 业务表 → `action='create'`, `after_json=新记录`
- **所有 UPDATE** 业务表 → `action='update'`, `before_json/after_json` 只存**变了的字段**
- **所有 DELETE**（含软删 status='deleted'）→ `action='delete'`
- **所有状态变更** → `action='status_change'`, `before_json={status:'draft'}`, `after_json={status:'pending'}`
- **登录 / 登出** → `action='login'` / `'logout'`, `meta_json={ip, ua}`
- **数据导出**（涉及批量导出敏感数据）→ `action='export'`, `meta_json={filter, rowCount}`

### 不用记

- 纯 SELECT / 详情查看
- 前端菜单点击
- 密码修改（**要另单独记**，`before/after` 不存明文）

## §3 audit 工具函数

```js
// server/src/lib/audit.js
import { db } from '../db.js';
import { now } from '../util.js';

export function audit({ user, entity, entityId, action, before, after, meta, remark }) {
  db.prepare(`
    INSERT INTO audit_log(ts, actor_id, actor_name, entity, entity_id, action, before_json, after_json, meta_json, remark)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
  `).run(
    now(),
    user.id,
    user.name ?? '',
    entity,
    String(entityId),
    action,
    before ? JSON.stringify(diffOnly(before, after)) : null,
    after  ? JSON.stringify(diffOnly(after, before))  : null,
    meta   ? JSON.stringify(meta) : null,
    remark ?? ''
  );
}

// 只保留有变化的字段，减少存储
function diffOnly(a, b) {
  if (!b) return a;
  const out = {};
  for (const k of Object.keys(a)) {
    if (JSON.stringify(a[k]) !== JSON.stringify(b[k])) out[k] = a[k];
  }
  return out;
}
```

## §4 用法示例

### 新建订单

```js
db.transaction(() => {
  const orderId = insertOrder(...);
  audit({ user, entity: 'sales_order', entityId: orderId, action: 'create',
          after: { order_no, items_count: items.length, status: 'draft' }});
})();
```

### 修改订单

```js
db.transaction(() => {
  const before = db.prepare('SELECT * FROM sales_order WHERE id=?').get(id);
  db.prepare('UPDATE sales_order SET remark=?, updated_at=?, updated_by=? WHERE id=?').run(remark, now(), user.id, id);
  const after = db.prepare('SELECT * FROM sales_order WHERE id=?').get(id);
  audit({ user, entity: 'sales_order', entityId: id, action: 'update', before, after });
})();
```

### 状态迁移

```js
db.transaction(() => {
  const before = { status: 'pending' };
  assertTransition('pending', 'released');
  db.prepare("UPDATE sales_order SET status='released', updated_at=?, updated_by=? WHERE id=?").run(now(), user.id, id);
  audit({ user, entity: 'sales_order', entityId: id, action: 'status_change',
          before, after: { status: 'released' }, remark: '车间审核通过' });
})();
```

## §5 查看审计（工具页面）

后台建一个页面 `/admin/audit`，能按：

- 实体类型 + 单号查（车间说"这单被谁改过"）
- 操作人查（账号异常时看这人做了啥）
- 时间范围查
- action 类型筛选

**这个页面权限**：只给管理员 + 信息中心。

## §6 保留周期

- **业务表 audit_log**：保留 **3 年**（够审计和车间对账）
- **登录 audit_log**：保留 **1 年**
- **导出 audit_log**：保留 **永久**（合规上有用）

超过周期的定期归档到 `.db` 快照文件后从主库清理，别让 audit_log 占满 SQLite（超过 10GB 影响性能）。

## §7 什么时候看 audit_log 就能解决问题

| 车间反馈 | 你去查什么 |
|---|---|
| "我明明改了啊，怎么没保存" | `entity=<单据> entity_id=<单号> ORDER BY ts DESC` 看最后一次 update |
| "这单不是我下的" | `entity=<单据> action='create'` 看 actor_id |
| "这批货量怎么变了" | `entity=stock action='update'` 时间窗口内 |
| "有人半夜登录做事" | `action='login' ts BETWEEN <凌晨>` |
| "报表数据对不上" | 涉及的表 update/status_change 记录做差 |

**遇到"扯不清"的争议，第一反应就是 audit_log**。这是白鲨内部系统最有价值的一层保险。
