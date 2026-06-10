# File Browser 国际化（i18n）与本地化分析文档

## 一、概述

File Browser 采用前后端分离架构：
- **前端**：Vue 3 + vue-i18n 11 实现界面国际化
- **后端**：Go 语言存储用户语言偏好设置
- **日期本地化**：dayjs 库处理日期格式本地化
- **RTL 支持**：内置希伯来语（he）和阿拉伯语（ar）的从右到左布局支持

---

## 二、技术栈与依赖

### 2.1 核心依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| `vue-i18n` | ^11.1.10 | Vue 3 国际化核心库 |
| `@intlify/unplugin-vue-i18n` | ^11.0.1 | Vite 构建时自动导入语言 JSON 资源 |
| `dayjs` | ^1.11.13 | 日期处理与本地化 |

定义位置：[package.json](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/package.json#L20-L76)

---

## 三、语言资源加载流程

### 3.1 构建时配置（Vite 插件层）

**文件**：[vite.config.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/vite.config.ts#L1-L81)

```typescript
import VueI18nPlugin from "@intlify/unplugin-vue-i18n/vite";

VueI18nPlugin({
  include: [path.resolve(__dirname, "./src/i18n/**/*.json")],
})
```

**作用**：
- 该插件在构建时扫描 `src/i18n/` 目录下所有 JSON 文件
- 将它们打包为虚拟模块 `@intlify/unplugin-vue-i18n/messages`
- 所有语言资源**一次性**全部加载（无按需懒加载）

**构建优化**：在 `build` 模式下，i18n 相关模块被单独拆分为 `i18n` chunk，dayjs 相关模块拆分为 `dayjs` chunk：

```typescript
manualChunks: (id) => {
  if (id.includes("dayjs/")) {
    return "dayjs";
  } else if (id.includes("i18n/")) {
    return "i18n";
  }
}
```

### 3.2 语言资源文件

**目录**：[frontend/src/i18n/](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n)

支持的语言文件（共 34 种）：

| 语言代码 | 语言名称 |
|---------|---------|
| ar | العربية（阿拉伯语） |
| bg | български език（保加利亚语） |
| ca | Català（加泰罗尼亚语） |
| cs | Čeština（捷克语） |
| de | Deutsch（德语） |
| el | Ελληνικά（希腊语） |
| en | English（英语，默认） |
| es | Español（西班牙语） |
| fa | فارسی（波斯语） |
| fr | Français（法语） |
| he | עברית（希伯来语） |
| hr | Hrvatski（克罗地亚语） |
| hu | Magyar（匈牙利语） |
| is | Icelandic（冰岛语） |
| it | Italiano（意大利语） |
| ja | 日本語（日语） |
| ko | 한국어（韩语） |
| lv | Latviešu（拉脱维亚语） |
| nl | Nederlands（荷兰语） |
| nl-be | Nederlands (België)（荷兰语-比利时） |
| no | Norsk（挪威语） |
| pl | Polski（波兰语） |
| pt | Português（葡萄牙语） |
| pt-br | Português (Brasil)（葡萄牙语-巴西） |
| pt-pt | Português (Portugal)（葡萄牙语-葡萄牙） |
| ro | Romanian（罗马尼亚语） |
| ru | Русский（俄语） |
| sk | Slovenčina（斯洛伐克语） |
| sv-se | Swedish (Sweden)（瑞典语） |
| tr | Türkçe（土耳其语） |
| uk | Українська（乌克兰语） |
| vi | Tiếng Việt（越南语） |
| zh-cn | 中文 (简体) |
| zh-tw | 中文 (繁體) |

**资源格式示例**（[en.json](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/en.json#L1-L100)）：

```json
{
  "buttons": {
    "cancel": "Cancel",
    "update": "Update",
    "share": "Share"
  },
  "settings": {
    "language": "Language",
    "profileSettings": "Profile Settings"
  }
}
```

### 3.3 i18n 实例初始化

**文件**：[frontend/src/i18n/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L1-L195)

#### 3.3.1 dayjs 语言包预加载

```typescript
import("dayjs/locale/ar");
import("dayjs/locale/bg");
import("dayjs/locale/zh-cn");
import("dayjs/locale/zh-tw");
// ... 共约 33 种 dayjs locale
```

采用动态 `import()` 语法，确保 dayjs 所有语言包在初始化时可用。

#### 3.3.2 浏览器语言自动检测 `detectLocale()`

**核心逻辑**：从 `navigator.language` 读取 RFC 5646 语言标签，通过正则匹配映射到系统支持的语言代码。

```typescript
export function detectLocale() {
  let locale = navigator.language.toLowerCase();
  switch (true) {
    case /^ar\b/.test(locale):    locale = "ar";    break;
    case /^zh-tw\b/.test(locale): locale = "zh-tw"; break;
    case /^zh-cn\b/.test(locale):
    case /^zh\b/.test(locale):    locale = "zh-cn"; break;
    case /^pt-br\b/.test(locale): locale = "pt-br"; break;
    case /^pt-pt\b/.test(locale):
    case /^pt\b/.test(locale):    locale = "pt-pt"; break;
    // ... 更多语言分支
    default: locale = "en";  // 默认回退到英语
  }
  return locale;
}
```

**特殊映射规则**：
- `zh` / `zh-cn` → `zh-cn`（简体中文）
- `zh-tw` → `zh-tw`（繁体中文）
- `pt` / `pt-pt` → `pt-pt`（葡萄牙语-葡萄牙）
- `pt-br` → `pt-br`（葡萄牙语-巴西）
- `sv` / `sv-se` → `sv`（瑞典语）
- `nb` / `no` → `no`（挪威语）
- 其他未匹配 → `en`（英语）

#### 3.3.3 创建 vue-i18n 实例

```typescript
export const i18n = createI18n({
  locale: detectLocale(),      // 初始 locale 使用浏览器检测结果
  fallbackLocale: "en",         // 翻译缺失时回退到英语
  messages,                     // 从虚拟模块导入的所有语言资源
  legacy: true,                 // 启用 legacy 模式，暴露 i18n.global 供非组件使用
});
```

**关键配置说明**：
- `legacy: true`：允许在组件外部通过 `i18n.global.t()` 访问翻译函数（用于 Toast 等非组件场景）
- `fallbackLocale: "en"`：当翻译键在当前语言中不存在时，自动查找英语版本

#### 3.3.4 RTL 语言支持

```typescript
export const rtlLanguages = ["he", "ar"];  // 希伯来语和阿拉伯语

export const isRtl = (locale?: string) => {
  return rtlLanguages.includes(locale || i18n.global.locale.value);
};
```

---

## 四、应用入口注册与初始化

### 4.1 主入口注册

**文件**：[frontend/src/main.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/main.ts#L1-L106)

```typescript
import i18n, { isRtl } from "@/i18n";

app.use(i18n);  // 将 i18n 实例注册到 Vue 应用
```

**全局 Toast 中的使用**：

```typescript
app.provide("$showSuccess", (message: string) => {
  $toast.success(..., { ...toastConfig, rtl: isRtl() });
});

app.provide("$showError", (error: Error | string, displayReport = true) => {
  $toast.error({
    props: {
      reportText: i18n.global.t("buttons.reportIssue"),  // 非组件中使用全局 t()
    }
  }, { ...toastConfig, rtl: isRtl() });
});
```

### 4.2 dayjs 插件注册

```typescript
import dayjs from "dayjs";
import localizedFormat from "dayjs/plugin/localizedFormat";
import relativeTime from "dayjs/plugin/relativeTime";
import duration from "dayjs/plugin/duration";

dayjs.extend(localizedFormat);  // 支持本地化日期格式（如 L, LL, LLL 等）
dayjs.extend(relativeTime);     // 支持相对时间（如 "2 hours ago"）
dayjs.extend(duration);         // 支持时长格式化
```

---

## 五、界面语言切换完整链路

### 5.1 语言切换核心函数

**文件**：[frontend/src/i18n/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L180-L193)

#### 5.1.1 `setLocale(locale)` - 切换应用语言

```typescript
export function setLocale(locale: string) {
  dayjs.locale(locale);                                  // 同步切换 dayjs 语言
  i18n.global.locale.value = locale;                     // 切换 vue-i18n 语言
}
```

**注意**：由于使用 `legacy: true` 模式，`locale` 实际是一个 Ref 对象，需要通过 `.value` 赋值。

#### 5.1.2 `setHtmlLocale(locale)` - 更新 HTML 文档属性

```typescript
export function setHtmlLocale(locale: string) {
  const html = document.documentElement;
  html.lang = locale;                                    // 设置 <html lang="...">
  if (isRtl(locale)) html.dir = "rtl";                   // RTL 语言设置 dir="rtl"
  else html.dir = "ltr";                                 // 其他语言设置 dir="ltr"
}
```

### 5.2 应用启动时的语言初始化链路

#### 5.2.1 App.vue 根组件监听

**文件**：[frontend/src/App.vue](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/App.vue#L1-L33)

```typescript
import { useI18n } from "vue-i18n";
import { setHtmlLocale } from "./i18n";

const { locale } = useI18n();

onMounted(() => {
  setTheme(userTheme.value);
  setHtmlLocale(locale.value);          // 首次挂载时设置 HTML lang/dir
});

watch(locale, (newValue) => {           // 监听 locale 变化，自动更新 HTML 属性
  newValue && setHtmlLocale(newValue);
});
```

#### 5.2.2 路由守卫与认证初始化

**文件**：[frontend/src/router/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/router/index.ts#L156-L222)

**首次访问触发链路**：

```
router.beforeResolve()
  └─ from.name == null (首次路由) → initAuth()
       ├─ 有登录页配置 → validateLogin() (使用本地存储的 JWT 续期)
       └─ 无登录页配置 → login("", "", "") (无认证模式自动登录)
            └─ parseToken()
                 └─ authStore.setUser(data.user)
                      └─ setLocale(user.locale || detectLocale())
```

**路由标题国际化**：

```typescript
const titles = {
  Login: "sidebar.login",
  Files: "files.files",
  Settings: "sidebar.settings",
  // ...
};

router.beforeResolve(async (to, from, next) => {
  const title = i18n.global.t(titles[to.name as keyof typeof titles]);
  document.title = title + " - " + name;  // 动态设置页面标题
});
```

#### 5.2.3 认证流程中的语言设置

**文件**：[frontend/src/utils/auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/utils/auth.ts#L9-L141)

**JWT Token 解析**：

```typescript
export function parseToken(token: string) {
  const data = jwtDecode<JwtPayload & { user: IUser }>(token);
  
  const authStore = useAuthStore();
  authStore.jwt = token;
  authStore.setUser(data.user);   // ← 触发语言设置
  // ...
}
```

**Auth Store 中的语言应用**：

**文件**：[frontend/src/stores/auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/stores/auth.ts#L1-L46)

```typescript
import { defineStore } from "pinia";
import { detectLocale, setLocale } from "@/i18n";

export const useAuthStore = defineStore("auth", {
  actions: {
    setUser(user: IUser) {
      setLocale(user.locale || detectLocale());  // 用户有偏好则用，否则检测浏览器
      this.user = user;
    },
    updateUser(user: Partial<IUser>) {
      if (user.locale) {
        setLocale(user.locale);                  // 更新用户信息时同步切换语言
      }
      this.user = { ...this.user, ...cloneDeep(user) } as IUser;
    },
  },
});
```

---

### 5.3 用户手动切换语言（个人设置页面）

#### 5.3.1 语言选择组件

**文件**：[frontend/src/components/settings/Languages.vue](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/components/settings/Languages.vue#L1-L69)

```html
<select name="selectLanguage" v-on:change="change" :value="locale">
  <option v-for="(language, value) in locales" :key="value" :value="value">
    {{ language }}
  </option>
</select>
```

**组件特点**：
- 使用原生 `<select>` 元素
- `locales` 对象硬编码了 34 种语言的**原生名称**（如 "中文 (简体)"、"日本語"）
- 使用 `markRaw()` 防止 Vue 3 响应式系统破坏对象配置
- 通过 `v-model:locale` 实现双向绑定，使用 `update:locale` 事件

#### 5.3.2 个人设置页面集成

**文件**：[frontend/src/views/settings/Profile.vue](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/views/settings/Profile.vue#L1-L221)

**页面加载时**：

```typescript
onMounted(async () => {
  locale.value = authStore.user.locale;      // 从当前用户数据读取语言偏好
  // ... 其他设置
});
```

**用户保存时**：

```typescript
const updateSettings = async (event: Event) => {
  event.preventDefault();
  const data = {
    ...authStore.user,
    id: authStore.user.id,
    locale: locale.value,                     // 包含新选择的语言
    // ... 其他设置
  };

  await api.update(data, [
    "locale",                                 // 告诉后端只更新这些字段
    "hideDotfiles",
    // ...
  ]);
  authStore.updateUser(data);                 // 更新 store，触发 setLocale()
  $showSuccess(t("settings.settingsUpdated"));
};
```

#### 5.3.3 后端用户更新 API

**文件**：[frontend/src/api/users.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/api/users.ts#L29-L43)

```typescript
export async function update(
  user: Partial<IUser>,
  which = ["all"],
  currentPassword: string | null = null
) {
  await fetchURL(`/api/users/${user.id}`, {
    method: "PUT",
    body: JSON.stringify({
      what: "user",
      which: which,                           // 受控字段列表
      ...(currentPassword != null ? { current_password: currentPassword } : {}),
      data: user,
    }),
  });
}
```

**后端处理**：[http/users.go](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/http/users.go#L180-L269)

```go
var userPutHandler = withSelfOrAdmin(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
  // 1. 解析请求，验证权限
  // 2. 敏感字段修改需要验证当前密码
  // 3. Locale 不在 NonModifiableFieldsForNonAdmin 中，普通用户可自行修改
  err = d.store.Users.Update(req.Data, req.Which...)
  // 4. 更新持久化存储（BoltDB）
})
```

**权限说明**：
- `NonModifiableFieldsForNonAdmin` 包含：Username、Scope、LockPassword、Perm、Commands、Rules
- **Locale 不在此列表中**，因此普通用户可以自由修改自己的语言设置

---

### 5.4 管理员设置用户语言

#### 5.4.1 用户表单组件

**文件**：[frontend/src/components/settings/UserForm.vue](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/components/settings/UserForm.vue#L1-L122)

```html
<p>
  <label for="locale">{{ t("settings.language") }}</label>
  <languages
    class="input input--block"
    id="locale"
    v-model:locale="user.locale"
  ></languages>
</p>
```

该组件同时用于：
1. **创建新用户**（管理员）
2. **编辑现有用户**（管理员）
3. **设置默认用户配置**（全局设置 → 用户默认值）

#### 5.4.2 全局设置中的默认语言

**文件**：[frontend/src/views/settings/Global.vue](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/views/settings/Global.vue#L168-L192)

```html
<form class="card" @submit.prevent="save">
  <div class="card-title">
    <h2>{{ t("settings.userDefaults") }}</h2>
  </div>
  <div class="card-content">
    <user-form
      :isNew="false"
      :isDefault="true"
      v-model:user="settings.defaults"
    />
  </div>
</form>
```

这里设置的 `settings.defaults.locale` 会作为**新创建用户**的默认语言。

---

## 六、后端数据模型与持久化

### 6.1 用户模型中的 Locale 字段

**文件**：[users/users.go](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/users/users.go#L20-L39)

```go
type User struct {
    ID                    uint            `storm:"id,increment" json:"id"`
    Username              string          `storm:"unique" json:"username"`
    Password              string          `json:"password"`
    Scope                 string          `json:"scope"`
    Locale                string          `json:"locale"`  // ← 语言偏好字段
    // ...
}
```

注意：`Locale` 字段**未包含**在 `checkableFields` 中，因此 `Clean()` 方法不会对其进行校验或默认值设置。

### 6.2 用户默认配置

**文件**：[settings/defaults.go](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/settings/defaults.go#L8-L37)

```go
type UserDefaults struct {
    Scope                 string            `json:"scope"`
    Locale                string            `json:"locale"`  // ← 默认语言
    ViewMode              users.ViewMode    `json:"viewMode"`
    // ...
}

func (d *UserDefaults) Apply(u *users.User) {
    u.Locale = d.Locale  // 创建用户时应用默认语言
    // ...
}
```

### 6.3 前端类型定义

**文件**：[frontend/src/types/user.d.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/types/user.d.ts#L1-L18)

```typescript
interface IUser {
  id: number;
  username: string;
  password: string;
  scope: string;
  locale: string;  // ← 类型定义
  // ...
}
```

---

## 七、完整调用链时序图

### 7.1 应用启动与首次语言初始化

```
用户访问页面
    │
    ▼
main.ts 启动
    ├─ app.use(i18n)           // 注册 i18n，初始 locale=detectLocale() 结果
    ├─ dayjs.extend(...)       // 注册 dayjs 插件
    │
    ▼
router.beforeResolve()        // 首次路由，from.name == null
    │
    ├─ initAuth()
    │    ├─ validateLogin() / login()
    │    │    └─ parseToken(jwt)
    │    │         └─ authStore.setUser(data.user)
    │    │              └─ setLocale(user.locale || detectLocale())
    │    │                   ├─ dayjs.locale(locale)
    │    │                   └─ i18n.global.locale.value = locale
    │    │
    ▼
App.vue onMounted()
    └─ setHtmlLocale(locale.value)   // 设置 <html lang="xx" dir="ltr|rtl">
         │
         ▼
App.vue watch(locale)         // 建立长期监听，后续切换都会更新 HTML
```

### 7.2 用户通过设置修改语言

```
用户访问 /settings/profile
    │
    ▼
Profile.vue onMounted()
    └─ locale.value = authStore.user.locale   // 读取当前用户语言到下拉框
    │
    ▼
用户选择新语言 → 点击 Update
    │
    ▼
updateSettings()
    ├─ 构造 data = { ...user, locale: 新值 }
    ├─ api.update(data, ["locale", ...])      // PUT /api/users/:id
    │    └─ 后端：d.store.Users.Update()      // 持久化到 BoltDB
    │
    └─ authStore.updateUser(data)
         └─ setLocale(user.locale)
              ├─ dayjs.locale(locale)         // 日期本地化
              └─ i18n.global.locale.value = locale
                   │
                   ▼
App.vue watch(locale) 触发
    └─ setHtmlLocale(newLocale)
         ├─ html.lang = newLocale
         └─ html.dir = "ltr" / "rtl"
              │
              ▼
所有 Vue 组件重新渲染，t() 函数返回新语言翻译
```

---

## 八、翻译使用方式

### 8.1 组件内使用（Composition API）

```typescript
import { useI18n } from "vue-i18n";

const { t } = useI18n();

// 简单翻译
t("buttons.update")

// 带参数翻译（示例见 en.json 中的 showingRows）
t("files.showingRows", { count: 10 })
```

### 8.2 组件内使用（Options API / 模板）

```html
<h2>{{ t("settings.profileSettings") }}</h2>
<input :value="t('buttons.update')" type="submit" />
```

### 8.3 组件插值翻译

使用 `<i18n-t>` 组件实现带 HTML 或组件的复杂翻译：

```html
<i18n-t
  keypath="settings.brandingHelp"
  tag="p"
  class="small"
  scope="global"
>
  <a class="link" target="_blank" href="...">{{ t("settings.documentation") }}</a>
</i18n-t>
```

### 8.4 非组件代码中使用

```typescript
import i18n from "@/i18n";

// 使用全局实例
i18n.global.t("buttons.reportIssue")
```

---

## 九、RTL（从右到左）适配要点

| 适配位置 | 处理方式 |
|---------|---------|
| `<html>` 元素 | `dir="rtl"` 或 `dir="ltr"`，由 `setHtmlLocale()` 控制 |
| Toast 消息 | 通过 `rtl: isRtl()` 参数传递给 vue-toastification |
| CSS 样式 | 依赖浏览器原生 `dir` 属性的自动布局调整 |

---

## 十、关键文件索引

| 类别 | 文件路径 |
|------|---------|
| i18n 核心逻辑 | [frontend/src/i18n/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts) |
| 语言资源目录 | [frontend/src/i18n/](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n) |
| Vite i18n 配置 | [frontend/vite.config.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/vite.config.ts) |
| 应用入口 | [frontend/src/main.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/main.ts) |
| 根组件（locale watch） | [frontend/src/App.vue](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/App.vue) |
| 路由守卫（初始化） | [frontend/src/router/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/router/index.ts) |
| Auth Store（setLocale 触发点） | [frontend/src/stores/auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/stores/auth.ts) |
| 认证工具（parseToken） | [frontend/src/utils/auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/utils/auth.ts) |
| 语言选择下拉组件 | [frontend/src/components/settings/Languages.vue](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/components/settings/Languages.vue) |
| 个人设置页面 | [frontend/src/views/settings/Profile.vue](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/views/settings/Profile.vue) |
| 全局设置页面 | [frontend/src/views/settings/Global.vue](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/views/settings/Global.vue) |
| 用户表单组件 | [frontend/src/components/settings/UserForm.vue](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/components/settings/UserForm.vue) |
| 用户 API | [frontend/src/api/users.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/api/users.ts) |
| 后端用户模型 | [users/users.go](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/users/users.go) |
| 后端用户默认配置 | [settings/defaults.go](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/settings/defaults.go) |
| 后端用户更新 Handler | [http/users.go](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/http/users.go) |
| 前端用户类型定义 | [frontend/src/types/user.d.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/types/user.d.ts) |

---

## 十一、设计特点总结

1. **全量加载策略**：所有语言资源在构建时打包，应用启动一次性加载，切换时无需网络请求，但首屏体积较大
2. **两级默认机制**：浏览器检测 → 用户个性化设置，后者优先级更高
3. **翻译回退链**：当前语言缺失 → `fallbackLocale: "en"` 回退到英语
4. **持久化存储**：用户语言偏好通过 JWT Token 下发，保存在后端 BoltDB 中
5. **生态协同**：vue-i18n 负责界面翻译，dayjs 负责日期本地化，两者通过 `setLocale()` 统一调度
6. **无动态导入**：dayjs locale 和语言 JSON 均为全量加载，未使用代码分割进行按需加载
