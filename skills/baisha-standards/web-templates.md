# 前端模板（Vue 3 + Vite 5 + Element Plus + Pinia）

> 纯架构骨架，不带业务字段。`Resource` 是占位资源名，新建业务页面照样画葫芦。

## 0. web/package.json

```json
{
  "name": "{{PROJECT}}-web",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@element-plus/icons-vue": "^2.3.1",
    "axios": "^1.7.7",
    "dayjs": "^1.11.13",
    "element-plus": "^2.8.7",
    "pinia": "^2.2.4",
    "vue": "^3.5.12",
    "vue-router": "^4.4.5"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.1.4",
    "vite": "^5.4.10"
  },
  "pnpm": {
    "onlyBuiltDependencies": ["esbuild", "vue-demi"]
  }
}
```

## 1. vite.config.js（**两段 proxy 都要**）

```js
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
  plugins: [vue()],
  server: {
    host: '0.0.0.0',     // 监听所有网卡，局域网可访问
    port: 5173,
    strictPort: true,
    proxy: {
      '/api': { target: 'http://localhost:4000', changeOrigin: true },
    },
  },
  // ★ 生产用 vite preview 跑前端时这段才生效；不写就 /api 404
  preview: {
    host: '0.0.0.0',
    port: 5173,
    strictPort: true,
    proxy: {
      '/api': { target: 'http://localhost:4000', changeOrigin: true },
    },
  },
});
```

**坑**：`server.proxy` 只对 `vite dev` 生效，`preview.proxy` 才对 `vite preview` 生效。如果生产用 nginx 反代（推荐），preview.proxy 可不写。

## 2. index.html

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>{{PROJECT_TITLE}}</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.js"></script>
  </body>
</html>
```

## 3. src/main.js

```js
import { createApp } from 'vue';
import { createPinia } from 'pinia';
import ElementPlus from 'element-plus';
import 'element-plus/dist/index.css';
import zhCn from 'element-plus/dist/locale/zh-cn.mjs';
import App from './App.vue';
import router from './router';
import './styles.css';

const app = createApp(App);
app.use(createPinia());
app.use(router);
app.use(ElementPlus, { locale: zhCn });
app.mount('#app');
```

## 4. src/App.vue

```vue
<script setup>
import { onMounted } from 'vue';
import { useUserStore } from './stores/user';

const userStore = useUserStore();
onMounted(() => { if (userStore.token) userStore.fetchMe(); });
</script>

<template>
  <router-view />
</template>
```

## 5. src/router.js

```js
import { createRouter, createWebHistory } from 'vue-router';
import { useUserStore } from './stores/user';

const routes = [
  { path: '/login', name: 'login', component: () => import('./views/Login.vue') },
  {
    path: '/',
    component: () => import('./components/Layout.vue'),
    redirect: '/resource',
    children: [
      { path: 'resource',         name: 'resource-list',   component: () => import('./views/resource/List.vue') },
      { path: 'resource/new',     name: 'resource-new',    component: () => import('./views/resource/Edit.vue') },
      { path: 'resource/:id',     name: 'resource-detail', component: () => import('./views/resource/Detail.vue') },
      { path: 'resource/:id/edit', name: 'resource-edit',  component: () => import('./views/resource/Edit.vue') },

      // 系统管理
      { path: 'system/accounts', name: 'accounts', component: () => import('./views/system/Accounts.vue') },
      { path: 'system/roles',    name: 'roles',    component: () => import('./views/system/Roles.vue') },
    ],
  },
];

const router = createRouter({ history: createWebHistory(), routes });

router.beforeEach(async (to) => {
  const userStore = useUserStore();
  if (to.path === '/login') return true;
  if (!userStore.isLoggedIn) return { path: '/login', query: { redirect: to.fullPath } };
  if (!userStore.user) await userStore.fetchMe();
  return true;
});

export default router;
```

## 6. src/styles.css（基线样式）

```css
* { box-sizing: border-box; }
html, body, #app { height: 100%; margin: 0; }
body {
  font-family: 'PingFang SC', 'Microsoft YaHei', -apple-system, sans-serif;
  background: #f5f7fa;
  color: #303133;
}
.page-card { background: #fff; padding: 16px; border-radius: 4px; border: 1px solid #ebeef5; }
```

## 7. src/api/index.js（axios 实例）

```js
import axios from 'axios';
import { ElMessage } from 'element-plus';

const api = axios.create({ baseURL: '/api', timeout: 15000 });

api.interceptors.request.use((cfg) => {
  const token = localStorage.getItem('token');
  if (token) cfg.headers.Authorization = `Bearer ${token}`;
  return cfg;
});

api.interceptors.response.use(
  (res) => res.data,
  (err) => {
    const msg = err.response?.data?.error || err.message || '请求失败';
    if (err.response?.status === 401) {
      localStorage.removeItem('token');
      if (!location.pathname.startsWith('/login')) location.href = '/login';
    } else {
      ElMessage.error(msg);
    }
    return Promise.reject(err);
  }
);

export default api;
```

**要点**：
- `baseURL: '/api'` 用**相对路径**，让 Vite proxy / nginx 反代统一处理，跨环境零修改
- 401 自动跳登录页 + 清 token
- 其他错误用 ElMessage 提示，业务代码不用每次 catch

## 8. src/api/resources.js（接口聚合）

按资源分组，每个资源一个对象，新业务照葫芦画瓢：

```js
import api from './index';

export const authApi = {
  changePassword: (data) => api.post('/auth/change-password', data),
};

export const personsApi = {
  list:   (params) => api.get('/persons', { params }),
  detail: (id)     => api.get(`/persons/${id}`),
  create: (data)   => api.post('/persons', data),
  update: (id, data) => api.put(`/persons/${id}`, data),
  remove: (id)     => api.delete(`/persons/${id}`),
};

export const rolesApi = {
  list:   () => api.get('/roles'),
  create: (data) => api.post('/roles', data),
  update: (id, data) => api.put(`/roles/${id}`, data),
  remove: (id) => api.delete(`/roles/${id}`),
};

export const accountsApi = {
  list:   (params) => api.get('/accounts', { params }),
  create: (data) => api.post('/accounts', data),
  update: (id, data) => api.put(`/accounts/${id}`, data),
};

// 新业务资源：复制上面任一个改名即可
// export const resourceApi = { list: ..., detail: ..., create: ..., update: ..., remove: ... };
```

## 9. src/stores/user.js（Pinia 登录态）

```js
import { defineStore } from 'pinia';
import api from '../api';

export const useUserStore = defineStore('user', {
  state: () => ({
    user: null,
    token: localStorage.getItem('token') || '',
  }),
  getters: {
    isLoggedIn: (s) => !!s.token,
    roleCodes:  (s) => (s.user?.roles || []).map((r) => r.code),
    isAdmin:    (s) => (s.user?.roles || []).some((r) => r.code === 'admin'),
  },
  actions: {
    async login(username, password) {
      const data = await api.post('/auth/login', { username, password });
      this.token = data.token;
      this.user = data.user;
      localStorage.setItem('token', data.token);
    },
    async fetchMe() {
      if (!this.token) return null;
      try { this.user = await api.get('/auth/me'); }
      catch { this.logout(); }
      return this.user;
    },
    logout() {
      this.token = ''; this.user = null;
      localStorage.removeItem('token');
    },
    hasRole(...codes) { return (this.user?.roles || []).some((r) => codes.includes(r.code)); },
  },
});
```

## 10. src/views/Login.vue

```vue
<script setup>
import { reactive } from 'vue';
import { useRouter } from 'vue-router';
import { ElMessage } from 'element-plus';
import { useUserStore } from '../stores/user';

const router = useRouter();
const userStore = useUserStore();
const form = reactive({ username: '', password: '' });
const loading = ref(false);

async function submit() {
  if (!form.username || !form.password) return ElMessage.warning('请填账号和密码');
  loading.value = true;
  try {
    await userStore.login(form.username, form.password);
    router.push('/');
  } catch {} finally { loading.value = false; }
}
</script>

<template>
  <div style="display:flex; align-items:center; justify-content:center; height:100vh; background:#f5f7fa;">
    <el-card style="width: 360px;">
      <h2 style="text-align:center; margin: 0 0 16px;">{{PROJECT_TITLE}}</h2>
      <el-form :model="form" @submit.prevent="submit">
        <el-form-item><el-input v-model="form.username" placeholder="账号" /></el-form-item>
        <el-form-item><el-input v-model="form.password" type="password" placeholder="密码" show-password /></el-form-item>
        <el-form-item>
          <el-button type="primary" style="width:100%;" :loading="loading" @click="submit">登录</el-button>
        </el-form-item>
      </el-form>
    </el-card>
  </div>
</template>
```

## 11. src/components/Layout.vue（顶部导航 + 角色过滤）

```vue
<script setup>
import { computed } from 'vue';
import { useRouter, useRoute } from 'vue-router';
import { ArrowDown } from '@element-plus/icons-vue';
import { useUserStore } from '../stores/user';

const router = useRouter();
const route = useRoute();
const userStore = useUserStore();

const menus = computed(() => {
  const all = [
    { name: 'resource-list', label: '资源',     roles: null },           // null = 全员可见
    { name: 'accounts',      label: '系统管理', roles: ['admin'] },
  ];
  return all.filter((m) => !m.roles || userStore.hasRole(...m.roles));
});

function logout() { userStore.logout(); router.push('/login'); }
</script>

<template>
  <el-container style="height: 100vh;">
    <el-header style="display:flex; align-items:center; gap:24px; background:#fff; border-bottom:1px solid #eee;">
      <strong>{{PROJECT_TITLE}}</strong>
      <el-menu mode="horizontal" :default-active="route.name" :ellipsis="false"
               style="flex:1; border-bottom:none;"
               @select="(idx) => router.push({ name: idx })">
        <el-menu-item v-for="m in menus" :key="m.name" :index="m.name">{{ m.label }}</el-menu-item>
      </el-menu>
      <el-dropdown>
        <span>
          {{ userStore.user?.person?.name || userStore.user?.username }}
          <el-icon><ArrowDown /></el-icon>
        </span>
        <template #dropdown>
          <el-dropdown-menu>
            <el-dropdown-item @click="logout">退出登录</el-dropdown-item>
          </el-dropdown-menu>
        </template>
      </el-dropdown>
    </el-header>
    <el-main style="background:#f5f7fa; padding:16px;">
      <router-view />
    </el-main>
  </el-container>
</template>
```

## 12. 三模板：List / Detail / Edit

新业务资源直接复制改字段名。下面的 `Resource` 是占位资源。

### 12.1 views/resource/List.vue

```vue
<script setup>
import { ref, reactive, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import { ElMessage, ElMessageBox } from 'element-plus';
import { resourceApi } from '../../api/resources';

const router = useRouter();
const list = ref([]);
const total = ref(0);
const loading = ref(false);
const filter = reactive({ keyword: '', page: 1, page_size: 10 });

async function load() {
  loading.value = true;
  try {
    const r = await resourceApi.list(filter);
    list.value = r.list; total.value = r.total;
  } finally { loading.value = false; }
}
function reset() { Object.assign(filter, { keyword: '', page: 1, page_size: 10 }); load(); }
onMounted(load);

async function remove(row) {
  await ElMessageBox.confirm(`确认删除 ${row.id}？`, '提示', { type: 'warning' });
  await resourceApi.remove(row.id);
  ElMessage.success('已删除'); load();
}
</script>

<template>
  <div>
    <el-card shadow="never" style="margin-bottom: 12px;">
      <el-form :inline="true" :model="filter" @submit.prevent="load">
        <el-form-item label="关键词">
          <el-input v-model="filter.keyword" placeholder="搜索..." clearable />
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="load">筛选</el-button>
          <el-button @click="reset">重置</el-button>
        </el-form-item>
      </el-form>
    </el-card>

    <el-card shadow="never">
      <div style="display:flex; align-items:center; margin-bottom:12px;">
        <div style="flex:1; font-weight:600;">共 {{ total }} 条</div>
        <el-button type="primary" @click="router.push({ name: 'resource-new' })">+ 新建</el-button>
      </div>

      <el-table :data="list" v-loading="loading" max-height="600" border>
        <el-table-column prop="id" label="ID" width="80" />
        <!-- 业务字段在这里加 el-table-column -->
        <el-table-column prop="created_at" label="创建时间" width="160" />
        <el-table-column label="操作" width="180" fixed="right">
          <template #default="{ row }">
            <el-button link type="primary" size="small"
                       @click="router.push({ name: 'resource-detail', params: { id: row.id } })">查看</el-button>
            <el-button link type="primary" size="small"
                       @click="router.push({ name: 'resource-edit', params: { id: row.id } })">编辑</el-button>
            <el-button link type="danger" size="small" @click="remove(row)">删除</el-button>
          </template>
        </el-table-column>
      </el-table>

      <div style="margin-top:12px; display:flex; justify-content:flex-end;">
        <el-pagination v-model:current-page="filter.page" v-model:page-size="filter.page_size"
                       :page-sizes="[10]" :total="total"
                       layout="total, prev, pager, next" @current-change="load" />
      </div>
    </el-card>
  </div>
</template>
```

### 12.2 views/resource/Detail.vue

```vue
<script setup>
import { ref, computed, watch } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { resourceApi } from '../../api/resources';

const route = useRoute();
const router = useRouter();
const id = computed(() => +route.params.id);
const data = ref(null);
const loading = ref(false);

async function load() {
  loading.value = true;
  try { data.value = await resourceApi.detail(id.value); }
  finally { loading.value = false; }
}

// ★ 关键：route.params.id 切换时组件不卸载，必须 watch 手动刷新
watch(id, load, { immediate: true });
</script>

<template>
  <div v-loading="loading">
    <el-card v-if="data" shadow="never">
      <div style="display:flex; align-items:center; gap:12px; margin-bottom:16px;">
        <strong style="font-size:16px;">详情 #{{ data.id }}</strong>
        <div style="flex:1"></div>
        <el-button type="primary"
                   @click="router.push({ name: 'resource-edit', params: { id: data.id } })">编辑</el-button>
        <el-button @click="router.back()">关闭</el-button>
      </div>
      <el-descriptions :column="3" border>
        <el-descriptions-item label="ID">{{ data.id }}</el-descriptions-item>
        <!-- 业务字段 -->
        <el-descriptions-item label="创建时间">{{ data.created_at }}</el-descriptions-item>
      </el-descriptions>
    </el-card>
  </div>
</template>
```

### 12.3 views/resource/Edit.vue（创建 + 编辑共用）

```vue
<script setup>
import { ref, reactive, computed, onMounted } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { ElMessage } from 'element-plus';
import { resourceApi } from '../../api/resources';

const route = useRoute();
const router = useRouter();
const id = computed(() => +route.params.id || null);
const mode = computed(() => id.value ? 'edit' : 'new');
const title = computed(() => mode.value === 'edit' ? '编辑' : '新建');

const form = reactive({
  // 业务字段在这里加
});
const submitting = ref(false);

async function load() {
  if (!id.value) return;
  const data = await resourceApi.detail(id.value);
  Object.assign(form, data);
}

async function save() {
  submitting.value = true;
  try {
    if (id.value) await resourceApi.update(id.value, form);
    else await resourceApi.create(form);
    ElMessage.success('已保存');
    router.push({ name: 'resource-list' });
  } finally { submitting.value = false; }
}

onMounted(load);
</script>

<template>
  <el-card shadow="never">
    <div style="display:flex; align-items:center; gap:12px; margin-bottom:16px;">
      <strong>{{ title }}</strong>
      <div style="flex:1"></div>
      <el-button @click="router.back()">关闭</el-button>
      <el-button type="primary" @click="save" :loading="submitting">保存</el-button>
    </div>
    <el-form :model="form" label-width="120px">
      <!-- 业务字段表单 -->
    </el-form>
  </el-card>
</template>
```

## 13. 三个值得记住的前端陷阱

| 陷阱 | 修法 |
|---|---|
| 路由 `:id` 切换不刷新 | `watch(id, load, { immediate: true })` 替代 `onMounted` |
| 表格横向滚动条只在底部出现 | `el-table` 加 `max-height` |
| HTTP 请求要换路径 | 永远用相对路径 `/api/xxx`，让 vite proxy / nginx 转 |
