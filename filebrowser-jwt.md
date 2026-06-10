# File Browser JWT 认证与会话刷新协作分析

## 概述

File Browser 采用 JWT（JSON Web Token）作为无状态身份认证机制，并结合服务端主动提示 + 前端被动刷新的策略实现会话续期。本文从**登录签发**、**请求校验**、**刷新失效**三个核心路径，结合代码分析前后端的协作关系。

---

## 一、JWT 登录签发路径

### 1.1 前端发起登录

用户在 [Login.vue](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/frontend/src/views/Login.vue) 中输入用户名密码后提交：

```typescript
// Login.vue submit()
await auth.login(username.value, password.value, captcha);
```

[utils/auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/frontend/src/utils/auth.ts#L51-L76) 中的 `login()` 函数向 `/api/login` 发送 POST 请求：

```typescript
export async function login(username: string, password: string, recaptcha: string) {
  const res = await fetch(`${baseURL}/api/login`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ username, password, recaptcha }),
  });
  const body = await res.text();
  if (res.status === 200) {
    parseToken(body);  // 解析并存储 JWT
  }
}
```

### 1.2 后端认证并签发 Token

路由在 [http.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/http/http.go#L49) 中注册：

```go
api.Handle("/login", monkey(loginHandler(tokenExpirationTime), ""))
```

[http/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/http/auth.go#L123-L144) 中的 `loginHandler` 处理流程：

1. 通过 `d.store.Auth.Get(d.settings.AuthMethod)` 获取对应认证器（Auther）
2. 调用 `auther.Auth(r, d.store.Users, d.settings, d.server)` 校验用户凭据
3. 认证通过后调用 `printToken()` 签发 JWT

**认证器（Auther）接口**定义在 [auth/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/auth/auth.go#L11-L16)：

```go
type Auther interface {
    Auth(r *http.Request, usr users.Store, stg *settings.Settings, srv *settings.Server) (*users.User, error)
    LoginPage() bool
}
```

系统支持三种认证实现：

| 认证方式 | 文件 | 说明 |
|---------|------|------|
| JSON 认证 | [auth/json.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/auth/json.go) | 默认方式，从请求 Body 解析用户名密码，比对 bcrypt 哈希 |
| Hook 认证 | [auth/hook.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/auth/hook.go) | 调用外部命令脚本进行认证，支持自动创建/更新用户 |
| Proxy 认证 | [auth/proxy.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/auth/proxy.go) | 信任反向代理 HTTP 头中的用户名，自动创建不存在的用户 |
| No Auth | [auth/none.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/auth/none.go) | 无认证，始终返回默认用户 |

### 1.3 Token 签发核心逻辑

[http/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/http/auth.go#L223-L257) 中的 `printToken()` 完成 JWT 构建：

```go
func printToken(w http.ResponseWriter, _ *http.Request, d *data, user *users.User, tokenExpirationTime time.Duration) (int, error) {
    claims := &authToken{
        User: userInfo{ /* 用户信息序列化 */ },
        RegisteredClaims: jwt.RegisteredClaims{
            IssuedAt:  jwt.NewNumericDate(time.Now()),           // 签发时间 iat
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(tokenExpirationTime)), // 过期时间 exp
            Issuer:    "File Browser",
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, err := token.SignedString(d.settings.Key)  // 使用 settings.Key（512位随机密钥）HS256 签名
    w.Write([]byte(signed))
    return 0, nil
}
```

**Token 结构**（[authToken](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/http/auth.go#L42-L45)）：

```go
type authToken struct {
    User userInfo  // 用户信息（ID、权限、偏好设置等）
    jwt.RegisteredClaims  // iat, exp, iss 标准字段
}
```

默认有效期：`DefaultTokenExpirationTime = time.Hour * 2`（2 小时），可通过 `Server.TokenExpirationTime` 配置。

### 1.4 前端接收并存储 Token

[utils/auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/frontend/src/utils/auth.ts#L9-L38) 中的 `parseToken()`：

```typescript
export function parseToken(token: string) {
  const data = jwtDecode<JwtPayload & { user: IUser }>(token);

  // 三处持久化存储
  document.cookie = `auth=${token}; Path=/; SameSite=Strict;`;  // Cookie（供 GET 请求读取）
  localStorage.setItem("jwt", token);                            // localStorage（页面刷新恢复）
  authStore.jwt = token;                                         // Pinia 状态（运行时使用）

  authStore.setUser(data.user);  // 反序列化用户信息到状态

  // 设置自动登出计时器
  const expiresAt = new Date(data.exp! * 1000);
  const timeout = expiresAt.getTime() - Date.now();
  authStore.setLogoutTimer(setSafeTimeout(() => logout("inactivity"), timeout));
}
```

**签发路径完整流程图**：

```
用户输入凭据 → Login.vue submit()
    ↓
auth.login() → POST /api/login
    ↓ (后端)
loginHandler()
    ├─→ 获取 Auther（json/hook/proxy/none）
    ├─→ auther.Auth() 校验凭据 → 返回 *users.User
    └─→ printToken()
         ├─ 构建 authToken{User + RegisteredClaims}
         ├─ HS256 签名（settings.Key）
         └─ 返回纯文本 JWT
    ↓ (前端)
parseToken()
    ├─ jwtDecode 解析 payload
    ├─ 存储：Cookie + localStorage + Pinia
    ├─ 设置用户状态
    └─ 设置到期自动登出计时器
```

---

## 二、JWT 请求校验路径

### 2.1 前端附带 Token

所有 API 请求通过 [api/utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/frontend/src/api/utils.ts#L17-L64) 中的 `fetchURL()` 统一发送，自动在 Header 中携带 JWT：

```typescript
export async function fetchURL(url: string, opts: ApiOpts, auth = true): Promise<Response> {
  const authStore = useAuthStore();
  res = await fetch(`${baseURL}${url}`, {
    headers: {
      "X-Auth": authStore.jwt,  // JWT 放入 X-Auth 请求头
      ...headers,
    },
    ...rest,
  });
  // ... 响应处理
}
```

### 2.2 后端 Token 提取与校验

受保护的路由使用 `withUser` 中间件（或其变体 `withAdmin`）。核心校验逻辑在 [http/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/http/auth.go#L85-L111)：

```go
func withUser(fn handleFunc) handleFunc {
    return func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        keyFunc := func(_ *jwt.Token) (interface{}, error) {
            return d.settings.Key, nil  // 获取签名密钥
        }

        var tk authToken
        p := jwt.NewParser(
            jwt.WithValidMethods([]string{jwt.SigningMethodHS256.Alg()}),  // 仅允许 HS256
            jwt.WithExpirationRequired(),                                   // 强制要求 exp 字段
        )
        token, err := request.ParseFromRequest(r, &extractor{}, keyFunc,
            request.WithClaims(&tk), request.WithParser(p))

        if (err != nil || !token.Valid) && !renewableErr(err, d) {
            return http.StatusUnauthorized, nil  // 401 未授权
        }

        // ... 刷新检测（见下节）

        d.user, err = d.store.Users.Get(d.server.Root, tk.User.ID)  // 加载最新用户数据
        return fn(w, r, d)
    }
}
```

### 2.3 Token 提取器（Extractor）

自定义的 [extractor](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/http/auth.go#L47-L67) 按优先级从两处读取 Token：

```go
func (e extractor) ExtractToken(r *http.Request) (string, error) {
    // 1. 优先从 X-Auth 请求头获取（所有请求）
    token, _ := request.HeaderExtractor{"X-Auth"}.ExtractToken(r)
    if token != "" && strings.Count(token, ".") == 2 {
        return token, nil
    }

    // 2. GET 请求回退到 Cookie 中的 auth（用于直接点击下载链接等场景）
    if r.Method == http.MethodGet {
        cookie, _ := r.Cookie("auth")
        if cookie != nil && strings.Count(cookie.Value, ".") == 2 {
            return cookie.Value, nil
        }
    }
    return "", request.ErrNoTokenInRequest
}
```

> **设计意图**：浏览器直接发起的 GET 请求（如 `<a href="...">` 下载、`<img src="...">` 预览）无法自定义 Header，此时通过 Cookie 作为兜底传输通道。

### 2.4 Proxy 认证的过期宽容

[renewableErr](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/http/auth.go#L69-L83) 对 Proxy 认证模式做了特殊处理：

```go
func renewableErr(err error, d *data) bool {
    // 仅 Proxy 认证 + 自定义登出页 + Token 刚好过期 时，不立即返回 401
    if d.settings.AuthMethod != fbAuth.MethodProxyAuth || err == nil {
        return false
    }
    if d.settings.LogoutPage == settings.DefaultLogoutPage {
        return false
    }
    if !errors.Is(err, jwt.ErrTokenExpired) {
        return false
    }
    return true  // 让请求继续通过，依赖外部代理会话管理
}
```

**校验路径完整流程图**：

```
前端 API 调用 → fetchURL()
    ↓
请求头添加 X-Auth: <jwt>
    ↓ (后端)
withUser 中间件
    ├─→ extractor.ExtractToken()
    │    ├─ 优先 X-Auth Header
    │    └─ GET 请求兜底 Cookie(auth)
    ├─→ jwt.NewParser(HS256 only, exp required)
    ├─→ request.ParseFromRequest() 验证签名 + exp
    │    └─ Proxy 认证过期宽容（renewableErr）
    ├─→ 检测是否需要刷新（见第三节）
    ├─→ d.store.Users.Get() 从 DB 加载最新用户
    └─→ 执行业务处理函数
```

---

## 三、JWT 刷新与失效路径

### 3.1 服务端主动提示刷新

`withUser` 中间件在校验 Token 有效后，会检测两种需要刷新的条件（[http/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/http/auth.go#L98-L103)）：

```go
expiresSoon := tk.ExpiresAt != nil && time.Until(tk.ExpiresAt.Time) < time.Hour
updated := tk.IssuedAt != nil && tk.IssuedAt.Unix() < d.store.Users.LastUpdate(tk.User.ID)

if expiresSoon || updated {
    w.Header().Add("X-Renew-Token", "true")  // 通过响应头提示前端
}
```

| 触发条件 | 判断逻辑 | 设计意图 |
|---------|---------|---------|
| **即将过期** | Token 剩余有效期 < 1 小时 | 提前续期，避免用户操作中途掉线 |
| **用户信息变更** | Token 签发时间 (iat) < 用户最后更新时间 | 确保前端持有最新权限/偏好设置 |

**用户更新时间戳机制**：[users/storage.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/users/storage.go#L76-L90) 在每次 `Update()` 时记录：

```go
func (s *Storage) Update(user *User, fields ...string) error {
    // ... 更新数据库
    s.mux.Lock()
    s.updated[user.ID] = time.Now().Unix()  // 内存中记录更新时间戳
    s.mux.Unlock()
    return nil
}
```

### 3.2 前端被动响应刷新

[api/utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/frontend/src/api/utils.ts#L45-L47) 在收到响应后检查 Header：

```typescript
if (auth && res.headers.get("X-Renew-Token") === "true") {
    await renew(authStore.jwt);  // 携带旧 Token 请求新 Token
}
```

[utils/auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/frontend/src/utils/auth.ts#L78-L96) 中的 `renew()`：

```typescript
export async function renew(jwt: string) {
  const res = await fetch(`${baseURL}/api/renew`, {
    method: "POST",
    headers: { "X-Auth": jwt },  // 用旧 Token 认证
  });
  const body = await res.text();
  if (res.status === 200) {
    parseToken(body);  // 重新解析并存储新 Token
  }
}
```

### 3.3 后端 Renew 端点

[http/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/http/auth.go#L216-L221) 中的 `renewHandler`：

```go
func renewHandler(tokenExpireTime time.Duration) handleFunc {
    return withUser(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        w.Header().Set("X-Renew-Token", "false")  // 清除刷新提示
        return printToken(w, r, d, d.user, tokenExpireTime)  // 签发全新 Token
    })
}
```

> **关键点**：`renewHandler` 使用 `withUser` 包裹，意味着必须持有效 Token 才能刷新——这是一种**惰性刷新**策略：只要用户仍在活跃（产生 API 请求），会话就会自动续期。

### 3.4 失效处理：401 与自动登出

当 Token 真正失效（过期、签名错误、被篡改等）时：

1. **后端**：`withUser` 返回 `http.StatusUnauthorized`（401）
2. **前端**：[api/utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/frontend/src/api/utils.ts#L56-L58) 检测到 401 自动登出：

```typescript
if (auth && res.status == 401) {
    logout();  // 清除状态并跳转登录页
}
```

[utils/auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/frontend/src/utils/auth.ts#L118-L141) 中的 `logout()`：

```typescript
export function logout(reason?: string) {
  document.cookie = "auth=; Max-Age=0; Path=/; SameSite=Strict;";  // 清除 Cookie
  authStore.clearUser();                                            // 清除 Pinia 状态
  localStorage.setItem("jwt", "");                                  // 清除 localStorage
  // 跳转到登录页或自定义登出页
}
```

此外，前端还通过 `parseToken()` 中设置的 `logoutTimer`，在 Token 到期时刻主动触发登出（原因标记为 `"inactivity"`）。

### 3.5 页面初始化的静默续期

[router/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/frontend/src/router/index.ts#L156-L176) 在首次路由时调用 `initAuth()`：

```typescript
async function initAuth() {
  if (loginPage) {
    await validateLogin();  // 有登录页：尝试用 localStorage 中的 JWT 续期
  } else {
    await login("", "", "");  // 无登录页（no auth / proxy）：直接获取 Token
  }
}
```

[utils/auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/frontend/src/utils/auth.ts#L40-L49) 中的 `validateLogin()`：

```typescript
export async function validateLogin() {
  if (localStorage.getItem("jwt")) {
    await renew(<string>localStorage.getItem("jwt"));  // 用存储的旧 Token 换新 Token
  }
}
```

> 这意味着：只要 localStorage 中还存有未过期的 JWT，用户刷新页面时就会静默完成一次续期，无需重新登录。

**刷新失效完整流程图**：

```
┌─ 服务端检测 ─────────────────────────────────────┐
│ withUser 中间件                                   │
│   ├─ expiresSoon: 剩余 < 1h ?                     │
│   └─ updated: iat < LastUpdate ?                  │
│   └─→ 是 → 响应头 X-Renew-Token: true             │
└───────────────────────────────────────────────────┘
                         ↓
┌─ 前端响应 ───────────────────────────────────────┐
│ fetchURL() 检测响应头                             │
│   X-Renew-Token === "true"                        │
│   └─→ await renew(authStore.jwt)                  │
│        → POST /api/renew (携带旧 JWT)             │
└───────────────────────────────────────────────────┘
                         ↓
┌─ 后端 Renew ─────────────────────────────────────┐
│ renewHandler (withUser 包裹)                      │
│   ├─ 验证旧 Token 有效                            │
│   ├─ 从 DB 加载最新用户                           │
│   ├─ printToken() 签发新 Token (iat/exp 重置)     │
│   └─ X-Renew-Token: false                         │
└───────────────────────────────────────────────────┘
                         ↓
┌─ 前端接收 ───────────────────────────────────────┐
│ parseToken()                                      │
│   ├─ 更新 Cookie / localStorage / Pinia           │
│   ├─ 更新用户信息（权限/偏好变更同步）              │
│   └─ 重置 logoutTimer                             │
└───────────────────────────────────────────────────┘

失效场景：
  Token 过期/无效 → withUser 返回 401
    → fetchURL 检测 401 → logout()
    → 清除所有存储 → 跳转 /login

页面刷新场景：
  router.beforeResolve → initAuth()
    → validateLogin() → renew(localStorage.jwt)
    → 静默续期成功 → 用户无感
```

---

## 四、协作关系总结

### 4.1 核心设计要点

| 设计决策 | 实现方式 | 优势 |
|---------|---------|------|
| **无状态认证** | JWT 自包含用户信息，服务端不存会话 | 水平扩展友好，无需共享 session 存储 |
| **惰性刷新** | 正常请求响应头捎带刷新提示，不单独心跳 | 节省请求，只有活跃用户才续期 |
| **双触发刷新** | 即将过期 + 用户信息变更 | 兼顾会话续期与数据一致性 |
| **双通道传输** | X-Auth Header + Cookie (GET fallback) | 兼顾 SPA API 调用与浏览器原生资源请求 |
| **前端三重存储** | Pinia + localStorage + Cookie | 运行时高效、刷新可恢复、GET 请求兜底 |

### 4.2 前后端交互时序

```
前端                                    后端
 │                                       │
 │── POST /api/login (用户名密码) ──────→│
 │                                       │── auther.Auth() 校验
 │                                       │── printToken() 签发 JWT
 │←────────── 200 JWT 纯文本 ───────────│
 │── parseToken()                        │
 │    (存三处 + 设登出计时器)             │
 │                                       │
 │── GET /api/resources/... ────────────→│
 │    X-Auth: <jwt>                      │
 │                                       │── withUser 校验
 │                                       │    ├─ 提取 Token
 │                                       │    ├─ 验签+过期检查
 │                                       │    ├─ expiresSoon?
 │                                       │    └─ updated?
 │←──── 200 + X-Renew-Token: true ──────│
 │                                       │
 │── detect X-Renew-Token: true          │
 │── POST /api/renew ───────────────────→│
 │    X-Auth: <old jwt>                  │
 │                                       │── withUser 校验旧 Token
 │                                       │── printToken() 发新 Token
 │←────────── 200 NEW JWT ──────────────│
 │── parseToken()                        │
 │    (更新三处 + 重置计时器)             │
 │                                       │
 │── (时间流逝，Token 过期)              │
 │── GET /api/resources/... ────────────→│
 │                                       │── withUser → 401 Unauthorized
 │←────────────── 401 ──────────────────│
 │── detect 401 → logout()               │
 │    (清除存储 + 跳转登录)              │
```

### 4.3 关键文件索引

| 角色 | 文件 | 核心函数/类型 |
|------|------|--------------|
| 后端签发 | [http/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/http/auth.go) | `loginHandler`, `renewHandler`, `printToken`, `authToken` |
| 后端校验 | [http/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/http/auth.go) | `withUser`, `extractor.ExtractToken`, `renewableErr` |
| 认证接口 | [auth/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/auth/auth.go) | `Auther` interface |
| JSON 认证 | [auth/json.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/auth/json.go) | `JSONAuth.Auth` |
| Hook 认证 | [auth/hook.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/auth/hook.go) | `HookAuth.Auth` |
| Proxy 认证 | [auth/proxy.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/auth/proxy.go) | `ProxyAuth.Auth` |
| 用户更新追踪 | [users/storage.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/users/storage.go) | `Storage.Update`, `Storage.LastUpdate` |
| 路由注册 | [http/http.go](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/http/http.go) | `/api/login`, `/api/renew` |
| 前端登录/续期 | [frontend/src/utils/auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/frontend/src/utils/auth.ts) | `login`, `renew`, `parseToken`, `logout`, `validateLogin` |
| 前端请求封装 | [frontend/src/api/utils.ts](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/frontend/src/api/utils.ts) | `fetchURL` (X-Auth 注入 + X-Renew-Token 响应) |
| 前端状态 | [frontend/src/stores/auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/frontend/src/stores/auth.ts) | `useAuthStore` |
| 前端路由守卫 | [frontend/src/router/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/165-filebrowser/frontend/src/router/index.ts) | `initAuth`, `beforeResolve` |
