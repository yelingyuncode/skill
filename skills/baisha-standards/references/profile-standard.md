# Profile · Standard（标准）

**适用**：MES 主系统、订单管理主系统、任何中等以上业务系统。看 [stack-selection.md](stack-selection.md) 判断你是不是应该走这个 profile。

## 导览

- `§1-2`：技术栈锁定和目录结构
- `§3-5`：建表、事务、幂等
- `§6-8`：并发防护、审计、错误处理
- `§9`：部署
- `§10-11`：Lite 升 Standard 和继续升 Scale 的信号

---

## §1 技术栈锁定

| 层 | 选型 | 版本 | 锁定原因 |
|---|---|---|---|
| Node（前端构建） | LTS | 20.x | 前端构建工具需要新 Node，与部署无关 |
| pnpm | 8+ | 前端包管理 |
| **前端框架** | Vue | 3.x | 与 Lite 一致，团队复用 |
| 构建 | Vite | 5.x | |
| UI 组件 | Element Plus | 2.x | |
| 状态 | Pinia | 2.x | |
| 路由 | Vue Router | 4.x | |
| HTTP | axios | 1.x | |
| **后端语言** | Python | **3.11**（不是 3.12 也不是 3.10） | 3.11 显著性能提升 + 生态稳；3.12 部分包还不稳 |
| **后端框架** | FastAPI | 0.115+ | 类型友好、OpenAPI 自动生成、async 原生 |
| ASGI 服务器 | uvicorn + gunicorn | 最新 | uvicorn 跑 async，gunicorn 管进程 |
| ORM | SQLAlchemy | **2.x**（不是 1.4 老 API） | 2.x async + typed 官方 |
| 数据校验 | Pydantic | **2.x** | v2 rust 内核，比 v1 快数倍 |
| **数据库** | MySQL | **8.0 LTS** | InnoDB / JSON 字段 / CTE 齐 |
| DB 驱动 | aiomysql 或 asyncmy | 最新 | 配合 SQLAlchemy 2 async |
| 迁移 | Alembic | 最新 | SQLAlchemy 官方 |
| 缓存 / 会话 | Redis | 7.x | 可选，需要 SSE / 分布式锁 / 队列时用 |
| JWT | python-jose | 最新 | |
| 密码哈希 | passlib\[bcrypt\] | 最新 | |
| 定时任务 | APScheduler | 最新 或 独立 celery | 简单场景 APScheduler 够 |
| 日志 | structlog | 最新 | JSON 结构化日志便于聚合 |
| 包管理 | uv 或 pip + venv | uv 是新一代 | uv 快很多，但 pip 也可 |
| 部署 | Docker Compose + nginx | | 或 CentOS 8+ systemd |

## §2 目录结构

```
{{PROJECT}}/
├── README.md
├── CLAUDE.md
├── docker-compose.yml           mysql + redis + api + web + nginx
├── .env.example
│
├── server/                      Python 后端
│   ├── pyproject.toml           uv / pip 都吃
│   ├── alembic.ini
│   ├── alembic/                 migration 脚本
│   │   ├── env.py
│   │   └── versions/
│   ├── uv.lock
│   ├── Dockerfile
│   └── app/
│       ├── main.py              FastAPI 入口 + include_router + 全局异常
│       ├── config.py            pydantic Settings
│       ├── db.py                async engine + session factory + Base
│       ├── auth.py              JWT + password + get_current_user
│       ├── deps.py              依赖注入（db session / user / permissions）
│       │
│       ├── models/              SQLAlchemy 模型（一个业务实体一个文件）
│       │   ├── __init__.py
│       │   ├── base.py          BaseModel with id/created_at/updated_at/status/remark
│       │   ├── user.py
│       │   ├── order.py
│       │   ├── work_order.py
│       │   └── audit_log.py
│       │
│       ├── schemas/             Pydantic 请求 / 响应 schema
│       │   ├── order.py         OrderCreate / OrderUpdate / OrderRead
│       │   └── ...
│       │
│       ├── crud/                纯 DB 操作（不含业务规则）
│       │   ├── order.py
│       │   └── ...
│       │
│       ├── services/            业务逻辑（跨表 / 状态机 / 事务）
│       │   ├── order_service.py
│       │   └── ...
│       │
│       ├── routers/             FastAPI router
│       │   ├── auth.py
│       │   ├── order.py
│       │   └── ...
│       │
│       └── lib/
│           ├── errors.py        BusinessError + CODES
│           ├── audit.py         audit 工具
│           ├── status_transition.py
│           └── sequence.py      单号生成
│
├── web/                         Vue 前端（跟 profile-lite 完全一致）
│   ├── package.json
│   ├── vite.config.js
│   └── src/  ...
│
└── deploy/
    ├── docker-compose.prod.yml
    ├── nginx.conf.template
    └── backup.sh
```

**关键差异 vs Lite**：
- 后端多了 `models/schemas/crud/services/routers` 五层
- **注意**：这不是过度设计。Python + SQLAlchemy + Pydantic 的场景下这些抽象是有回报的（类型安全、异步 session、复用 CRUD、schema 复用给 OpenAPI 文档）
- **仍然禁止**再加 `domain/`、`application/`、`infrastructure/` 六边形架构分层——那是 100+ 人的东西

## §3 建表规范（对应 [business-fields.md](business-fields.md) 的 MySQL 版本）

**BaseModel**（所有业务表继承）：

```python
# app/models/base.py
from sqlalchemy import BigInteger, String, Integer
from sqlalchemy.orm import Mapped, mapped_column, DeclarativeBase
from sqlalchemy.sql import func

class Base(DeclarativeBase):
    pass

class BusinessBase(Base):
    __abstract__ = True

    id:         Mapped[int]  = mapped_column(BigInteger, primary_key=True, autoincrement=True)
    created_at: Mapped[int]  = mapped_column(BigInteger, nullable=False, default=lambda: int(time.time()))
    updated_at: Mapped[int]  = mapped_column(BigInteger, nullable=False, default=lambda: int(time.time()),
                                              onupdate=lambda: int(time.time()))
    created_by: Mapped[int]  = mapped_column(BigInteger, nullable=False)
    updated_by: Mapped[int]  = mapped_column(BigInteger, nullable=False)
    status:     Mapped[str]  = mapped_column(String(32), nullable=False, default='draft', index=True)
    remark:     Mapped[str]  = mapped_column(String(500), nullable=False, default='')
```

**业务实体**：

```python
# app/models/order.py
from sqlalchemy import String, BigInteger, Integer
from sqlalchemy.orm import Mapped, mapped_column
from .base import BusinessBase

class Order(BusinessBase):
    __tablename__ = 'sales_order'

    order_no:      Mapped[str]  = mapped_column(String(32), nullable=False, unique=True, index=True)
    customer_id:   Mapped[int]  = mapped_column(BigInteger, nullable=False, index=True)
    total_amount:  Mapped[int]  = mapped_column(BigInteger, nullable=False, default=0)   # 分
    version:       Mapped[int]  = mapped_column(Integer, nullable=False, default=0)      # 乐观锁
```

**MySQL 特定注意**：

- 用 **`utf8mb4`** 字符集 + `utf8mb4_unicode_ci` 排序
- 表和字段名统一 **`snake_case`**（与 Lite 一致）
- 大字段（备注 500+）用 `TEXT`，小的用 `VARCHAR`
- 金额用 **INTEGER 分**（不用 DECIMAL，规避浮点）
- 时间用 **`BIGINT` unix 秒**（不用 DATETIME，跨时区省事，跟 Lite 一致）
- 每个 InnoDB 表 **必须有主键**（否则复制走不通）
- **不用外键约束**（DDL 迁移麻烦、性能损失），在应用层保证

## §4 事务边界

**FastAPI 依赖注入拿 session，路由函数里显式 begin**：

```python
# app/routers/order.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from ..deps import get_db, get_current_user
from ..services import order_service
from ..lib.errors import BusinessError

router = APIRouter(prefix='/orders')

@router.post('')
async def create_order(
    payload: OrderCreate,
    db: AsyncSession = Depends(get_db),
    user = Depends(get_current_user),
):
    try:
        async with db.begin():           # ← 事务边界
            result = await order_service.create_order(db, user, payload)
        return {"ok": True, "data": result}
    except BusinessError as e:
        return {"ok": False, "code": e.code, "msg": e.msg}
    # 系统错交给全局 exception handler
```

对应 [transactions.md](transactions.md) 里 `db.transaction()` 的语义。**一次业务动作 = 一个 `async with db.begin()`**。

## §5 幂等（对应 [transactions.md](transactions.md#idempotency)）

**状态检查幂等** —— 跟 Lite 一样：

```python
async def release_order(db, user, order_id):
    order = await db.get(Order, order_id)
    if order.status == 'released':
        return {"already_done": True}   # 已发布，直接成功
    assert_transition(order.status, 'released')
    order.status = 'released'
    order.updated_at = int(time.time())
    order.updated_by = user.id
    await write_audit(db, user, 'sales_order', order_id, 'status_change',
                      before={'status': order.status}, after={'status': 'released'})
```

**client_request_id 幂等** —— 用 `idempotency_key` 表 + MySQL 唯一索引：

```sql
CREATE TABLE idempotency_key (
  `key`       VARCHAR(64) PRIMARY KEY,
  action      VARCHAR(64) NOT NULL,
  result_json JSON NOT NULL,
  created_at  BIGINT NOT NULL,
  INDEX idx_created (created_at)
);
```

## §6 并发防护

**乐观锁**（跟 Lite 一样，用 `version` 字段）：

```python
result = await db.execute(
    update(Order)
    .where(Order.id == id, Order.version == client_version)
    .values(remark=remark, updated_at=..., updated_by=user.id, version=Order.version + 1)
)
if result.rowcount == 0:
    raise BusinessError('CONFLICT', '该单已被他人修改，请刷新后重试')
```

**行锁**（涉及库存扣减的写场景）：

```python
# MySQL InnoDB 用 SELECT ... FOR UPDATE
stock = await db.execute(
    select(Stock).where(Stock.product_id == pid).with_for_update()
)
stock_row = stock.scalar_one()
if stock_row.qty < need_qty:
    raise BusinessError('INSUFFICIENT_STOCK', '库存不足')
stock_row.qty -= need_qty
```

**⚠️ 注意**：MySQL 的 `FOR UPDATE` 必须在事务里、必须走索引，否则锁全表。

## §7 audit_log 表（对应 [audit-trail.md](audit-trail.md)）

```sql
CREATE TABLE audit_log (
  id           BIGINT PRIMARY KEY AUTO_INCREMENT,
  ts           BIGINT NOT NULL,
  actor_id     BIGINT NOT NULL,
  actor_name   VARCHAR(64) NOT NULL,
  entity       VARCHAR(64) NOT NULL,
  entity_id    VARCHAR(64) NOT NULL,
  action       VARCHAR(32) NOT NULL,
  before_json  JSON,
  after_json   JSON,
  meta_json    JSON,
  remark       VARCHAR(500) DEFAULT '',
  INDEX idx_entity (entity, entity_id, ts DESC),
  INDEX idx_actor  (actor_id, ts DESC),
  INDEX idx_ts     (ts DESC)
) ENGINE=InnoDB CHARSET=utf8mb4;
```

比 SQLite 版本多了 `JSON` 字段类型——**MySQL 8 原生支持 JSON 查询**，追溯页面可以直接 `WHERE JSON_EXTRACT(after_json, '$.status') = 'cancelled'`。

## §8 错误处理

`app/lib/errors.py`：

```python
class BusinessError(Exception):
    def __init__(self, code: str, msg: str):
        self.code = code
        self.msg = msg
        super().__init__(msg)
```

全局 handler：

```python
@app.exception_handler(BusinessError)
async def biz_handler(request, exc: BusinessError):
    return JSONResponse({"ok": False, "code": exc.code, "msg": exc.msg})

@app.exception_handler(Exception)
async def sys_handler(request, exc: Exception):
    logger.error("system error", exc_info=exc, path=request.url.path)
    return JSONResponse({"ok": False, "code": "INTERNAL", "msg": "系统繁忙，请稍后重试"}, status_code=500)
```

跟 Lite 的语义完全一致，前端 axios 拦截器不用改。

## §9 部署

**Docker Compose 起手**（推荐）：

```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
    volumes:
      - ./data/mysql:/var/lib/mysql
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - ./data/redis:/data
    restart: unless-stopped

  api:
    build: ./server
    environment:
      DB_URL: mysql+aiomysql://root:${MYSQL_ROOT_PASSWORD}@mysql/${DB_NAME}
      REDIS_URL: redis://redis:6379/0
      JWT_SECRET: ${JWT_SECRET}
    depends_on: [mysql, redis]
    restart: unless-stopped

  web:
    build: ./web
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports: ['80:80']
    volumes:
      - ./deploy/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on: [api, web]
    restart: unless-stopped
```

**备份**：`mysqldump` + `mysql_config_editor` 定时任务，每日备份到 `deploy/backup/`。**只备 database，不备 mysql 系统表**。

**监控**：结构化日志 → 定时聚合到 CSV → 关注 500 数量 / 慢查询数 / DB 连接池饱和度。

## §10 迁移路径（从 Lite 升到 Standard）

如果你项目从 Lite 起步、后来数据量业务量都涨了，升 Standard 走这个 checklist：

- [ ] 数据从 SQLite 导出到 MySQL：`sqlite3 data.db .dump` → 用 SQL 兼容工具（如 `pgloader` 或人工改语法）导入 MySQL
- [ ] Node routes → FastAPI routers：**逻辑几乎不变**，只是语法翻译（都是 handler 拿 request、访问 DB、返回 JSON）
- [ ] `db.transaction()` → `async with db.begin()`
- [ ] `db.prepare().run()` → SQLAlchemy `session.execute()`
- [ ] `businessError()` → `BusinessError()`（同名概念）
- [ ] audit / state-machine / 状态枚举 **文本值不变**
- [ ] 前端 axios / stores / components **一行不改**（因为 API 契约不变）
- [ ] 部署链路换成 Docker Compose

**关键**：只要业务代码严格遵守本 skill 的规范（[state-machine](state-machine.md) / [audit-trail](audit-trail.md) / [errors](errors.md)），迁移成本以人天计，不是人月。

## §11 什么时候还要向 Scale 升

参考 [stack-selection.md](stack-selection.md#§6 什么信号该升档) 的信号。到 Scale 就要专业架构师和 ADR 了，本 skill 不写 profile-scale 模板。
