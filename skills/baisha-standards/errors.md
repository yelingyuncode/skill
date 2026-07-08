# 业务错 vs 系统错

**AI 生成代码最容易混淆的一件事**：把业务错当系统错抛出，导致前端弹"系统繁忙，请稍后重试"——用户完全不知道自己错在哪里。

## §1 一句话区分

| 类型 | 举例 | 谁的问题 | 如何处理 |
|---|---|---|---|
| **业务错** | 库存不足、状态不对、权限不够、单号已存在、密码错误、余额不够、订单已作废 | 用户操作错 / 数据当前不满足条件 | HTTP 200 + `{ok:false, code, msg}`，前端 warning 提示 |
| **系统错** | 数据库连不上、undefined.prop、JSON 解析失败、外部服务超时 | 代码或基础设施出问题 | HTTP 500 + 日志，前端统一"系统繁忙" |

## §2 统一返回结构

**所有业务接口**统一返回体：

```ts
// 成功
{ ok: true, data: <任何东西> }

// 业务错
{ ok: false, code: 'INSUFFICIENT_STOCK', msg: '库存不足' }

// 分页
{ ok: true, data: { list: [...], total: 123, page: 1, size: 20 } }
```

**HTTP 状态码**：

- 业务错 → **200**（是的，就是 200，因为 HTTP 层面请求处理成功了，业务是另一回事）
- 未登录 → **401**（自动跳登录）
- 无权限 → **403**（弹权限提示）
- 找不到路由 → **404**
- 系统崩了 → **500**（走全局错误处理）

## §3 后端实现

### 业务错：用带 `businessError` 标记的 Error

```js
// server/src/lib/errors.js
export function businessError(code, msg) {
  const err = new Error(msg);
  err.code = code;
  err.businessError = true;
  return err;
}

// 常用业务错码，集中管理
export const CODES = Object.freeze({
  INSUFFICIENT_STOCK:  '库存不足',
  INVALID_TRANSITION:  '不允许的状态迁移',
  DUPLICATE_ORDER_NO:  '单号重复',
  NO_PERMISSION:       '无操作权限',
  NOT_FOUND:           '数据不存在',
  ALREADY_CANCELLED:   '该单已作废',
  CONCURRENT_UPDATE:   '该单已被他人修改，请刷新后重试',
});
```

### route 里怎么用

```js
fastify.post('/orders/:id/release', async (req, reply) => {
  const { id } = req.params;
  const { user } = req;

  try {
    return db.transaction(() => {
      const order = db.prepare('SELECT * FROM `order` WHERE id=?').get(id);
      if (!order) throw businessError('NOT_FOUND', '订单不存在');
      if (order.status === 'cancelled') throw businessError('ALREADY_CANCELLED', '订单已作废');
      assertTransition(order.status, 'released');  // 内部也会 throw businessError

      db.prepare("UPDATE `order` SET status='released', updated_at=?, updated_by=? WHERE id=?").run(now(), user.id, id);
      audit({ user, entity:'order', entityId:id, action:'status_change',
              before:{status:order.status}, after:{status:'released'} });

      return { ok: true, data: { id, status: 'released' } };
    })();
  } catch (err) {
    if (err.businessError) {
      req.log.warn({ code: err.code, msg: err.message }, 'business error');
      return { ok: false, code: err.code, msg: err.message };
    }
    throw err; // 系统错交给全局 handler
  }
});
```

### 全局错误处理（系统错）

```js
// server/src/index.js
fastify.setErrorHandler((err, req, reply) => {
  req.log.error({ err, path: req.raw.url }, 'system error');
  reply.code(500).send({ ok: false, code: 'INTERNAL', msg: '系统繁忙，请稍后重试' });
});
```

## §4 前端处理（axios 拦截器）

```js
// web/src/api/index.js
import axios from 'axios';
import { ElMessage } from 'element-plus';
import router from '../router.js';
import { useUserStore } from '../stores/user.js';

const http = axios.create({ baseURL: '/api', timeout: 15000 });

http.interceptors.request.use((cfg) => {
  const token = useUserStore().token;
  if (token) cfg.headers.Authorization = `Bearer ${token}`;
  return cfg;
});

http.interceptors.response.use(
  (resp) => {
    const body = resp.data;
    if (body && body.ok === false) {
      // ↓ 这是业务错，走 warning
      ElMessage.warning(body.msg || '操作失败');
      return Promise.reject({ businessError: true, code: body.code, msg: body.msg });
    }
    return body.data;   // 直接返回 data，页面不用再 .data
  },
  (err) => {
    // ↓ 这里都是 HTTP 层面错（401 / 500 / 网络）
    const status = err.response?.status;
    if (status === 401) {
      useUserStore().logout();
      router.push('/login');
      return Promise.reject(err);
    }
    if (status === 403) {
      ElMessage.error('无权限');
    } else if (status >= 500) {
      ElMessage.error('系统繁忙，请稍后重试');
    } else if (!err.response) {
      ElMessage.error('网络异常');
    } else {
      ElMessage.error(err.message || '请求失败');
    }
    return Promise.reject(err);
  }
);

export default http;
```

**页面代码里怎么用**：

```js
async function submit() {
  submitting.value = true;
  try {
    const data = await http.post('/orders', form.value); // 成功直接拿 data
    ElMessage.success('提交成功');
    // 走下一步
  } catch (e) {
    // 业务错：拦截器已 warning，这里只要不继续走成功路径
    // 系统错：拦截器已 error，这里也是同样处理
  } finally {
    submitting.value = false;
  }
}
```

**页面里不用自己判 `ok`**，拦截器统一处理。

## §5 常见 AI 犯的错

### ❌ 把业务错 500 抛出去

```js
if (stock.qty < qty) throw new Error('库存不足');   // ← 变成 500
```

用户看到"系统繁忙"，明明是自己填多了。

**改**：`throw businessError('INSUFFICIENT_STOCK', '库存不足');`

### ❌ 把系统错捕获成业务错

```js
try {
  await someOperation();
} catch (e) {
  return { ok:false, code:'UNKNOWN', msg: e.message };  // ← 系统崩溃被"包装"了
}
```

Bug 被吞掉，用户看到"undefined is not a function"这种消息。

**改**：只 catch 你能识别的业务错（`err.businessError`）。系统错让全局 handler 处理并写日志。

### ❌ 前端弹 alert 而不是 Message

```vue
<button @click="submit">提交</button>

<script>
async submit() {
  const r = await axios.post('/api/orders', form);
  if (!r.data.ok) alert(r.data.msg);   // ← 阻塞式，体验差
}
</script>
```

**改**：走拦截器 + `ElMessage.warning`。

### ❌ HTTP 状态码 400 传业务错

```js
reply.code(400).send({ msg: '库存不足' });
```

前端 axios 认为是 HTTP 错，走的是 error 分支不是 response 分支，拦截器逻辑对不上。**业务错一律 200**。

### ❌ 错误消息里带技术细节

```js
throw businessError('DB_ERROR', 'SQLITE_BUSY: database is locked');
```

用户看不懂。**msg 必须是中文、面向业务方**（"操作繁忙，请稍后重试"）；技术细节写日志。

## §6 code 命名约定

- **全大写下划线**：`INSUFFICIENT_STOCK` / `INVALID_TRANSITION`
- **前缀分类**（可选）：`STOCK_INSUFFICIENT` / `ORDER_ALREADY_CANCELLED`——按业务模块加前缀，或不加都行，团队统一
- **别用中文 code**（`'库存不足'`）——中文 code 不能给国际化，也不能改 msg 而不动 code

## §7 checklist

AI 写完接口后自查：

- [ ] 校验失败是 `businessError`（不是 `throw new Error`）
- [ ] `if (err.businessError)` 分支返回 `{ok:false, code, msg}`
- [ ] 其他异常 `throw err` 走全局 handler
- [ ] msg **中文、面向业务方**
- [ ] code **英文大写下划线**
- [ ] HTTP 状态码：业务错 **200**，未登录 **401**，无权限 **403**，系统错 **500**
- [ ] 前端 `axios` 拦截器统一处理，页面不自己判 `ok`
