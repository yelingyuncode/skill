# 架构选型（先做选择，再套 profile）

**核心态度**：架构没有一成不变的。写代码之前先花 2 分钟做选型，选定 profile 后才**在这个 profile 内部**锁死细节。

---

## §1 十问速判

给业务方（或自己）问下面 10 个问题，把答案填进右列：

| # | 问题 | 你的答案 |
|---:|---|---|
| 1 | **同时在线人数**峰值大概多少？ | ___ |
| 2 | **写操作**峰值 QPS 大概多少？（车间同时报工/提交订单的密度） | ___ |
| 3 | **数据总量**一年后会到多少 GB？ | ___ |
| 4 | 需要**跨系统集成**吗？（ERP / OA / 财务 / IoT / PLC / MES 硬件） | ___ |
| 5 | 有**大量分析/报表/BI**吗？（复杂聚合、跨表 join、跑批） | ___ |
| 6 | 有**机器学习 / 图像 / OCR / AI 推理**吗？ | ___ |
| 7 | 需要**实时推送**（车间大屏、机台状态实时刷新）？ | ___ |
| 8 | **部署环境**是啥？（CentOS 7 / RHEL 8 / Ubuntu / K8s / 云） | ___ |
| 9 | **团队语言栈**哪个熟？（Node / Python / Java / Go） | ___ |
| 10 | **数据一致性要求**（丢一条能不能忍）？ | ___ |

## §2 选型决策矩阵

按答案落到哪个区间，就走哪个 profile：

| 维度 | Lite（轻量） | Standard（标准） | Scale（规模化） |
|---|---|---|---|
| 同时在线 | ≤ 15 人 | 15–100 人 | 100+ 人 |
| 写 QPS 峰值 | ≤ 5 | 5–50 | 50+ |
| 数据总量 / 年 | < 1 GB | 1–100 GB | 100 GB+ |
| 跨系统集成 | 无 / 只是 Excel 导入导出 | 1–3 个内部系统对接 | 多系统 + IoT + 数据总线 |
| 分析 / 报表 | 简单 GROUP BY 够 | 需要 JOIN 多表 + 定时跑批 | 需要 OLAP / 数仓 |
| AI / ML / 图像 | 无 | 有一点（OCR 单据、异常检测） | 深度学习流水线 |
| 实时推送 | 无 或 轮询即可 | SSE / 少量 WebSocket | 大量 WebSocket + 消息队列 |
| 部署环境 | CentOS 7 单机 | CentOS 7/8 单机或双机 | 多机 / K8s |
| 一致性 | 事务级即可 | 事务 + 幂等 + 补偿 | 分布式事务 / 事件溯源 |
| **推荐** | **[profile-lite](profile-lite.md)** | **[profile-standard](profile-standard.md)** | 单独讨论 |

**判定原则**：**只要有 ≥3 个维度落到更右边的档，就用更右边的 profile**。

## §3 三个 profile 概览

### Profile A · Lite（轻量）—— [profile-lite.md](profile-lite.md)

```
前端：Vue 3 + Element Plus + Pinia
后端：Node 18 + Fastify 5
数据库：SQLite + better-sqlite3 v10
部署：CentOS 7 + systemd + nginx
```

**适合场景**：内部工具、后台管理、单车间小系统、原型验证。

**典型代表**：
- 光山白鲨"活动盖板订单管理系统"（10 人用、日增数据 < 1MB）
- 曹荣国 vibe coding 练手项目
- 小型看板 / 简单表单 / 内部审批

**为什么锁 Node 18 + SQLite**：CentOS 7 glibc 2.17，Node 20+ 跑不动、better-sqlite3 v11 装不上。这不是保守，是环境约束。

---

### Profile B · Standard（标准）—— [profile-standard.md](profile-standard.md)

```
前端：Vue 3 + Element Plus + Pinia
后端：Python 3.11 + FastAPI + SQLAlchemy 2 + Pydantic 2
数据库：MySQL 8 + Redis 7（可选）
部署：Docker Compose + nginx（或 CentOS 8+ 上 systemd）
```

**适合场景**：**大部分 MES / 生产订单主系统**。

**典型代表**：
- 光山白鲨"生产 MES"（多车间、多机台、要报表、日增数据 100MB+）
- 订单管理系统扩容后
- 涉及 OCR / 图像 / 简单 ML（Python 生态优势明显）
- 需要跟 ERP / OA / 财务系统对接

**为什么切 Python**：
- 报表 / 数据分析 pandas + numpy 无敌
- 有 OCR / 图像 / ML 需求时 Python 生态无争议
- SQLAlchemy 处理复杂 ORM 关系比手写 SQL 好维护
- Python 团队人才市场更宽

**为什么切 MySQL**：
- 100+ GB 数据 + 复杂查询超 SQLite
- 多机部署 SQLite 无法跨机器共享
- 需要标准备份 / 主从 / 权限管理

---

### Profile C · Scale（规模化）

到这个量级（100+ 人 / 100GB+ 数据 / 多机 / IoT）就**不适合复制 profile 走**了：

- 数据库分片 / 读写分离
- 微服务 / 消息队列（Kafka / RabbitMQ）
- 时序库（InfluxDB / TDengine）存 IoT 数据
- OLAP（ClickHouse / Doris）跑分析
- 网关 / 服务发现 / 配置中心

**到这个阶段**：别照抄 profile，要找专业架构师+按项目单独设计+做 ADR。本 skill 不提供 profile-scale 模板。

---

## §4 选型的常见误区

| 误区 | 真相 |
|---|---|
| "反正 SQLite 便宜先用，以后再迁" | 数据一多 + 业务代码耦合 SQLite 语法 = 迁库要重写 30% 代码。选型别怕多花 1 天想清楚 |
| "上来就 MySQL 保险" | 20 人用的后台上 MySQL = 每天多花 30 分钟运维（备份/权限/连接池）没意义 |
| "反正 Node 会，全用 Node" | 报表跑批、pandas 计算、ML 推理这类，Node 生态弱于 Python，勉强用会累死自己 |
| "Java 才是企业级" | 5-15 人项目上 Spring = 引入 20 个不必要的抽象层 = 前端等你 2 周 |
| "先按 Standard 起手保险" | 起手代价：Docker + MySQL + Python 环境 = 部署链路多 3 倍。**先 Lite，业务增长了升到 Standard 是可预期路径**，不算过度设计 |
| "从 Lite 迁到 Standard 会很痛" | 只要按本 skill 的"业务规范"写（[state-machine](state-machine.md) / [audit-trail](audit-trail.md) / [errors](errors.md)），大部分业务代码逻辑不变，只是把 `db.transaction()` 换成 SQLAlchemy session，SQL 语法调 5% |

---

## §5 决策记录（ADR）

**选定 profile 后**，在项目根目录 `docs/ADR/001-choose-profile.md` 写清楚：

```markdown
# ADR-001: 选择 Profile-<lite|standard> 作为技术栈

日期：YYYY-MM-DD
决策人：xxx

## 背景
（业务方 / 团队现状 / 数据量 / 集成需求）

## 选型对答（对着 stack-selection.md §1）
1. 同时在线人数：xxx
2. 写 QPS：xxx
3. 数据总量：xxx
...

## 决策
选择 Profile-<xxx>，因为 <3-5 条主要原因>

## 备选与放弃
- 考虑过 Profile-<xxx>，放弃因为 xxx
- 考虑过 xxx 单独选型，放弃因为 xxx

## 未来何时重评
- 数据量突破 xxx GB 时
- 同时在线突破 xxx 人时
- 需要 xxx 集成时
```

**没 ADR 就动手写代码 = 半年后要背锅**。5 分钟写 ADR，省半年扯皮。

---

## §6 什么信号该"升档"

已经在 Lite，出现下面任一信号，认真评估升到 Standard：

1. **单库超 500 MB** 且日增 > 10 MB → 6 个月内到 GB 级
2. **报表页面耗时 > 3 秒**，SQLite 优化空间被榨干
3. **业务要求跟其他系统对接**（不只是 CSV 导入导出）
4. **需要跑数据科学 / 图像识别 / ML** 且没法丢外部服务
5. **多个车间同时接入，写高峰有阻塞**
6. **数据要跨机房 / 多机可访问**

升档不是灾难，是里程碑。**准备好了 audit_log + 状态机 + 错误分类**，业务代码逻辑迁移成本可控。
