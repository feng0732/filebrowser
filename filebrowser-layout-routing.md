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

路由配置定义在 `frontend/src/router/index.ts`。

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

路由守卫在 `frontend/src/router/index.ts` 第 183–222 行中定义，执行以下流程：

1. **设置页面标题**：根据路由名称查找 i18n key，设置 `document.title`
2. **首次访问初始化认证**：`from.name == null` 时调用 `initAuth()`
3. **登录态用户访问登录页**：重定向到 `/files/`
4. **认证检查**：
   - `requiresAuth` 路由：未登录则跳转 `/login?redirect=...`
   - `requiresAdmin` 路由：非管理员则跳转 `/403`
5. **放行**：`next()`

### 2.4 通配重定向策略

未匹配路径统一重定向到 `/files/` 前缀，见 `frontend/src/router/index.ts` 第 147–153 行：

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

`frontend/src/App.vue` 极其简洁，只包含一个 `<router-view>`，处理主题和语言初始化，**不承担任何布局职责**。

```vue
<template>
  <div>
    <router-view></router-view>
  </div>
</template>
```

### 3.2 主布局组件 Layout.vue

`frontend/src/views/Layout.vue` 是**所有需要导航的页面的外壳**，结构如下：

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

- **路由切换监听**（`frontend/src/views/Layout.vue` 第 47–53 行）：每次路由变化清空文件选择、关闭非 success 弹窗
- **条件渲染 Shell**：需同时满足 `enableExec` 常量、已登录、用户有 `execute` 权限

### 3.3 登录页 Login.vue（独立布局）

`frontend/src/views/Login.vue` 不嵌套在 Layout 中，直接渲染登录表单，包含：
- 用户名/密码输入
- 注册模式切换（signup 启用时）
- reCAPTCHA 验证
- 登录成功后跳转到 `query.redirect` 或 `/files/`

### 3.4 文件视图容器 Files.vue

`frontend/src/views/Files.vue` 是 `/files/:path*` 的实际渲染组件，结构：

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

**动态视图选择逻辑**（`frontend/src/views/Files.vue` 第 65–86 行）：

```
req.type === undefined → null（加载中）
req.isDir === true     → FileListing（目录列表）
extension === .csv     → Preview（表格），?edit=true 时 → Editor
type === text / textImmutable → Editor
其他                   → Preview
```

**数据获取**：监听 `route` 和 `reload`，调用 `fetchData()` 通过 API 获取当前路径的文件资源。

### 3.5 设置视图容器 Settings.vue

`frontend/src/views/Settings.vue` 是 `/settings/*` 的容器，含**二级子导航**：

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

`frontend/src/views/Share.vue` 渲染公开共享链接，结构类似 Files.vue 但更简单，不包含操作菜单，只有下载/预览功能和密码保护输入。

---

## 四、核心交互机制详解

### 4.1 面包屑：由当前路径生成层级

`frontend/src/components/Breadcrumbs.vue` 通过 **响应式计算属性** 从 `route.path` 自动生成面包屑层级。

#### 4.1.1 核心计算逻辑

`items` 计算属性（`frontend/src/components/Breadcrumbs.vue` 第 35–72 行）的生成流程：

```
route.path (如 /files/documents/work/report/)
  ↓ 1. 去除 base 前缀
relativePath = /documents/work/report/
  ↓ 2. 按 "/" 分割为数组
parts = ["", "documents", "work", "report", ""]
  ↓ 3. 移除首尾空字符串
parts = ["documents", "work", "report"]
  ↓ 4. 遍历构建面包屑层级
  - 第 0 级: { name: "documents", url: "/files/documents/" }
  - 第 1 级: { name: "work",      url: "/files/documents/work/" }
  - 第 2 级: { name: "report",    url: "/files/documents/work/report/" }
  ↓ 5. 超过 3 层时截断（保留最后 3 层，第 1 层显示为 "..."）
  - 若层级 > 3，循环 shift() 直到只剩 4 项
  - 第 0 项 name 替换为 "..."
```

#### 4.1.2 关键设计要点

| 特性 | 实现方式 | 作用 |
|------|----------|------|
| **Base 路径** | `props.base` 作为根路径前缀 | 支持在 Files (`/files`) 和 Share (`/share/:hash`) 等不同场景复用 |
| **自动解码** | `decodeURIComponent(parts[i])` | 正确显示中文或特殊字符文件名 |
| **层级截断** | 超过 3 层时首项显示为 "..." | 避免深层级目录面包屑过长溢出 |
| **无链接模式** | `noLink` prop 控制渲染 `span` 或 `router-link` | 用于只读展示场景（如编辑器顶部） |
| **首页图标** | 首项为 home 图标，链接到 `base` | 提供快速返回根目录的入口 |

#### 4.1.3 与路由的响应式联动

由于 `items` 是 computed 属性，依赖 `route.path`，因此**路由变化时面包屑会自动更新**，无需手动监听。

### 4.2 顶部菜单与侧栏：通过弹窗栈联动

顶部菜单栏（HeaderBar）和侧边栏（Sidebar）通过 **layoutStore 的弹窗栈** 实现显示/隐藏联动，二者共享同一套状态机制。

#### 4.2.1 弹窗栈核心状态

`frontend/src/stores/layout.ts` 中的 `prompts` 数组是弹窗栈的核心：

```ts
state: () => ({
  prompts: PopupProps[],  // 弹窗栈，数组末尾为栈顶
  loading: boolean,
  showShell: boolean,
})

getters: {
  currentPrompt() { return prompts[prompts.length - 1] },  // 取栈顶
  currentPromptName() { return this.currentPrompt?.prompt }, // 栈顶弹窗名
}

actions: {
  showHover(value) { prompts.push(value) },   // 压栈
  closeHovers() { prompts.pop()?.close?.() }, // 弹栈并调用 close 回调
}
```

#### 4.2.2 侧栏打开/关闭联动

**打开侧栏（HeaderBar → Sidebar）**：

1. HeaderBar 的菜单按钮点击 → 调用 `layoutStore.showHover('sidebar')`（`frontend/src/components/header/HeaderBar.vue` 第 8–10 行）
2. `showHover('sidebar')` 向 `prompts` 栈压入 `{ prompt: "sidebar", ... }`
3. Sidebar 组件通过 `currentPromptName === 'sidebar'` 判断是否激活 → 添加 `active` 类（`frontend/src/components/Sidebar.vue` 第 155–157 行）
4. CSS 过渡动画使侧边栏从左侧滑入

**关闭侧栏（多种触发方式）**：

| 触发方式 | 实现位置 | 效果 |
|----------|----------|------|
| 点击遮罩层 | `frontend/src/components/Sidebar.vue` 第 2 行 | 点击 `.overlay` → `closeHovers()` |
| 点击菜单项跳转 | `frontend/src/components/Sidebar.vue` 第 193–201 行 | `toRoot/toAccountSettings` 等方法内调用 `closeHovers()` |
| 路由切换 | `frontend/src/views/Layout.vue` 第 47–53 行 | Layout 的 `watch(route)` → `closeHovers()`（非 success 弹窗） |

#### 4.2.3 顶部"更多"菜单的同一机制

HeaderBar 右侧的 `more_vert` 按钮也使用弹窗栈机制（`frontend/src/components/header/HeaderBar.vue` 第 14–33 行）：

```
点击 more 按钮
  → layoutStore.showHover('more')
  → #dropdown 元素添加 active 类（显示下拉菜单）
  → .overlay 显示，点击遮罩调用 closeHovers() 关闭
```

#### 4.2.4 Action 组件的 show 属性

`frontend/src/components/header/Action.vue` 封装了按钮 + 弹窗触发的通用逻辑：

```ts
const action = () => {
  if (props.show) {
    layoutStore.showHover(props.show);  // 自动触发弹窗
  }
  emit("action");
};
```

这样使用方只需传入 `show="share"` 或 `show="rename"` 等属性，按钮点击就会自动打开对应弹窗，无需重复写触发逻辑。

#### 4.2.5 弹窗栈与路由的关系

弹窗栈**完全独立于路由**状态：
- 弹窗状态不反映在 URL 中，刷新页面弹窗会消失
- 路由切换时 Layout 会主动调用 `closeHovers()` 清理弹窗
- 这是有意的设计选择：弹窗是临时 UI 状态，不应通过路由管理

### 4.3 侧栏跳转与用量刷新：响应路由变化

`frontend/src/components/Sidebar.vue` 既是导航的发起者，也是路由变化的响应者。

#### 4.3.1 侧栏菜单项的跳转行为

各菜单项点击后执行路由跳转并关闭侧栏（`frontend/src/components/Sidebar.vue` 第 191–205 行）：

```ts
toRoot() {
  this.$router.push({ path: "/files" });
  this.closeHovers();   // 跳转后立即关闭侧栏
},

toAccountSettings() {
  this.$router.push({ path: "/settings/profile" });
  this.closeHovers();
},

toGlobalSettings() {
  this.$router.push({ path: "/settings/global" });
  this.closeHovers();
},
```

**设计特点**：
- 跳转 + 关侧栏是原子操作，避免侧栏残留
- 使用 `$router.push` 而非 `router-link`，因为需要在跳转后执行额外逻辑（关闭弹窗）

#### 4.3.2 磁盘用量的路由响应式刷新

侧栏底部显示磁盘使用量，该数据**只在文件页面才获取**，通过 `watch.$route` 实现（`frontend/src/components/Sidebar.vue` 第 208–217 行）：

```
watch: {
  $route: {
    handler(to) {
      if (to.path.includes("/files")) {
        this.fetchUsage();   // 进入 /files 路径时获取用量
      }
    },
    immediate: true,         // 组件挂载时立即执行一次
  }
}
```

**用量获取的细节**：

| 特性 | 实现 | 作用 |
|------|------|------|
| **条件触发** | `to.path.includes("/files")` | 设置页等非文件页面不请求用量数据 |
| **请求取消** | `AbortController` + `abortOngoingFetch()` | 路由快速切换时取消未完成的请求，避免竞态 |
| **展示条件** | `isFiles && !disableUsedPercentage` | 只有文件页面且启用用量统计时才显示进度条 |

#### 4.3.3 菜单项的权限控制

侧栏菜单项根据用户权限动态显示/隐藏（`frontend/src/components/Sidebar.vue` 第 19–51 行）：

```
已登录状态显示：
  ├── 用户名（所有人）
  ├── "我的文件"（所有人）
  ├── 新建文件夹/新建文件（perm.create）
  ├── 设置（perm.admin）
  └── 退出登录（!noAuth && 配置允许注销）

未登录状态显示：
  ├── 登录按钮（!hideLoginButton）
  └── 注册按钮（signup 启用）
```

这些权限条件通过 computed 属性从 `authStore.user` 派生，权限变化时菜单项会自动更新。

### 4.4 路由变化：选择与多选状态的重置

文件选择状态（`fileStore.selected`）和多选模式（`fileStore.multiple`）在路由切换时会被**多级重置**，确保不同页面/目录间选择状态不混淆。

#### 4.4.1 重置触发点汇总

| 层级 | 触发位置 | 重置内容 | 时机 |
|------|----------|----------|------|
| **布局层** | `frontend/src/views/Layout.vue` 第 47–53 行 | `selected = []`, 关闭非 success 弹窗 | 任意路由切换 |
| **视图层** | `frontend/src/views/Files.vue` 第 146–148 行 `fetchData()` | `selected = []`, `multiple = false`, 关闭 hover | 路径变化需重新获取数据时 |
| **视图层** | `frontend/src/views/Share.vue` 第 394–397 行 `fetchData()` | `selected = []`, `multiple = false`, 关闭 hover | 共享路径变化时 |
| **列表层** | `frontend/src/views/files/FileListing.vue` 第 392–394 行 `onBeforeRouteUpdate` | 隐藏右键菜单 | 路由更新前 |
| **数据层** | `frontend/src/stores/file.ts` 第 41–54 行 `updateRequest()` | `selected = []` 后尝试恢复同名项 | 请求数据更新时 |

#### 4.4.2 Layout 层：全局路由监听重置

最顶层的重置在 Layout.vue 的 `watch(route)` 中（`frontend/src/views/Layout.vue` 第 47–53 行）：

```ts
watch(route, () => {
  fileStore.selected = [];           // 清空所有选中项
  fileStore.multiple = false;        // 退出多选模式
  if (layoutStore.currentPromptName !== "success") {
    layoutStore.closeHovers();       // 关闭非成功弹窗
  }
});
```

**作用范围**：所有嵌套在 Layout 下的路由（`/files/*`、`/settings/*`、`/share/*`）切换时都会触发。

**注意**：`success` 弹窗不关闭，用于操作成功后的提示延续（如复制/移动成功提示）。

#### 4.4.3 Files 视图层：数据获取时重置

Files.vue 的 `fetchData()` 函数在每次获取数据前重置选择状态（`frontend/src/views/Files.vue` 第 143–149 行）：

```ts
const fetchData = async () => {
  fileStore.reload = false;
  fileStore.selected = [];     // 清空选择
  fileStore.multiple = false;  // 退出多选
  layoutStore.closeHovers();   // 关闭弹窗
  // ... 发起 API 请求
};
```

触发时机包括：
- `watch(route)` 监听到路由变化
- `watch(reload)` 监听到 `fileStore.reload = true`（操作后刷新）
- 组件 `onMounted` 首次挂载

#### 4.4.4 File Store 层：数据更新时的选择保留

`frontend/src/stores/file.ts` 的 `updateRequest` 方法有一个巧妙的设计：**先清空再尝试按 URL 恢复选择**（`frontend/src/stores/file.ts` 第 41–54 行）：

```ts
updateRequest(value: Resource | null) {
  const selectedItems = this.selected.map((i) => this.req?.items[i]);
  this.oldReq = this.req;
  this.req = value;

  this.selected = [];  // 先清空

  if (!this.req?.items) return;
  // 尝试按 URL 匹配恢复之前选中的项
  this.selected = this.req.items
    .filter((item) => selectedItems.some((rItem) => rItem?.url === item.url))
    .map((item) => item.index);
}
```

**应用场景**：刷新目录（如上传、删除文件后）时，相同文件名的文件会保持选中状态，提升操作连贯性。

#### 4.4.5 目录切换后的预选（Preselection）

Files.vue 的 `applyPreSelection` 函数（`frontend/src/views/Files.vue` 第 117–141 行）在目录切换后会预选一个文件，提供两种预选逻辑：

```
场景一：有 preselect 目标（操作后指定选中）
  → fileStore.preselect 不为 null
  → 在 req.items 中查找 path 匹配的项
  → 将该项加入 selected

场景二：从子目录返回父目录（oldReq 是子目录路径）
  → fileStore.oldReq.path 以 req.path 开头
  → 提取 oldReq 路径中紧接的下一级目录名
  → 在 req.items 中查找并选中该项（高亮刚离开的子目录）
```

这个机制让用户在目录间导航时能保持视觉连续性，知道自己从哪里来。

#### 4.4.6 FileListing 内的右键菜单重置

`frontend/src/views/files/FileListing.vue` 使用 `onBeforeRouteUpdate` 钩子在路由更新前隐藏右键菜单（`frontend/src/views/files/FileListing.vue` 第 392–394 行）：

```ts
onBeforeRouteUpdate(() => {
  hideContextMenu();
});
```

这是更细粒度的重置，确保路由变化前上下文菜单被正确收起。

### 4.5 编辑器未保存内容拦截路由更新

`frontend/src/views/files/Editor.vue` 实现了**两道拦截**：组件内路由守卫拦截 Vue Router 的同组件路径切换，浏览器事件拦截页面关闭/刷新。

#### 4.5.1 拦截入口一：`onBeforeRouteUpdate`

当用户在编辑器中打开了另一个文件（同组件内路由更新），`onBeforeRouteUpdate` 守卫会检查编辑器是否有未保存的修改（`frontend/src/views/files/Editor.vue` 第 201–219 行）：

```ts
onBeforeRouteUpdate((to, from, next) => {
  if (editor.value?.session.getUndoManager().isClean()) {
    next();          // 无修改，直接放行
    return;
  }

  layoutStore.showHover({     // 有修改，弹出确认对话框
    prompt: "discardEditorChanges",
    confirm: (event: Event) => {
      event.preventDefault();
      next();                 // 用户选择"放弃修改"→ 放行
    },
    saveAction: async () => {
      await save();           // 用户选择"保存修改"→ 先保存再放行
      next();
    },
  });
});
```

**判断依据**：Ace Editor 的 `UndoManager.isClean()` 方法——当 `markClean()` 被调用（保存成功后）之后，`isClean()` 返回 `true`，表示没有未保存的修改。

**弹窗交互**：`discardEditorChanges` 弹窗（`frontend/src/components/prompts/DiscardEditorChanges.vue`）提供三个按钮：

| 按钮 | 触发回调 | 效果 |
|------|----------|------|
| **取消** | `closeHovers()` | 关闭弹窗，路由更新被阻止，用户留在当前编辑状态 |
| **保存修改** | `currentPrompt.saveAction()` | 调用 `save()` 保存文件 → `markClean()` → `next()` 放行 |
| **放弃修改** | `currentPrompt.confirm()` | 直接 `next()` 放行，不保存 |

**关键点**：`next()` 是 Vue Router 传入的回调，只有在用户做出选择后才调用。如果不调用 `next()`，路由更新就会被阻止——这正是拦截的核心机制。

#### 4.5.2 拦截入口二：`beforeunload` 浏览器事件

当用户关闭浏览器标签页或刷新页面时，`beforeunload` 事件提供浏览器原生拦截（`frontend/src/views/files/Editor.vue` 第 260–267 行）：

```ts
const handlePageChange = (event: BeforeUnloadEvent) => {
  if (!editor.value?.session.getUndoManager().isClean()) {
    event.preventDefault();
    event.returnValue = true;  // 兼容旧浏览器
  }
};
```

这个事件在 `onMounted` 中注册（`frontend/src/views/files/Editor.vue` 第 158–159 行），在 `onBeforeUnmount` 中移除（`frontend/src/views/files/Files.vue` 第 196–197 行），确保只在编辑器存活期间生效。

**与 Vue Router 拦截的区别**：
- `beforeunload`：浏览器级拦截，只能弹出浏览器原生确认框，无法自定义 UI
- `onBeforeRouteUpdate`：Vue Router 级拦截，可以使用自定义弹窗（DiscardEditorChanges）

#### 4.5.3 关闭按钮的独立拦截

编辑器的关闭按钮（`<action icon="close" @action="close()" />`）也有独立的未保存检查（`frontend/src/views/files/Editor.vue` 第 298–317 行）：

```ts
const close = () => {
  if (!editor.value?.session.getUndoManager().isClean()) {
    layoutStore.showHover({
      prompt: "discardEditorChanges",
      confirm: (event: Event) => {
        event.preventDefault();
        editor.value?.session.getUndoManager().reset();  // 重置 undo 栈
        finishClose();
      },
      saveAction: async () => {
        try { await save(true); finishClose(); }
        catch {}
      },
    });
    return;
  }
  finishClose();
};

const finishClose = () => {
  const uri = url.removeLastDir(route.path) + "/";
  router.push({ path: uri });  // 跳转到父目录
};
```

**与 `onBeforeRouteUpdate` 的区别**：关闭按钮的"放弃修改"回调会额外调用 `UndoManager.reset()`，彻底清空撤销历史，而路由守卫中的 `confirm` 只调用 `next()` 直接放行。这是因为关闭按钮主动退出编辑器时需要重置状态，而路由守卫放行后编辑器组件会被复用并重新初始化。

#### 4.5.4 完整拦截流程图

```
用户触发离开编辑器的操作
  │
  ├── 点击关闭按钮 → close()
  │     ├── isClean() → finishClose() → router.push(父目录)
  │     └── !isClean() → DiscardEditorChanges 弹窗
  │           ├── 取消 → closeHovers()（留在编辑器）
  │           ├── 保存 → save() + markClean() + finishClose()
  │           └── 放弃 → reset() + finishClose()
  │
  ├── 在同组件内切换文件（如面包屑导航）
  │     └── onBeforeRouteUpdate
  │           ├── isClean() → next()（直接放行）
  │           └── !isClean() → DiscardEditorChanges 弹窗
  │                 ├── 取消 → closeHovers()（路由不更新）
  │                 ├── 保存 → save() + markClean() + next()
  │                 └── 放弃 → next()
  │
  └── 关闭/刷新浏览器标签
        └── beforeunload 事件
              ├── isClean() → 无拦截
              └── !isClean() → 浏览器原生确认框
```

### 4.6 文件页离开时的清理：键盘监听与请求状态

文件浏览涉及的组件在离开时会**主动清理**事件监听器和网络请求，防止内存泄漏和幽灵回调。

#### 4.6.1 Files.vue：请求中断与状态重置

`frontend/src/views/Files.vue` 在组件卸载时执行三步清理（第 95–106 行）：

```ts
onUnmounted(() => {
  fileStore.isFiles = false;             // 1. 重置文件页标识
  if (layoutStore.showShell) {
    layoutStore.toggleShell();           // 2. 关闭 Shell 终端
  }
  fileStore.updateRequest(null);         // 3. 清空当前文件资源
  fetchDataController.abort();           // 4. 中止进行中的 API 请求
});
```

**请求中止机制**（`fetchDataController`）：

```ts
let fetchDataController = new AbortController();  // 组件级控制器

const fetchData = async () => {
  fetchDataController.abort();                     // 中止上次请求
  fetchDataController = new AbortController();     // 创建新控制器
  try {
    const res = await api.fetch(url, fetchDataController.signal);
    // ...
  } catch (err) {
    if (err instanceof StatusError && err.is_canceled) {
      return;   // 主动取消的请求不视为错误
    }
  }
};
```

**双重保障**：
1. 路由切换时 `fetchData()` 被重新调用，会 `abort()` 上次请求
2. 组件卸载时 `onUnmounted` 再次 `abort()`，确保离开页面后不会有残留请求

**键盘监听**：Files.vue 在 `onMounted` 注册 `keyEvent`（F1 打开帮助），在 `onBeforeUnmount` 中移除（第 95–97 行）。

#### 4.6.2 Editor.vue：键盘监听 + 浏览器事件 + Ace 销毁

`frontend/src/views/files/Editor.vue` 在组件卸载时清理三个资源（第 195–199 行）：

```ts
onBeforeUnmount(() => {
  window.removeEventListener("keydown", keyEvent);           // 1. 移除快捷键监听（Ctrl+S 保存、Esc 关闭）
  window.removeEventListener("beforeunload", handlePageChange); // 2. 移除浏览器关闭拦截
  editor.value?.destroy();                                    // 3. 销毁 Ace Editor 实例
});
```

**如果不清理会发生什么**：
- `keydown` 监听残留：离开编辑器后按 Ctrl+S 仍会尝试调用已销毁的 `editor.value` 方法
- `beforeunload` 监听残留：在其他页面关闭浏览器时仍弹出"未保存修改"提示
- Ace Editor 实例残留：DOM 节点和定时器不被释放，导致内存泄漏

#### 4.6.3 Preview.vue：键盘监听清理

`frontend/src/views/files/Preview.vue` 同样注册了键盘监听（方向键切换前后文件、Esc 关闭），在卸载时清理（第 344 行）：

```ts
onMounted(async () => {
  window.addEventListener("keydown", key);
  // ...
});

onBeforeUnmount(() => window.removeEventListener("keydown", key));
```

Preview 的 `key` 事件处理函数还会检查弹窗状态（第 383–384 行）：

```ts
const key = (event: KeyboardEvent) => {
  if (layoutStore.currentPrompt !== null) {
    return;   // 有弹窗时不响应快捷键，避免冲突
  }
  // ...
};
```

#### 4.6.4 FileListing.vue：多重事件监听器的清理

`frontend/src/views/files/FileListing.vue` 注册了大量事件监听器，卸载时逐一清理（第 538–549 行）：

```ts
onBeforeUnmount(() => {
  window.removeEventListener("keydown", keyEvent);     // 快捷键（Delete/F2/Ctrl+A 等）
  window.removeEventListener("scroll", scrollEvent);   // 无限滚动加载
  window.removeEventListener("resize", windowsResize); // 窗口大小变化

  if (authStore.user && !authStore.user?.perm.create) return;
  document.removeEventListener("dragover", preventDefault); // 拖放
  document.removeEventListener("dragenter", dragEnter);
  document.removeEventListener("dragleave", dragLeave);
  document.removeEventListener("drop", drop);
});
```

#### 4.6.5 Share.vue：请求清理

`frontend/src/views/Share.vue` 的 `fetchData` 函数同样使用 `AbortController` 来管理请求生命周期。虽然 Share 页面不像 Files 那样在组件卸载时显式调用 `abort()`，但路由切换时 Layout 的 `watch(route)` 会先关闭弹窗，且 Share 组件卸载后回调即使触发也不会有副作用（因为组件已销毁，状态更新无效）。

#### 4.6.6 各组件清理对照表

| 组件 | 清理内容 | 清理时机 |
|------|----------|----------|
| **Files.vue** | `keydown` 监听、`isFiles` 重置、Shell 关闭、`fileStore.req` 清空、`AbortController` 中止 | `onBeforeUnmount` + `onUnmounted` |
| **Editor.vue** | `keydown` 监听、`beforeunload` 监听、Ace Editor 实例销毁 | `onBeforeUnmount` |
| **Preview.vue** | `keydown` 监听 | `onBeforeUnmount` |
| **FileListing.vue** | `keydown` + `scroll` + `resize` + 拖放系列监听 | `onBeforeUnmount` |
| **Sidebar.vue** | `fetchUsage` 的 `AbortController` | `unmounted` |

---

## 五、状态管理与路由布局的配合

### 5.1 Pinia 注入 Router

在 `frontend/src/stores/index.ts` 中，Pinia 通过插件把 `router` 实例注入到每个 store：

```ts
pinia.use(({ store }) => {
  store.router = markRaw(router);
});
```

这样所有 store 都可以通过 `this.router` 访问路由实例（当前代码中主要为预留能力）。

### 5.2 Layout Store 控制全局 UI

`frontend/src/stores/layout.ts` 管理跨页面共享的 UI 状态：

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

### 5.3 Auth Store 控制路由守卫

`frontend/src/stores/auth.ts` 的 `isLoggedIn` getter 和 `user.perm.admin` 是路由守卫判断的核心依据。

### 5.4 File Store 驱动视图切换

`frontend/src/stores/file.ts` 的 `req` 字段存储当前请求的文件资源对象。Files.vue 的 `currentView` computed 完全依赖 `req` 的属性（`isDir`、`type`、`extension`）决定渲染哪个子视图。

---

## 六、路由切换的完整数据流

### 6.1 场景一：用户从登录页进入文件浏览

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
  ↓
Sidebar 监听到路由包含 /files
  └── fetchUsage() 获取磁盘用量并显示
```

### 6.2 场景二：在文件列表中点击子目录

```
用户点击目录 /documents
  ↓
ListingItem.open() → router.push("/files/documents")
  ↓
beforeResolve（from.name 非空，跳过 initAuth）
  ↓
Layout.vue watch(route)
  ├── fileStore.selected = []（清空选择）
  ├── fileStore.multiple = false（退出多选）
  └── layoutStore.closeHovers()（关闭弹窗）
  ↓
FileListing.vue onBeforeRouteUpdate → hideContextMenu()
  ↓
Files.vue watch(route) → fetchData()
  ├── 中止上次请求（AbortController）
  ├── fileStore.selected = []
  ├── fileStore.multiple = false
  ├── 请求 /api/resources/documents
  └── fileStore.updateRequest() 更新 req + 预选旧目录
  ↓
currentView 仍为 FileListing.vue，但数据已更新
  ↓
Breadcrumbs 自动计算：/files/ + documents → 更新面包屑
  ↓
Sidebar 监听到路由仍在 /files 下
  └── abortOngoingFetchUsage() + fetchUsage()（重新获取用量）
```

### 6.3 场景三：从文件列表进入设置页

```
Sidebar 点击 "设置" → router.push("/settings/global")
  ↓
Sidebar.toGlobalSettings()
  ├── $router.push({ path: "/settings/global" })
  └── closeHovers()（关闭侧栏）
  ↓
Layout.vue watch(route)
  ├── fileStore.selected = []
  ├── fileStore.multiple = false
  └── layoutStore.closeHovers()
  ↓
Settings.vue 渲染
  ↓
Settings 内部导航高亮 "全局设置"（通过 $route.path 判断 active 类）
  ↓
Settings > router-view → GlobalSettings.vue
  ↓
Sidebar 监听到路由不含 /files
  └── 不请求磁盘用量，用量区域不显示
```

### 6.4 场景四：未登录用户访问受保护路由

```
直接访问 /settings/profile
  ↓
beforeResolve 守卫
  ├── initAuth() 首次初始化（检查 cookie/token）
  ├── authStore.isLoggedIn 为 false
  └── next({ path: "/login", query: { redirect: "/settings/profile" } })
  ↓
Login.vue 渲染（独立布局，无 Sidebar）
  ↓
用户登录成功 → router.push(query.redirect || "/files/")
  ↓
回到场景一的流程
```

### 6.5 场景五：侧栏开闭交互

```
点击 HeaderBar 菜单按钮
  ↓
Action @action → layoutStore.showHover('sidebar')
  ↓
prompts 栈压入 { prompt: "sidebar" }
  ↓
currentPromptName = "sidebar"
  ↓
Sidebar: active = true → 侧边栏滑入 + 遮罩显示
  ↓
（用户点击遮罩 / 点击菜单项 / 路由切换）
  ↓
layoutStore.closeHovers()
  ↓
prompts 栈弹出 → currentPromptName 变化
  ↓
Sidebar: active = false → 侧边栏滑出
```

### 6.6 场景六：编辑器中未保存修改时切换文件

```
用户在编辑 /files/notes/readme.md 时通过面包屑点击 /files/notes/config.json
  ↓
onBeforeRouteUpdate 触发
  ├── editor.session.getUndoManager().isClean() → false（有修改）
  └── layoutStore.showHover({ prompt: "discardEditorChanges" })
  ↓
DiscardEditorChanges 弹窗显示
  ├── 用户点"取消" → closeHovers()，next() 不被调用 → 路由不更新
  ├── 用户点"保存" → save() → markClean() → next() → 路由更新，编辑器重新初始化
  └── 用户点"放弃" → next() → 路由更新，编辑器重新初始化（修改丢失）
```

---

## 七、关键设计模式总结

### 7.1 嵌套路由 + 嵌套布局

整个应用通过三层嵌套 `router-view` 实现布局复用：

| 层级 | 组件 | 职责 |
|------|------|------|
| 第一层 | App.vue | 主题/语言初始化，无布局 |
| 第二层 | Layout.vue | 全局框架（Sidebar、Prompts、Shell、上传进度） |
| 第三层 | Files.vue / Settings.vue / Share.vue | 页面容器（导航栏、面包屑、子视图） |
| 第四层 | FileListing / Editor / Preview / ProfileSettings... | 具体内容 |

### 7.2 动态组件 + Pinia 驱动视图

Files.vue 不使用子路由区分文件/目录/编辑器，而是通过 `fileStore.req` 的元数据动态 `<component :is="currentView">`，实现了**单路由多视图**的模式，避免了路由过度碎片化。

### 7.3 弹窗栈（Prompt Stack）

全局弹窗（包括 Sidebar、more 菜单、各种对话框）不使用路由控制，而是通过 `layoutStore.prompts` 数组作为栈管理，Prompts.vue 作为统一容器渲染栈顶弹窗。优势：
- 弹窗状态与 URL 解耦，刷新不残留
- 支持弹窗叠加（栈结构）
- 统一的开关、回调机制
- HeaderBar 和 Sidebar 通过同一状态联动

### 7.4 路由守卫集中控制权限

所有权限检查集中在 `beforeResolve` 单一守卫中，通过 `meta.requiresAuth` 和 `meta.requiresAdmin` 声明式配置，避免每个组件重复鉴权逻辑。

### 7.5 响应式路由监听驱动数据

Files.vue、Share.vue、Sidebar.vue 均使用 `watch(route)` 触发数据重新获取，Layout.vue 用 watch(route) 做全局 UI 重置，确保路由变化时各组件能正确响应。

### 7.6 多级状态重置保障一致性

文件选择状态通过**布局层 → 视图层 → 数据层 → 列表层**四级重置，配合数据更新时的智能恢复，既保证了跨页面选择状态不混淆，又在同目录刷新时保留用户选择。

### 7.7 面包屑的纯计算属性设计

面包屑完全通过 `computed` 从 `route.path` 派生，无需任何手动同步逻辑，天然响应式且无副作用。

### 7.8 双层编辑器拦截（路由级 + 浏览器级）

编辑器通过 `onBeforeRouteUpdate` 拦截 Vue Router 导航（自定义弹窗），通过 `beforeunload` 拦截浏览器关闭/刷新（原生确认框），两层拦截互补，确保未保存修改在任何离开场景下都不会静默丢失。

### 7.9 组件卸载时的完备清理

每个注册了 `window.addEventListener` 的组件都在 `onBeforeUnmount` / `onUnmounted` 中移除监听；每个发起网络请求的组件都使用 `AbortController` 在路由切换或组件卸载时中止请求。这避免了内存泄漏、幽灵回调和请求竞态。

---

## 八、核心文件速查

| 文件 | 作用 |
|------|------|
| `frontend/src/router/index.ts` | 路由定义 + 全局守卫 |
| `frontend/src/views/Layout.vue` | 主布局框架（Sidebar + Main + Prompts） |
| `frontend/src/views/Files.vue` | 文件浏览容器，动态切换 Listing/Editor/Preview，管理请求生命周期 |
| `frontend/src/views/Settings.vue` | 设置页容器 + 二级导航 |
| `frontend/src/views/files/Editor.vue` | 文件编辑器，未保存拦截（onBeforeRouteUpdate + beforeunload），快捷键与 Ace 销毁 |
| `frontend/src/views/files/Preview.vue` | 文件预览，前后切换，键盘导航与监听清理 |
| `frontend/src/views/files/FileListing.vue` | 文件列表，选择/多选/右键菜单/拖放，多重事件监听清理 |
| `frontend/src/components/Breadcrumbs.vue` | 面包屑路径生成 |
| `frontend/src/components/Sidebar.vue` | 侧边栏导航，根据权限显示菜单项，响应路由刷新用量 |
| `frontend/src/components/header/HeaderBar.vue` | 顶部操作栏，菜单按钮触发侧栏 |
| `frontend/src/components/header/Action.vue` | 通用按钮组件，支持 show 属性触发弹窗 |
| `frontend/src/components/prompts/Prompts.vue` | 全局弹窗渲染容器 |
| `frontend/src/components/prompts/DiscardEditorChanges.vue` | 编辑器未保存修改确认弹窗 |
| `frontend/src/stores/layout.ts` | 全局 UI 状态（弹窗栈、加载、Shell） |
| `frontend/src/stores/file.ts` | 当前文件资源状态，驱动视图切换，管理选择状态 |
| `frontend/src/stores/auth.ts` | 用户认证状态，驱动权限守卫 |
