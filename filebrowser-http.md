# File Browser HTTP 请求处理链深度分析

---

## 一、用户管理接口的权限判断：管理员或本人

用户管理接口位于 [http/users.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/users.go)，注册路由在 [http/http.go#L53-L58](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/http.go#L53-L58)。

### 1.1 路由与装饰器对应关系

| 路由 | 方法 | 装饰器 | 权限 |
|------|------|--------|------|
| `/api/users` | GET | `withAdmin` | 仅管理员可列出全部用户 |
| `/api/users` | POST | `withAdmin` | 仅管理员可创建新用户 |
| `/api/users/{id}` | GET | `withSelfOrAdmin` | 本人或管理员可查看 |
| `/api/users/{id}` | PUT | `withSelfOrAdmin` | 本人或管理员可修改（带字段级限制） |
| `/api/users/{id}` | DELETE | `withSelfOrAdmin` | 本人或管理员可删除 |

### 1.2 两层权限装饰器

#### 第一层：withAdmin —— 纯管理员

[withAdmin](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/auth.go#L113-L121) 嵌套了 `withUser`，在用户认证通过后额外校验 `Perm.Admin`：

```go
func withAdmin(fn handleFunc) handleFunc {
    return withUser(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        if !d.user.Perm.Admin {
            return http.StatusForbidden, nil   // 非管理员直接 403
        }
        return fn(w, r, d)
    })
}
```

#### 第二层：withSelfOrAdmin —— 本人或管理员

[withSelfOrAdmin](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/users.go#L57-L71) 是用户管理接口的核心权限门控：

```go
func withSelfOrAdmin(fn handleFunc) handleFunc {
    return withUser(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        // 1. 从 URL 路径中解析目标用户 ID
        id, err := getUserID(r)          // mux.Vars(r)["id"] → uint
        if err != nil {
            return http.StatusInternalServerError, err
        }

        // 2. 核心判断：当前登录用户 ID ≠ 目标用户 ID 且非管理员 → 403
        if d.user.ID != id && !d.user.Perm.Admin {
            return http.StatusForbidden, nil
        }

        // 3. 将解析出的目标用户 ID 放入 d.raw 传递给下游
        d.raw = id
        return fn(w, r, d)
    })
}
```

**判断逻辑真值表**：

| d.user.ID == id | d.user.Perm.Admin | 结果 |
|-----------------|-------------------|------|
| true | 任意 | ✅ 通过（本人操作） |
| false | true | ✅ 通过（管理员操作他人） |
| false | false | ❌ 403 Forbidden |

### 1.3 PUT /api/users/{id} 的字段级权限控制

[userPutHandler](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/users.go#L180-L269) 在 `withSelfOrAdmin` 基础上还有更细粒度的限制：

#### A. 当前密码验证（JSONAuth 模式下）

如果修改的是"敏感字段"（username、password、scope、lockPassword、commands、perm、all），需要验证当前登录用户的密码：

```go
sensibleFields := {"all", "username", "password", "scope", "lockPassword", "commands", "perm"}
for _, field := range req.Which {
    if _, ok := sensibleFields[strings.ToLower(field)]; ok {
        if !users.CheckPwd(req.CurrentPassword, d.user.Password) {
            return http.StatusBadRequest, fberrors.ErrCurrentPasswordIncorrect
        }
        break
    }
}
```

#### B. 非管理员的字段黑名单

[NonModifiableFieldsForNonAdmin](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/users.go#L21-L23) 定义了 6 个非管理员不可修改的字段：

```go
NonModifiableFieldsForNonAdmin = []string{
    "Username", "Scope", "LockPassword", "Perm", "Commands", "Rules",
}
```

遍历检查逻辑：
```go
for _, f := range NonModifiableFieldsForNonAdmin {
    if !d.user.Perm.Admin && v == f {
        return http.StatusForbidden, nil
    }
}
```

#### C. 全量更新必须是管理员

如果 `req.Which` 为空（等价于 "all"），要求必须是管理员：
```go
if len(req.Which) == 0 || (len(req.Which) == 1 && req.Which[0] == "all") {
    if !d.user.Perm.Admin {
        return http.StatusForbidden, nil
    }
    // ...
}
```

#### D. 非管理员修改密码的额外限制

如果用户 `LockPassword = true`（密码被锁定），即使是本人也不能改密码：
```go
if v == "Password" {
    if !d.user.Perm.Admin && d.user.LockPassword {
        return http.StatusForbidden, nil
    }
    // ...
}
```

#### E. 分享权限与下载权限的绑定约束

任何情况下设置 `Perm.Share = true` 都必须同时有 `Perm.Download = true`：
```go
if req.Data.Perm.Share && !req.Data.Perm.Download {
    return http.StatusBadRequest, fberrors.ErrShareRequiresDownload
}
```

### 1.4 GET /api/users/{id} 的返回字段脱敏

[userGetHandler](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/users.go#L90-L105) 中，非管理员查看自己的信息时会隐藏 `Scope`：

```go
u.Password = ""                       // 始终清空密码哈希
if !d.user.Perm.Admin {
    u.Scope = ""                       // 非管理员看不到自己的文件系统根路径
}
```

---

## 二、令牌续期入口与完整流程

### 2.1 续期机制的三个关键点

令牌续期涉及 [http/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/auth.go) 中的以下代码：

| 组件 | 位置 | 作用 |
|------|------|------|
| `renewableErr` | [auth.go#L69-L83](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/auth.go#L69-L83) | 判断过期错误是否允许续期 |
| `withUser` 中的续期提示 | [auth.go#L98-L103](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/auth.go#L98-L103) | 检测到需要续期时设置响应头 |
| `renewHandler` | [auth.go#L216-L221](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/auth.go#L216-L221) | `/api/renew` 显式续期入口 |
| `printToken` | [auth.go#L223-L257](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/auth.go#L223-L257) | 签发新 Token 的公共函数 |

### 2.2 隐式续期提示：withUser 中的 X-Renew-Token

每次经过 `withUser` 认证的请求都会检查是否需要提示续期：

```go
// 条件1：Token 过期时间 < 1 小时
expiresSoon := tk.ExpiresAt != nil && time.Until(tk.ExpiresAt.Time) < time.Hour

// 条件2：Token 签发时间早于用户信息最后更新时间
//        （管理员修改了该用户的权限/密码等信息）
updated := tk.IssuedAt != nil && tk.IssuedAt.Unix() < d.store.Users.LastUpdate(tk.User.ID)

// 满足任一条件，在响应头中提示前端续期
if expiresSoon || updated {
    w.Header().Add("X-Renew-Token", "true")
}
```

**重要**：这里只是**设置响应头提示**，并不实际签发新 Token。前端看到 `X-Renew-Token: true` 后需要主动调用 `/api/renew`。

### 2.3 显式续期入口：/api/renew

路由注册：[http.go#L51](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/http.go#L51)

```go
func renewHandler(tokenExpireTime time.Duration) handleFunc {
    return withUser(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
        w.Header().Set("X-Renew-Token", "false")   // 明确告知：已续期，无需再提示
        return printToken(w, r, d, d.user, tokenExpireTime)   // 签发新 Token
    })
}
```

流程：
1. `withUser` 验证旧 Token（只要没过期或属于可再生错误都放行）
2. 设置 `X-Renew-Token: false`（清除续期提示）
3. 调用 `printToken` 用当前最新的用户信息签发全新 Token

### 2.4 过期 Token 的特殊放行：renewableErr

[renewableErr](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/auth.go#L69-L83) 决定了**已过期 Token 能否被接受**（仅用于 ProxyAuth 模式）：

```go
func renewableErr(err error, d *data) bool {
    // 条件1：必须是 ProxyAuth 认证方式，且确实发生了错误
    if d.settings.AuthMethod != fbAuth.MethodProxyAuth || err == nil {
        return false
    }
    // 条件2：必须配置了非默认的 LogoutPage（自定义登出页）
    if d.settings.LogoutPage == settings.DefaultLogoutPage {
        return false
    }
    // 条件3：错误类型必须是 JWT 过期
    if !errors.Is(err, jwt.ErrTokenExpired) {
        return false
    }
    return true   // 三个条件全满足 → 过期 Token 暂不放回 401，交给 renewHandler 处理
}
```

在 `withUser` 中的使用：
```go
if (err != nil || !token.Valid) && !renewableErr(err, d) {
    return http.StatusUnauthorized, nil      // 正常情况下过期直接 401
}
// 如果 renewableErr 返回 true，则继续执行，过期 Token 也能通过 withUser
// 这样 /api/renew 就能用已过期的 ProxyAuth Token 换取新 Token
```

### 2.5 新 Token 签发：printToken

[printToken](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/auth.go#L223-L257) 被 `loginHandler` 和 `renewHandler` 共同调用：

```go
func printToken(w http.ResponseWriter, _ *http.Request, d *data, user *users.User, tokenExpirationTime time.Duration) (int, error) {
    claims := &authToken{
        User: userInfo{ /* 从 user 对象拷贝字段 */ },
        RegisteredClaims: jwt.RegisteredClaims{
            IssuedAt:  jwt.NewNumericDate(time.Now()),              // 签发时间 = 现在
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(tokenExpirationTime)),  // 默认 2 小时后
            Issuer:    "File Browser",
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, err := token.SignedString(d.settings.Key)   // 用 settings.Key 做 HS256 签名
    // ...
    w.Header().Set("Content-Type", "text/plain")
    w.Write([]byte(signed))
    return 0, nil
}
```

### 2.6 令牌续期完整时序图

```
前端                                后端
  |                                   |
  |-- GET /api/resources/xxx ------->|
  |                                   |  withUser 解析 Token
  |                                   |  → 发现 expiresSoon = true
  |<-- 200 + X-Renew-Token: true ----|
  |                                   |
  |  检测到 X-Renew-Token 头          |
  |-- POST /api/renew -------------->|  (携带旧 Token)
  |                                   |
  |                                   |  withUser 验证旧 Token
  |                                   |  → renewableErr 判定 (ProxyAuth 专用)
  |                                   |  renewHandler:
  |                                   |    1. X-Renew-Token = "false"
  |                                   |    2. printToken 签发新 Token
  |<-- 200 新 Token 文本 -------------|
  |                                   |
  |  保存新 Token，后续请求使用        |
```

---

## 三、基础路径剥离与路由前缀剥离的先后顺序

### 3.1 涉及的三层剥离函数

整个系统存在 **三层** 路径前缀剥离，按**请求到达先后顺序**排列：

| 层级 | 执行时机 | 函数/位置 | 剥离内容 | 剥离对象 |
|------|----------|-----------|----------|----------|
| 第 1 层 | 最外层，进入 mux.Router 之前 | [http.go#L93](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/http.go#L93) `stripPrefix(server.BaseURL, r)` | 应用基础路径 BaseURL（如 `/files`） | 全局所有请求 |
| 第 2 层 | mux.Router 匹配后，进入业务 handler 之前 | [data.go#L100](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/data.go#L100) `stripPrefix(prefix, handler)` | 路由注册前缀（如 `/api/resources` 无尾斜杠 → 结果带 `/`；如 `/static/` 有尾斜杠 → 结果无 `/`） | 单个路由组 |
| 第 3 层 | 分享场景中，进入业务逻辑前 | [public.go#L70](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/public.go#L70) `files.NewScopedFs` + `checkerPrefix` | 分享根路径（如 `/docs/share/`） | 分享场景的文件系统路径 |

### 3.2 代码层面的精确执行顺序

以 `NewHandler` 中 `/api/resources` 路由为例：

```go
// 【构造阶段】倒序注册，层层包裹
// Step 3: 最外层包装 BaseURL 剥离
return stripPrefix(server.BaseURL, r), nil
                        ↑
                        |
// Step 2: 注册路由时，monkey 包装 handle，handle 内部再包 stripPrefix
api.PathPrefix("/resources").Handler(
    monkey(resourceGetHandler, "/api/resources")
    // monkey 内部: handle(fn, prefix, ...)
    //   handle 内部构造 HandlerFunc 后，外层包: stripPrefix("/api/resources", handler)
    //                                            ↑
    //                                            |
    // Step 1: 最内层业务逻辑（withUser 装饰链）
    // withUser(withUser(func(...))) → 业务函数
).Methods("GET")
```

### 3.3 实际请求路径变换示例

**核心区别**：prefix 是否带末尾斜杠，决定了剥离后路径是否带前导斜杠。

- API 路由 prefix **不带末尾斜杠**（如 `"/api/resources"`）→ 剥离后路径**带前导斜杠**
- 静态资源 prefix **带末尾斜杠**（如 `"/static/"`）→ 剥离后路径**无前导斜杠**
- public 分享 prefix **带末尾斜杠**（如 `"/api/public/dl/"`）→ 剥离后路径**无前导斜杠**

#### 示例 A：API 资源路由（prefix 无末尾斜杠 → 带前导斜杠）

配置：`server.BaseURL = "/app"`，请求 URL: `http://host/app/api/resources/docs/report.pdf`

```
原始 URL.Path:
    /app/api/resources/docs/report.pdf

─────────────── 第 1 层剥离: stripPrefix("/app") ───────────────
    /api/resources/docs/report.pdf

    mux.Router 匹配: PathPrefix("/api/resources") 成功

─────────────── 第 2 层剥离: stripPrefix("/api/resources") ───────────────
    prefix = "/api/resources"（无末尾斜杠）
    TrimPrefix("/api/resources/docs/report.pdf", "/api/resources")
    = "/docs/report.pdf"              ← 带前导斜杠

─────────────── 业务 handler 内部 ───────────────
    resourceGetHandler 内 r.URL.Path = "/docs/report.pdf"
    → d.user.Fs.Open("/docs/report.pdf")  // 用户文件系统，路径带 / 正常工作
```

#### 示例 B：静态资源路由（prefix 有末尾斜杠 → 无前导斜杠）

配置：`server.BaseURL = ""`，请求 URL: `GET /static/js/app.js`

```
原始 URL.Path:
    /static/js/app.js

─────────────── 第 1 层剥离: stripPrefix("") ───────────────
    /static/js/app.js（无变化）

    mux.Router 匹配: PathPrefix("/static") 成功

─────────────── 第 2 层剥离: stripPrefix("/static/") ───────────────
    prefix = "/static/"（有末尾斜杠）
    TrimPrefix("/static/js/app.js", "/static/")
    = "js/app.js"                      ← 无前导斜杠

─────────────── 业务 handler 内部 ───────────────
    static handler 内 r.URL.Path = "js/app.js"
    → assetsFs.Open("js/app.js.gz")    // fs.FS 要求无前导斜杠，正常工作
```

#### 示例 C：public 分享路由（prefix 有末尾斜杠 → 无前导斜杠）

请求 URL: `GET /api/public/share/abc123/docs/readme.md`

```
─────────────── 第 2 层剥离: stripPrefix("/api/public/share/") ───────────────
    prefix = "/api/public/share/"（有末尾斜杠）
    TrimPrefix("/api/public/share/abc123/docs/readme.md", "/api/public/share/")
    = "abc123/docs/readme.md"          ← 无前导斜杠

─────────────── withHashFile 内部 ───────────────
    解析 "abc123" 为分享 ID，剩余路径 "docs/readme.md"
```

### 3.4 stripPrefix 的内部实现

[stripPrefix](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/utils.go#L55-L80) 相比标准库 `http.StripPrefix` 有两处重要差异：

```go
func stripPrefix(prefix string, h http.Handler) http.Handler {
    // 快速路径：无前缀直接透传
    if prefix == "" || prefix == "/" {
        return h
    }

    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        p := strings.TrimPrefix(r.URL.Path, prefix)
        rp := strings.TrimPrefix(r.URL.RawPath, prefix)

        // 差异1: 恰好等于前缀时重定向到带尾斜杠的版本
        //        防止 mux 收到空路径后重定向到站点根
        if p == "" {
            http.Redirect(w, r, prefix+"/", http.StatusMovedPermanently)
            return
        }

        // 差异2: 不做 404，没有前缀也继续转发
        //        标准库 StripPrefix 这里是 http.NotFound(w, r)
        r2 := new(http.Request)
        *r2 = *r
        r2.URL = new(url.URL)
        *r2.URL = *r.URL
        r2.URL.Path = p
        r2.URL.RawPath = rp
        h.ServeHTTP(w, r2)
    })
}
```

### 3.5 静态资源与 NotFoundHandler 的路径处理

`NewHandler` 中的特殊路由注册：

```go
r.PathPrefix("/static").Handler(static)   // static 内部带 stripPrefix("/static/", ...)
r.NotFoundHandler = index                 // index 无内部前缀剥离
```

- `/static` 路由：进入 `getStaticHandlers` 返回的 `static` handler，其内部的 `handle` 包装又会执行 `stripPrefix("/static/", ...)`，所以业务 handler 看到的是无前导斜杠的相对路径（如 `js/app.js`、`css/app.css`）
- `NotFoundHandler`：所有未匹配的请求都走 SPA 的 `index` handler，**不会剥离任何前缀**，因为注册时传给 `handle` 的 prefix 是 `""`

---

## 四、状态码和错误文本的统一返回方式

### 4.1 统一返回约定：handleFunc 签名

```go
// [data.go#L18]
type handleFunc func(w http.ResponseWriter, r *http.Request, d *data) (int, error)
```

**双返回值协议**：

| status | err | 含义 | 后续行为 |
|--------|-----|------|----------|
| `0` | `nil` | 正常返回，handler 已自行写入响应体 | 直接结束，不做处理 |
| `0` | 非 nil | handler 写响应中途出错 | 记录日志（err != nil 触发），但不向客户端输出 |
| 非 0（< 400） | 任意 | 业务状态码（如 201 Created） | 由 handle 统一用 `http.Error` 输出 |
| 非 0（>= 400） | 任意 | 错误状态码 | 记录日志 + 统一错误输出 |

### 4.2 唯一出口：handle 函数中的错误处理

[data.go#L85-L97](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/data.go#L85-L97) 是**所有 HTTP 错误响应的唯一出口**（WebSocket 除外）：

```go
status, err := fn(w, r, &data{...})

// ─── 阶段 1：日志记录 ───
// 触发条件：status >= 400  或者  err != nil
// 记录内容：请求路径、HTTP 状态码、客户端真实 IP、错误信息字符串
if status >= 400 || err != nil {
    clientIP := realip.FromRequest(r)
    log.Printf("%s: %v %s %v", r.URL.Path, status, clientIP, err)
}

// ─── 阶段 2：响应输出 ───
if status != 0 {
    txt := http.StatusText(status)              // 标准 HTTP 状态文本，如 "Bad Request"

    // ★ 重要：只有 400 Bad Request 才会附加错误详情
    // 其他状态码一律不泄露内部错误信息给客户端
    if status == http.StatusBadRequest && err != nil {
        txt += " (" + err.Error() + ")"
    }

    // 最终输出格式：
    //   响应行: HTTP/1.1 <status> <StatusText>
    //   响应体: "<status> <StatusText>[(err)]"
    //   Content-Type: text/plain; charset=utf-8
    http.Error(w, strconv.Itoa(status)+" "+txt, status)
    return
}
```

### 4.3 不同状态码的响应体格式对照表

| status | err | 响应体示例 |
|--------|-----|------------|
| `201` | nil | `201 Created` |
| `204` | nil | `204 No Content` |
| `400` | nil | `400 Bad Request` |
| `400` | `errors.New("invalid JSON")` | `400 Bad Request (invalid JSON)` |
| `400` | `fberrors.ErrCurrentPasswordIncorrect` | `400 Bad Request (the current password is incorrect)` |
| `401` | nil | `401 Unauthorized` |
| `403` | `fberrors.ErrPermissionDenied` | `403 Forbidden`（**错误详情不返回**） |
| `404` | `os.ErrNotExist` | `404 Not Found`（**错误详情不返回**） |
| `409` | `fberrors.ErrExist` | `409 Conflict`（**错误详情不返回**） |
| `500` | `errors.New("db conn fail")` | `500 Internal Server Error`（**错误详情不返回**） |

### 4.4 错误码转换函数：errToStatus

[errToStatus](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/utils.go#L30-L51) 是业务层常用的辅助函数，将 Go 标准错误和领域错误映射为 HTTP 状态码：

```go
func errToStatus(err error) int {
    switch {
    case err == nil:
        return http.StatusOK                           // 200
    case os.IsPermission(err):
        return http.StatusForbidden                    // 403
    case os.IsNotExist(err), errors.Is(err, libErrors.ErrNotExist):
        return http.StatusNotFound                     // 404
    case os.IsExist(err), errors.Is(err, libErrors.ErrExist):
        return http.StatusConflict                     // 409
    case errors.Is(err, libErrors.ErrPermissionDenied):
        return http.StatusForbidden                    // 403
    case errors.Is(err, libErrors.ErrInvalidRequestParams):
        return http.StatusBadRequest                   // 400
    case errors.Is(err, libErrors.ErrRootUserDeletion):
        return http.StatusForbidden                    // 403
    case errors.Is(err, imgErrors.ErrImageTooLarge):
        return http.StatusRequestEntityTooLarge        // 413
    default:
        return http.StatusInternalServerError          // 500（兜底）
    }
}
```

**典型用法**：
```go
file, err := files.NewFileInfo(...)
if err != nil {
    return errToStatus(err), err     // 领域错误 → HTTP 状态码 + 原始错误
}
```

注意：`errToStatus` 只是转换状态码，**最终响应体是否包含错误详情仍由 handle 函数的 `status == 400` 判断决定**。例如 `ErrExist` 被转为 409，虽然 err 不为 nil，但因为 status ≠ 400，响应体只有 `409 Conflict`。

### 4.5 领域错误定义

[errors/errors.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/errors/errors.go) 中定义了全部领域错误变量：

| 错误变量 | 触发 400 详情返回？ | 说明 |
|----------|---------------------|------|
| `ErrExist` | ❌ (→ 409) | 资源已存在 |
| `ErrNotExist` | ❌ (→ 404) | 资源不存在 |
| `ErrPermissionDenied` | ❌ (→ 403) | 权限被拒 |
| `ErrInvalidRequestParams` | ✅ (→ 400) | 请求参数无效 |
| `ErrEmptyRequest` | ✅ (→ 400) | 请求体为空 |
| `ErrInvalidDataType` | ✅ (→ 400) | 数据类型无效 |
| `ErrInvalidOption` | ✅ (→ 400) | 选项无效 |
| `ErrEmptyPassword` | ✅ (→ 400) | 密码为空 |
| `ErrEasyPassword` | ✅ (→ 400) | 密码太简单 |
| `ErrEmptyUsername` | ✅ (→ 400) | 用户名为空 |
| `ErrCurrentPasswordIncorrect` | ✅ (→ 400) | 当前密码错误 |
| `ErrShareRequiresDownload` | ✅ (→ 400) | 分享权限需要下载权限 |
| `ErrSourceIsParent` | ✅ (→ 400) | 源是父目录（复制移动场景） |
| `ErrRootUserDeletion` | ❌ (→ 403) | 不能删除唯一管理员 |
| `ErrShortPassword` (struct) | ✅ (→ 400) | 密码太短，带最小长度信息 |

### 4.6 JSON 成功响应的返回模式

`renderJSON` 是成功响应的标准出口：

```go
func renderJSON(w http.ResponseWriter, _ *http.Request, data interface{}) (int, error) {
    marsh, err := json.Marshal(data)
    if err != nil {
        return http.StatusInternalServerError, err   // 序列化失败走统一错误出口
    }
    w.Header().Set("Content-Type", "application/json; charset=utf-8")
    if _, err := w.Write(marsh); err != nil {
        return http.StatusInternalServerError, err   // 写入失败走统一错误出口
    }
    return 0, nil   // status=0, err=nil → handle 不再处理
}
```

### 4.7 WebSocket 的独立错误处理（特殊情况）

命令执行接口 `/api/command` 使用 WebSocket 协议，**在升级前后走完全不同的错误出口**。

---

## 五、命令执行接口：WebSocket 升级前后的错误出口分析

命令执行接口路由注册：[http.go#L85](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/http.go#L85)
```go
api.PathPrefix("/command").Handler(monkey(commandsHandler, "/api/command")).Methods("GET")
```

`commandsHandler` 位于 [http/commands.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/commands.go)，整体结构是 `withUser` 装饰 + 业务逻辑。

### 5.1 升级前：HTTP 协议阶段 → handle 统一错误出口

在 `upgrader.Upgrade` 调用**之前**发生的所有错误，都走 **handle 函数的 HTTP 错误出口**（即 `http.Error` 写入 HTTP 响应体）：

| 错误位置 | 代码行 | 触发条件 | 返回值 | 实际错误出口 |
|----------|--------|----------|--------|-------------|
| `handle` 内 settings 加载 | data.go#L72-L76 | 数据库读取 settings 失败 | `log.Fatalf`（进程退出） | 无，进程崩溃 |
| `withUser` JWT 认证 | auth.go#L92-L94 | Token 无效、过期、签名伪造，且不满足 renewableErr 条件 | `return 401, nil` | handle → `http.Error(w, "401 Unauthorized", 401)` |
| `upgrader.Upgrade` | commands.go#L42-L44 | WebSocket 握手失败（方法非 GET、Upgrade 头缺失、Origin 校验失败等） | `return 500, err` | handle → `http.Error(w, "500 Internal Server Error", 500)` |

**注意**：gorilla/websocket 的 `Upgrade` 函数失败时**只返回 error，不写入响应**（可查看其源码，错误分支均为 `return nil, err`），因此 500 响应完全由 handle 统一出口输出。

**调用链（升级前 JWT 认证失败）**：
```
HTTP GET /api/command (带 Upgrade 头)
  │
  ├─ stripPrefix(BaseURL)           无变化
  ├─ mux.Router 匹配                 /api/command 匹配
  ├─ stripPrefix("/api/command")    路径变 ""
  │
  ▼
  handle 包装器
    ├─ 设置全局 Cache-Control 头
    ├─ store.Settings.Get()         读 DB 成功
    └─ 调用 fn = withUser(commandsHandler)
           │
           ▼
        withUser:
          ├─ 提取 X-Auth / Cookie 中的 JWT
          ├─ jwt.ParseFromRequest   → 签名无效
          └─ return 401, nil        ← 升级前就返回了
    │
    ▼
  handle 统一出口
    ├─ status=401 >= 400 → 打日志 (路径 + realip + err)
    └─ status != 0 → http.Error(w, "401 Unauthorized", 401)
```

### 5.2 升级后：WebSocket 协议阶段 → 三类 WebSocket 错误出口

`upgrader.Upgrade` 成功返回 `*websocket.Conn` 后，所有分支**最终都返回 `(0, nil)`**，handle 函数中 `status != 0` 判断为 false，**不会**调用 `http.Error`。错误通过 WebSocket 协议自身的三种通道返回：

#### 类型 A：wsErr 函数 → WebSocket Close Frame（1011 Internal Server Error）

[wsErr](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/commands.go#L31-L39) 用于**致命错误**（管道创建失败、进程启动失败、命令非零退出等），先发送 Close 控制帧（状态码 1011），再由 `return 0, nil` 触发 `defer conn.Close()` 关闭连接：

```go
func wsErr(ws *websocket.Conn, r *http.Request, status int, err error) {
    txt := http.StatusText(status)
    // 日志记录（格式与 handle 相同，但 IP 来源不同）
    if err != nil || status >= 400 {
        log.Printf("%s: %v %s %v", r.URL.Path, status, r.RemoteAddr, err)
        //                        注意这里用 r.RemoteAddr，而非 realip.FromRequest(r)
    }
    // Close Frame (1011) + 状态文本，如 "Internal Server Error"
    if err := ws.WriteControl(websocket.CloseInternalServerErr, []byte(txt),
                              time.Now().Add(WSWriteDeadline)); err != nil {
        log.Print(err)
    }
}
```

**与 HTTP 出口的差异**：`wsErr` 打日志用 `r.RemoteAddr`，而 handle 统一出口用 `realip.FromRequest(r)`（穿透代理时取真实 IP）。在有反向代理的部署中两者值不同。

#### 类型 B：conn.WriteMessage → WebSocket Text Frame（业务拒绝）

用于**可恢复的业务拒绝**（权限不够、命令不在白名单、命令解析失败），通过普通数据帧（Opcode=1 Text）返回错误文本，然后正常 `return 0, nil` 由 `defer conn.Close()` 关闭连接：

```go
// 以权限拒绝为例
if !d.server.EnableExec || !d.user.Perm.Execute {
    if err := conn.WriteMessage(websocket.TextMessage, cmdNotAllowed); err != nil {
        wsErr(conn, r, http.StatusInternalServerError, err)  // 写消息失败时回退到类型 A
    }
    return 0, nil   // 写消息成功时正常返回，前端收到 Text Frame 后等连接关闭
}
```

| 错误场景 | 代码行 | 写成功时发送给前端的内容 |
|----------|--------|-------------------------|
| 命令执行未启用 / 无 Execute 权限 | commands.go#L64-L69 | Text Frame: `"Command not allowed."` |
| `runner.ParseCommand` 命令解析失败 | commands.go#L72-L78 | Text Frame: `err.Error()` 具体描述 |
| 命令不在用户白名单 `d.user.Commands` | commands.go#L80-L86 | Text Frame: `"Command not allowed."` |

**注意**：这三个场景都有"写消息失败时回退到 wsErr"的降级逻辑——即类型 B 和类型 A 的错误点一一对应。

#### 类型 C：仅 log.Print，不主动关闭连接（运行时写消息失败）

命令正常执行过程中向客户端转发 stdout/stderr 时如果写失败，**不中断流程也不关闭连接**，仅记录日志后继续尝试后续输出：

```go
// commands.go#L108-L113
s := bufio.NewScanner(io.MultiReader(stdout, stderr))
for s.Scan() {
    if err := conn.WriteMessage(websocket.TextMessage, s.Bytes()); err != nil {
        log.Print(err)   // 仅日志，不 return，不关闭连接
    }
}
```

此时前端表现为部分输出丢失，直到命令结束后由 `cmd.Wait` 的 wsErr（如有错误）或自然 defer 关闭连接。

### 5.3 commandsHandler 全错误点分类表

[commands.go#L41-L120](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/commands.go#L41-L120) 内共有 **13 个** 错误分支点。以 `upgrader.Upgrade`（L42）为界，严格区分升级前和升级后：

> **注意**：5.1 节列出了调用链上 3 个升级前错误点（settings 加载、withUser JWT、upgrader.Upgrade），其中前两个在 commandsHandler 函数体之外。本表仅统计 commandsHandler **函数体内**的 13 个分支，其中 L42 是唯一的升级前错误点。

#### 升级前（HTTP 阶段）— 共 1 个，走 handle 统一 HTTP 出口

| 行号 | 错误点 | 返回值 | 实际错误出口 |
|------|--------|--------|-------------|
| L42 | `upgrader.Upgrade(w, r, nil)` 返回 err | `return http.StatusInternalServerError, err` | handle → `http.Error(w, "500 Internal Server Error", 500)`（HTTP 响应体） |

> 注意：此时 HTTP 响应尚未写入任何内容（Upgrade 失败时 gorilla/websocket 会自行写 403 吗？不——查看 Upgrade 源码，失败时它只返回 error，**不会**写响应。因此 500 完全由 handle 函数输出。

#### 升级后（WebSocket 阶段）— 共 12 个，全部 return `0, nil`，不再经过 handle

升级后所有分支最终都返回 `(0, nil)`，handle 函数中 `status != 0` 判断为 false，**不会**调用 `http.Error`。

##### 类型 A：wsErr → WebSocket Close Frame（1011 Internal Server Error）— 共 8 处

| 行号 | 错误点 | 返回值 | 发送给前端的内容 |
|------|--------|--------|-----------------|
| L53 | `conn.ReadMessage()` 失败（读取用户命令） | `return 0, nil`（内部先调 `wsErr`） | Close Frame(1011) + `"Internal Server Error"` |
| L66 | 权限拒绝时 `conn.WriteMessage(cmdNotAllowed)` 写入失败 | `return 0, nil`（内部先调 `wsErr`） | Close Frame(1011) + `"Internal Server Error"` |
| L75 | 解析失败时 `conn.WriteMessage(err.Error())` 写入失败 | `return 0, nil`（内部先调 `wsErr`） | Close Frame(1011) + `"Internal Server Error"` |
| L82 | 白名单拒绝时 `conn.WriteMessage(cmdNotAllowed)` 写入失败 | `return 0, nil`（内部先调 `wsErr`） | Close Frame(1011) + `"Internal Server Error"` |
| L93 | `cmd.StdoutPipe()` 创建管道失败 | `return 0, nil`（内部先调 `wsErr`） | Close Frame(1011) + `"Internal Server Error"` |
| L99 | `cmd.StderrPipe()` 创建管道失败 | `return 0, nil`（内部先调 `wsErr`） | Close Frame(1011) + `"Internal Server Error"` |
| L104 | `cmd.Start()` 启动进程失败 | `return 0, nil`（内部先调 `wsErr`） | Close Frame(1011) + `"Internal Server Error"` |
| L116 | `cmd.Wait()` 命令非零退出 | `return 0, nil`（隐式，内部先调 `wsErr`） | Close Frame(1011) + `"Internal Server Error"` |

##### 类型 B：conn.WriteMessage → WebSocket Text Frame（业务拒绝）— 共 3 处

| 行号 | 错误点 | 返回值 | 发送给前端的内容 |
|------|--------|--------|-----------------|
| L65 | `!EnableExec` 或 `!Perm.Execute`（写消息成功） | `return 0, nil` | Text Frame: `"Command not allowed."` |
| L74 | `runner.ParseCommand` 解析失败（写消息成功） | `return 0, nil` | Text Frame: `err.Error()` 具体错误描述 |
| L81 | `!slices.Contains(d.user.Commands, name)` 不在白名单（写消息成功） | `return 0, nil` | Text Frame: `"Command not allowed."` |

##### 类型 C：仅日志记录，不主动关闭连接 — 共 1 处

| 行号 | 错误点 | 返回值 | 前端感知 |
|------|--------|--------|---------|
| L111 | 运行时 `conn.WriteMessage(stdout/stderr)` 写入失败 | 无 return，继续循环/后续代码 | 仅服务端打 `log.Print(err)`，连接不主动关闭 |

### 5.4 完整时序对比图

#### 升级失败（HTTP 错误出口）
```
前端                                后端
  |-- GET /api/command (Upgrade) -->|
  |                                   |  handle: 读 settings OK
  |                                   |  withUser: JWT 认证 OK
  |                                   |  upgrader.Upgrade → 失败
  |                                   |  return 500, err
  |                                   |  handle 统一出口:
  |                                   |    log + http.Error
  |<-- HTTP 500 "Internal Server Error" --|
```

#### 升级成功 + 权限拒绝（WebSocket Text Frame）
```
前端                                后端
  |-- GET /api/command (Upgrade) -->|
  |<-- HTTP 101 Switching Protocols --|  Upgrade 成功
  |                                   |
  |-- [WS Text] "ls -la" ----------->|  发送命令
  |                                   |  检查: EnableExec=false
  |                                   |  conn.WriteMessage("Command not allowed.")
  |                                   |  return 0, nil
  |<-- [WS Text] "Command not allowed." --|
  |<-- [WS Close] 正常关闭 ----------|
```

#### 升级成功 + 管道创建失败（WebSocket Close Frame）
```
前端                                后端
  |-- GET /api/command (Upgrade) -->|
  |<-- HTTP 101 Switching Protocols --|
  |                                   |
  |-- [WS Text] "ls -la" ----------->|
  |                                   |  权限检查 OK
  |                                   |  命令解析 OK
  |                                   |  cmd.StdoutPipe() 失败
  |                                   |  wsErr(500, err):
  |                                   |    log + WriteControl(CloseInternalServerErr)
  |<-- [WS Close 1011] "Internal Server Error" --|
```

---

## 六、静态资源请求：前缀剥离后的相对路径分析

静态资源处理涉及四个关键位置的协作：
1. 前端资源嵌入：[frontend/assets.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/frontend/assets.go)
2. 子文件系统创建：[cmd/root.go#L233](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/cmd/root.go#L233)
3. 处理器构造：[http/static.go#L106-L181](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/static.go#L106-L181)
4. 路由注册：[http.go#L43](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/http.go#L43)

### 6.1 前端资源的嵌入结构

[frontend/assets.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/frontend/assets.go) 使用 `//go:embed` 嵌入 `dist/` 目录下的所有文件：

```go
//go:embed dist/*
var assets embed.FS
```

嵌入后 `assets` 内部的路径结构：
```
dist/
  ├─ index.html
  ├─ js/
  │   ├─ app.js
  │   ├─ app.js.gz
  │   └─ chunk-vendors.js
  ├─ css/
  │   └─ app.css
  └─ img/
      └─ logo.svg
```

### 6.2 assetsFs 的创建：fs.Sub 剥离 "dist/" 前缀

[cmd/root.go#L233](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/cmd/root.go#L233) 创建子文件系统，剥离 `dist/` 前缀：

```go
assetsFs, err := fs.Sub(frontend.Assets(), "dist")
```

**关键点**：`fs.Sub(frontend.Assets(), "dist")` 会将 `assetsFs` 的根映射到嵌入文件系统的 `dist/` 目录。此时 `assetsFs` 内部的有效路径（无前导斜杠）：
```
index.html
js/app.js
js/app.js.gz
css/app.css
img/logo.svg
```

`fs.FS` 接口的 `Open(name)` 方法要求 `name` 必须：
- 是相对路径，**不能以 `/` 开头**
- 是 `path.Clean` 过的（无 `..` 段）

### 6.3 static handler 的前缀剥离：/static/

路由注册时传给 `handle` 的 prefix 是 `"/static/"`（**注意末尾有斜杠，共 8 个字符**）：

[http/static.go#L178](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/static.go#L178)
```go
static = handle(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
    // 业务 handler：此处 r.URL.Path 已被 stripPrefix 处理，不带前导斜杠
    // ...
}, "/static/", store, server)
```

`handle` 函数在 [data.go#L66-L101](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/data.go#L66-L101) 中先构造 `http.HandlerFunc`（包含全局头、读 settings、调用 fn、错误处理），然后在 L100 调用 `stripPrefix(prefix, handler)` 将前缀剥离包装在外层。因此执行顺序是：**先 stripPrefix 剥离前缀 → 再进入 handle 的 HandlerFunc 执行全局头/读 settings/调用业务 handler**。

### 6.4 完整路径变换示例（每一步 r.URL.Path 的精确值）

请求 URL：`GET /static/js/app.js`，假设 `server.BaseURL = ""`。

因为 static 路由的 prefix 是 `"/static/"`（**有末尾斜杠**），剥离后路径为**无前导斜杠的相对路径**——这与 API 路由（prefix 无末尾斜杠 → 剥离后带前导斜杠）形成对比（见 3.3 节）。

下面是每一层处理后 `r.URL.Path` 的精确值：

| 处理层 | 代码位置 | 处理逻辑 | r.URL.Path 结果 |
|--------|----------|----------|----------------|
| Go net/http 入口 | — | 原始请求路径 | `"/static/js/app.js"` |
| 第 1 层剥离 | [http.go#L93](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/http.go#L93) `stripPrefix("", r)` | `BaseURL` 为空，直接透传 | `"/static/js/app.js"` |
| mux.Router 匹配 | [http.go#L43](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/http.go#L43) | `PathPrefix("/static")` 匹配成功，调用 monkey 返回的 handler | `"/static/js/app.js"`（此时仍未变） |
| 第 2 层剥离 | [data.go#L100](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/data.go#L100) `stripPrefix("/static/", handler)` | `prefix = "/static/"`（末尾带斜杠），`strings.TrimPrefix("/static/js/app.js", "/static/")` | `"js/app.js"`（**无前导斜杠**） |
| 业务 handler 内 | [static.go#L116](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/static.go#L116) | 业务代码看到的路径 | `"js/app.js"` |

**为什么结果没有前导斜杠？** 因为传给 `handle` 的 prefix 是 `"/static/"`（末尾带斜杠，共 8 个字符），`TrimPrefix` 完整剥离这 8 个字符，剩余字符串 `"js/app.js"` 自然不带前导斜杠。

如果 prefix 写成 `"/static"`（不带末尾斜杠），剥离结果会是 `"/js/app.js"`（带前导斜杠），这将导致 `.js` 文件分支的 `assetsFs.Open()` 失败。

### 6.5 剥离后的路径在业务 handler 中的三处使用

[http/static.go#L116-L177](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/static.go#L116-L177) 中，剥离后的 `r.URL.Path = "js/app.js"`（以请求不同资源为例）被用于以下三处，每一处都假设路径**不带前导斜杠**：

#### A. 品牌文件覆盖（Branding Override）

当 `d.settings.Branding.Files` 配置了自定义资源目录时，优先从该目录查找：

```go
// 示例 1: 请求 /static/img/logo.svg
//         r.URL.Path 剥离后 = "img/logo.svg"
if strings.HasPrefix(r.URL.Path, "img/") {
    fPath := filepath.Join(d.settings.Branding.Files, r.URL.Path)
    // filepath.Join("branding", "img/logo.svg")
    // → "branding/img/logo.svg" （Windows 下为 "branding\\img\\logo.svg"）
    if _, err := os.Stat(fPath); err == nil {
        http.ServeFile(w, r, fPath)   // 从文件系统返回覆盖文件
        return 0, nil
    }
}

// 示例 2: 请求 /static/custom.css
//         r.URL.Path 剥离后 = "custom.css"
if r.URL.Path == "custom.css" && d.settings.Branding.Files != "" {
    fPath := filepath.Join(d.settings.Branding.Files, "custom.css")
    // → "branding/custom.css"
    http.ServeFile(w, r, fPath)
    return 0, nil
}
```

两处判断均使用**无前导斜杠**的字符串字面量 `"img/"` 和 `"custom.css"` 做匹配。

#### B. 非 .js 文件：http.FileServer + http.FS 适配层

```go
// 示例: 请求 /static/css/app.css
//       r.URL.Path 剥离后 = "css/app.css"
if !strings.HasSuffix(r.URL.Path, ".js") {
    http.FileServer(http.FS(assetsFs)).ServeHTTP(w, r)
    return 0, nil
}
```

`http.FS(assetsFs)` 是标准库的 `fs.FS → http.FileSystem` 适配层，内部 `Open` 会自动规范化路径：
```go
// net/http/fs.go 中的 httpFS.Open 伪代码
func (f httpFS) Open(name string) (File, error) {
    if name == "" || name[0] != '/' {
        name = "/" + name     // "css/app.css" → "/css/app.css"
    }
    name = path.Clean(name)   // "/css/app.css" → "/css/app.css"
    return f.fs.Open(strings.TrimPrefix(name, "/"))
    // 去掉前导斜杠再调用 fs.FS.Open: "/css/app.css" → "css/app.css" → assetsFs.Open("css/app.css")
}
```

因此非 `.js` 分支对路径是否带前导斜杠有容错能力。

#### C. .js 文件：直接调用 fs.FS.Open 查找 .gz 压缩版本

```go
// 示例: 请求 /static/js/app.js
//       r.URL.Path 剥离后 = "js/app.js"
f, err := assetsFs.Open(r.URL.Path + ".gz")
// → assetsFs.Open("js/app.js" + ".gz")
// → assetsFs.Open("js/app.js.gz")  ✓ 查找成功

// 如果剥离后路径带前导斜杠 = "/js/app.js"：
// assetsFs.Open("/js/app.js.gz")   ✗ fs.FS 接口禁止前导斜杠，返回错误
```

**这是对路径格式要求最严格的分支**：没有 `http.FS` 适配层做容错，完全依赖 `stripPrefix` 剥离后的路径**不带前导斜杠**。

找到 `.gz` 文件后的处理：
- 客户端支持 gzip（`Accept-Encoding: gzip`）→ 直接写入 gzip 字节流，`Content-Encoding: gzip`
- 客户端不支持 gzip → `gzip.NewReader` 解压后写入明文 JS

**注意**：这里 `.gz` 打开失败时 `return http.StatusNotFound, err`，这个 `(404, err)` 返回给 handle 函数后，handle 会输出 `http.Error(w, "404 Not Found", 404)`。但此时**响应头已经部分写入**（`Cache-Control` 已在 L126 设置），虽然技术上仍可输出 404，但如果 `io.Copy` 过程中出错返回 `(500, err)`，响应体可能已经写入了部分字节，此时 `http.Error` 追加的内容会导致响应格式混乱。

### 6.6 index handler 的特殊处理（无前缀剥离）

与 static handler 不同，`index`（SPA 入口）注册时传给 `handle` 的 prefix 是 `""`：

[http/static.go#L107-L114](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/static.go#L107-L114)
```go
index = handle(func(w http.ResponseWriter, r *http.Request, d *data) (int, error) {
    if r.Method != http.MethodGet {
        return http.StatusNotFound, nil
    }
    w.Header().Set("x-xss-protection", "1; mode=block")
    // 硬编码 "public/index.html" 传给 handleWithStaticData
    return handleWithStaticData(w, r, d, assetsFs, "public/index.html", "text/html; charset=utf-8")
}, "", store, server)   // prefix = ""，无 stripPrefix
```

同时，`index` 被注册为 `NotFoundHandler`，所有未被其他路由匹配的请求都会走到这里，且**不会剥离任何前缀**。业务 handler 内部也不使用 `r.URL.Path`，而是硬编码 `"public/index.html"` 作为资源路径。

### 6.7 路径处理总结表

下表中 **stripPrefix 后路径** 即业务 handler 中实际读取到的 `r.URL.Path`：

| 请求 URL | 路由 prefix | stripPrefix 后 r.URL.Path | 资源查找代码 | 最终定位到的资源 |
|----------|------------|--------------------------|-------------|-----------------|
| `GET /static/js/app.js` | `"/static/"` | `"js/app.js"` | `assetsFs.Open(r.URL.Path + ".gz")` | `dist/js/app.js.gz` |
| `GET /static/js/vendors.js` | `"/static/"` | `"js/vendors.js"` | `assetsFs.Open("js/vendors.js.gz")` | `dist/js/vendors.js.gz` |
| `GET /static/css/app.css` | `"/static/"` | `"css/app.css"` | `http.FileServer(http.FS(assetsFs))` 间接查找 | `dist/css/app.css` |
| `GET /static/img/logo.svg` | `"/static/"` | `"img/logo.svg"` | 先 `os.Stat(branding/img/logo.svg)`，命中则 `http.ServeFile`，否则回退 `http.FileServer` | `branding/img/logo.svg` 或 `dist/img/logo.svg` |
| `GET /static/custom.css` | `"/static/"` | `"custom.css"` | `http.ServeFile(branding/custom.css)`（仅当配置了 Branding.Files） | `branding/custom.css`，未配置时走 `http.FileServer` 查 `dist/custom.css`（通常不存在 → 404） |
| `GET /settings/users` | `""`（index NotFoundHandler） | `"/settings/users"`（**不被使用**） | 硬编码 `assetsFs, "public/index.html"` | `dist/public/index.html`（SPA 路由由前端处理） |

---

## 七、请求处理链完整调用图（综合示例）

以 **PUT /api/users/5**（修改用户 5）为例，`server.BaseURL = ""`，当前登录用户 ID = 2，非管理员：

```
          HTTP 请求 PUT /api/users/5
                      │
                      ▼
    ┌──────────────────────────────────┐
    │  stripPrefix(BaseURL="")        │  第 1 层：无前缀，直接透传
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  CSP 中间件                     │  设置 Content-Security-Policy 头
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  mux.Router 路由匹配            │  匹配: /api/users/{id:[0-9]+} PUT
    └──────────────┬───────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────┐
    │  handle 包装器 (users.go 路由)  │
    │  ├─ 设置 Cache-Control 头       │
    │  ├─ store.Settings.Get()        │  从 DB 读取 settings
    │  ├─ 构造 &data{settings,...}    │  第一层上下文装配
    │  ├─ 调用 fn (即 userPutHandler) │
    │  └─ 统一错误处理 + 响应输出      │
    └──────────────┬───────────────────┘
                   │  stripPrefix("", handler) = 透传
                   ▼
    ┌──────────────────────────────────┐
    │  withSelfOrAdmin 装饰器         │  users.go#L57-L71
    │  └─ 嵌套 withUser               │
    │     ├─ 提取 JWT (X-Auth/Cookie) │
    │     ├─ 解析验证 HS256 签名      │
    │     ├─ 检查是否提示 X-Renew-Token│
    │     ├─ store.Users.Get(id=2)    │  加载当前用户 d.user
    │     └─ 返回 withSelfOrAdmin 体  │
    │        ├─ 解析路径 id = 5       │
    │        ├─ 判断: 2≠5 且 非Admin  │
    │        └─ return 403, nil       │  ← 命中无权限分支
    └──────────────┬───────────────────┘
                   │  返回 (403, nil)
                   ▼
    ┌──────────────────────────────────┐
    │  handle 统一出口                │
    │  ├─ status=403 >= 400 → 打日志 │
    │  ├─ status != 0 → 输出响应      │
    │  │   txt = "Forbidden"          │
    │  │   status≠400 → 不附加 err    │
    │  └─ http.Error(w, "403 Forbidden", 403)
    └──────────────────────────────────┘
```

---

## 八、关键文件索引

| 文件 | 核心职责 |
|------|----------|
| [http/http.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/http.go) | 路由注册、NewHandler 入口、monkey 包装函数 |
| [http/data.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/data.go) | data 上下文结构、handle 包装器（统一错误出口）、Check 规则检查 |
| [http/auth.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/auth.go) | JWT Token 提取解析、withUser/withAdmin 装饰器、renewHandler 续期入口、printToken 签发、renewableErr 过期特判 |
| [http/users.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/users.go) | withSelfOrAdmin 本人/管理员权限、5 个用户管理 handler 及字段级权限控制 |
| [http/share.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/share.go) | withPermShare 分享权限装饰器、分享 CRUD handler |
| [http/public.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/public.go) | withHashFile 分享鉴权 + 文件系统重绑定、健康检查 |
| [http/utils.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/utils.go) | errToStatus 错误码映射、stripPrefix 前缀剥离、renderJSON 成功响应 |
| [http/headers.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/headers.go) | 全局响应头（Cache-Control: no-cache） |
| [http/static.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/static.go) | 静态资源和 SPA index.html 处理器构造 |
| [http/commands.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/http/commands.go) | WebSocket 命令执行及 wsErr 独立错误处理 |
| [frontend/assets.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/frontend/assets.go) | 前端资源 embed.FS 嵌入 |
| [cmd/root.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/cmd/root.go) | assetsFs 子文件系统创建（fs.Sub 剥离 dist/） |
| [errors/errors.go](file:///d:/fz/0601/solo-dogfeeding/code/168-filebrowser/errors/errors.go) | 领域错误变量及 ErrShortPassword 自定义类型 |
