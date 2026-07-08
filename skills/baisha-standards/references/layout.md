# 目录 & 文件布局

**规矩**：AI 不许自己发明分层。所有项目按下面这个结构放，找不到位置的东西就来这里对答案。

## §1 项目根

```
{{PROJECT}}/
├── README.md                    快速启动 + 部署
├── CLAUDE.md                    AI 协作硬规矩（抄 claude-md-template.md）
├── package.json                 workspace 根，pnpm dev 同时起前后端
├── pnpm-lock.yaml
├── .env.example
│
├── server/                      后端
├── web/                         前端
├── deploy/                      部署脚本 + nginx 配 + systemd unit
└── docs/                        业务文档（不放代码文档）
```

## §2 后端 server/

```
server/
├── package.json
├── data.db                      SQLite 库（运行时生成，不入 git）
├── data.db-wal                  WAL 文件
├── data.db-shm                  共享内存
├── .env                         DB_PATH, JWT_SECRET, PORT
└── src/
    ├── index.js                 Fastify 入口 + 全局钩子 + 全局错误 handler
    ├── db.js                    getDb() 单例 + schema + safeExec ALTER 迁移
    ├── auth.js                  bcrypt + JWT + preHandler
    ├── seed.js                  种子账号（首次启动写默认 admin）
    ├── util.js                  now() / uuid() / diffOnly()
    │
    ├── lib/
    │   ├── audit.js             审计工具
    │   ├── errors.js            businessError() + CODES
    │   ├── status-transition.js 状态机校验
    │   ├── sequence.js          单号生成器
    │   └── idempotency.js       幂等键
    │
    └── routes/
        ├── auth.js              /auth/login /me /change-password
        ├── admin/               后台管理
        │   ├── persons.js
        │   ├── roles.js
        │   └── accounts.js
        ├── order.js             业务模块 - 订单
        ├── work-order.js        业务模块 - 工单
        ├── stock.js             业务模块 - 库存
        └── report.js            业务模块 - 报表
```

**routes 的层级**：
- **平铺**是默认（`routes/order.js`）
- 只有子模块 ≥3 个才拆目录（`routes/admin/persons.js`）
- 不允许 `routes/module/controller/x.js` 三层以上

**不许出现的目录**：
- `models/` `entities/` `dto/` `domain/` `application/` `infrastructure/` `services/` `repositories/`
- 上面这些都是 AI 从 Java Spring 里抄的**。5-15 人系统 = route 直接落库 = 完事。

## §3 前端 web/

```
web/
├── package.json
├── vite.config.js               server.proxy + preview.proxy 两段都要
├── index.html
├── .env.development             VITE_API_BASE
├── .env.production
└── src/
    ├── main.js
    ├── App.vue
    ├── router.js                路由 + 权限守卫
    ├── styles.css               全局 CSS reset + 变量
    │
    ├── api/
    │   ├── index.js             axios 实例 + 拦截器（业务错 warning / 401 跳登录）
    │   └── resources.js         按模块聚合接口封装（可选）
    │
    ├── stores/
    │   └── user.js              pinia：token / user / logout
    │
    ├── components/
    │   ├── Layout.vue           顶部导航 + 侧边菜单 + 角色过滤
    │   ├── StatusTag.vue        统一状态展示（用 state-machine.md 的颜色）
    │   ├── PageHeader.vue       页面标题 + 面包屑
    │   └── AuditDrawer.vue      审计追溯抽屉（点单据能查历史）
    │
    └── views/                   按业务模块分
        ├── auth/
        │   └── Login.vue
        ├── admin/
        │   ├── PersonList.vue
        │   ├── RoleList.vue
        │   └── AccountList.vue
        ├── order/
        │   ├── OrderList.vue
        │   ├── OrderDetail.vue
        │   └── OrderForm.vue
        └── ... 其他模块
```

**views 的规则**：
- 按**业务模块**分子目录（`order/` `work-order/` `stock/`）
- 每模块典型三件套：`XxxList.vue` / `XxxDetail.vue` / `XxxForm.vue`
- 不许出现 `pages/features/modules/sections/` 这类分类
- **组件超过 500 行拆**：拆到 `components/` 下的模块子目录

## §4 部署 deploy/

```
deploy/
├── nginx.conf.template          nginx 反代配置
├── {{project}}.service          systemd unit
├── build.sh                     打包脚本：pnpm build + 拷 dist + tar
├── install.sh                   服务器上跑：解压 + npm ci --prod + systemd restart
├── backup.sh                    每日备份 data.db
└── README.md                    部署步骤（内部人员看）
```

## §5 docs/

```
docs/
├── ADR/                         架构决策记录
│   ├── 001-lock-node18.md
│   └── 002-choose-sqlite.md
├── business/                    业务流程文档
│   ├── order-flow.md
│   └── work-order-flow.md
└── ops/                         运维文档
    ├── backup-restore.md
    └── incident-YYYYMMDD.md     事故复盘
```

**docs 只放业务和运维文档**，代码规范这类走 skill（本 skill）。

## §6 命名总规则

| 项 | 规则 | 示例 |
|---|---|---|
| 目录 | kebab-case | `work-order/` |
| 后端文件 | kebab-case.js | `work-order.js` |
| 前端 SFC 组件 | PascalCase.vue | `OrderList.vue` |
| 前端 utility js | kebab-case.js | `format-date.js` |
| 数据库表 | 单数 snake_case | `sales_order` / `work_order` |
| 数据库字段 | snake_case | `created_at` |
| API 路径 | kebab-case，复数资源 | `/api/work-orders` |
| API 参数 | camelCase（JSON body） | `{workshopId: 5}` |

**跨层不一致的地方**：
- URL 里 `work-orders`（英文复数）
- JSON body 里 `workshopId`（camelCase）
- 数据库 `workshop_id`（snake_case）
- 后端 handler 里 `req.body.workshopId`（跟 JSON 一致）

这三种命名互相翻译是有约定的，别一层一层转换的时候搞乱。

## §7 什么放在哪的速查

| 想放的东西 | 放哪 |
|---|---|
| 一个业务实体的路由 | `server/src/routes/<模块>.js` |
| 通用工具函数 | `server/src/lib/<用途>.js` |
| DB schema 或迁移 | `server/src/db.js` |
| 一个业务实体的列表页 | `web/src/views/<模块>/XxxList.vue` |
| 前端复用组件 | `web/src/components/<用途>.vue` |
| 一个业务错码 | `server/src/lib/errors.js` 的 `CODES` |
| 一个新状态 | `state-machine.md` 加边 + `server/src/lib/status-transition.js` 加 |
| 一次部署所需的 nginx 规则 | `deploy/nginx.conf.template` |
| 一次事故的复盘 | `docs/ops/incident-YYYYMMDD.md` |
