# 技术栈锁定

**不许换**。想换的人先在项目 CLAUDE.md 里写 ADR 说清楚为什么、代价谁承担。

## §1 版本锁死表

| 层 | 选型 | 版本 | 锁定原因 |
|---|---|---|---|
| Node | LTS | **18.x**（不是 20、不是 22） | CentOS 7 glibc 2.17 跑不动 Node 20+ |
| 包管理（开发） | pnpm | 8+ | 磁盘友好、workspace 简单 |
| 包管理（服务器） | npm | Node 内置 | 服务器上少装东西 |
| 后端框架 | Fastify | **5.x** | 比 Express 快、hook 模型清晰 |
| 数据库 | SQLite | 3.35+ | 内置 |
| DB 驱动 | better-sqlite3 | **v10.x**（不是 v11） | v11 需要 glibc 2.28+ |
| JWT | @fastify/jwt | 9.x | Fastify hook 无缝 |
| 密码哈希 | bcryptjs | 2.x | 纯 JS，免 native 编译 |
| 时间 | dayjs | 1.x | 小、API 兼容 moment |
| 前端框架 | Vue | **3.x** | Composition API |
| 构建 | Vite | 5.x | 快 |
| UI 组件 | Element Plus | **2.x** | 富表单、富表格生态完备 |
| 状态管理 | Pinia | 2.x | Vue 3 官方 |
| 路由 | Vue Router | 4.x | |
| HTTP 客户端 | axios | 1.x | 拦截器优雅 |
| 部署 | systemd + nginx | CentOS 自带 | 别装 pm2/docker |

## §2 为什么不能换

### 不换 Node 版本

- CentOS 7 的 glibc 只到 2.17
- Node 20+ 官方二进制要 glibc 2.28+，会直接跑不起来
- 想上 Node 20 = 换服务器 OS = 别的事

### 不换 SQLite → PostgreSQL

- 5-15 人内网，SQLite 单库并发够
- WAL 模式下读不阻塞
- 免维护（没 pg 实例、没 pg_dump 定时任务、没连接池调优）
- 备份 = cp data.db，恢复 = cp 回去
- 想上 PG 前先问：**当前 SQLite 具体哪个查询扛不住**？答不上来就别换

### 不换 Fastify → Express / Koa / Nest

- Fastify 5 生态成熟，plugin 体系清晰
- 上 Nest = 引入 IoC + Decorator + Module + Provider，5-15 人不需要
- 上 Koa = 中间件手动串，社区活跃度不如 Fastify
- Express 太老，Fastify 是 Express 的思路 + 现代化

### 不换 Vue → React

- 团队已 Vue，改栈 = 培训 + 重写
- Vue 3 + Element Plus 出后台 CRUD **是行业内最快的组合**

### 不加 Redis

- SQLite mmap 加会话缓存足够
- Redis = 多一个进程要监控、多一个故障点
- 除非同时活跃 > 200 人才需要考虑

### 不上 K8s / Docker

- 单机跑 = 部署 = tar + systemctl restart
- Docker + K8s = 增加运维复杂度 5×
- 内部工具不 SLA，运维简单 > 弹性

## §3 允许的依赖增补

以下情况允许**引一个新包**（其他不允许）：

| 场景 | 允许包 | 理由 |
|---|---|---|
| Excel 导入 / 导出 | `xlsx` | 车间要 Excel 是硬需求 |
| PDF 生成 | `pdfkit` 或 `@vue/print` | 报表 / 出货单打印 |
| 图表 | `echarts` 5 | 生产数据可视化 |
| 富文本 | `wangeditor` 5 | 少数场景 |
| 二维码 | `qrcode` | 追溯 / 扫码 |
| 中文拼音搜索 | `pinyin-pro` | 中文名称快速筛选 |

**不允许的**：
- lodash（Node/浏览器原生 + Vue 3 helper 已够）
- moment（用 dayjs）
- jquery、underscore、bluebird（时代过了）
- 任何 Rx / Redux / MobX 生态（Pinia 够了）
- ORM（Sequelize / TypeORM / Prisma / Mikro / Drizzle 一律不许）

## §4 环境要求

**开发机**：Mac / Windows / Linux 都可以，Node 18 + pnpm 8。

**部署服务器**（生产 / 内测）：
- CentOS 7.x（glibc 2.17）
- Node 18 LTS（用 [nodejs.org 官方 x64 tarball](https://nodejs.org/dist/latest-v18.x/) 解压到 `/opt/node18`）
- nginx 1.20+
- systemd
- **不用 Docker**

**内网 IP 段**：192.168.x.x，服务器一般在 192.168.6.x 或 192.168.7.x（内部约定，具体项目问信息中心）。

## §5 版本升级策略

- **打补丁版本**（1.2.x → 1.2.y）：跟着走，`pnpm up` 即可
- **小版本**（1.2 → 1.3）：季度评估一次，Test 环境跑 1 周再升
- **大版本**（Fastify 5 → 6，Vue 3 → 4）：**不追**，等公司项目全部稳定 + 有明确收益再讨论
