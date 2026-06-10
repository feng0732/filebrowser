# File Browser 前端布局与路由配合机制分析

## 一、项目整体架构概览

File Browser 是一个基于 **Vue 3 + TypeScript + Pinia + Vue Router** 的文件管理系统。前端代码位于 `frontend/src/` 目录，整体采用**嵌套路由 + 嵌套布局组件**的架构模式。

### 核心目录结构

```
frontend/src/
├── App.vue                 # 根组件，仅包含最外层 router-view
├── main.ts                 # 应用入口，注册插件并挂载
├── router/index.ts         # 路由配置与全局路由守卫
├── stores/                 # Pinia 状态管理
│   ├── index.ts            # Pinia 创建，注入 router
│   ├── auth.ts             # 认证状态
│   ├── layout.ts           # 布局 UI 状态（弹窗、加载、Shell）
│   ├── file.ts             # 文件数据状态
│   ├── router.ts           # 路由 store（空实现，预留）
│   └── upload.ts           # 上传状态
├── views/                  # 页面级视图组件
│   ├── Layout.vue          # 主布局框架（Sidebar + Main + Prompts）
│   ├── Login.vue           # 登录页（无 Layout）
│   ├── Files.vue           # 文件浏览容器视图
│   ├── Share.vue           # 共享文件视图
│   ├── Settings.vue        # 设置页容器视图（含二级导航）
│   ├── Errors.vue          # 错误页
│   ├── files/              # 文件子视图
│   │   ├── FileListing.vue # 目录列表
│   │   ├── Editor.vue      # 文件编辑器
│   │   └── Preview.vue     # 文件预览
│   └── settings/           # 设置子视图
│       ├── Profile.vue     # 个人设置
│       ├── Shares.vue      # 共享管理
│       ├── Global.vue      # 全局设置
│       ├── Users.vue       # 用户列表
│       └── User.vue        # 单个用户编辑
└── components/             # 可复用组件
    ├── Sidebar.vue         # 侧边栏导航
    ├── Shell.vue           # 终端 Shell
    ├── Breadcrumbs.vue     # 面包屑
    ├── header/HeaderBar.vue# 顶部操作栏
    └── prompts/Prompts.vue # 弹窗管理容器
```

---

## 二、路由配置详解

路由配置定义在 [router/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/router/index.ts)。

### 2.1 路由层级结构

路由采用**三层嵌套**设计：

```
/ (根路径 catchAll 重定向)
├── /login                        # 一级路由，独立布局
├── /share/:path*                 # 一级路由 Layout + 二级
│   └── Share 组件
├── /files/:path*                 # 一级路由 Layout + 二级（需认证）
│   └── Files 组件（内部动态选择 FileListing/Editor/Preview）
├── /settings                     # 一级路由 Layout + 二级（需认证）
│   └── Settings 组件（三级导航容器）
│       ├── /settings/profile         # ProfileSettings
│       ├── /settings/shares          # Shares
│       ├── /settings/global          # GlobalSettings（需管理员）
│       ├── /settings/users           # Users（需管理员）
│       └── /settings/users/:id       # User（需管理员）
├── /403, /404, /500             # 独立错误页
└── /:catchAll(.*)*              # 通配重定向到 /files/
```

### 2.2 路由元信息（meta）

| meta 字段 | 作用 | 应用路由 |
|-----------|------|----------|
| `requiresAuth` | 需要登录认证 | `/files/*`, `/settings/*` |
| `requiresAdmin` | 需要管理员权限 | `/settings/global`, `/settings/users`, `/settings/users/:id` |

### 2.3 全局路由守卫 `beforeResolve`

路由守卫在 [router/index.ts#L183-L222](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/router/index.ts#L183-L222) 中定义，执行以下流程：

1. **设置页面标题**：根据路由名称查找 i18n key，设置 `document.title`
2. **首次访问初始化认证**：`from.name == null` 时调用 `initAuth()`
3. **登录态用户访问登录页**：重定向到 `/files/`
4. **认证检查**：
   - `requiresAuth` 路由：未登录则跳转 `/login?redirect=...`
   - `requiresAdmin` 路由：非管理员则跳转 `/403`
5. **放行**：`next()`

### 2.4 通配重定向策略

未匹配路径统一重定向到 `/files/` 前缀，见 [router/index.ts#L147-L153](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/router/index.ts#L147-L153)：

```ts
{
  path: "/:catchAll(.*)*",
  redirect: (to: RouteLocation) => {
    const catchAll = to.params.catchAll;
    if (!catchAll) return "/files/";
    return `/files/${Array.isArray(catchAll) ? catchAll.join("/") : catchAll}`;
  },
}
```

---

## 三、布局组件层次

### 3.1 根组件 App.vue

[App.vue](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/App.vue) 极其简洁，只包含一个 `<router-view>`，处理主题和语言初始化，**不承担任何布局职责**。

```vue
<template>
  <div>
    <router-view></router-view>
  </div>
</template>
```

### 3.2 主布局组件 Layout.vue

[Layout.vue](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/views/Layout.vue) 是**所有需要导航的页面的外壳**，结构如下：

```
Layout.vue
├── 上传进度条（有上传时显示）
├── <Sidebar />          # 侧边栏导航
├── <main>
│   ├── <router-view />  # 子路由渲染出口（Files / Share / Settings）
│   └── <Shell />        # 终端 Shell（权限满足时显示）
├── <Prompts />          # 全局弹窗容器
└── <UploadFiles />      # 上传文件弹窗
```

关键逻辑：

- **路由切换监听**（[Layout.vue#L47-L53](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/views/Layout.vue#L47-L53)）：每次路由变化清空文件选择、关闭非 success 弹窗
- **条件渲染 Shell**：需同时满足 `enableExec` 常量、已登录、用户有 `execute` 权限

### 3.3 登录页 Login.vue（独立布局）

[Login.vue](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/views/Login.vue) 不嵌套在 Layout 中，直接渲染登录表单，包含：
- 用户名/密码输入
- 注册模式切换（signup 启用时）
- reCAPTCHA 验证
- 登录成功后跳转到 `query.redirect` 或 `/files/`

### 3.4 文件视图容器 Files.vue

[Files.vue](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/views/Files.vue) 是 `/files/:path*` 的实际渲染组件，结构：

```
Files.vue
├── <HeaderBar showMenu showLogo />
├── <Breadcrumbs base="/files" />
├── <Errors />           # 请求错误时显示
├── <component :is="currentView" />  # 动态组件
│   ├── FileListing.vue  # 目录
│   ├── Editor.vue       # 文本/可编辑文件（异步加载）
│   └── Preview.vue      # 其他文件/CSV 表格视图（异步加载）
└── 加载中 Spinner
```

**动态视图选择逻辑**（[Files.vue#L65-L86](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/views/Files.vue#L65-L86)）：

```
req.type === undefined → null（加载中）
req.isDir === true     → FileListing（目录列表）
extension === .csv     → Preview（表格），?edit=true 时 → Editor
type === text / textImmutable → Editor
其他                   → Preview
```

**数据获取**：监听 `route` 和 `reload`，调用 `fetchData()` 通过 API 获取当前路径的文件资源。

### 3.5 设置视图容器 Settings.vue

[Settings.vue](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/views/Settings.vue) 是 `/settings/*` 的容器，含**二级子导航**：

```
Settings.vue
├── <HeaderBar showMenu showLogo />
├── <div id="nav">        # 设置项导航栏
│   ├── /settings/profile     # 个人设置
│   ├── /settings/shares      # 共享管理（perm.share）
│   ├── /settings/global      # 全局设置（perm.admin）
│   └── /settings/users       # 用户管理（perm.admin）
├── 加载 Spinner
└── <router-view />      # 三级路由出口
```

默认重定向 `/settings` → `/settings/profile`（见路由配置）。

### 3.6 共享视图 Share.vue

[Share.vue](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/views/Share.vue) 渲染公开共享链接，结构类似 Files.vue 但更简单，不包含操作菜单，只有下载/预览功能和密码保护输入。

---

## 四、状态管理与路由布局的配合

### 4.1 Pinia 注入 Router

在 [stores/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/stores/index.ts) 中，Pinia 通过插件把 `router` 实例注入到每个 store：

```ts
pinia.use(({ store }) => {
  store.router = markRaw(router);
});
```

这样所有 store 都可以通过 `this.router` 访问路由实例（当前代码中主要为预留能力）。

### 4.2 Layout Store 控制全局 UI

[stores/layout.ts](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/stores/layout.ts) 管理跨页面共享的 UI 状态：

| 状态 | 作用 | 相关组件 |
|------|------|----------|
| `loading` | 全局加载状态 | Files.vue, Share.vue, Settings.vue |
| `prompts: PopupProps[]` | 弹窗栈（后进先出） | Prompts.vue, Sidebar.vue, HeaderBar.vue |
| `showShell` | Shell 终端显示/隐藏 | Layout.vue, Shell.vue |

**弹窗栈机制**：
- `showHover(value)` 压入一个弹窗
- `closeHovers()` 弹出并调用弹窗的 `close()` 回调
- `currentPromptName` getter 取栈顶弹窗名
- Prompts.vue 通过 `component :is="modal"` 动态渲染当前弹窗组件

### 4.3 Auth Store 控制路由守卫

[stores/auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/stores/auth.ts) 的 `isLoggedIn` getter 和 `user.perm.admin` 是路由守卫判断的核心依据。

### 4.4 File Store 驱动视图切换

[stores/file.ts](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/stores/file.ts) 的 `req` 字段存储当前请求的文件资源对象。Files.vue 的 `currentView` computed 完全依赖 `req` 的属性（`isDir`、`type`、`extension`）决定渲染哪个子视图。

---

## 五、路由切换的完整数据流

### 5.1 场景一：用户从登录页进入文件浏览

```
用户点击登录
  ↓
auth.login() → authStore.user 更新
  ↓
router.push("/files/")
  ↓
beforeResolve 守卫
  ├── 设置标题 "My Files - File Browser"
  ├── 检查 requiresAuth → 通过（已登录）
  └── next()
  ↓
路由匹配：/files → Layout.vue → 子路由 /files/ → Files.vue
  ↓
Layout.vue 渲染：
  ├── Sidebar（显示用户名、我的文件、设置等）
  └── main > router-view → Files.vue
  ↓
Files.vue onMounted
  ├── fetchData() 调用 API 获取根目录资源
  ├── layoutStore.loading = true
  └── fileStore.updateRequest(res) 存储资源
  ↓
currentView 计算 → req.isDir=true → 渲染 FileListing.vue
  ↓
layoutStore.loading = false，页面展示文件列表
```

### 5.2 场景二：在文件列表中点击子目录

```
用户点击目录 /documents
  ↓
router.push("/files/documents")
  ↓
beforeResolve（from.name 非空，跳过 initAuth）
  ↓
Layout.vue watch(route)
  ├── fileStore.selected = []（清空选择）
  └── layoutStore.closeHovers()（关闭弹窗）
  ↓
Files.vue watch(route) → fetchData()
  ├── 中止上次请求（AbortController）
  ├── 请求 /api/resources/documents
  └── fileStore.updateRequest() 更新 req
  ↓
currentView 仍为 FileListing.vue，但数据已更新
  ↓
Breadcrumbs 组件根据 route.params.path 自动更新面包屑
```

### 5.3 场景三：从文件列表进入设置页

```
Sidebar 点击 "设置" → router.push("/settings/global")
  ↓
Layout.vue watch(route)
  ├── fileStore.selected = []
  └── fileStore.multiple = false
  ↓
Settings.vue 渲染（其 watch 会触发子路由数据加载）
  ↓
Settings 内部导航高亮 "全局设置"（通过 $route.path 判断 active 类）
  ↓
Settings > router-view → GlobalSettings.vue
```

### 5.4 场景四：未登录用户访问受保护路由

```
直接访问 /settings/profile
  ↓
beforeResolve 守卫
  ├── initAuth() 首次初始化（检查 cookie/token）
  ├── authStore.isLoggedIn 为 false
  └── next({ path: "/login", query: { redirect: "/settings/profile" } })
  ↓
Login.vue 渲染
  ↓
用户登录成功 → router.push(query.redirect || "/files/")
  ↓
回到场景一的流程
```

---

## 六、关键设计模式总结

### 6.1 嵌套路由 + 嵌套布局

整个应用通过三层嵌套 `router-view` 实现布局复用：

| 层级 | 组件 | 职责 |
|------|------|------|
| 第一层 | App.vue | 主题/语言初始化，无布局 |
| 第二层 | Layout.vue | 全局框架（Sidebar、Prompts、Shell、上传进度） |
| 第三层 | Files.vue / Settings.vue / Share.vue | 页面容器（导航栏、面包屑、子视图） |
| 第四层 | FileListing / Editor / Preview / ProfileSettings... | 具体内容 |

### 6.2 动态组件 + Pinia 驱动视图

Files.vue 不使用子路由区分文件/目录/编辑器，而是通过 `fileStore.req` 的元数据动态 `<component :is="currentView">`，实现了**单路由多视图**的模式，避免了路由过度碎片化。

### 6.3 弹窗栈（Prompt Stack）

全局弹窗不使用路由控制，而是通过 `layoutStore.prompts` 数组作为栈管理，Prompts.vue 作为统一容器渲染栈顶弹窗。优势：
- 弹窗状态与 URL 解耦，刷新不残留
- 支持弹窗叠加（栈结构）
- 统一的开关、回调机制

### 6.4 路由守卫集中控制权限

所有权限检查集中在 `beforeResolve` 单一守卫中，通过 `meta.requiresAuth` 和 `meta.requiresAdmin` 声明式配置，避免每个组件重复鉴权逻辑。

### 6.5 响应式路由监听驱动数据

Files.vue、Share.vue、Sidebar.vue 均使用 `watch(route)` 触发数据重新获取，Layout.vue 用 watch(route) 做全局 UI 重置，确保路由变化时各组件能正确响应。

---

## 七、核心文件速查

| 文件 | 作用 |
|------|------|
| [router/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/router/index.ts) | 路由定义 + 全局守卫 |
| [views/Layout.vue](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/views/Layout.vue) | 主布局框架（Sidebar + Main + Prompts） |
| [views/Files.vue](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/views/Files.vue) | 文件浏览容器，动态切换 Listing/Editor/Preview |
| [views/Settings.vue](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/views/Settings.vue) | 设置页容器 + 二级导航 |
| [stores/layout.ts](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/stores/layout.ts) | 全局 UI 状态（弹窗栈、加载、Shell） |
| [stores/file.ts](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/stores/file.ts) | 当前文件资源状态，驱动视图切换 |
| [stores/auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/stores/auth.ts) | 用户认证状态，驱动权限守卫 |
| [components/Sidebar.vue](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/components/Sidebar.vue) | 侧边栏导航，根据权限显示菜单项 |
| [components/prompts/Prompts.vue](file:///d:/fz/0601/solo-dogfeeding/code/176-filebrowser/frontend/src/components/prompts/Prompts.vue) | 全局弹窗渲染容器 |
