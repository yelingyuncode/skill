# 单据类实体的统一状态机

**订单 / 工单 / 领料单 / 报工单 / 盘点单 / 质检单 / 采购单 / 出入库单** 一律走这个状态流。别的类型（用户、产品、角色）不属于"单据"，不套这个。

## 标准 6 态

```
        ┌──────────────┐
        │  草稿 draft  │  ← 新建时默认
        └──────┬───────┘
               │ 提交
               ▼
        ┌──────────────┐
   ┌────┤ 待审 pending │
   │    └──────┬───────┘
   │           │ 通过
   │           ▼
   │    ┌────────────────┐
   │    │ 已发布 released │   ← 可以下发到车间
   │    └──────┬─────────┘
   │           │ 开始执行
   │           ▼
   │    ┌──────────────────────┐
   │    │ 进行中 in_progress   │
   │    └──────┬───────────────┘
   │           │ 完成
   │           ▼
   │    ┌────────────────────┐
   │    │ 完成 completed     │  ← 终态
   │    └────────────────────┘
   │
   │           或任意状态取消
   └───────────► 作废 cancelled  ← 终态
```

**枚举字符串**（不许改）：

```js
const ORDER_STATUS = Object.freeze({
  DRAFT:       'draft',
  PENDING:     'pending',
  RELEASED:    'released',
  IN_PROGRESS: 'in_progress',
  COMPLETED:   'completed',
  CANCELLED:   'cancelled',
});
```

## 允许的状态迁移

只允许下面这 8 条边，其他一律拒绝：

| 从 | 到 | 触发 | 谁能触发 |
|---|---|---|---|
| draft | pending | 提交审核 | 创建人 |
| draft | cancelled | 直接作废 | 创建人 |
| pending | released | 审核通过 | 有审核权限的人 |
| pending | draft | 打回修改 | 有审核权限的人 |
| pending | cancelled | 审核拒绝并作废 | 有审核权限的人 |
| released | in_progress | 开始执行 | 车间执行人 |
| in_progress | completed | 完成 | 车间执行人 |
| released / in_progress | cancelled | 中途作废 | 有作废权限的人 |

**其他任意 from → to 都不合法**，比如 completed → draft、draft → completed、in_progress → pending。

## 后端强制校验（不能靠前端）

```js
// server/src/lib/status-transition.js
const ALLOWED = {
  draft:       new Set(['pending', 'cancelled']),
  pending:     new Set(['released', 'draft', 'cancelled']),
  released:    new Set(['in_progress', 'cancelled']),
  in_progress: new Set(['completed', 'cancelled']),
  completed:   new Set(),   // 终态
  cancelled:   new Set(),   // 终态
};

export function assertTransition(from, to) {
  const allowed = ALLOWED[from];
  if (!allowed || !allowed.has(to)) {
    const err = new Error(`不允许的状态迁移: ${from} → ${to}`);
    err.code = 'INVALID_TRANSITION';
    err.businessError = true;
    throw err;
  }
}
```

每次改 `status` 前调 `assertTransition(current, next)`。**前端 button 隐藏是好用户体验；后端 assert 是防绕过**。两个都要。

## 状态变更必留 audit

每次状态变更**必须**写一条到 `audit_log`（见 [audit-trail.md](audit-trail.md)），至少含：

- 单据类型 + 单据 ID
- from 状态 + to 状态
- 操作人 ID + 时间
- 备注（如"车间反映线不够先作废"）

## 前端展示

统一的 tag 颜色（用 Element Plus）：

| 状态 | 中文 | tag type |
|---|---|---|
| draft | 草稿 | info |
| pending | 待审 | warning |
| released | 已发布 | primary |
| in_progress | 进行中 | success（带脉动动画） |
| completed | 完成 | success |
| cancelled | 作废 | danger |

```vue
<el-tag :type="statusMap[row.status].type">
  {{ statusMap[row.status].label }}
</el-tag>
```

## 需要新增状态怎么办？

**不要在单点新增**。走这个流程：

1. 确认新状态是**真的必要**（问业务方，能不能用现有 6 态 + `sub_status` 或 `remark` 表达）
2. 需要的话，改这个 skill 的 `state-machine.md` + 全公司同步
3. 表加 `status` 枚举值，代码 `ALLOWED` 加边
4. 已在跑的旧单据数据要 migrate

**AI 常犯错**：看到需求"要支持退回"就直接加 `returned` 状态。**问自己：用 cancelled + remark 能表达吗？** 大多数情况能。
