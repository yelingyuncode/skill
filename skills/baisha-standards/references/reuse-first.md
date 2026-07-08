# 复用优先（不重复造轮子）

**核心信条**：**最好的代码是不需要写的代码**。任何新代码在写之前，先按下面 7 步阶梯往下判断，能停就停。

这条规则的收益：AI 生成代码时最容易犯"看到需求就动手写"的病，堆出一堆重复实现、无用抽象、绕开框架已有能力的自制轮子。**每停一步 = 少一片以后要维护的地方**。

## 导览

- `§1`：写代码前 7 步复用判断
- `§2-5`：骨架能力、Element Plus、已装依赖、允许依赖
- `§6`：AI 常见反面案例
- `§7-8`：提交前检查和允许造轮子的条件

---

## §1 复用判断阶梯（写代码前 30 秒走一遍）

| 顺序 | 问自己 | 停在这里的话怎么办 |
|---:|---|---|
| 1 | **这个功能真的需要存在吗？** | 让业务方确认，很多 AI 需求是自己脑补的 |
| 2 | **项目里（骨架里、其他模块里）已经有一模一样或 90% 相似的实现吗？** | 复用现有代码，需要就抽公用函数 |
| 3 | **技术栈的标准库 / 框架能力能不能做？** | 用它。举例：Node/Python 标准库、Fastify 内置 hook、FastAPI 内置 dependency、Vue 3 内置 reactive |
| 4 | **UI 库 / 平台原生能力能不能做？** | Element Plus 有 Table、Form、DatePicker、Cascader…别自己写；浏览器有 URL、Intl.DateTimeFormat…别 npm 包 |
| 5 | **已经装的依赖能不能做？** | dayjs 能格式化日期不要再引 moment；xlsx 能读能写不要再引 exceljs |
| 6 | **一行代码 / 一个 SQL 能不能解决？** | 别为一行代码抽 service 层 |
| 7 | **确实要写新代码，最小可行版本是什么？** | 只写完成需求所必需的那部分，不做"以后可能用到"的抽象 |

**任何一步答"是"就停下**，别为了显示"我做了工作"而绕过。

**不能省的**（不能因为"少写"而牺牲的）：
- ✅ 参数校验（前后端都要）
- ✅ 错误处理（业务错/系统错分类，见 [errors.md](errors.md)）
- ✅ 事务边界（见 [transactions.md](transactions.md)）
- ✅ 审计（见 [audit-trail.md](audit-trail.md)）
- ✅ 权限检查
- ✅ 5 类兜底字段（见 [business-fields.md](business-fields.md)）

**能省的**（AI 常写但不需要的）：
- ❌ 空的 try/catch 只是为了"看起来严谨"
- ❌ 单元测试之外再套一层集成测试的 wrapper
- ❌ 每个 model 一个 factory + builder + validator
- ❌ 为一次 CRUD 建 5 个文件（controller/service/repo/dto/entity）
- ❌ 一切"以后可能会 X"的抽象

---

## §2 骨架里已经有的东西（先看这里，别重写）

不管是 Lite 还是 Standard profile，都从 baisha-standards 的骨架起手。**看到下面需求先想能不能复用骨架里已有的**：

### 后端已有

| 你想做的 | 骨架里已有的 | 位置 |
|---|---|---|
| 用户名/密码登录 | 完整 `/auth/login /me /change-password` | server-templates §Auth |
| JWT 拦截 | preHandler + 全局 hook | server-templates §Auth |
| 单表 CRUD | `routes/persons.js` 骨架，替换表名字段就能用 | server-templates |
| 审计写日志 | `audit()` 工具函数 | audit-trail.md §3 |
| 业务错处理 | `businessError(code, msg)` + `CODES` 常量表 | errors.md §3 |
| 状态迁移校验 | `assertTransition(from, to)` | state-machine.md §3 |
| 单号生成 | `nextSeq(prefix)` 走 sequence 表 | business-fields.md §5 |
| 幂等 | 状态检查 / client_request_id | transactions.md §2 |
| 分页 | 统一 `?page=&size=` 参数 + `{list, total, page, size}` 返回 | server-templates |
| 时间戳 | `now()` 工具函数 | server-templates |
| 事务包裹 | `db.transaction()`（Lite）/ `async with db.begin()`（Standard） | transactions.md §1 |

### 前端已有

| 你想做的 | 骨架里已有的 | 位置 |
|---|---|---|
| 登录页 | `views/auth/Login.vue` | web-templates §Login |
| 顶部导航 + 侧边菜单 | `components/Layout.vue`（自带角色过滤） | web-templates §Layout |
| 401 自动跳登录 | axios 拦截器已配 | errors.md §4 |
| 业务错弹 Message | axios 拦截器已配，页面不用自己判 | errors.md §4 |
| 登录态 | `stores/user.js`（Pinia） | web-templates §Store |
| 路由守卫 | `router.js` 已配 | web-templates §Router |
| 单据状态 tag 展示 | `components/StatusTag.vue`（配色统一） | state-machine.md §5 |
| 审计追溯抽屉 | `components/AuditDrawer.vue`（点单据看历史） | audit-trail.md |

**看到这些需求还从零写 = 复用打回**。

---

## §3 Element Plus 已经内置的（不要再写）

**AI 最爱造的轮子第一名：造 Element Plus 已经有的组件**。下面这些**看到需求直接用官方**：

| 你想要 | Element Plus 组件 | 不要写的东西 |
|---|---|---|
| 表格列表 + 排序 + 筛选 + 多选 + 分页 | `el-table` + `el-pagination` | 自己写 tr/td，自己写全选逻辑 |
| 表单校验 | `el-form` + `rules` prop | 自己写正则校验、自己写错误提示 |
| 日期选择 | `el-date-picker`（含范围选择） | 自己组合 3 个 select |
| 时间选择 | `el-time-picker` | |
| 级联下拉（省市区、分类树） | `el-cascader` | 自己写 select 联动 |
| 树形选择 | `el-tree-select` | |
| 文件上传 | `el-upload`（含进度、限制、拖拽） | 自己写 `<input type=file>` |
| 图片查看 | `el-image` + `preview-src-list` | |
| 弹窗 | `el-dialog` | |
| 抽屉 | `el-drawer` | |
| 消息提示（成功/警告/错误） | `ElMessage.success/warning/error` | `alert()` / 自制 toast |
| 通知（右上角） | `ElNotification` | |
| 确认对话框 | `ElMessageBox.confirm` | 自己写弹窗组件 |
| 加载中遮罩 | `v-loading` 指令 | 自己写全屏 loading |
| 步骤条 | `el-steps` | |
| 折叠面板 | `el-collapse` | |
| Tab 页 | `el-tabs` | |
| 描述列表（详情页展示） | `el-descriptions` | 自己拿 flex 对齐 label/value |
| 空状态 | `el-empty` | |
| 面包屑 | `el-breadcrumb` | |
| 分割线 | `el-divider` | |
| 图片裁剪 | `el-upload` + 结合 crop 插件（如实在要） | 自己写 canvas |

**基本原则**：先在 [Element Plus 文档](https://element-plus.org/) 搜一下，**没有的组件才自己写**。

---

## §4 已经装的第三方能做的（别再引新包）

Element Plus + Vue 3 + 骨架里默认引的这些包：

| 你想做的 | 用什么 | 不要引的 |
|---|---|---|
| 日期格式化 / 比较 / 加减 | **dayjs**（骨架自带） | moment、date-fns、js-joda |
| Excel 导入导出 | **xlsx**（允许） | exceljs、node-xlsx |
| PDF 生成 | **pdfkit** 或 **@vue/print**（允许） | jsPDF |
| 二维码 | **qrcode**（允许） | qrcode.vue、qrcode-generator |
| 图表 | **echarts 5**（允许） | chart.js、d3、highcharts |
| 中文拼音搜索 | **pinyin-pro**（允许） | pinyin、pinyinjs |
| 数字格式化 | `Intl.NumberFormat`（**浏览器原生**） | numeral.js |
| 深拷贝 | `structuredClone(obj)`（**浏览器/Node 原生**） | lodash.cloneDeep |
| 对象合并 | `{...a, ...b}` / `Object.assign` | lodash.merge |
| 数组去重 | `[...new Set(arr)]` | lodash.uniq |
| 防抖 / 节流 | Vue 3 的 `useDebounceFn` / `useThrottleFn`（如果用 vueuse）；或手写 5 行 setTimeout | lodash.debounce（14 KB） |
| UUID | `crypto.randomUUID()`（**Node 18+/浏览器原生**） | uuid 包 |
| 字符串模板 | 反引号 `` `${a}` `` | template literal 库 |

**Node.js / 浏览器现代 API 已经能做的 = 不用装包**。安装一个包意味着：
- 打包体积增加
- 供应链风险
- 一个额外要跟踪安全公告的地方

---

## §5 允许引的少数第三方（其他一律不允许）

见 [profile-lite.md §3](profile-lite.md) 和 [profile-standard.md §1](profile-standard.md)。**不在清单里的包一律先问自己 5 遍"我真的需要吗"**。

如果确实需要装新包，走这个流程：

1. 在 [Element Plus](https://element-plus.org/) / 标准库 / 已装依赖里再确认一次没有
2. 在 `docs/ADR/NNN-add-<package>.md` 写清楚为什么、评估过什么替代、包大小 / 安全记录
3. PR 里 @ 信息中心 review

---

## §6 反面案例集（AI 常写但不该写的）

### ❌ 造 UI 轮子

```vue
<!-- AI 写：手动组合 3 个 select 做日期选择 -->
<template>
  <select v-model="year">...</select> 年
  <select v-model="month">...</select> 月
  <select v-model="day">...</select> 日
</template>
```

```vue
<!-- 应该：直接用 -->
<el-date-picker v-model="date" type="date" />
```

### ❌ 造工具轮子

```js
// AI 写：手写日期比较
function isSameDay(a, b) {
  const da = new Date(a);
  const db = new Date(b);
  return da.getFullYear() === db.getFullYear()
      && da.getMonth() === db.getMonth()
      && da.getDate() === db.getDate();
}
```

```js
// 应该：dayjs 一行
import dayjs from 'dayjs';
dayjs(a).isSame(b, 'day');
```

### ❌ 造分层轮子

```
❌ AI 生成的目录
routes/order.js
├── controllers/OrderController.js
├── services/OrderService.js
├── repositories/OrderRepository.js
├── entities/Order.js
├── dto/CreateOrderDto.js
├── mappers/OrderMapper.js
└── validators/OrderValidator.js
```

```
✅ 5-15 人系统真实需要的
routes/order.js   ← Fastify handler 里直接 db.prepare + db.transaction
```

Standard profile 允许 `models / schemas / crud / services / routers`，**但也就这 5 层**，不要再多。

### ❌ 造抽象轮子

```js
// AI 写：为了"以后可能有别的存储"抽了个 Repository
class OrderRepository {
  constructor(db) { this.db = db; }
  async findById(id) { ... }
  async save(order) { ... }
}
class OrderService {
  constructor(repo) { this.repo = repo; }
  async createOrder(payload) { return this.repo.save(...) }
}
```

```js
// 应该：直接在 route 里搞定
fastify.post('/orders', async (req) => {
  return db.transaction(() => { /* 落库 */ })();
});
```

**"以后可能有别的存储"是 YAGNI 违反**。真的换了 DB 再重构，不是提前一年抽。

### ❌ 造依赖轮子

```json
// AI 写：为了简单的字符串操作装 lodash
"dependencies": {
  "lodash": "^4.17.21",
  ...
}
```

```js
// 应该：浏览器/Node 原生够用
[...new Set(arr)]                      // uniq
{...a, ...b}                            // merge
Object.entries(obj).filter(([k,v])=>v) // pickBy
```

### ❌ 造测试轮子

```js
// AI 写：为一个 pure function 写 50 行 mock 框架
describe('formatAmount', () => {
  let mockFormatter, mockConfig, mockCurrency;
  beforeEach(() => { ... 30 行 setup ... });
  ...
});
```

```js
// 应该：3 行
test('formatAmount 分转元保留 2 位', () => {
  expect(formatAmount(12345)).toBe('123.45');
});
```

---

## §7 复用检查动作

在 PR / MR 提交前，用下面清单**问自己 5 秒**：

- [ ] 我加的每个新文件 / 新函数 / 新组件，骨架 / 已装依赖 / Element Plus / 标准库里**都没有**类似能力？
- [ ] 我加的新依赖，写了 ADR？
- [ ] 我造的抽象**当前**就需要（而不是"以后可能"）？
- [ ] 我的新代码是**最小可行**的（不是"更完整"的）？
- [ ] 我没为"看起来严谨"加空 try/catch、无用参数、假配置项？
- [ ] 我没重写业务错处理、审计、状态机、事务包裹这些骨架已有的能力？

一条对不上就砍。**每砍一行 = 少一个未来要维护的地方**。

---

## §8 什么时候可以造轮子

**允许**造轮子的场景：

1. **明确证据表明现有能力不够**（比如 Element Plus Table 不支持某种奇葩滚动方案，且业务方坚决要）
2. **性能有实测数据支撑**（现有实现测过 300ms，业务方要 100ms 以内）
3. **安全 / 合规硬要求**（第三方包无法审计源码）

上面任一条成立时，**写 ADR** 说明为什么不复用，然后开始造。

**不允许**的场景：
- "我觉得 Element Plus 那个组件不够优雅"
- "自己写学得快"
- "老板/其他人喜欢这种风格"
- "AI 生成看起来更完整"

---

## §9 小结

不重复造轮子不是"偷懒"，是**只把精力花在真正业务上**。

- **业务逻辑（车间怎么走、单据怎么流）**：你必须自己想清楚
- **技术底子（表格怎么排、日期怎么选、状态怎么留）**：**用已经有的**

把节省下来的时间留给：
- 跟业务方多聊两轮
- 多想一遍 audit_log 会记录什么
- 多测一次状态机的边界情况
- 多写一段 CLAUDE.md 让后续 AI 少出错

**这才是白鲨内部系统的正确投资方向**。
