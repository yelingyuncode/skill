# 业务字段兜底规范

每一张业务表都必须先有这 5 项，才允许加业务字段。缺一项 = review 打回。

## 5 项兜底

```sql
-- 每张业务表的最小骨架
CREATE TABLE {{业务表名}} (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,   -- 或 TEXT PRIMARY KEY，全局唯一
  created_at    INTEGER NOT NULL,                    -- 秒级 unix ts，用 now() 工具
  updated_at    INTEGER NOT NULL,
  created_by    INTEGER NOT NULL,                    -- 账号 ID，不允许 NULL
  updated_by    INTEGER NOT NULL,
  status        TEXT NOT NULL DEFAULT 'draft',       -- 业务状态枚举，见 state-machine.md
  remark        TEXT DEFAULT '',                     -- 备注，车间常写线下沟通
  -- ↓ 下面才是业务字段
  ...
);
```

### 每项的用途和硬约束

| 字段 | 类型 | NULL | 谁写 | 用途 |
|---|---|---|---|---|
| `id` | INTEGER PK 自增 或 TEXT PK | ❌ | 数据库/代码 | 主键，不许业务字段做主键 |
| `created_at` | INTEGER 秒级 | ❌ | 后端 `now()` | 报表/审计基准 |
| `updated_at` | INTEGER 秒级 | ❌ | 后端 `now()` | 每次 UPDATE 必更新 |
| `created_by` | INTEGER 账号 ID | ❌ | 后端 req.user.id | 追责 |
| `updated_by` | INTEGER 账号 ID | ❌ | 后端 req.user.id | 追责 |
| `status` | TEXT 枚举 | ❌ | 业务代码 | 状态机，即使当前只有 1 个状态也留 |
| `remark` | TEXT | ✅（默认 `''`） | 前端/后端 | 车间线下沟通、订单特殊要求 |

### 常见踩坑

❌ **用 `DATETIME` / `TIMESTAMP`**：SQLite 没这类型，会存成 TEXT，排序/比较全出错。**用秒级 INTEGER**。
❌ **`created_by` 允许 NULL**：那怎么追责？连"系统操作"也要留一个 `system` 用户 ID。
❌ **`status` 用 boolean `is_active`**：一年后必然要加 `pending` / `frozen` / `archived`，重构成本极高。
❌ **忘了 `remark`**：车间会私聊你要求"能不能加个备注啊"，你要连夜加。**先留着**。
❌ **`updated_at` 用触发器自动更新**：Better-sqlite3 触发器要小心事务嵌套。**在应用层写**，明确可控。

## 命名规则

- 表名：**单数 snake_case**：`order` / `work_order` / `stock_move`（不用 `orders`）
- 字段名：**snake_case**：`created_at` / `assigned_user_id`
- 外键：**关联表名 + `_id`**：`order_id` / `product_id`
- 布尔字段：**尽量避免**；实在要用 `is_xxx` 且明确"没有第三种可能"（`is_locked` 而非 `is_active`）
- 金额：**INTEGER，单位分**（避免浮点误差）
- 日期字符串（前端展示）：**在后端 `dayjs().format()` 转好后返回**；数据库不存字符串日期

## 索引规范

**默认加索引的字段**：
- 所有外键：`order_id`, `product_id`, `workshop_id` 等
- `status`（经常按状态筛列表）
- `created_at`（默认按创建时间倒序列表）
- 业务查询高频字段：单号 `order_no`, 批号 `batch_no`, 工号 `emp_no`

```sql
CREATE INDEX idx_order_status_created ON `order`(status, created_at DESC);
CREATE INDEX idx_order_created_by     ON `order`(created_by);
CREATE INDEX idx_order_no             ON `order`(order_no);
```

**别过度索引**：SQLite 每个索引都要维护，5000 行以下的表可以不加索引。

## 业务字段设计的问自己

新增一个业务字段前问 4 个问题：

1. **是不是能从别的字段推出来？**（比如 `order_total = sum(order_item.price * qty)`）能推就不存
2. **业务方是不是明天就想改这个值？**（是就存字段，别硬编码常量）
3. **要不要留历史？**（会变的价格 → 单独历史表；不变的产品名 → 直接字段）
4. **NULL 会不会让后端崩？**（会就 `DEFAULT ''` 或 `DEFAULT 0`；别相信"业务方会填的"）

## 单据号（order_no）生成规则

单据类实体（订单/工单/领料单/报工单）**必须有人类可读的单号**，规则统一：

```
类型前缀 + YYMMDD + 3 位流水
例如：
  订单     SO250923001
  工单     WO250923001
  领料单   ML250923001
  报工单   WR250923001
```

**用一张 `sequence` 表按天累加**，不要用 `count(*)+1`（并发下重号）。

```sql
CREATE TABLE sequence (
  prefix    TEXT NOT NULL,       -- 'SO' / 'WO' 等
  date_key  TEXT NOT NULL,       -- '250923'
  seq       INTEGER NOT NULL,    -- 当天最新流水
  PRIMARY KEY (prefix, date_key)
);
```

单号生成走事务：`SELECT … FOR UPDATE 效果` 通过 SQLite 的单线程写保证，只要在 `db.transaction()` 里即可。
