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

---

## 十二、三层架构对应关系全景分析

国际化系统涉及三个独立的"语言代码"来源，它们之间存在不一致甚至冲突。本节逐一拆解。

### 12.1 全量数据对照表（5 个语言链路维度 + 2 个 RTL 样式维度）

**列说明**：第 1 列是语言代码（索引），第 2-6 列是 5 个语言链路维度（资源、检测、下拉、dayjs、RTL列表），第 7-8 列是 2 个 RTL 样式维度，最后 1 列是综合可达性评估。

| 语言代码 | 资源文件(JSON) | detectLocale返回 | Languages下拉key | dayjs加载 | RTL列表 | html[dir]样式(24处) | body.rtl样式(2处) | 完整可达 |
|---------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| ar | ✅ ar.json | ✅ "ar" | ✅ "ar" | ✅ ar | ✅ 是 | ✅ 生效 | ❌ 不匹配（死代码） | ⚠️ 基本正常（2处RTL样式不生效） |
| bg | ✅ bg.json | ✅ "bg" | ✅ "bg" | ✅ bg | ❌（非RTL） | N/A | N/A | ✅ 是 |
| ca | ✅ ca.json | ❌ 无分支→en | ✅ "ca" | ✅ ca | ❌（非RTL） | N/A | N/A | ⚠️ 仅手动，不可自动检测 |
| cs | ✅ cs.json | ✅ "cs" | ✅ "cs" | ✅ cs | ❌（非RTL） | N/A | N/A | ✅ 是 |
| de | ✅ de.json | ✅ "de" | ✅ "de" | ✅ de | ❌（非RTL） | N/A | N/A | ✅ 是 |
| el | ✅ el.json | ✅ "el" | ✅ "el" | ✅ el | ❌（非RTL） | N/A | N/A | ✅ 是 |
| en | ✅ en.json | ✅ "en" | ✅ "en" | ✅ en | ❌（非RTL） | N/A | N/A | ✅ 是 |
| es | ✅ es.json | ✅ "es" | ✅ "es" | ✅ es | ❌（非RTL） | N/A | N/A | ✅ 是 |
| **fa** | ✅ fa.json | ❌ 无分支→en | ❌ 不存在 | ❌ 未加载 | ❌ 不在列表 | ❌ dir=ltr（应为rtl） | ❌ 不匹配 | ❌ 完全不可达 |
| fr | ✅ fr.json | ✅ "fr" | ✅ "fr" | ✅ fr | ❌（非RTL） | N/A | N/A | ✅ 是 |
| he | ✅ he.json | ✅ "he" | ✅ "he" | ✅ he | ✅ 是 | ✅ 生效 | ❌ 不匹配（死代码） | ⚠️ 基本正常（2处RTL样式不生效） |
| hr | ✅ hr.json | ✅ "hr" | ✅ "hr" | ✅ hr | ❌（非RTL） | N/A | N/A | ✅ 是 |
| hu | ✅ hu.json | ✅ "hu" | ✅ "hu" | ✅ hu | ❌（非RTL） | N/A | N/A | ✅ 是 |
| is | ✅ is.json | ✅ "is" | ✅ "is" | ✅ is | ❌（非RTL） | N/A | N/A | ✅ 是 |
| it | ✅ it.json | ✅ "it" | ✅ "it" | ✅ it | ❌（非RTL） | N/A | N/A | ✅ 是 |
| ja | ✅ ja.json | ✅ "ja" | ✅ "ja" | ✅ ja | ❌（非RTL） | N/A | N/A | ✅ 是 |
| ko | ✅ ko.json | ✅ "ko" | ✅ "ko" | ✅ ko | ❌（非RTL） | N/A | N/A | ✅ 是 |
| lv | ✅ lv.json | ✅ "lv" | ✅ "lv" | ✅ lv | ❌（非RTL） | N/A | N/A | ✅ 是 |
| nl | ✅ nl.json | ✅ "nl" | ✅ "nl" | ✅ nl | ❌（非RTL） | N/A | N/A | ✅ 是 |
| **nl-be** | ✅ nl-be.json | ⚠️ 被nl截断 | ✅ "nl-be" | ✅ nl-be | ❌（非RTL） | N/A | N/A | ⚠️ 仅手动，自动检测时用nl |
| no | ✅ no.json | ✅ "no"(nb/no) | ✅ "no" | ⚠️ 加载了nb（代码不匹配） | ❌（非RTL） | N/A | N/A | ⚠️ 日期不本地化 |
| pl | ✅ pl.json | ✅ "pl" | ✅ "pl" | ✅ pl | ❌（非RTL） | N/A | N/A | ✅ 是 |
| **pt** | ✅ pt.json | ⚠️ 映射到pt-pt | ❌ 不存在 | ✅ pt | ❌（非RTL） | N/A | N/A | ❌ 不可达（冗余文件） |
| pt-br | ✅ pt-br.json | ✅ "pt-br" | ✅ "pt-br" | ✅ pt-br | ❌（非RTL） | N/A | N/A | ✅ 是 |
| pt-pt | ✅ pt-pt.json | ✅ "pt-pt" | ✅ "pt-pt" | ⚠️ 加载了pt（代码不匹配） | ❌（非RTL） | N/A | N/A | ⚠️ 日期不本地化 |
| ro | ✅ ro.json | ✅ "ro" | ✅ "ro" | ✅ ro | ❌（非RTL） | N/A | N/A | ✅ 是 |
| ru | ✅ ru.json | ✅ "ru" | ✅ "ru" | ✅ ru | ❌（非RTL） | N/A | N/A | ✅ 是 |
| sk | ✅ sk.json | ✅ "sk" | ✅ "sk" | ✅ sk | ❌（非RTL） | N/A | N/A | ✅ 是 |
| **sv-se** | ✅ sv-se.json | ⚠️ 返回"sv"不匹配 | ✅ "sv-se" | ✅ sv(而非sv-se) | ❌（非RTL） | N/A | N/A | ⚠️ 复杂错位 |
| tr | ✅ tr.json | ✅ "tr" | ✅ "tr" | ✅ tr | ❌（非RTL） | N/A | N/A | ✅ 是 |
| uk | ✅ uk.json | ✅ "uk" | ✅ "uk" | ✅ uk | ❌（非RTL） | N/A | N/A | ✅ 是 |
| vi | ✅ vi.json | ✅ "vi" | ✅ "vi" | ✅ vi | ❌（非RTL） | N/A | N/A | ✅ 是 |
| zh-cn | ✅ zh-cn.json | ✅ "zh-cn" | ✅ "zh-cn" | ✅ zh-cn | ❌（非RTL） | N/A | N/A | ✅ 是 |
| zh-tw | ✅ zh-tw.json | ✅ "zh-tw" | ✅ "zh-tw" | ✅ zh-tw | ❌（非RTL） | N/A | N/A | ✅ 是 |

**可达性说明**（按 5 个语言链路维度 + 2 个 RTL 样式维度评估）：
- ✅ 是：非 RTL 语言的资源文件、浏览器检测、下拉可选、dayjs本地化均正常；RTL列表正确标记为非RTL，RTL样式列不适用
- ⚠️ 部分：某个环节存在缺陷但整体仍可用
- ❌ 不可达：翻译资源存在但没有任何入口可以触发
- **RTL 语言额外评估**：ar/he 虽然 24 处 `html[dir="rtl"]` 样式正常，但 2 处 `body.rtl` 样式（卡片按钮对齐、终端内容方向）永不生效，因此不是"完全正常"
- **N/A**：非 RTL 语言，html[dir]样式 和 body.rtl样式 不适用

---

### 12.2 messages 对象的 key 生成机制

`@intlify/unplugin-vue-i18n` 插件根据 JSON 文件名生成 messages 的 key。

以 [vite.config.ts](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/vite.config.ts#L10-L12) 的配置为例：

```typescript
VueI18nPlugin({
  include: [path.resolve(__dirname, "./src/i18n/**/*.json")],
})
```

对于 `src/i18n/` 下的文件，key 规则为 **文件名（去扩展名）**：

| 文件名 | messages 中的 key |
|--------|-------------------|
| en.json | "en" |
| zh-cn.json | "zh-cn" |
| sv-se.json | "sv-se" |
| nl-be.json | "nl-be" |
| fa.json | "fa" |

这意味着：**setLocale() 传入的 locale 参数必须与 JSON 文件名完全一致**，否则 vue-i18n 找不到该语言的翻译字典，触发 fallbackLocale 回退到英语。

---

## 十三、重点边界语言深度分析

### 13.1 瑞典语（sv / sv-se）——四层错位

#### 13.1.1 各层代码值对比

| 层级 | 位置 | 值 | 代码依据 |
|------|------|----|---------|
| 资源文件 | `frontend/src/i18n/sv-se.json` | `"sv-se"` | 文件名即 messages key |
| 浏览器检测 | [detectLocale() L129-L132](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L129-L132) | `"sv"` | `/^sv-se\b/` 或 `/^sv\b/` 均返回 `"sv"` |
| 下拉选项 | [Languages.vue L44](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/components/settings/Languages.vue#L44) | `"sv-se"` | `locales["sv-se"] = "Swedish (Sweden)"` |
| dayjs locale | [i18n/index.ts L30](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L30) | `"sv"` | `import("dayjs/locale/sv")` |

#### 13.1.2 可达入口分析

**入口 1：浏览器自动检测**

```
navigator.language = "sv-SE" 或 "sv"
    │
    ▼
detectLocale():
    /^sv-se\b/.test("sv-se")  → true  → locale = "sv"   // ← 注意是"sv"不是"sv-se"
    /^sv\b/.test("sv")        → true  → locale = "sv"
    │
    ▼
setLocale("sv"):
    ├─ dayjs.locale("sv")        → ✅ 正常，dayjs/sv 已加载
    └─ i18n.global.locale = "sv" → ❌ messages 中只有 key "sv-se"，没有 "sv"
         │
         ▼
    vue-i18n fallback 到 fallbackLocale = "en"
         │
         ▼
    界面显示英语，但日期显示为瑞典语格式
```

**实际效果**：用户浏览器语言为瑞典语时，界面显示英语，日期却是瑞典语格式，造成不一致。

**入口 2：用户手动下拉选择**

```
用户选择 "Swedish (Sweden)"
    │
    ▼
Profile.vue → api.update(data, ["locale", ...])
    │
    ▼
authStore.updateUser({ locale: "sv-se", ... })
    │
    ▼
setLocale("sv-se"):
    ├─ dayjs.locale("sv-se")    → ❌ dayjs 中没有 "sv-se"，只有 "sv"
    │                           →   dayjs 静默回退到英语日期格式
    └─ i18n.global.locale = "sv-se" → ✅ 匹配 messages key，翻译正常
```

**实际效果**：用户手动选择后界面翻译正常（瑞典语），但日期格式退化为英语，造成另一种不一致。

#### 13.1.3 切换影响总结

| 切换方式 | 界面翻译 | 日期格式 | 一致性 |
|---------|:---:|:---:|:---:|
| 浏览器检测 sv/sv-SE | ❌ 英语 | ✅ 瑞典语 | ❌ 错位 |
| 手动选择 Swedish (Sweden) | ✅ 瑞典语 | ❌ 英语 | ❌ 错位 |

**根因**：四层代码使用了三个不同的标识符 —— `sv-se`（资源/下拉）、`sv`（检测/dayjs），互不统一。

---

### 13.2 荷兰语比利时（nl-be）——正则顺序 BUG

#### 13.2.1 正则匹配顺序分析

关键代码位于 [i18n/index.ts L133-L138](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L133-L138)：

```typescript
case /^nl\b/.test(locale):      // ← 先匹配
    locale = "nl";
    break;
case /^nl-be\b/.test(locale):   // ← 后匹配（永远不会执行）
    locale = "nl-be";
    break;
```

**问题核心**：`switch(true)` 是按 `case` 顺序依次判断的，第一个匹配的 case 执行后 `break` 跳出。

正则 `/^nl\b/` 的含义是"以 nl 开头且 nl 后是单词边界"。对于字符串 `"nl-be"`：
- `^nl` 匹配开头的 "nl"
- `\b` 匹配 "nl" 和 "-" 之间的位置（连字符属于非单词字符，`\w` 与 `\W` 之间存在单词边界）
- 因此 **`/^nl\b/.test("nl-be")` 返回 `true`**

**匹配结果**：

| navigator.language | 命中的 case | 返回值 | 期望返回 |
|-------------------|:---:|:---:|:---:|
| `"nl"` | `/^nl\b/` | `"nl"` | ✅ `"nl"` |
| `"nl-NL"` | `/^nl\b/` | `"nl"` | ✅ `"nl"` |
| `"nl-be"` | `/^nl\b/` ← 错误命中 | `"nl"` | ❌ 应为 `"nl-be"` |
| `"nl-BE"` | `/^nl\b/` ← 错误命中 | `"nl"` | ❌ 应为 `"nl-be"` |

#### 13.2.2 可达入口分析

**入口 1：浏览器自动检测**

比利时荷兰语用户的 `navigator.language` 通常是 `"nl-BE"` 或 `"nl-be"`，会被错误归类为 `"nl"`（荷兰本土荷兰语），自动加载 `nl.json` 而非 `nl-be.json`。

两种荷兰语翻译存在差异（如 `nl-be.json` 中部分词汇使用比利时惯用表达），用户看到的是错误的地域变体。

**入口 2：用户手动下拉选择**

下拉中存在 `"nl-be": "Nederlands (België)"` 选项，用户可以手动选择，此时：

```
setLocale("nl-be"):
    ├─ dayjs.locale("nl-be")     → ✅ dayjs/locale/nl-be 已加载
    └─ i18n.global.locale = "nl-be" → ✅ 匹配 nl-be.json
         │
         ▼
    界面和日期都正常显示比利时荷兰语
```

#### 13.2.3 切换影响总结

| 切换方式 | 界面翻译 | 日期格式 | 地域正确性 |
|---------|:---:|:---:|:---:|
| 浏览器检测 nl-BE | ⚠️ 荷兰本土荷兰语 | ⚠️ 荷兰本土格式 | ❌ 错误 |
| 手动选择 Nederlands (België) | ✅ 比利时荷兰语 | ✅ 比利时格式 | ✅ 正确 |

**根因**：正则匹配顺序错误——更具体的 `nl-be` 分支应该放在更通用的 `nl` 分支之前。这与同文件中的 `pt-br` 放在 `pt` 之前、`zh-tw` 放在 `zh-cn` 之前的正确模式形成鲜明对比（见 [L85-L91](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L85-L91) 和 [L95-L101](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L95-L101)）。

---

### 13.3 波斯语（fa）——翻译资源存在但完全不可达

#### 13.3.1 各层检查结果

波斯语（فارسی，Farsi）是伊朗的官方语言，属于 RTL（从右到左）书写系统。

| 检查项 | 状态 | 证据 |
|-------|:---:|-----|
| JSON 翻译资源 | ✅ 存在 | [frontend/src/i18n/fa.json](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/fa.json)，翻译完整（按钮、设置等均有波斯语译文） |
| detectLocale 分支 | ❌ 完全缺失 | [i18n/index.ts L45-L146](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L45-L146) 的 switch 中无任何 `/^fa\b/` 分支 |
| Languages 下拉选项 | ❌ 完全缺失 | [Languages.vue L17-L50](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/components/settings/Languages.vue#L17-L50) 的 locales 对象中无 `"fa"` 键 |
| dayjs locale 加载 | ❌ 未加载 | [i18n/index.ts L4-L35](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L4-L35) 的 import 列表中无 `dayjs/locale/fa` |
| RTL 语言列表 | ❌ 未包含 | [i18n/index.ts L164](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L164)：`rtlLanguages = ["he", "ar"]`，fa 不在其中 |

#### 13.3.2 可达入口分析

**入口 1：浏览器自动检测**

伊朗用户 `navigator.language = "fa-IR"` 或 `"fa"`：

```
detectLocale():
    遍历所有 case，无一匹配 /^fa\b/
        │
        ▼
    default: locale = "en"
        │
        ▼
    界面显示英语
```

**入口 2：设置页手动选择**

下拉框中根本没有波斯语选项，用户无法选择。

**入口 3：管理员设置用户默认语言**

全局设置 → 用户默认值 → UserForm 中使用同一个 Languages 组件，同样没有 fa 选项。

**入口 4：直接调用 API 修改用户 locale**

即使通过 API 直接将用户的 locale 设为 `"fa"`（绕过前端），登录后：

```
setLocale("fa"):
    ├─ dayjs.locale("fa")          → ❌ dayjs/fa 未加载，日期退化为英语
    └─ i18n.global.locale = "fa"   → ✅ fa.json 存在，翻译正常
         │
         ▼
    setHtmlLocale("fa"):
        isRtl("fa") → rtlLanguages.includes("fa") → false
            │
            ▼
        html.dir = "ltr"   ← ❌ 波斯语应该是 RTL，但实际是 LTR
```

**结果**：翻译显示为波斯语，但布局是从左到右（文字方向错误，阅读困难），日期也是英语格式。

#### 13.3.3 切换影响总结

| 场景 | 界面翻译 | 日期格式 | 文字方向 |
|-----|:---:|:---:|:---:|
| 正常访问（检测/手动均不可达） | ❌ 英语 | ❌ 英语 | ❌ LTR（英语方向） |
| 强制API设置 locale="fa" | ✅ 波斯语 | ❌ 英语 | ❌ LTR（应为RTL） |

**根因**：fa.json 翻译文件被贡献者提交，但后续的四个集成点（检测分支、下拉选项、dayjs加载、RTL列表）均未同步添加，形成"孤立资源"。

---

### 13.4 葡萄牙语系列（pt / pt-br / pt-pt）——三叉戟错位

葡萄牙语有三个资源文件，但三者在各层的支持程度不一致。

#### 13.4.1 资源文件对比

| 资源文件 | 代表地区 | 翻译特点示例（buttons.cancel） |
|---------|---------|---------|
| [pt.json](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/pt.json#L3) | 通用/欧洲葡萄牙语 | "Cancelar" |
| [pt-br.json](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/pt-br.json) | 巴西葡萄牙语 | （有独立翻译） |
| [pt-pt.json](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/pt-pt.json) | 葡萄牙（欧洲）葡萄牙语 | （有独立翻译） |

注意：`pt.json` 的翻译内容（如 `"Cancelar"`, `"Eliminar"`, `"Alterar nome"`）使用的是欧洲葡萄牙语拼写，与 `pt-pt.json` 应基本一致。

#### 13.4.2 detectLocale 映射分析

关键代码 [i18n/index.ts L85-L91](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L85-L91)：

```typescript
case /^pt-br\b/.test(locale):    // 巴西，必须放在前面
    locale = "pt-br";
    break;
case /^pt-pt\b/.test(locale):    // 葡萄牙（欧洲）
case /^pt\b/.test(locale):       // 其他所有 pt 变体（如 pt-AO 安哥拉、pt-MZ 莫桑比克）
    locale = "pt-pt";            // ← 都强制映射到 pt-pt
    break;
```

**映射结果**：

| navigator.language | 返回 locale | 实际使用的翻译文件 |
|-------------------|:---:|:---:|
| `"pt-BR"` / `"pt-br"` | `"pt-br"` | ✅ pt-br.json |
| `"pt-PT"` / `"pt-pt"` | `"pt-pt"` | ✅ pt-pt.json |
| `"pt"` / `"pt-AO"` / `"pt-MZ"` / ... | `"pt-pt"` | ⚠️ pt-pt.json（而非 pt.json） |

**关键发现**：`pt.json` 虽然存在，但浏览器自动检测永远不会返回 `"pt"`——所有非巴西的葡萄牙语都被强制映射到 `"pt-pt"`。

#### 13.4.3 下拉选项分析

[Languages.vue L39-L40](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/components/settings/Languages.vue#L39-L40)：

```javascript
"pt-br": "Português (Brasil)",
"pt-pt": "Português (Portugal)",
```

下拉中只有两个选项，没有单独的 `"pt"`。因此 `pt.json` 在手动选择场景下也不可达。

#### 13.4.4 dayjs 加载分析

[i18n/index.ts L25-L26](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L25-L26)：

```typescript
import("dayjs/locale/pt-br");   // ✅ 巴西葡萄牙语
import("dayjs/locale/pt");      // ⚠️ 这是通用葡萄牙语（即欧洲葡萄牙语）
// ❌ 没有 import("dayjs/locale/pt-pt")
```

dayjs 库中：
- `dayjs/locale/pt` = 欧洲葡萄牙语（葡萄牙）
- `dayjs/locale/pt-br` = 巴西葡萄牙语
- dayjs **不存在** `pt-pt` 这个 locale（欧洲葡萄牙语用 `pt` 即可）

#### 13.4.5 可达入口与切换影响

**入口 1：巴西用户浏览器检测 pt-BR**

```
setLocale("pt-br"):
    ├─ dayjs.locale("pt-br")  → ✅ 已加载，日期使用巴西格式
    └─ i18n: "pt-br"          → ✅ pt-br.json，翻译正常
```

✅ 完全正常。

**入口 2：葡萄牙用户浏览器检测 pt-PT**

```
setLocale("pt-pt"):
    ├─ dayjs.locale("pt-pt")  → ❌ dayjs 中无 pt-pt 这个 locale
    │                          →   dayjs 静默回退到英语格式
    └─ i18n: "pt-pt"          → ✅ pt-pt.json，翻译正常
```

**结果**：界面翻译正确（欧洲葡萄牙语），但日期显示为英语格式，不一致。

**入口 3：用户手动选择 "Português (Portugal)"**

与入口 2 相同，dayjs 日期格式退化为英语。

**入口 4：安哥拉/莫桑比克等其他葡语国家用户**

浏览器语言 `pt-AO` → 被检测映射到 `"pt-pt"`，结果同入口 2（翻译正确，日期英语）。

**pt.json 的命运**：

无论自动检测还是手动选择，都无法触发 `pt.json`。它与 `pt-pt.json` 内容可能重复（都是欧洲葡萄牙语），属于冗余文件。

#### 13.4.6 切换影响总结

| 场景 | 返回 locale | 界面翻译 | 日期格式 | 一致性 |
|-----|:---:|:---:|:---:|:---:|
| 浏览器 pt-BR | "pt-br" | ✅ 巴西葡语 | ✅ 巴西格式 | ✅ 一致 |
| 浏览器 pt-PT | "pt-pt" | ✅ 欧洲葡语 | ❌ 英语 | ❌ 不一致 |
| 浏览器 pt（其他） | "pt-pt" | ✅ 欧洲葡语 | ❌ 英语 | ❌ 不一致 |
| 手动选 Português (Brasil) | "pt-br" | ✅ 巴西葡语 | ✅ 巴西格式 | ✅ 一致 |
| 手动选 Português (Portugal) | "pt-pt" | ✅ 欧洲葡语 | ❌ 英语 | ❌ 不一致 |
| pt.json（通用） | — | ❌ 不可达 | ❌ 不可达 | — |

**根因**：vue-i18n 使用 `pt-pt` 作为欧洲葡萄牙语的标识，而 dayjs 使用 `pt`，两者命名约定不同但未做映射。

---

## 十四、边界语言问题汇总与修复建议

### 14.1 问题总表

| 语言 | 问题类型 | 严重程度 | 影响范围 |
|-----|---------|:---:|---------|
| fa 波斯语 | 完全不可达（4处缺失） | 🔴 高 | 所有伊朗/波斯语用户 |
| nl-be 荷兰语比利时 | 正则顺序BUG（自动检测失效） | 🟠 中 | 比利时荷兰语用户（自动检测） |
| sv-se 瑞典语 | 四层代码标识符错位 | 🟠 中 | 所有瑞典语用户 |
| pt-pt 葡萄牙语(葡萄牙) | dayjs locale名称不匹配 | 🟡 低 | 欧洲葡语用户的日期显示 |
| pt 通用葡萄牙语 | 翻译文件不可达 | 🟡 低 | 冗余文件，无用户影响 |
| ca 加泰罗尼亚语 | 缺少detectLocale分支 | 🟡 低 | 仅手动可选，无法自动检测 |

### 14.2 具体修复建议

**修复 1：瑞典语统一标识符（sv vs sv-se）**

建议方案：统一使用 `"sv-se"`（与资源文件名和下拉选项一致）

```typescript
// i18n/index.ts
- case /^sv-se\b/.test(locale):
- case /^sv\b/.test(locale):
-     locale = "sv";
+ case /^sv-se\b/.test(locale):
+ case /^sv\b/.test(locale):
+     locale = "sv-se";

- import("dayjs/locale/sv");
+ import("dayjs/locale/sv");  // 保留（dayjs无sv-se）
// 在 setLocale 中增加映射
+ const dayjsLocaleMap: Record<string, string> = {
+   "sv-se": "sv",
+   "pt-pt": "pt",
+ };
  export function setLocale(locale: string) {
+   dayjs.locale(dayjsLocaleMap[locale] || locale);
-   dayjs.locale(locale);
    i18n.global.locale.value = locale;
  }
```

**修复 2：荷兰语比利时正则顺序**

```typescript
// i18n/index.ts  - 调换两个 case 的顺序
- case /^nl\b/.test(locale):
-     locale = "nl";
-     break;
  case /^nl-be\b/.test(locale):
      locale = "nl-be";
      break;
+ case /^nl\b/.test(locale):
+     locale = "nl";
+     break;
```

**修复 3：波斯语集成（4处同步添加）**

```typescript
// i18n/index.ts - 增加检测分支（放在default之前）
+ case /^fa\b/.test(locale):
+     locale = "fa";
+     break;

// i18n/index.ts - 增加 dayjs 加载
+ import("dayjs/locale/fa");

// i18n/index.ts - RTL 列表添加波斯语
- export const rtlLanguages = ["he", "ar"];
+ export const rtlLanguages = ["he", "ar", "fa"];

// Languages.vue - locales 对象中增加
+ fa: "فارسی",
```

**修复 4：葡萄牙语 dayjs 映射**

见修复 1 中的 `dayjsLocaleMap` 方案，将 `"pt-pt"` 映射到 dayjs 的 `"pt"`。

**修复 5：检测分支补充（加泰罗尼亚语等）**

```typescript
// i18n/index.ts
+ case /^ca\b/.test(locale):
+     locale = "ca";
+     break;
```

---

## 十五、三层架构完整代码流程对照

以下是将语言代码从用户浏览器到最终界面渲染的完整数据流，标注了每层的输入输出和可能的丢失/错位点。

```
用户访问
    │
    │ navigator.language (RFC 5646, e.g. "sv-SE", "nl-BE", "fa-IR")
    ▼
┌───────────────────────────────────────────────────┐
│ 第1层: detectLocale() [i18n/index.ts L41-L149]      │
│   输入: navigator.language                          │
│   逻辑: switch(true) + 正则匹配（顺序敏感！）        │
│   输出: 规范化 locale code                          │
│   ❌ 丢失风险: fa/ca无分支、nl-be被nl截断、          │
│              sv→"sv"与资源不匹配                    │
└───────────────────────┬───────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────┐
│ 第2层: i18n 实例初始化 [i18n/index.ts L166-L172]    │
│   messages key = JSON文件名（如 sv-se.json→"sv-se"） │
│   ❌ 不匹配风险: detectLocale输出与文件名不一致时，   │
│                fallback到 en                       │
└───────────────────────┬───────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────┐
│ 第3层: setLocale() 触发                           │
│   a) dayjs.locale(locale)                          │
│      ❌ 风险: dayjs无对应locale（如pt-pt、sv-se）    │
│   b) i18n.global.locale.value = locale             │
│      ✅ 与 messages key 一致则正常翻译               │
└───────────────────────┬───────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────┐
│ 第4层: App.vue watch(locale) → setHtmlLocale()     │
│   html.lang = locale                               │
│   html.dir = isRtl(locale) ? "rtl" : "ltr"         │
│   ❌ 风险: fa不在rtlLanguages，dir错误为ltr         │
└───────────────────────┬───────────────────────────┘
                        │
                        ▼
              界面渲染 + Toast RTL + 日期格式
```

用户手动设置语言时（通过 Profile.vue），跳过第1层（detectLocale），从 Languages.vue 下拉选项的 key 开始进入第2层，因此问题表现与自动检测场景有所不同（如瑞典语手动选择反而能正确翻译，但日期出问题）。

---

## 十六、视频播放控件语言包分析

Video.js 播放器拥有独立的语言包加载机制，不依赖 vue-i18n，而是直接从 `document.documentElement.lang` 读取当前语言。

### 16.1 加载流程

**文件**：[VideoPlayer.vue](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/components/files/VideoPlayer.vue#L59-L91)

```typescript
const initVideoPlayer = async () => {
  const lang = document.documentElement.lang;  // 从 HTML 标签读取，由 setHtmlLocale() 设置
  const languagePack = await (
    languageImports[lang] || languageImports.en
  )?.();
  const code = languageImports[lang] ? lang : "en";
  videojs.addLanguage(code, languagePack.default);  // 动态注册到 videojs
  
  const langOpt = { language: code };
  const options = getOptions(props.options, langOpt, ...);
  player.value = videojs(videoPlayer.value!, options, () => {});
};
```

**关键逻辑**：
1. 语言代码来源是 `document.documentElement.lang`，即 `setHtmlLocale()` 设置的值
2. 如果 `languageImports` 中没有对应 key，则回退到 `en`
3. 调用 `videojs.addLanguage()` 注册语言包
4. 初始化播放器时传入 `{ language: code }`

### 16.2 languageImports 映射表

[VideoPlayer.vue L149-L179](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/components/files/VideoPlayer.vue#L149-L179) 定义了 29 个 key 的动态导入映射（注意：其中 `nb` 这个 key 永远不会被访问，因为 locale 值是 `"no"`）：

| languageImports 的 key | video.js 语言包路径 | 能否被实际访问 | 说明 |
|-----------------------|-------------------|:---:|------|
| ar | `video.js/dist/lang/ar.json` | ✅ | 匹配 locale "ar" |
| bg | `video.js/dist/lang/bg.json` | ✅ | 匹配 locale "bg" |
| cs | `video.js/dist/lang/cs.json` | ✅ | 匹配 locale "cs" |
| de | `video.js/dist/lang/de.json` | ✅ | 匹配 locale "de" |
| el | `video.js/dist/lang/el.json` | ✅ | 匹配 locale "el" |
| en | `video.js/dist/lang/en.json` | ✅ | 匹配 locale "en"（fallback） |
| es | `video.js/dist/lang/es.json` | ✅ | 匹配 locale "es" |
| fr | `video.js/dist/lang/fr.json` | ✅ | 匹配 locale "fr" |
| he | `video.js/dist/lang/he.json` | ✅ | 匹配 locale "he" |
| hr | `video.js/dist/lang/hr.json` | ✅ | 匹配 locale "hr" |
| hu | `video.js/dist/lang/hu.json` | ✅ | 匹配 locale "hu" |
| it | `video.js/dist/lang/it.json` | ✅ | 匹配 locale "it" |
| ja | `video.js/dist/lang/ja.json` | ✅ | 匹配 locale "ja" |
| ko | `video.js/dist/lang/ko.json` | ✅ | 匹配 locale "ko" |
| lv | `video.js/dist/lang/lv.json` | ✅ | 匹配 locale "lv" |
| **nb** | `video.js/dist/lang/nb.json` | ❌ 死代码 | key 是 "nb" 但 locale 是 "no"，永远访问不到 |
| nl | `video.js/dist/lang/nl.json` | ✅ | 匹配 locale "nl" |
| **nl-be** | `video.js/dist/lang/nl.json` | ✅ | 匹配 locale "nl-be"，回退到 nl 包 |
| pl | `video.js/dist/lang/pl.json` | ✅ | 匹配 locale "pl" |
| pt-br | `video.js/dist/lang/pt-BR.json` | ✅ | 匹配 locale "pt-br"，文件名与 video.js 官方一致 |
| pt-pt | `video.js/dist/lang/pt-PT.json` | ✅ | 匹配 locale "pt-pt"，文件名与 video.js 官方一致 |
| ro | `video.js/dist/lang/ro.json` | ✅ | 匹配 locale "ro" |
| ru | `video.js/dist/lang/ru.json` | ✅ | 匹配 locale "ru" |
| sk | `video.js/dist/lang/sk.json` | ✅ | 匹配 locale "sk" |
| tr | `video.js/dist/lang/tr.json` | ✅ | 匹配 locale "tr" |
| uk | `video.js/dist/lang/uk.json` | ✅ | 匹配 locale "uk" |
| vi | `video.js/dist/lang/vi.json` | ✅ | 匹配 locale "vi" |
| zh-cn | `video.js/dist/lang/zh-CN.json` | ✅ | 匹配 locale "zh-cn"，文件名与 video.js 官方一致 |
| zh-tw | `video.js/dist/lang/zh-TW.json` | ✅ | 匹配 locale "zh-tw"，文件名与 video.js 官方一致 |

**实际可访问 key 数**：28 个（29 个 key 减去 1 个死代码 `nb`）；**唯一 video.js 语言包数**：27 个（`nl-be` 与 `nl` 共用同一语言包）

### 16.3 缺失的语言

以下 6 种 vue-i18n 支持的语言在 VideoPlayer 中没有可直接访问的映射，播放视频时控件将显示英语：

| 语言代码 | 缺失原因 | 实际效果 |
|---------|---------|---------|
| **ca** | `languageImports` 无 ca 键 | 视频控件英语 |
| **fa** | `languageImports` 无 fa 键 | 视频控件英语 |
| **is** | `languageImports` 无 is 键 | 视频控件英语 |
| **no** | `languageImports` 只有 "nb" 键，没有 "no" 键 | 视频控件英语（死代码问题） |
| **pt** | `languageImports` 无 pt 键 | （但 pt 本身也不可达，影响为 0） |
| **sv-se** | `languageImports` 无 sv-se 键 | 视频控件英语 |

### 16.4 特殊映射说明

1. **挪威语（no）——死代码 BUG**：
   - video.js 中挪威书面语的代码是 `nb`
   - 代码中写了 `nb: () => import("video.js/dist/lang/nb.json")`
   - 但 `document.documentElement.lang` 的值是 `"no"`（来自 `detectLocale()`）
   - 查找 `languageImports["no"]` → undefined → 回退英语
   - **`nb` 这个 key 永远不会被访问到，属于死代码**

   **实际效果**：挪威用户播放视频时，界面是挪威语，但视频控件显示英语。

2. **荷兰语比利时（nl-be）**：video.js 没有独立的 nl-be 语言包，代码中显式映射到 `nl.json`（荷兰本土）。这是合理的，因为两种变体在视频控件的简单术语上差异极小。

3. **地区代码大小写**：
   - video.js 官方语言包文件名使用大写地区代码（`pt-BR.json`、`pt-PT.json`、`zh-CN.json`、`zh-TW.json`）
   - 代码中的 import 路径写法与 video.js 官方完全一致，**不存在大小写错误**
   - Vite 构建时会按照实际文件路径解析，在所有平台上都能正常工作
   - 之前认为"大小写不同可能在 Linux 上失败"是错误判断

---

## 十七、日期本地化映射核对

dayjs 语言包与 vue-i18n locale 代码之间存在命名不一致问题，且 `setLocale()` 中没有映射层。

### 17.1 dayjs 语言包加载清单

**文件**：[i18n/index.ts L4-L35](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L4-L35)

dayjs 预加载了 33 种语言包（按导入顺序）：

| dayjs locale 代码 | 对应 vue-i18n locale | 匹配状态 |
|------------------|---------------------|:---:|
| ar | ar | ✅ 匹配 |
| bg | bg | ✅ 匹配 |
| ca | ca | ⚠️ 加载了但 detectLocale 无分支 |
| cs | cs | ✅ 匹配 |
| de | de | ✅ 匹配 |
| el | el | ✅ 匹配 |
| en | en | ✅ 匹配 |
| es | es | ✅ 匹配 |
| fr | fr | ✅ 匹配 |
| he | he | ✅ 匹配 |
| hr | hr | ✅ 匹配 |
| hu | hu | ✅ 匹配 |
| is | is | ✅ 匹配 |
| it | it | ✅ 匹配 |
| ja | ja | ✅ 匹配 |
| ko | ko | ✅ 匹配 |
| lv | lv | ✅ 匹配 |
| nb | no | ⚠️ dayjs 用 nb，vue-i18n 用 no |
| nl | nl | ✅ 匹配 |
| nl-be | nl-be | ✅ 匹配 |
| pl | pl | ✅ 匹配 |
| pt-br | pt-br | ✅ 匹配 |
| **pt** | pt-pt | ⚠️ dayjs 用 pt，vue-i18n 用 pt-pt |
| ro | ro | ✅ 匹配 |
| ru | ru | ✅ 匹配 |
| sk | sk | ✅ 匹配 |
| **sv** | sv-se | ⚠️ dayjs 用 sv，vue-i18n 用 sv-se |
| tr | tr | ✅ 匹配 |
| uk | uk | ✅ 匹配 |
| vi | vi | ✅ 匹配 |
| zh-cn | zh-cn | ✅ 匹配 |
| zh-tw | zh-tw | ✅ 匹配 |
| **（无 fa）** | fa | ❌ dayjs 未加载 fa |

### 17.2 setLocale() 调用链

**文件**：[i18n/index.ts L180-L186](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L180-L186)

```typescript
export function setLocale(locale: string) {
  dayjs.locale(locale);  // ← 直接传入，无映射
  i18n.global.locale.value = locale;
}
```

**问题核心**：`dayjs.locale(locale)` 直接传入 vue-i18n 的 locale 代码，没有任何映射转换。

### 17.3 不匹配场景的实际行为

| 场景 | 传入 setLocale 的 locale | dayjs.locale() 实际行为 | 日期显示效果 |
|-----|-------------------------|------------------------|:---:|
| 浏览器检测瑞典语 | `"sv"` | ✅ dayjs/sv 已加载 | 瑞典语格式 |
| 手动选瑞典语 | `"sv-se"` | ❌ dayjs 无 sv-se locale | 回退英语格式 |
| 浏览器/手动选 pt-pt | `"pt-pt"` | ❌ dayjs 无 pt-pt locale | 回退英语格式 |
| 浏览器/手动选 no | `"no"` | ❌ dayjs 无 no locale（只有 nb） | 回退英语格式 |
| 强制API设置 fa | `"fa"` | ❌ dayjs 未加载 fa | 回退英语格式 |

**dayjs 回退机制**：当 `dayjs.locale("xxx")` 传入未加载的 locale 时，dayjs 不会报错，而是**静默回退到英语**（默认 locale）。这与 vue-i18n 显式的 `fallbackLocale` 配置不同，dayjs 的回退是隐式的。

### 17.4 修复建议

在 `setLocale()` 中增加一个映射表，将 vue-i18n 的 locale 代码转换为 dayjs 的对应代码：

```typescript
// i18n/index.ts
const DAYJS_LOCALE_MAP: Record<string, string> = {
  "sv-se": "sv",      // 瑞典语
  "pt-pt": "pt",      // 欧洲葡萄牙语
  "no": "nb",         // 挪威语（书面语）
  "fa": "fa",         // 需先添加 import("dayjs/locale/fa")
};

export function setLocale(locale: string) {
  const dayjsLocale = DAYJS_LOCALE_MAP[locale] || locale;
  dayjs.locale(dayjsLocale);
  i18n.global.locale.value = locale;
}
```

---

## 十八、RTL 样式选择器深度分析

项目中存在**两套** RTL 样式选择器系统，但其中一套从未被激活。

### 18.1 两套选择器对比

| 选择器模式 | CSS 规则数量 | 设置位置 | 是否生效 |
|-----------|:---:|---------|:---:|
| `html[dir="rtl"]` | 24 处 | [setHtmlLocale()](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/i18n/index.ts#L188-L193) 设置 `html.dir = "rtl"` | ✅ 正常生效 |
| `body.rtl` | 2 处 | **从未设置** | ❌ 死代码 |

### 18.2 html[dir="rtl"] 生效样式分布

| CSS 文件 | 规则位置 | 样式效果 |
|---------|---------|---------|
| [base.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/base.css#L61-L79) | L61-L64 | `nav` 从左侧固定变为右侧固定 |
| [base.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/base.css#L61-L79) | L77-L79 | `nav .action` 文本右对齐 |
| [base.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/base.css#L117-L119) | L117-L119 | `.breadcrumbs a` 左移 16em |
| [styles.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/styles.css#L361-L363) | L361-L363 | `.breadcrumbs .chevron` 水平翻转 + 右移 |
| [styles.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/styles.css#L450-L463) | L450-L453 | `.card-content textarea` 强制 LTR + 左对齐 |
| [styles.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/styles.css#L450-L463) | L455-L458 | `.card-content .small + input` 强制 LTR + 左对齐 |
| [styles.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/styles.css#L450-L463) | L460-L463 | `.card.floating .file-list` 强制 LTR + 左对齐 |
| [header.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/header.css#L145-L158) | L145-L158 | `#search #result` 定位调整 |
| [header.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/header.css#L248-L249) | L248-L249 | `#search .boxes h3` 方向调整 |
| [listing.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/listing.css#L1-L3) | L1-L3 | `#listing` 方向调整 |
| [dashboard.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/dashboard.css#L11-L13) | L11-L13 | `.dashboard .row` margin 调整 |
| [dashboard.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/dashboard.css#L67-L69) | L67-L69 | `#nav .wrapper` margin 调整 |
| [dashboard.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/dashboard.css#L148-L150) | L148-L150 | `table tr > *` 文本右对齐 |
| [dashboard.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/dashboard.css#L201-L203) | L201-L203 | `.card .card-title > *:first-child` 方向调整 |
| [dashboard.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/dashboard.css#L502-L503) | L502-L503 | `.credits` 方向调整 |
| [mobile.css](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/mobile.css#L59-L165) | 8 处 | 移动端下拉菜单、导航、文件列表、面包屑等定位 |

### 18.3 body.rtl 死代码分析

以下两条规则**永远不会匹配**，因为代码中从未给 `<body>` 添加 `rtl` 类：

1. **[dashboard.css L226-L228](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/dashboard.css#L226-L228)**：
   ```css
   body.rtl .card .card-action {
     text-align: left;  /* RTL 下卡片操作按钮应左对齐 */
   }
   ```
   原规则 L222-L224 是 `text-align: right`，RTL 下应翻转。

2. **[_shell.css L47-L49](file:///d:/fz/0601/solo-dogfeeding/code/178-filebrowser/frontend/src/css/_shell.css#L47-L49)**：
   ```css
   body.rtl .shell-content {
     direction: ltr;  /* RTL 下终端内容应保持 LTR（命令行从左到右） */
   }
   ```

### 18.4 RTL 样式切换的完整链路

```
用户切换到 RTL 语言（he/ar）
    │
    ▼
setLocale(locale)
    │
    ▼
App.vue watch(locale) 触发
    │
    ▼
setHtmlLocale(locale) [i18n/index.ts L188-L193]
    ├─ html.lang = locale
    └─ isRtl(locale) → true
         │
         ▼
    html.dir = "rtl"  ← 唯一设置的属性
         │
         ▼
    html[dir="rtl"] 选择器匹配（24处CSS生效）
         │
         ▼
    ✅ 侧边栏右置、面包屑箭头翻转、表单输入方向调整
    ❌ body.rtl 选择器不匹配（2处CSS不生效）
         │
         ▼
    卡片操作按钮仍右对齐（应为左对齐）
    终端内容方向未修正
```

### 18.5 修复建议

方案 A（推荐）：在 `setHtmlLocale()` 中同步设置 `body.rtl` 类：

```typescript
export function setHtmlLocale(locale: string) {
  const html = document.documentElement;
  const body = document.body;
  html.lang = locale;
  if (isRtl(locale)) {
    html.dir = "rtl";
    body.classList.add("rtl");
  } else {
    html.dir = "ltr";
    body.classList.remove("rtl");
  }
}
```

方案 B：将 2 处 `body.rtl` 选择器改为 `html[dir="rtl"]`，保持一致性。

---

## 十九、全量语言矩阵统一核对表（34 种语言 × 9 个数据维度 + 综合状态）

**列说明**：语言列只是索引，不计入维度；其后 9 列是数据维度（JSON资源、detectLocale、Languages下拉、dayjs加载、dayjs匹配、videojs加载、RTL列表、html[dir]样式、body.rtl样式），最后 1 列是综合状态。

| 语言 | JSON资源 | detectLocale返回 | Languages下拉 | dayjs加载 | dayjs匹配 | videojs加载 | RTL列表 | html[dir]样式 | body.rtl样式 | 综合状态 |
|-----|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| ar | ✅ ar.json | ✅ "ar" | ✅ "ar" | ✅ ar | ✅ 匹配 | ✅ ar.json | ✅ 是 | ✅ 生效 | ❌ 死代码 | ⚠️ 基本正常，2处RTL样式不生效 |
| bg | ✅ bg.json | ✅ "bg" | ✅ "bg" | ✅ bg | ✅ 匹配 | ✅ bg.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| ca | ✅ ca.json | ❌ 无→en | ✅ "ca" | ✅ ca | ✅ 匹配 | ❌ 无 | ❌（非RTL） | N/A | N/A | ⚠️ 仅手动，视频控件英语 |
| cs | ✅ cs.json | ✅ "cs" | ✅ "cs" | ✅ cs | ✅ 匹配 | ✅ cs.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| de | ✅ de.json | ✅ "de" | ✅ "de" | ✅ de | ✅ 匹配 | ✅ de.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| el | ✅ el.json | ✅ "el" | ✅ "el" | ✅ el | ✅ 匹配 | ✅ el.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| en | ✅ en.json | ✅ "en" | ✅ "en" | ✅ en | ✅ 匹配 | ✅ en.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| es | ✅ es.json | ✅ "es" | ✅ "es" | ✅ es | ✅ 匹配 | ✅ es.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| **fa** | ✅ fa.json | ❌ 无→en | ❌ 无 | ❌ 未加载 | — | ❌ 无 | ❌ 不在列表 | ❌ dir=ltr（应为rtl） | ❌ 不匹配 | ❌ 完全不可达 |
| fr | ✅ fr.json | ✅ "fr" | ✅ "fr" | ✅ fr | ✅ 匹配 | ✅ fr.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| he | ✅ he.json | ✅ "he" | ✅ "he" | ✅ he | ✅ 匹配 | ✅ he.json | ✅ 是 | ✅ 生效 | ❌ 死代码 | ⚠️ 基本正常，2处RTL样式不生效 |
| hr | ✅ hr.json | ✅ "hr" | ✅ "hr" | ✅ hr | ✅ 匹配 | ✅ hr.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| hu | ✅ hu.json | ✅ "hu" | ✅ "hu" | ✅ hu | ✅ 匹配 | ✅ hu.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| is | ✅ is.json | ✅ "is" | ✅ "is" | ✅ is | ✅ 匹配 | ❌ 无 | ❌（非RTL） | N/A | N/A | ⚠️ 视频控件英语 |
| it | ✅ it.json | ✅ "it" | ✅ "it" | ✅ it | ✅ 匹配 | ✅ it.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| ja | ✅ ja.json | ✅ "ja" | ✅ "ja" | ✅ ja | ✅ 匹配 | ✅ ja.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| ko | ✅ ko.json | ✅ "ko" | ✅ "ko" | ✅ ko | ✅ 匹配 | ✅ ko.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| lv | ✅ lv.json | ✅ "lv" | ✅ "lv" | ✅ lv | ✅ 匹配 | ✅ lv.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| nl | ✅ nl.json | ✅ "nl" | ✅ "nl" | ✅ nl | ✅ 匹配 | ✅ nl.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| **nl-be** | ✅ nl-be.json | ⚠️ 被nl截断→"nl" | ✅ "nl-be" | ✅ nl-be | ✅ 匹配 | ✅ nl.json(回退) | ❌（非RTL） | N/A | N/A | ⚠️ 自动检测时用nl，手动正常 |
| no | ✅ no.json | ✅ "no"(nb/no) | ✅ "no" | ✅ nb(挪威) | ❌ no→nb | ❌ 无（nb key是死代码） | ❌（非RTL） | N/A | N/A | ⚠️ 日期/视频控件均英语 |
| pl | ✅ pl.json | ✅ "pl" | ✅ "pl" | ✅ pl | ✅ 匹配 | ✅ pl.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| **pt** | ✅ pt.json | ⚠️ 映射到pt-pt | ❌ 无 | ✅ pt | ❌ pt→pt-pt | ❌ 无 | ❌（非RTL） | N/A | N/A | ❌ 不可达（冗余） |
| pt-br | ✅ pt-br.json | ✅ "pt-br" | ✅ "pt-br" | ✅ pt-br | ✅ 匹配 | ✅ pt-BR.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| **pt-pt** | ✅ pt-pt.json | ✅ "pt-pt" | ✅ "pt-pt" | ✅ pt(加载) | ❌ pt-pt→pt | ✅ pt-PT.json | ❌（非RTL） | N/A | N/A | ⚠️ 日期英语 |
| ro | ✅ ro.json | ✅ "ro" | ✅ "ro" | ✅ ro | ✅ 匹配 | ✅ ro.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| ru | ✅ ru.json | ✅ "ru" | ✅ "ru" | ✅ ru | ✅ 匹配 | ✅ ru.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| sk | ✅ sk.json | ✅ "sk" | ✅ "sk" | ✅ sk | ✅ 匹配 | ✅ sk.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| **sv-se** | ✅ sv-se.json | ⚠️ 返回"sv"→不匹配 | ✅ "sv-se" | ✅ sv(加载) | ❌ 双向错配 | ❌ 无 | ❌（非RTL） | N/A | N/A | ❌ 检测→界面英语，手动→日期英语 |
| tr | ✅ tr.json | ✅ "tr" | ✅ "tr" | ✅ tr | ✅ 匹配 | ✅ tr.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| uk | ✅ uk.json | ✅ "uk" | ✅ "uk" | ✅ uk | ✅ 匹配 | ✅ uk.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| vi | ✅ vi.json | ✅ "vi" | ✅ "vi" | ✅ vi | ✅ 匹配 | ✅ vi.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| zh-cn | ✅ zh-cn.json | ✅ "zh-cn" | ✅ "zh-cn" | ✅ zh-cn | ✅ 匹配 | ✅ zh-CN.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |
| zh-tw | ✅ zh-tw.json | ✅ "zh-tw" | ✅ "zh-tw" | ✅ zh-tw | ✅ 匹配 | ✅ zh-TW.json | ❌（非RTL） | N/A | N/A | ✅ 正常 |

**状态说明**（按 9 个数据维度评估）：
- ✅ 正常：非 RTL 语言 6 个基础 i18n 维度（JSON资源、detectLocale、Languages下拉、dayjs加载、dayjs匹配、videojs加载）全部匹配，RTL相关列为非RTL或 N/A
- ⚠️ 部分缺陷：某个维度存在问题，但整体仍可用
- ❌ 严重缺陷：完全不可达或多个维度不匹配
- **N/A**：非 RTL 语言，html[dir]样式 和 body.rtl样式 不适用
- **RTL 语言额外要求**：RTL列表 + html[dir]样式 + body.rtl样式 三项均需满足才算"完全正常"，ar/he 因 body.rtl 样式不生效只能评"基本正常"

---

## 二十、重点边界语言可达性与显示效果汇总

### 20.1 瑞典语（sv / sv-se）完整效果矩阵

| 场景 | 触发方式 | locale值 | 界面翻译 | 日期格式 | 视频控件 | RTL样式 | 综合体验 |
|-----|---------|:---:|:---:|:---:|:---:|:---:|:---:|
| 浏览器检测 sv-SE | 自动 | `"sv"` | ❌ 英语（messages无"sv"key） | ✅ 瑞典语（dayjs/sv） | ❌ 英语（无sv-se映射） | N/A | 混乱：界面英语但日期瑞典语 |
| 浏览器检测 sv | 自动 | `"sv"` | ❌ 英语 | ✅ 瑞典语 | ❌ 英语 | N/A | 同上 |
| 手动选 "Swedish (Sweden)" | 设置页 | `"sv-se"` | ✅ 瑞典语 | ❌ 英语（dayjs无sv-se） | ❌ 英语（无sv-se映射） | N/A | 部分可用：翻译正确但日期/视频英语 |
| 管理员设用户locale=sv-se | API | `"sv-se"` | ✅ 瑞典语 | ❌ 英语 | ❌ 英语 | N/A | 同上 |

**根因**：四层代码使用了三个不同标识符 —— `sv-se`（资源/下拉）、`sv`（检测/dayjs）。

### 20.2 荷兰语比利时（nl-be）完整效果矩阵

| 场景 | 触发方式 | locale值 | 界面翻译 | 日期格式 | 视频控件 | RTL样式 | 综合体验 |
|-----|---------|:---:|:---:|:---:|:---:|:---:|:---:|
| 浏览器检测 nl-BE | 自动 | `"nl"`（被正则截断） | ⚠️ 荷兰本土语 | ⚠️ 荷兰本土格式 | ⚠️ 荷兰本土语 | N/A | 地域错误：显示荷兰而非比利时变体 |
| 浏览器检测 nl-be | 自动 | `"nl"`（被正则截断） | ⚠️ 荷兰本土语 | ⚠️ 荷兰本土格式 | ⚠️ 荷兰本土语 | N/A | 同上 |
| 手动选 "Nederlands (België)" | 设置页 | `"nl-be"` | ✅ 比利时荷兰语 | ✅ 比利时格式（dayjs/nl-be） | ⚠️ 荷兰本土语（回退nl） | N/A | 基本可用：翻译/日期正确，视频用荷兰本土语 |

**根因**：`/^nl\b/` 放在 `/^nl-be\b/` 之前，前者匹配 `"nl-be"` 后 break，后者永远不执行。

### 20.3 波斯语（fa）完整效果矩阵

| 场景 | 触发方式 | locale值 | 界面翻译 | 日期格式 | 视频控件 | RTL样式 | 综合体验 |
|-----|---------|:---:|:---:|:---:|:---:|:---:|:---:|
| 浏览器检测 fa-IR | 自动 | `"en"`（无分支） | ❌ 英语 | ❌ 英语 | ❌ 英语 | ⚠️ LTR（应为RTL） | 完全不可用 |
| 浏览器检测 fa | 自动 | `"en"`（无分支） | ❌ 英语 | ❌ 英语 | ❌ 英语 | ⚠️ LTR | 完全不可用 |
| 手动选择 | 设置页 | — | ❌ 下拉无fa选项 | — | — | — | 无法选择 |
| 强制API设locale=fa | API | `"fa"` | ✅ 波斯语 | ❌ 英语（dayjs/fa未加载） | ❌ 英语（无fa映射） | ⚠️ LTR（fa不在rtl列表） | 严重缺陷：翻译正确但文字方向错误，日期/视频英语 |

**根因**：fa.json 翻译文件被贡献，但四个集成点（检测分支、下拉选项、dayjs加载、RTL列表）均未同步。

### 20.4 葡萄牙语系列完整效果矩阵

| 场景 | 触发方式 | locale值 | 界面翻译 | 日期格式 | 视频控件 | RTL样式 | 综合体验 |
|-----|---------|:---:|:---:|:---:|:---:|:---:|:---:|
| 浏览器检测 pt-BR | 自动 | `"pt-br"` | ✅ 巴西葡语 | ✅ 巴西格式 | ✅ 巴西葡语 | N/A | ✅ 完全正常 |
| 浏览器检测 pt-PT | 自动 | `"pt-pt"` | ✅ 欧洲葡语 | ❌ 英语（dayjs无pt-pt） | ✅ 欧洲葡语 | N/A | 部分可用：翻译/视频正确，日期英语 |
| 浏览器检测 pt | 自动 | `"pt-pt"`（被映射） | ✅ 欧洲葡语 | ❌ 英语 | ✅ 欧洲葡语 | N/A | 同上 |
| 浏览器检测 pt-AO(安哥拉) | 自动 | `"pt-pt"`（被映射） | ✅ 欧洲葡语 | ❌ 英语 | ✅ 欧洲葡语 | N/A | 同上 |
| 手动选 "Português (Brasil)" | 设置页 | `"pt-br"` | ✅ 巴西葡语 | ✅ 巴西格式 | ✅ 巴西葡语 | N/A | ✅ 完全正常 |
| 手动选 "Português (Portugal)" | 设置页 | `"pt-pt"` | ✅ 欧洲葡语 | ❌ 英语 | ✅ 欧洲葡语 | N/A | 部分可用：日期英语 |
| pt.json（通用） | 任何方式 | — | ❌ 不可达 | ❌ 不可达 | ❌ 不可达 | — | 冗余文件 |

**根因**：vue-i18n 使用 `pt-pt` 表示欧洲葡语，dayjs 使用 `pt`，两者命名约定不同且无映射。

### 20.5 RTL 语言效果对比（he vs fa）

| 语言 | RTL列表 | html.dir | 24处 html[dir] 样式 | 2处 body.rtl 样式 | 文字方向 | 卡片按钮对齐 | 终端内容方向 |
|-----|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| he（希伯来语） | ✅ 在列表 | ✅ "rtl" | ✅ 全部生效 | ❌ 不匹配 | ✅ RTL | ❌ 仍右对齐（应为左） | ❌ 未强制LTR |
| ar（阿拉伯语） | ✅ 在列表 | ✅ "rtl" | ✅ 全部生效 | ❌ 不匹配 | ✅ RTL | ❌ 仍右对齐 | ❌ 未强制LTR |
| fa（波斯语） | ❌ 不在列表 | ❌ "ltr" | ❌ 不匹配 | ❌ 不匹配 | ❌ LTR（错误） | ❌ 右对齐 | ❌ 未强制LTR |

**RTL 缺陷影响**：
- ✅ 生效的：侧边栏右置、面包屑箭头翻转、搜索框定位、表格右对齐
- ❌ 不生效的：卡片操作按钮左对齐、终端内容保持 LTR（命令行应为左到右）

---

## 二十一、全链路问题汇总与修复优先级

| 优先级 | 问题 | 影响语言 | 修复复杂度 | 涉及文件 |
|:---:|-----|---------|:---:|---------|
| 🔴 高 | 波斯语 fa 完全不可达（4处缺失：检测分支、下拉选项、dayjs加载、RTL列表） | fa | 低 | i18n/index.ts ×3, Languages.vue |
| 🔴 高 | body.rtl 样式选择器永不生效（2处CSS死代码） | he, ar | 极低 | i18n/index.ts |
| 🟠 中 | 瑞典语四层标识符错位（检测返回"sv"但资源是"sv-se"，dayjs也是"sv"） | sv-se | 低 | i18n/index.ts |
| 🟠 中 | 荷兰语比利时正则顺序BUG（nl在nl-be前面导致被截断） | nl-be | 极低 | i18n/index.ts |
| 🟠 中 | dayjs 无映射层导致4种语言日期不本地化 | sv-se, pt-pt, no, fa | 低 | i18n/index.ts |
| 🟠 中 | VideoPlayer 中 nb key 是死代码（挪威语视频控件英语） | no | 极低 | VideoPlayer.vue |
| 🟡 低 | VideoPlayer 缺少 6 种语言包 | ca, fa, is, no, pt, sv-se | 中 | VideoPlayer.vue |
| 🟡 低 | 加泰罗尼亚语 ca 无 detectLocale 分支（只能手动选，不能自动检测） | ca | 极低 | i18n/index.ts |
| 🟡 低 | pt.json 冗余不可达（永远不会被使用） | pt | 低 | 资源文件 |
| 🟡 低 | 希腊语 el 正则写法与其他语言不一致（用 `.*` 而非 `\b`） | el | 极低 | i18n/index.ts |

> **勘误说明**：之前认为 "video.js 语言包大小写可能在 Linux 失败" 是错误判断。经核实，video.js 官方语言包文件名确实使用大写地区代码（`pt-BR.json`、`zh-CN.json` 等），代码中的写法与官方一致，不存在大小写问题。

### 21.1 建议一次性修复代码

```typescript
// i18n/index.ts - 完整版修复

// 1. 增加缺失的 dayjs locale 导入
import("dayjs/locale/fa");  // 波斯语

// 2. 修正 detectLocale()
//    a) 调换 nl-be 和 nl 的顺序（nl-be 必须在 nl 之前）
//    b) 增加 fa 分支
//    c) 增加 ca 分支（加泰罗尼亚语）
//    d) 瑞典语返回 "sv-se" 而非 "sv"（与资源文件名一致）
//    e) 希腊语正则可保持现状，但建议统一为 \b 风格

// 3. 增加 dayjs locale 映射表
const DAYJS_LOCALE_MAP: Record<string, string> = {
  "sv-se": "sv",      // 瑞典语（vue-i18n用sv-se，dayjs用sv）
  "pt-pt": "pt",      // 欧洲葡萄牙语（vue-i18n用pt-pt，dayjs用pt）
  "no": "nb",         // 挪威语（vue-i18n用no，dayjs用nb挪威书面语）
  "fa": "fa",         // 波斯语（需先添加import）
};

// 4. 修正 setLocale() 使用映射表
export function setLocale(locale: string) {
  const dayjsLocale = DAYJS_LOCALE_MAP[locale] || locale;
  dayjs.locale(dayjsLocale);
  i18n.global.locale.value = locale;
}

// 5. 修正 RTL 列表（增加波斯语）
export const rtlLanguages = ["he", "ar", "fa"];

// 6. 修正 setHtmlLocale() 增加 body.rtl 类
export function setHtmlLocale(locale: string) {
  const html = document.documentElement;
  const body = document.body;
  html.lang = locale;
  if (isRtl(locale)) {
    html.dir = "rtl";
    body.classList.add("rtl");
  } else {
    html.dir = "ltr";
    body.classList.remove("rtl");
  }
}
```

```javascript
// Languages.vue - 只增加 fa 选项（ca 已存在于 L20）
const locales = {
  ar: "العربية",
  bg: "български език",
  ca: "Català",
  // ... 中间已有语言 ...
+ fa: "فارسی",
  // ... 其余语言 ...
};
```

```typescript
// VideoPlayer.vue - 修复和补充语言包映射
const languageImports: LanguageImports = {
  ar: () => import("video.js/dist/lang/ar.json"),
  bg: () => import("video.js/dist/lang/bg.json"),
+ ca: () => import("video.js/dist/lang/ca.json"),      // 新增：加泰罗尼亚语
  cs: () => import("video.js/dist/lang/cs.json"),
  // ... 中间已有语言 ...
+ fa: () => import("video.js/dist/lang/fa.json"),      // 新增：波斯语
  // ...
+ is: () => import("video.js/dist/lang/is.json"),      // 新增：冰岛语
  // ...
- nb: () => import("video.js/dist/lang/nb.json"),      // 删除：死代码，key不对
+ no: () => import("video.js/dist/lang/nb.json"),      // 新增：改为正确的 key
  // ...
+ "sv-se": () => import("video.js/dist/lang/sv.json"), // 新增：瑞典语（回退到sv）
  // ...
  // 注意：pt-BR / pt-PT / zh-CN / zh-TW 的大小写不需要改，
  // 这与 video.js 官方文件名一致，所有平台都能正常工作
};
```

### 21.2 修复后预期效果（边界语言）

| 语言 | 修复前问题 | 修复后状态 |
|-----|-----------|:---:|
| fa 波斯语 | 4处缺失，完全不可达 | ✅ 完全正常（含RTL、日期、视频控件） |
| sv-se 瑞典语 | 检测→界面英语，手动→日期英语 | ✅ 检测和手动均正常，日期瑞典语 |
| nl-be 荷兰语比利时 | 自动检测被截断为 nl | ✅ 自动检测和手动均为比利时变体 |
| pt-pt 欧洲葡萄牙语 | 日期显示英语 | ✅ 日期显示欧洲葡语格式 |
| no 挪威语 | 日期英语 + 视频控件英语 | ✅ 日期和视频控件均为挪威语 |
| ca 加泰罗尼亚语 | 只能手动选，不能自动检测 | ✅ 浏览器检测也能自动匹配 |
| he/ar 希伯来语/阿拉伯语 | 2处RTL样式（卡片按钮、终端）不生效 | ✅ 全部RTL样式生效 |
