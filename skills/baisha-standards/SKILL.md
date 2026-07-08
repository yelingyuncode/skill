---
name: baisha-standards
description: 用于光山白鲨内部管理系统的开发、评审和部署，覆盖 MES、订单、生产、车间、采购、库存、质量、工单、报工、追溯、盘点、报表等场景；规范 Lite/Standard profile、字段兜底、状态机、事务、审计、错误处理和目录结构。
---

# 白鲨内部系统开发规范

只要在白鲨内部管理系统里写、改、审、部署业务代码，先按本文件走判断流程；需要细节时再读对应子文档。

**核心信条**：
1. **最好的代码是不需要写的代码**——写前先复用。见 [reuse-first.md](references/reuse-first.md)
2. **架构不是一成不变的**——先选型，再锁定。见 [stack-selection.md](references/stack-selection.md)
3. **业务规范是跨栈的**——字段兜底 / 状态机 / 事务 / 审计 / 错误分类，两个 profile 都一样
4. **profile 内锁死，跨 profile 有清晰迁移路径**

---

## §1 适用边界

**一定用**
- 公司任何**内部管理系统**代码（MES / 订单 / 生产 / 采购 / 库存 / 质量 / 领料 / 工单 / 报工 / 追溯 / 盘点 / 报表 / 后台工具）
- AI vibe coding 生成的业务代码在合并前必须过一遍此规范

**不用**
- C 端高并发场景
- 多租户 SaaS
- 强一致金融账务

---

## §2 判断阶梯（AI 动手前必走）

在写任何业务代码前，按顺序回答（跳级就是过度设计 / 或选错方向）：

| 顺序 | 问自己 | 若"是" | 若"否" |
|---:|---|---|---|
| **0** | **这个项目的 profile 已经确定了吗？（Lite / Standard / Scale）** | 继续 | **先做选型**，见 [stack-selection.md](references/stack-selection.md) |
| **1** | **这个功能真的需要存在吗？** | 继续 | 停手，业务方确认 |
| **2** | **骨架 / 已有模块里已经有一样或 90% 相似的实现吗？** | **复用**，看 [reuse-first.md §2](references/reuse-first.md) | 继续 |
| **3** | **标准库 / Element Plus / 已装依赖能不能做？** | **用它**，看 [reuse-first.md §3-4](references/reuse-first.md) | 继续 |
| **4** | **一张表 + 一组 CRUD 端点能解决吗？** | 只写这些，别加多余抽象 | 继续 |
| 5 | **业务字段兜底 5 类齐了吗？**（详见 [business-fields.md](references/business-fields.md)） | 继续 | 补齐 |
| 6 | **单据类实体，状态机画清楚了吗？**（详见 [state-machine.md](references/state-machine.md)） | 用统一状态枚举 | 停手画流程 |
| 7 | **写操作的事务边界在哪里？**（详见 [transactions.md](references/transactions.md)） | 一次业务 = 一次事务 | 拆错，重新想 |
| 8 | **失败会怎样？该报业务错还是系统错？**（详见 [errors.md](references/errors.md)） | 分类正确 | 补 |

任何一步"否"都先补齐或找业务方确认，别硬写。

---

## §3 硬规矩（不许改）

### 3.0 复用优先（写代码前必过一遍）→ [reuse-first.md](references/reuse-first.md)

**每加一个函数 / 组件 / 依赖前**必须问：骨架里有吗？Element Plus 有吗？已装依赖能做吗？标准库能做吗？一行能吗？——**任何一步"是"就停下**。反面清单看 [reuse-first.md §6](references/reuse-first.md)。

### 3.1 先选型，再锁定 → [stack-selection.md](references/stack-selection.md)

**新项目开工前必须选定 profile 并写 ADR**（`docs/ADR/001-choose-profile.md`）。选定后，**profile 内部的技术栈锁死**：

- **Lite（轻量 / 5-15 人 / <1GB）** → Vue 3 + Fastify 5 + SQLite（[profile-lite.md](references/profile-lite.md)）
- **Standard（标准 / 15-100 人 / 1-100GB）** → Vue 3 + FastAPI + MySQL 8（[profile-standard.md](references/profile-standard.md)）
- **Scale（规模化 / 100+ 人）** → 单独找架构师设计，本 skill 不提供模板

选定后**不许在项目内混栈**。想换 profile = 新写 ADR + 走迁移 checklist。

### 3.2 目录结构按 profile → [layout.md](references/layout.md)

**AI 不许自己发明** `domain/`、`application/`、`infrastructure/` 分层——那是给 100+ 人团队的。**每个 profile 有自己的目录约定**，照抄，别自作聪明。

### 3.3 业务字段 5 类兜底（跨 profile 一致）→ [business-fields.md](references/business-fields.md)

**每一张业务表都必须有 5 类兜底字段**（不管 SQLite 还是 MySQL）：
1. 主键：`id`
2. 时间：`created_at` + `updated_at`（秒级 INTEGER / BIGINT，不用 DATETIME）
3. 操作人：`created_by` + `updated_by`（账号 ID，不许 NULL）
4. 状态：`status`（业务状态枚举，即使当前只有 1 个状态也留字段）
5. 备注：`remark`（车间常写线下沟通信息）

缺一项 = review 打回。

### 3.4 单据类实体走统一状态机（跨 profile 一致）→ [state-machine.md](references/state-machine.md)

**订单、工单、领料单、报工单、盘点单、质检单** 一律走：

`草稿 → 待审 → 已发布 → 进行中 → 完成 / 作废`

需要额外状态先在 skill 里补，别单点扩展。

### 3.5 事务边界 = 业务动作（跨 profile 一致）→ [transactions.md](references/transactions.md)

- Lite 里：一次业务 = 一次 `db.transaction(() => {...})`
- Standard 里：一次业务 = 一次 `async with db.begin(): ...`

**关键动作强制幂等**（提交/确认/完成走状态检查或 client_request_id）。

### 3.6 写操作全部留审计（跨 profile 一致）→ [audit-trail.md](references/audit-trail.md)

INSERT / UPDATE / DELETE / 状态变更 都在 `audit_log` 表留一行。**用户改错单能追、车间对账能查、财务复核能证**——这是白鲨环境下最有价值的一件事。

### 3.7 业务错 ≠ 系统错（跨 profile 一致）→ [errors.md](references/errors.md)

- **业务错**：库存不够 / 状态不对 / 权限不足 → HTTP 200 + `{ok:false, code:'...', msg:'...'}`，前端 Message.warning
- **系统错**：数据库挂了 / undefined.prop → HTTP 500 + 日志，前端"系统繁忙"

前端 axios 拦截器区分处理，AI 常混淆。

---

## §4 vibe coding 时的"AI 提示词模板"

给 AI 下开发任务时**必须**带上：

```text
按 baisha-standards 规范：
0. 项目 profile 是 <lite | standard>（已在 docs/ADR/001 定），按对应 references/profile-*.md 锁定的技术栈，不换
1. 写之前先按 references/reuse-first.md 复用阶梯自查：骨架 / Element Plus / 已装依赖 / 标准库能做的就不写新代码
2. 表加字段满足 5 类兜底（id / 时间 / 人 / status / remark）
3. 单据类走草稿→待审→已发布→进行中→完成/作废 状态机
4. 一次业务动作 = 一次事务
5. 写操作留 audit_log
6. 报错区分业务错（返回 code+msg）和系统错（抛异常）
7. 目录按 references/layout.md 放，不发明新分层
8. 不新增依赖（不在 profile 允许清单里的），不做超出需求的抽象
```

写完让 AI **自查这 8 条各在哪里体现**，一条对不上就重来。

## §5 完工自查输出格式

写完业务代码后，必须用下面 8 项自查，说明每一项体现在哪里；不适用也要写原因：

```text
baisha-standards 自查：
1. profile：
2. 复用检查：
3. 表字段 5 类兜底：
4. 状态机：
5. 事务边界：
6. audit_log：
7. 错误处理：
8. 新依赖：
```

---

## §6 常用模式速查

| 场景 | 我该干嘛 | 去哪查 |
|---|---|---|
| **新项目开工** | 先选 profile 并写 ADR | [stack-selection.md](references/stack-selection.md) |
| **写任何新代码前** | 走一遍复用阶梯 | [reuse-first.md](references/reuse-first.md) |
| **看到 UI 需求** | 先查 Element Plus 有没有 | [reuse-first.md §3](references/reuse-first.md) |
| **想引一个新包** | 先看已装依赖能不能做 | [reuse-first.md §4](references/reuse-first.md) |
| 新建一个业务表 | 5 类兜底字段 + 状态字段 + 索引常用查询字段 | [business-fields.md](references/business-fields.md) |
| 单据从草稿到完成 | 用统一状态机 + 每次状态变更留 audit | [state-machine.md](references/state-machine.md) |
| 涉及库存/账户的更新 | 事务包起来 + 乐观锁 | [transactions.md](references/transactions.md) |
| 前端点两下同一个提交按钮 | 后端幂等 + 前端 loading 禁用 | [transactions.md](references/transactions.md) |
| 用户报"我明明改过啊" | 查 audit_log 表 | [audit-trail.md](references/audit-trail.md) |
| 前端如何区分业务错 vs 系统错 | axios 拦截器 + `ok` 字段 | [errors.md](references/errors.md) |
| Lite 新增一个后端路由 | 抄 routes/persons.js 骨架 | [server-templates.md](templates/server-templates.md) |
| Standard 新增一个后端路由 | 抄 routers/order.py 骨架 | [profile-standard.md](references/profile-standard.md) |
| 新增一个后台页面 | 抄 views/ 里 CRUD 页面 | [web-templates.md](templates/web-templates.md) |
| 项目部署上服务器 | 按 profile 部署 | [deploy-templates.md](templates/deploy-templates.md) / [profile-standard.md#§9](references/profile-standard.md) |
| 遇到"跑不起来"的问题 | 先翻踩坑清单 | [pitfalls.md](references/pitfalls.md) |
| 新项目的 CLAUDE.md 怎么写 | 抄模板 | [claude-md-template.md](templates/claude-md-template.md) |
| **从 Lite 想升 Standard** | 走迁移 checklist | [profile-standard.md#§10](references/profile-standard.md) |

---

## §7 反面清单（AI 常犯）

### 选型层面
1. ❌ **上来不选 profile 就写代码**——半年后必然乱
2. ❌ **20 人后台上 Docker + MySQL + Kubernetes**——过度
3. ❌ **100 人 MES 用 SQLite**——不够，选型不到位
4. ❌ **想上前沿栈（Deno / Bun / SurrealDB）**——不锁定 = 无人维护

### 架构层面（跨 profile）
5. ❌ 一上手就加 `domain/entities/dto/repositories/`——**5-15 人系统不需要，100 人系统才可能需要**
6. ❌ 换 UI 库（Ant / TDesign / Vuetify）——**Element Plus 锁死**
7. ❌ 换状态库（Vuex / Redux）——**Pinia 锁死**
8. ❌ 状态字段设成布尔 `is_active`——**用枚举 status**，未来会加状态
9. ❌ 表里不留 `remark` 字段——**车间要写线下情况**
10. ❌ 事务拆到多个 handler / setTimeout / Promise.all——**一次业务 = 一次事务**

### 错误处理层面
11. ❌ `throw new Error('库存不足')`——那是**业务错**，用 `businessError('INSUFFICIENT_STOCK', '库存不足')`
12. ❌ 前端弹 `alert("系统错误：Cannot read property 'x' of undefined")`——**AI 生成代码最丢人的一行**
13. ❌ 用 `DELETE FROM ...`——**用 status='deleted' 软删**，除非明确要清库

---

## §8 文档地图

**判断优先级层**（每次写代码前扫）：
- [reuse-first.md](references/reuse-first.md) — 7 步复用阶梯 + 白鲨骨架清单 + Element Plus 内置能力对照 + 反面案例

**选型层**（新项目开工时）：
- [stack-selection.md](references/stack-selection.md) — 十问 + 决策矩阵 + ADR 模板

**profile 层**（选定后照抄）：
- [profile-lite.md](references/profile-lite.md) — Vue + Fastify + SQLite 详细规格
- [profile-standard.md](references/profile-standard.md) — Vue + FastAPI + MySQL 详细规格 + Lite→Standard 迁移

**业务规范层**（跨 profile 一致）：
- [layout.md](references/layout.md) — 目录规范 + 命名 + "什么放哪"速查（含两个 profile 差异）
- [business-fields.md](references/business-fields.md) — 建表规范 + 单号 + 索引
- [state-machine.md](references/state-machine.md) — 单据 6 态 + 8 条合法迁移边
- [transactions.md](references/transactions.md) — 事务 + 幂等 + 并发（Lite / Standard 双写法）
- [audit-trail.md](references/audit-trail.md) — audit_log + 必记场景 + 典型追溯查询
- [errors.md](references/errors.md) — 业务错 vs 系统错 + 前端拦截器

**Lite profile 代码模板**：
- [server-templates.md](templates/server-templates.md) — Fastify 后端可抄骨架
- [web-templates.md](templates/web-templates.md) — Vue 前端可抄骨架（跨 profile 通用）
- [deploy-templates.md](templates/deploy-templates.md) — CentOS 7 systemd + nginx 部署

**通用**：
- [pitfalls.md](references/pitfalls.md) — 踩坑清单
- [claude-md-template.md](templates/claude-md-template.md) — 新项目根 CLAUDE.md 模板

---

## §9 使用心态

- **不迷信任何"最好的技术栈"**——只有"当下这个项目最合适的 profile"
- **profile 内锁定**是为了**多人协作时不用反复讨论技术选型**
- **规范跨 profile 一致**是为了**升档时业务代码可以最小改动地迁移**
- 出现下面情况就停下来：**"我想给这个项目引入 X 技术，但 profile 里没有 X"**——先做 ADR 再动手
