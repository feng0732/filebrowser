# File Browser JWT 认证与会话刷新协作分析

## 概述

File Browser 采用 JWT（JSON Web Token）作为无状态身份认证机制，并结合服务端主动提示 + 前端被动刷新的策略实现会话续期。本文从**登录签发**、**请求校验**、**刷新失效**三个核心路径，结合代码分析前后端的协作关系，并重点说明 Proxy 认证 + 自定义登出页场景下的特殊分支处理。

---

## 一、JWT 登录签发路径

### 1.1 前端发起登录

用户在 [Login.vue](frontend/src/views/Login.vue) 中输入用户名密码后提交：

```typescript
// Login.vue submit()
await auth.login(username.value, password.value, captcha);
```

[utils/auth.ts](frontend/src/utils/auth.ts#L51-L76) 中的 `login()` 函数向 `/api/login` 发送 POST 请求：

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

路由在 [http.go](http/http.go#L49) 中注册：

```go
api.Handle("/login", monkey(loginHandler(tokenExpirationTime), ""))
```

[http/auth.go](http/auth.go#L123-L144) 中的 `loginHandler` 处理流程：

1. 通过 `d.store.Auth.Get(d.settings.AuthMethod)` 获取对应认证器（Auther）
2. 调用 `auther.Auth(r, d.store.Users, d.settings, d.server)` 校验用户凭据
3. 认证通过后调用 `printToken()` 签发 JWT

**认证器（Auther）接口**定义在 [auth/auth.go](auth/auth.go#L11-L16)：

```go
type Auther interface {
    Auth(r *http.Request, usr users.Store, stg *settings.Settings, srv *settings.Server) (*users.User, error)
    LoginPage() bool
}
```

系统支持四种认证实现：

| 认证方式 | 文件 | 说明 |
|---------|------|------|
| JSON 认证 | [auth/json.go](auth/json.go) | 默认方式，从请求 Body 解析用户名密码，比对 bcrypt 哈希 |
| Hook 认证 | [auth/hook.go](auth/hook.go) | 调用外部命令脚本进行认证，支持自动创建/更新用户 |
| Proxy 认证 | [auth/proxy.go](auth/proxy.go) | 信任反向代理 HTTP 头中的用户名，自动创建不存在的用户 |
| No Auth | [auth/none.go](auth/none.go) | 无认证，始终返回默认用户 |

**Proxy 认证补充**：[ProxyAuth.LoginPage()](auth/proxy.go#L68-L71) 返回 `false`，意味着前端路由层会跳过登录页，直接走 `initAuth()` 中的 `login("", "", "")` 分支获取 Token。

### 1.3 Token 签发核心逻辑

[http/auth.go](http/auth.go#L223-L257) 中的 `printToken()` 完成 JWT 构建：

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

**Token 结构**（[authToken](http/auth.go#L42-L45)）：

```go
type authToken struct {
    User userInfo  // 用户信息（ID、权限、偏好设置等）
    jwt.RegisteredClaims  // iat, exp, iss 标准字段
}
```

默认有效期：`DefaultTokenExpirationTime = time.Hour * 2`（2 小时），可通过 `Server.TokenExpirationTime` 配置。

### 1.4 前端接收并存储 Token

[utils/auth.ts](frontend/src/utils/auth.ts#L9-L38) 中的 `parseToken()`，包含 **Proxy 认证 + 自定义登出页的计时器禁用分支**：

```typescript
export function parseToken(token: string) {
  const data = jwtDecode<JwtPayload & { user: IUser }>(token);

  // 三处持久化存储
  document.cookie = `auth=${token}; Path=/; SameSite=Strict;`;  // Cookie（供 GET 请求读取）
  localStorage.setItem("jwt", token);                            // localStorage（页面刷新恢复）
  authStore.jwt = token;                                         // Pinia 状态（运行时使用）

  authStore.setUser(data.user);  // 反序列化用户信息到状态

  // ─── Proxy 认证 + 自定义登出页：禁用空闲登出计时器 ───
  // 反向代理场景下，会话生命周期由外部代理（如 OIDC/SAML）统一管理，
  // File Browser 的 JWT 只是内部标识，不应根据自己的 exp 主动登出。
  if (logoutPage !== "/login" && authMethod === "proxy") {
    console.warn("idle timeout disabled with proxy auth and custom logout");
    return;  // 提前返回，不设置 logoutTimer
  }

  if (authStore.logoutTimer) {
    clearTimeout(authStore.logoutTimer);
  }

  // 设置自动登出计时器
  const expiresAt = new Date(data.exp! * 1000);
  const timeout = expiresAt.getTime() - Date.now();
  authStore.setLogoutTimer(
    setSafeTimeout(() => logout("inactivity"), timeout)
  );
}
```

**判断条件来源**：`authMethod` 和 `logoutPage` 均来自 [utils/constants.ts](frontend/src/utils/constants.ts#L12-L14)，由后端在 HTML 模板中注入到全局 `window.FileBrowser` 对象：

```typescript
const authMethod = window.FileBrowser.AuthMethod;
const logoutPage: string = window.FileBrowser.LogoutPage;
```

| 组合 | authMethod | logoutPage | 是否禁用计时器 |
|------|-----------|-----------|--------------|
| 默认 JSON 认证 | `"json"` | `"/login"` | ❌ 不禁用 |
| Proxy + 默认登出页 | `"proxy"` | `"/login"` | ❌ 不禁用 |
| Proxy + 自定义登出页 | `"proxy"` | `"https://sso.example.com/logout"` | ✅ **禁用** |
| 其他 + 自定义登出页 | `"json"` | `"https://..."` | ❌ 不禁用 |

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
    └─ 分支判断：
         ├─ authMethod=="proxy" && logoutPage!="/login"
         │   └─→ 禁用空闲登出计时器（return，不设 logoutTimer）
         └─ 其他 → 设置到期自动登出计时器
```

---

## 二、JWT 请求校验路径

### 2.1 前端附带 Token

所有 API 请求通过 [api/utils.ts](frontend/src/api/utils.ts#L17-L64) 中的 `fetchURL()` 统一发送，自动在 Header 中携带 JWT：

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

受保护的路由使用 `withUser` 中间件（或其变体 `withAdmin`）。核心校验逻辑在 [http/auth.go](http/auth.go#L85-L111)，包含 **Proxy + 自定义登出页场景下的过期令牌宽容分支**：

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

        // ─── 核心校验 + Proxy 过期宽容 ───
        // jwt-go/v5 只要签名正确，即使过期也会把 claims 解析进 tk，
        // 只是 token.Valid==false 且 err==jwt.ErrTokenExpired。
        // 此时 renewableErr() 可以决定是否放行。
        if (err != nil || !token.Valid) && !renewableErr(err, d) {
            return http.StatusUnauthorized, nil  // 401 未授权
        }

        // ─── 刷新检测（即使过期宽容通过也执行，见下节） ───
        expiresSoon := tk.ExpiresAt != nil && time.Until(tk.ExpiresAt.Time) < time.Hour
        updated := tk.IssuedAt != nil && tk.IssuedAt.Unix() < d.store.Users.LastUpdate(tk.User.ID)
        if expiresSoon || updated {
            w.Header().Add("X-Renew-Token", "true")
        }

        d.user, err = d.store.Users.Get(d.server.Root, tk.User.ID)  // 加载最新用户数据
        return fn(w, r, d)
    }
}
```

### 2.3 Token 提取器（Extractor）

自定义的 [extractor](http/auth.go#L47-L67) 按优先级从两处读取 Token：

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

### 2.4 Proxy 认证的过期宽容（核心分支）

[renewableErr](http/auth.go#L69-L83) 对 Proxy 认证模式做了特殊处理，**允许过期 Token 在特定条件下继续通过校验**：

```go
func renewableErr(err error, d *data) bool {
    // 条件 1：认证方式必须是 Proxy 认证
    if d.settings.AuthMethod != fbAuth.MethodProxyAuth || err == nil {
        return false
    }

    // 条件 2：登出页不是默认的 /login（即配置了自定义登出页）
    // settings.DefaultLogoutPage 定义见 settings/settings.go L14
    if d.settings.LogoutPage == settings.DefaultLogoutPage {
        return false
    }

    // 条件 3：错误类型恰好是 Token 过期（不是签名错误、篡改等）
    if !errors.Is(err, jwt.ErrTokenExpired) {
        return false
    }

    // 三个条件同时满足：放行过期 Token，信任外部代理的会话管理
    return true
}
```

**三层条件同时满足才能放行**，缺一不可：

| 条件 | 变量/表达式 | 代码行 | 说明 |
|-----|-----------|-------|------|
| 认证方式 | `AuthMethod == "proxy"` | [http/auth.go L70](http/auth.go#L70) | 仅 Proxy 模式信任外部会话 |
| 自定义登出页 | `LogoutPage != "/login"` | [http/auth.go L74](http/auth.go#L74) | 有外部 SSO 登出端点才启用宽容 |
| 仅过期错误 | `errors.Is(err, ErrTokenExpired)` | [http/auth.go L78](http/auth.go#L78) | 签名错误/篡改仍会被拦截 |

**过期宽容的设计意图**：
- 在反向代理 + SSO 架构中，真正的会话边界在代理层（例如 Nginx + OIDC 会在请求到达 File Browser 前校验 SSO Cookie）
- File Browser 的 JWT 只是内部的用户标识，其 `exp` 不应阻断正常请求
- 如果 JWT 过期就直接 401，会导致用户的外部 SSO 会话仍有效但 File Browser 已登出的不一致状态
- 但签名正确性仍严格校验，防止伪造 Token

**过期令牌如何续签**：
即使通过了 `renewableErr` 的宽容放行，`withUser` 中的刷新检测逻辑仍然执行：
1. **expiresSoon**：对已过期 Token，`time.Until(expiredTime)` 返回**负数**，而**负数 < 1 小时恒成立**，因此 `expiresSoon=true`
2. **updated**：用户信息变更时仍会触发 `X-Renew-Token: true`
3. 以上两个条件**任一满足**就会设置响应头 `X-Renew-Token: true`
4. 前端 `fetchURL()` 收到响应头后调用 `renew()` → `POST /api/renew`
5. **关键点**：`renewHandler` 也被 `withUser` 包裹，而 `withUser` 对 Proxy + 自定义登出页场景下的过期 Token 是放行的，因此**过期 Token 也能成功换取新 Token**

```
请求携带过期 JWT（proxy + 自定义登出页场景）
    ↓
withUser 中间件
    ├─ ParseFromRequest：签名正确但 err=ErrTokenExpired
    ├─ renewableErr() → true（三条件均满足）
    ├─ 跳过 401，继续执行
    ├─ expiresSoon=true（过期返回负数，负数 < 1h 恒成立 ★）
    ├─ updated 判断（用户有变更也触发，双保险）
    ├─ d.store.Users.Get(ID) → 加载用户
    └─ 业务函数正常返回 + X-Renew-Token:true（几乎必然触发）
    ↓
前端 fetchURL() 检测 X-Renew-Token:true
    └─→ renew(过期JWT)
         └─→ POST /api/renew（仍由 withUser 包裹，同样走过期宽容）
              └─→ printToken() → 返回全新未过期 JWT
```

> 这就是**过期令牌续签**的完整链条：服务端对过期 Token 放行（`renewableErr`）→ `expiresSoon=true` 几乎必然触发刷新提示（过期后恒成立）→ 前端发起 renew → renewHandler 凭借同样的过期宽容机制通过校验并签发新 Token。
>
> **修正前的错误理解**：曾认为过期 Token 的 `time.Until` 为负不会触发 `< time.Hour`，实际是**负数一定小于正数**，所以过期后 `expiresSoon` 恒为 `true`。

### 2.5 校验路径完整流程图

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
    ├─→ request.ParseFromRequest() 解析出 tk + err
    ├─→ 校验分支：
    │    ├─ err==nil && valid==true → 直接通过
    │    └─ 过期/无效 → 检查 renewableErr(err, d)：
    │         ├─ 非 proxy → false → 401 Unauthorized
    │         ├─ logoutPage=="default" → false → 401
    │         ├─ 非 ErrTokenExpired → false → 401（签名错误不宽容）
    │         └─ proxy + 自定义登出 + 仅过期 → true → ★ 放行过期 Token
    ├─→ 刷新检测（expiresSoon || updated）→ X-Renew-Token 响应头
    │    （过期 Token 的 expiresSoon 恒为 true，负数 < 1h）
    ├─→ d.store.Users.Get() 从 DB 加载最新用户
    └─→ 执行业务处理函数
```

---

## 三、JWT 刷新与失效路径

### 3.1 服务端主动提示刷新

`withUser` 中间件在校验通过（含过期宽容）后，会检测两种需要刷新的条件（[http/auth.go](http/auth.go#L98-L103)）：

```go
expiresSoon := tk.ExpiresAt != nil && time.Until(tk.ExpiresAt.Time) < time.Hour
updated := tk.IssuedAt != nil && tk.IssuedAt.Unix() < d.store.Users.LastUpdate(tk.User.ID)

if expiresSoon || updated {
    w.Header().Add("X-Renew-Token", "true")  // 通过响应头提示前端
}
```

| 触发条件 | 判断逻辑 | 设计意图 | Proxy+自定义登出页场景下 |
|---------|---------|---------|----------------------|
| **即将过期（含已过期）** | `time.Until(ExpiresAt) < 1h` | 提前续期，避免用户操作中途掉线 | **过期后恒触发**（`time.Until` 返回负数，**负数 < 1 小时恒成立**），这是过期 Token 续签的**主要触发路径** |
| **用户信息变更** | Token 签发时间 (iat) < 用户最后更新时间 | 确保前端持有最新权限/偏好设置 | 仍会触发，与 expiresSoon 形成**双保险** |

> **关键修正**：`time.Until(t)` 返回 `t - now`，当 `t` 已过时时返回负值。而**任何负数都小于 1 小时**，因此：
> ```go
> time.Until(已过期的时间)  // 返回 -1h30m 这样的负数
> -1h30m < time.Hour        // 结果为 true！
> ```

**用户更新时间戳机制**：[users/storage.go](users/storage.go#L76-L90) 在每次 `Update()` 时记录：

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

[api/utils.ts](frontend/src/api/utils.ts#L45-L47) 在收到响应后检查 Header：

```typescript
if (auth && res.headers.get("X-Renew-Token") === "true") {
    await renew(authStore.jwt);  // 携带旧 Token 请求新 Token（旧 Token 可以是过期的）
}
```

[utils/auth.ts](frontend/src/utils/auth.ts#L78-L96) 中的 `renew()`：

```typescript
export async function renew(jwt: string) {
  const res = await fetch(`${baseURL}/api/renew`, {
    method: "POST",
    headers: { "X-Auth": jwt },  // 用旧 Token 认证（可以过期，服务端宽容）
  });
  const body = await res.text();
  if (res.status === 200) {
    parseToken(body);  // 重新解析并存储新 Token，重置计时器逻辑
  }
}
```

### 3.3 后端 Renew 端点

[http/auth.go](http/auth.go#L216-L221) 中的 `renewHandler`：

```go
func renewHandler(tokenExpireTime time.Duration) handleFunc {
    return withUser(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        w.Header().Set("X-Renew-Token", "false")  // 清除刷新提示
        return printToken(w, r, d, d.user, tokenExpireTime)  // 签发全新 Token（参数名与形参一致）
    })
}
```

> **关键点**：`renewHandler` 使用 `withUser` 包裹，意味着必须能通过 `withUser` 校验才能刷新。在 **Proxy + 自定义登出页**场景下，过期 Token 也能通过 `withUser`（靠 `renewableErr`），因此过期 Token 可以成功换取新 Token——这就是**过期令牌续签**的核心机制。
>
> 对其他认证方式，`renewHandler` 要求持**有效（未过期）** Token 才能刷新——这是一种**惰性刷新**策略：只要用户仍在活跃（产生 API 请求），会话就会在过期前自动续期。

### 3.4 失效处理：401 与自动登出

当 Token 真正失效（过期但不满足宽容条件、签名错误、被篡改等）时：

1. **后端**：`withUser` 返回 `http.StatusUnauthorized`（401）
2. **前端**：[api/utils.ts](frontend/src/api/utils.ts#L56-L58) 检测到 401 自动登出：

```typescript
if (auth && res.status == 401) {
    logout();  // 清除状态并跳转登录页
}
```

[utils/auth.ts](frontend/src/utils/auth.ts#L118-L141) 中的 `logout()`，包含**自定义登出页跳转分支**：

```typescript
export function logout(reason?: string) {
  document.cookie = "auth=; Max-Age=0; Path=/; SameSite=Strict;";  // 清除 Cookie
  authStore.clearUser();                                            // 清除 Pinia 状态
  localStorage.setItem("jwt", "");                                  // 清除 localStorage

  if (noAuth) {
    window.location.reload();
  } else if (logoutPage !== "/login") {
    // ─── 自定义登出页：跳转到外部 SSO 登出端点 ───
    // Proxy 认证场景下通常配置为外部系统统一登出 URL，
    // 确保用户同时从外部 SSO 和 File Browser 登出。
    document.location.href = `${logoutPage}`;
  } else {
    // 默认登出页：带 reason 参数跳转内部登录页
    if (typeof reason === "string" && reason.trim() !== "") {
      router.push({ path: "/login", query: { "logout-reason": reason } });
    } else {
      router.push({ path: "/login" });
    }
  }
}
```

此外，**非 Proxy+自定义登出页**场景下，前端还通过 `parseToken()` 中设置的 `logoutTimer`，在 Token 到期时刻主动触发登出（原因标记为 `"inactivity"`）。

### 3.5 页面初始化的静默续期

[router/index.ts](frontend/src/router/index.ts#L156-L176) 在首次路由时调用 `initAuth()`：

```typescript
async function initAuth() {
  if (loginPage) {
    // 有登录页（JSON/Hook/Proxy+默认登出页）：用 localStorage 中 JWT 续期
    await validateLogin();
  } else {
    // 无登录页（Proxy+自定义登出页 / No Auth）：
    // 直接 POST /api/login 空凭据，后端通过 ProxyAuth.Auth() 从请求头
    // 读取代理注入的用户名，自动签发新 JWT。
    await login("", "", "");
  }
}
```

> **注意**：`loginPage` 变量（[utils/constants.ts L14](frontend/src/utils/constants.ts#L14)）来自后端的 `Auther.LoginPage()` 返回值。Proxy 认证下 [ProxyAuth.LoginPage()](auth/proxy.go#L68-L71) 返回 `false`，因此**只要是 Proxy 认证就走 `login("","","")` 分支**，不管 logoutPage 是否自定义。

[utils/auth.ts](frontend/src/utils/auth.ts#L40-L49) 中的 `validateLogin()`：

```typescript
export async function validateLogin() {
  if (localStorage.getItem("jwt")) {
    await renew(<string>localStorage.getItem("jwt"));  // 用存储的旧 Token 换新 Token
  }
}
```

> 对于满足宽容条件的场景，即使 localStorage 中的 JWT 已过期，`renew()` 也能成功（靠 `renewableErr` 放行 + `renewHandler` 签发新 Token），实现真正的**刷新页面无感续期**。

**刷新失效完整流程图**：

```
┌─ 服务端检测 ───────────────────────────────────────────────┐
│ withUser 中间件                                             │
│   ├─ expiresSoon: 剩余 < 1h ?                               │
│   └─ updated: iat < LastUpdate ?                            │
│   └─→ 是 → 响应头 X-Renew-Token: true                       │
│                                                             │
│ 过期宽容分支（proxy + 自定义登出页 + ErrTokenExpired）：      │
│   └─→ renewableErr=true → 跳过 401，仍执行上述刷新检测       │
│      （expiresSoon 恒成立 + updated 双保险触发续签）          │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─ 前端响应 ─────────────────────────────────────────────────┐
│ fetchURL() 检测响应头                                       │
│   X-Renew-Token === "true"                                  │
│   └─→ await renew(authStore.jwt)                            │
│        → POST /api/renew（携带旧 JWT，可过期）               │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─ 后端 Renew ───────────────────────────────────────────────┐
│ renewHandler (withUser 包裹)                                │
│   ├─ 再次走 withUser 校验逻辑                               │
│   │   ├─ 有效 Token → 直接通过                              │
│   │   └─ 过期 Token → 满足 renewableErr 三条件则通过        │
│   ├─ 从 DB 加载最新用户                                     │
│   ├─ printToken() 签发新 Token (iat/exp 全重置)             │
│   └─ X-Renew-Token: false                                   │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─ 前端接收 parseToken() ────────────────────────────────────┐
│   ├─ 更新 Cookie / localStorage / Pinia                     │
│   ├─ 更新用户信息（权限/偏好变更同步）                        │
│   └─ 计时器分支：                                            │
│        ├─ proxy + 自定义登出页 → 禁用，return               │
│        └─ 其他 → 重置 logoutTimer                            │
└─────────────────────────────────────────────────────────────┘

失效场景（非宽容条件命中）：
  Token 过期但不满足宽容三条件 / 签名错误 / 被篡改
    → withUser 返回 401
    → fetchURL 检测 401 → logout()
    → 清除所有存储
    → logout() 跳转分支：
         ├─ logoutPage != "/login" → 跳外部登出 URL
         └─ 默认 → 跳内部 /login（带 logout-reason）

页面刷新场景：
  router.beforeResolve → initAuth()
    ├─ loginPage==true → validateLogin() → renew(localStorage.jwt)
    │   └─ 过期 JWT + 宽容三条件 → 仍可续签成功
    └─ loginPage==false (proxy/noauth) → login("","","")
         └─ ProxyAuth 从代理头取用户名 → 自动签发新 JWT
```

---

## 四、Proxy 认证 + 自定义登出页分支专项总结

### 4.1 三处代码改动 + 一处恒成立逻辑的协同关系

Proxy 认证 + 自定义登出页配置同时触发前后端**三处独立但互相配合**的逻辑分支，再结合 `expiresSoon` 的固有行为，共同完成过期令牌续签：

| 位置 | 代码点 | 作用 |
|------|-------|------|
| 后端校验 | [`renewableErr()`](http/auth.go#L69-L83) | 过期 JWT 仍可通过校验，不返回 401 |
| 后端刷新 | [`expiresSoon`](http/auth.go#L98) | 过期后恒为 `true`（负数 < 1h），触发 `X-Renew-Token: true` |
| 前端解析 | [`parseToken()` early return](frontend/src/utils/auth.ts#L22-L25) | 禁用 JWT 自身的空闲登出计时器，交还给外部 SSO 管理 |
| 前端登出 | [`logout()` redirect](frontend/src/utils/auth.ts#L127-L128) | 登出时跳转到外部 SSO 统一登出端点 |

> **注意**：`renewHandler` 的形参名是 `tokenExpireTime`（无 'a'），与 `printToken` 的 `tokenExpirationTime`（有 'a'）不同，调用时需保持参数名一致：`printToken(..., tokenExpireTime)`。

### 4.2 为什么需要三处同时改动？

1. **只改后端（renewableErr）不改前端计时器**：
   前端 `logoutTimer` 会在 `exp` 时刻准时调用 `logout("inactivity")`，即使后端仍接受过期 Token，用户仍会被前端强制登出——**单边修改无效**。

2. **只改前端计时器不改后端校验**：
   前端不主动登出了，但 API 请求到达后端后，`withUser` 仍返回 401，`fetchURL` 检测到 401 照样调用 `logout()`——**单边修改无效**。

3. **只改前两处不改 logout 跳转**：
   用户从外部 SSO 登出后，File Browser 登出仍只跳内部 `/login`，用户得手动从外部 SSO 登出，形成**幽灵会话**。

因此三处必须**同时配置**（authMethod=proxy + logoutPage=外部URL）才能形成完整的 SSO 集成体验。

### 4.3 Proxy + 自定义登出页的完整会话生命周期

```
                     外部反向代理 (Nginx + OIDC)
                     ┌───────────────────────────────┐
 用户访问             │                               │
 ───────────────────→│  校验 SSO Cookie 有效          │
                     │  └─→ 注入 X-Remote-User 头     │
                     │      └─→ 转发到 File Browser   │
                     └───────────────┬───────────────┘
                                     ↓
                     File Browser 后端
                     ┌───────────────────────────────┐
   首次请求 /        │                               │
 ───────────────────→│  initAuth() 走 login("","","") │
                     │    POST /api/login             │
                     │      ProxyAuth.Auth() 读头     │
                     │      → 自动创建/获取用户       │
                     │      → printToken() 发 JWT     │
                     │  parseToken()                  │
                     │    proxy + 自定义登出页         │
                     │    → ★ 禁用空闲计时器          │
                     └───────────────┬───────────────┘
                                     ↓
            用户活跃操作（N 次 API 请求）
                     ┌───────────────────────────────┐
  GET /api/...      │                               │
 ───────────────────→│  withUser 校验 JWT             │
                     │    ├─ 已过期 → renewableErr   │
                     │    │    (三条件命中则放行)     │
                     │    ├─ expiresSoon（过期恒 true）│
                     │    └─ updated（双保险）        │
                     │    → X-Renew-Token: true（几乎必发）│
                     │  fetchURL 检测到后 renew()     │
                     │    renewHandler 签发新 JWT     │
                     │    parseToken()（仍禁用计时）  │
                     └───────────────┬───────────────┘
                                     ↓
              用户点击外部 SSO 登出按钮
                     ┌───────────────────────────────┐
  SSO Cookie 失效   │                               │
 ───────────────────→│  下一次请求到达 Nginx           │
                     │  Nginx 重定向到 SSO 登录页     │
                     │  （用户不再能到达 File Browser）│
                     │                               │
                     │  或者用户主动触发 File Browser  │
                     │  的登出按钮 → logout()         │
                     │    → ★ 跳外部登出 URL          │
                     │    → 外部 SSO 统一销毁会话     │
                     └───────────────────────────────┘
```

---

## 五、协作关系总结

### 5.1 核心设计要点

| 设计决策 | 实现方式 | 优势 |
|---------|---------|------|
| **无状态认证** | JWT 自包含用户信息，服务端不存会话 | 水平扩展友好，无需共享 session 存储 |
| **惰性刷新** | 正常请求响应头捎带刷新提示，不单独心跳 | 节省请求，只有活跃用户才续期 |
| **双触发刷新** | 即将过期 + 用户信息变更 | 兼顾会话续期与数据一致性 |
| **双通道传输** | X-Auth Header + Cookie (GET fallback) | 兼顾 SPA API 调用与浏览器原生资源请求 |
| **前端三重存储** | Pinia + localStorage + Cookie | 运行时高效、刷新可恢复、GET 请求兜底 |
| **Proxy 过期宽容** | `renewableErr()` 三条件判断 | 与外部 SSO 会话保持一致，避免伪登出 |
| **Proxy 禁用计时** | `parseToken()` 条件 return | 不让 JWT 自身的 exp 干扰 SSO 会话管理 |
| **Proxy 外部登出** | `logout()` redirect 分支 | 登出闭环，消除幽灵会话 |

### 5.2 前后端交互时序（含 Proxy 分支）

```
前端                                    后端
 │                                       │
 │── POST /api/login (proxy 下为空) ────→│
 │                                       │── ProxyAuth.Auth() 读 X-Remote-User
 │                                       │── printToken() 签发 JWT
 │←────────── 200 JWT 纯文本 ───────────│
 │── parseToken()                        │
 │    ├─ 存三处                           │
 │    └─ proxy + custom-logout ?         │
 │         ├─ YES → ★ 禁用计时器 return  │
 │         └─ NO  → 设置 logoutTimer     │
 │                                       │
 │── GET /api/resources/... ────────────→│
 │    X-Auth: <jwt>                      │
 │                                       │── withUser 校验
 │                                       │    ├─ 提取 Token
 │                                       │    ├─ 验签 + exp 检查
 │                                       │    ├─ 过期 ?
 │                                       │    │  └─ renewableErr 三条件 ?
 │                                       │    │       ├─ YES → ★ 放行
 │                                       │    │       └─ NO  → 401
 │                                       │    ├─ expiresSoon?（过期则恒 true）
 │                                       │    └─ updated?（双保险）
 │←─ 200 + (可能 X-Renew-Token:true) ──│
 │                                       │
 │── detect X-Renew-Token:true           │
 │── POST /api/renew ───────────────────→│  (可带过期 JWT)
 │    X-Auth: <old jwt>                  │
 │                                       │── withUser (同上，过期仍可放行)
 │                                       │── printToken() 发新 JWT
 │←────────── 200 NEW JWT ──────────────│
 │── parseToken()                        │
 │    (计时器逻辑同上)                    │
 │                                       │
 │── (SSO Cookie 失效 / 主动登出)        │
 │── logout()                             │
 │    ├─ 清除存储                         │
 │    └─ logoutPage != "/login" ?        │
 │         ├─ YES → ★ 跳外部登出 URL     │
 │         └─ NO  → 跳 /login
```

### 5.3 关键文件索引

| 角色 | 文件 | 核心函数/类型 |
|------|------|--------------|
| 后端签发 | [http/auth.go](http/auth.go) | `loginHandler`, `renewHandler`, `printToken`, `authToken` |
| 后端校验 | [http/auth.go](http/auth.go) | `withUser`, `extractor.ExtractToken`, `renewableErr` |
| 认证接口 | [auth/auth.go](auth/auth.go) | `Auther` interface |
| JSON 认证 | [auth/json.go](auth/json.go) | `JSONAuth.Auth` |
| Hook 认证 | [auth/hook.go](auth/hook.go) | `HookAuth.Auth` |
| Proxy 认证 | [auth/proxy.go](auth/proxy.go) | `ProxyAuth.Auth`, `ProxyAuth.LoginPage` |
| No Auth | [auth/none.go](auth/none.go) | `NoAuth.Auth` |
| 用户更新追踪 | [users/storage.go](users/storage.go) | `Storage.Update`, `Storage.LastUpdate` |
| 路由注册 | [http/http.go](http/http.go) | `/api/login`, `/api/renew` |
| 配置定义 | [settings/settings.go](settings/settings.go) | `AuthMethod`, `DefaultLogoutPage`, `Server.TokenExpirationTime` |
| 前端配置常量 | [frontend/src/utils/constants.ts](frontend/src/utils/constants.ts) | `authMethod`, `logoutPage`, `loginPage` |
| 前端登录/续期/登出 | [frontend/src/utils/auth.ts](frontend/src/utils/auth.ts) | `login`, `renew`, `parseToken` (含禁用计时器分支), `logout` (含外部跳转分支), `validateLogin` |
| 前端请求封装 | [frontend/src/api/utils.ts](frontend/src/api/utils.ts) | `fetchURL` (X-Auth 注入 + X-Renew-Token 响应 + 401 登出) |
| 前端状态 | [frontend/src/stores/auth.ts](frontend/src/stores/auth.ts) | `useAuthStore` |
| 前端路由守卫 | [frontend/src/router/index.ts](frontend/src/router/index.ts) | `initAuth`, `beforeResolve` |
