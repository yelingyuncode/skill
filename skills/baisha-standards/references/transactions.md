# 事务 + 幂等 + 并发

**核心一句话**：一次业务操作 = 一次 `db.transaction()`。跨表更新绝不允许拆到多个 handler。

## §1 事务边界 = 业务动作

### 判定"一次业务动作"

不是"一次点击"，也不是"一次 HTTP 请求"，而是"业务上不可分割的一件事"。

| 业务动作 | 涉及表 | 事务边界 |
|---|---|---|
| 提交订单 | sales_order + order_item + audit_log | 全部一个 transaction |
| 审核订单 | sales_order.status + audit_log | 一个 transaction |
| 发布工单到车间 | work_order + machine_assignment + audit_log | 一个 transaction |
| 领料 | material_move + stock + audit_log | 一个 transaction |
| 报工完成 | work_order.status + production_log + stock（成品入库） + audit_log | 一个 transaction |
| 盘点差异调整 | stock_snapshot + stock + audit_log | 一个 transaction |

### 代码模式

**better-sqlite3** 的 transaction 是同步的，写起来最简单：

```js
// server/src/routes/order.js
fastify.post('/orders', async (req, reply) => {
  const { user } = req;
  const { items, remark } = req.body;

  // 参数校验放事务外
  if (!items?.length) return { ok:false, code:'NO_ITEMS', msg:'请至少选择一个物料' };

  try {
    const result = db.transaction(() => {
      // ① 生成单号
      const orderNo = nextSeq('SO');

      // ② 插主表
      const orderId = db.prepare(`
        INSERT INTO sales_order(order_no, created_at, updated_at, created_by, updated_by, status, remark)
        VALUES (?, ?, ?, ?, ?, 'draft', ?)
      `).run(orderNo, now(), now(), user.id, user.id, remark ?? '').lastInsertRowid;

      // ③ 插明细
      const insItem = db.prepare(`
        INSERT INTO order_item(order_id, product_id, qty, created_at, updated_at, created_by, updated_by, status, remark)
        VALUES (?, ?, ?, ?, ?, ?, ?, 'active', '')
      `);
      for (const it of items) insItem.run(orderId, it.product_id, it.qty, now(), now(), user.id, user.id);

      // ④ 审计
      audit(user.id, 'sales_order', orderId, 'create', { orderNo, itemCount: items.length });

      return { orderId, orderNo };
    })();

    return { ok: true, data: result };
  } catch (err) {
    if (err.businessError) return { ok: false, code: err.code, msg: err.message };
    throw err; // 系统错交给全局错误处理
  }
});
```

### 反面（❌ AI 常犯）

```js
// ❌ 拆到 route 外做，主表已插，明细失败——脏数据
async function createOrder(req) {
  const orderId = insertOrder(req.body);          // 提交 1
  for (const it of req.body.items) {
    insertItem(orderId, it);                       // 提交 2、3、4…
  }
  audit('create', orderId);                        // 提交 N
}
```

```js
// ❌ setTimeout / Promise.all 破坏事务边界
db.transaction(() => {
  insertOrder(...);
  setTimeout(() => insertItem(...), 100);   // ← 已在事务外！
})();
```

```js
// ❌ 事务里调外部 HTTP —— SQLite 锁住整个库
db.transaction(() => {
  await fetch('http://mes/api/...');   // ← 事务里等 IO，其他写全阻塞
  updateOrder(...);
})();
```

**规则**：事务里只做本地 DB 操作，外部调用（HTTP / 邮件 / 短信）**放事务提交后**做（可靠性差点，用 outbox 表补偿；99% 场景可以接受）。

## §2 幂等（idempotency）

### 哪些动作要幂等

**改变现实世界状态的关键动作**，用户可能重复点：

- 订单提交 / 审核 / 发布
- 工单发布 / 报工 / 完成
- 领料 / 出入库
- 支付 / 结算

**只读列表 / 详情 / 打印**不需要幂等。

### 三种做法（按简单到复杂排序）

#### A. 状态检查（最简单，够用 80%）

利用状态机的单向性：**如果当前状态已经是目标状态，直接返回成功**。

```js
db.transaction(() => {
  const order = db.prepare('SELECT status FROM sales_order WHERE id=?').get(id);
  if (order.status === 'released') return { alreadyDone: true }; // 已发布，直接成功
  assertTransition(order.status, 'released');
  db.prepare("UPDATE sales_order SET status='released', updated_at=?, updated_by=? WHERE id=?").run(now(), user.id, id);
})();
```

用户狂点"发布"按钮 → 第一次改状态 → 第二次进来发现已经 released → 返回成功。**没有副作用**。

#### B. 前端 loading 禁用（配合 A 用）

```vue
<el-button :loading="submitting" @click="submit">提交</el-button>
```

```js
async function submit() {
  if (submitting.value) return;
  submitting.value = true;
  try { await api.submit(...) } finally { submitting.value = false }
}
```

99% 场景 A + B 已经够。

#### C. client_request_id（金额敏感/严格场景）

前端每次发起请求生成 UUID，后端插 `idempotency_key` 表，主键冲突就返回原结果。用于**领料 / 出入库 / 报工数量提交**这类"数量会加倍"的场景。

```sql
CREATE TABLE idempotency_key (
  key         TEXT PRIMARY KEY,        -- 前端 UUID
  action      TEXT NOT NULL,           -- 'material_out' / 'work_report' 等
  result_json TEXT NOT NULL,
  created_at  INTEGER NOT NULL
);
-- 24h 后清理
```

```js
db.transaction(() => {
  try {
    db.prepare('INSERT INTO idempotency_key(key, action, result_json, created_at) VALUES (?, ?, ?, ?)').run(clientKey, 'material_out', '', now());
  } catch (e) {
    if (e.code === 'SQLITE_CONSTRAINT_PRIMARYKEY') {
      const prev = db.prepare('SELECT result_json FROM idempotency_key WHERE key=?').get(clientKey);
      return JSON.parse(prev.result_json); // 直接返回上次结果
    }
    throw e;
  }
  // ... 正常业务 ...
  const result = { moveId, qty };
  db.prepare('UPDATE idempotency_key SET result_json=? WHERE key=?').run(JSON.stringify(result), clientKey);
  return result;
})();
```

## §3 并发（内网 5-15 人场景）

### 内部系统的并发规模

不是 C 端。真实场景：

- 早上上班同时打卡查看订单：**读并发 10–20 QPS**
- 车间报工高峰同时提交：**写并发 2–5 QPS**
- 后台数据导出：**单个大查询**

SQLite WAL 模式下**读不阻塞**、**写串行**。5-15 人日常够用，**别提前上 Postgres**。

### 什么时候会出并发问题

- 两个人同时改同一张单（覆盖问题）
- 车间同一台机器双人报同一工单（重复报工）
- 库存并发扣减（可能超扣）

### 三个防护手段（按需要程度）

#### A. 版本号乐观锁（推荐做的）

给经常并发编辑的单据加 `version INTEGER DEFAULT 0` 字段：

```js
// 前端把 version 传回来
const r = db.prepare(`
  UPDATE sales_order SET remark=?, updated_at=?, updated_by=?, version=version+1
  WHERE id=? AND version=?
`).run(remark, now(), user.id, id, clientVersion);

if (r.changes === 0) {
  return { ok:false, code:'CONFLICT', msg:'该单已被他人修改，请刷新后重试' };
}
```

**成本几乎为零，效果显著**，值得给核心单据加。

#### B. 数据库层唯一约束（防重复业务）

比如"同一工单同一天不许两次报工"：

```sql
CREATE UNIQUE INDEX idx_work_report_unique
  ON work_report(work_order_id, report_date);
```

INSERT 时冲突 → 返回业务错。

#### C. 显式行锁（SQLite 用 IMMEDIATE 事务）

极端场景，比如"扣库存"。**better-sqlite3 默认已经是 IMMEDIATE**，串行写。

```js
// SELECT + UPDATE 同一事务内即可保证一致
db.transaction(() => {
  const stock = db.prepare('SELECT qty FROM stock WHERE product_id=?').get(pid);
  if (stock.qty < needQty) {
    const err = new Error('库存不足');
    err.code = 'INSUFFICIENT_STOCK';
    err.businessError = true;
    throw err;
  }
  db.prepare('UPDATE stock SET qty=qty-?, updated_at=?, updated_by=? WHERE product_id=?').run(needQty, now(), user.id, pid);
})();
```

SQLite 单进程写 → 天然串行 → 上面这段绝对不会超扣。**不用另外加锁**。

## §4 checklist（AI 写完后自查）

- [ ] 一次业务操作是不是包在**一个 `db.transaction()`** 里？
- [ ] 事务里是不是**没有外部 IO**（HTTP / 邮件 / 长计算）？
- [ ] 关键动作有没有**状态检查幂等**（已完成状态直接返回成功）？
- [ ] 前端提交按钮有没有 `:loading` 禁用？
- [ ] 涉及数量增减的（库存、报工），有没有想过**双人重复提交**怎么办？
- [ ] 并发编辑的单据加没加 `version` 字段？
